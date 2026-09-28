# 04：数据、状态、接口与流式响应

前端最核心的问题是：数据在哪里保存，谁修改它，修改后如何反映到界面。

## 1. 三类状态

### 组件局部状态

只属于一个组件，例如聊天输入框文字：

```ts
const inputText = ref('');
```

放在 [ChatInputArea.vue](../../app/components/chat/ChatInputArea.vue) 内最合适，因为别的组件不需要共享它。

### 页面流程状态

例如 CRUD 页面里的 loading、弹窗、表单，可由页面持有，也可抽到 [useCrudPage](../../app/composables/useCrudPage/index.ts) 复用。

### 跨组件共享状态

聊天侧栏、消息列表、输入区都需要会话和流式状态，因此放在 [stores/chat.ts](../../app/stores/chat.ts)。Pinia store 同时提供：

- state：`sessions`、`currentMessages`、`isStreaming` 等。
- action：`loadSessions`、`sendMessage`、`stopStreaming` 等。

选择规则：状态的最小共同使用范围在哪里，就尽量放在哪里。不要把所有变量都塞进全局 store。

## 2. 普通 HTTP 请求链路

以创建智能体为例：

```text
用户点击“创建智能体”
-> pages/agent/new.vue 的 createAgent()
-> 校验表单并构造 payload
-> agentService.create(payload)
-> axios.post('/api/agent', payload)
-> Nuxt 代理到 localhost:8065
-> Java 后端响应
-> 显示提示并跳到 /chat?agentId=新 id
```

[services/agent/index.ts](../../app/services/agent/index.ts) 把 API 地址、请求方法、参数和响应类型集中起来。页面只表达业务流程，不应到处拼接 URL。

```ts
async create(agent: Omit<Agent, 'id'>): Promise<Agent> {
  const response = await axios.post<Agent>('/api/agent', agent);
  return response.data;
}
```

Axios 的泛型 `<Agent>` 是对响应 data 的 TypeScript 约束，不会验证服务器真的返回了 Agent。真实项目仍要保证前后端接口约定一致。

## 3. HTTP 方法与 CRUD

| 目的 | 常见方法 | 本项目例子 |
| --- | --- | --- |
| 查询 | GET | 获取智能体列表 |
| 创建 | POST | 新建智能体、生成 API Key |
| 完整或部分更新 | PUT/PATCH | 修改智能体、重命名会话 |
| 删除 | DELETE | 删除智能体或会话 |

这只是常见约定，最终以服务端接口为准。

## 4. 加载、成功、空数据、失败

一个可靠页面至少考虑四种状态：

```ts
const loading = ref(false);
const items = ref<Item[]>([]);
const errorMessage = ref('');

async function load() {
  loading.value = true;
  errorMessage.value = '';
  try {
    items.value = await service.list();
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '加载失败';
  } finally {
    loading.value = false;
  }
}
```

本项目有些请求在 `catch` 中选择忽略，以允许页面降级显示。阅读时要问：这个失败是否真的可忽略？用户是否能知道发生了什么？开发者是否至少能从控制台看到原因？

## 5. Pinia 聊天 Store 的结构

`defineStore('chat', () => { ... })` 使用 setup 风格：

- `ref` 相当于 state。
- 内部函数相当于 action。
- `return` 的内容才会公开给组件。

组件调用 `const store = useChatStore()` 后，在 template 中可直接使用 `store.isStreaming`。在 script 中 Pinia 会保持 store 属性的响应性；如果解构 state，通常应使用 `storeToRefs`，否则可能丢失响应式连接。

## 6. SSE：为什么聊天不是普通请求

普通 HTTP 往往是请求一次、响应一次。DataAgent 执行分析时会逐步产生节点结果，所以 [services/graph/index.ts](../../app/services/graph/index.ts) 使用 `EventSource` 建立 SSE：

```ts
const eventSource = new EventSource(url);
eventSource.onmessage = (event) => {
  const response = JSON.parse(event.data);
  onMessage(response);
};
```

聊天完整链路：

```text
ChatInputArea.handleSend
-> chatStore.sendMessage
-> 先保存用户消息
-> chatStore._sendGraphRequest
-> graphService.streamSearch
-> 浏览器持续收到 GraphNodeResponse
-> store 更新 nodeBlocks / 报告文本
-> ChatMessageList 响应式重绘
-> complete 事件到达
-> 保存时间线和最终回复
-> 重新加载当前会话消息
```

`GraphNodeResponse` 中几个重要字段：

- `eventType`：节点输出、最终答案、需要人工反馈。
- `stepId`：本次节点执行标识，用于把同一步的增量内容归组。
- `nodeName`：哪个图节点产生了内容。
- `textType`：SQL、普通文本、Markdown、HTML、结果集等。
- `complete` / `error`：完成或错误状态。

## 7. 流式 UI 为什么要节流

后端可能很密集地发送小片段。如果每个片段都立即让复杂消息列表完整重绘，会卡顿。[stores/chat.ts](../../app/stores/chat.ts) 使用两种批处理：

- `requestAnimationFrame`：最多跟随浏览器一帧更新一次时间线视图。
- `setTimeout(..., 80)`：报告内容约每 80ms 推送一次。

这不是业务正确性的必要条件，而是性能优化。初读 store 时可以先忽略 `scheduleViewSync` 等细节，先抓住“收到事件 -> 更新 sessionState -> 同步到当前视图”的主线。

## 8. 为什么必须清理连接和监听

组件离开后，如果 EventSource、定时器或 document 事件仍存在，会造成：

- 重复请求或重复回调。
- 已离开页面的状态仍被修改。
- 内存不能释放。
- 再次进入页面后一个操作执行多次。

所以要成对记忆：

```text
addEventListener  <-> removeEventListener
setInterval       <-> clearInterval
setTimeout        <-> clearTimeout（需要取消时）
new EventSource   <-> close
```

## 9. 安全地渲染后端内容

[ChatMessageList.vue](../../app/components/chat/ChatMessageList.vue) 会把 Markdown 转成 HTML，并在 `v-html` 前使用 DOMPurify 清洗。原因是 `v-html` 会让字符串成为真实 DOM；如果直接渲染不可信内容，可能产生 XSS。

原则：普通文本优先使用 `{{ text }}`；确实需要 `v-html` 时，先使用可信规则清洗，并谨慎允许标签和属性。
