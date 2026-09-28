# 06：沿三条业务主线阅读源码

这一章不逐行解释，而是示范如何跨文件追踪真实功能。每条主线先跑一遍功能，再按顺序打开文件。

## 主线一：新建智能体

### 第一步：找到入口

根路由在 [nuxt.config.ts](../../nuxt.config.ts) 中重定向到 `/agent/new`，对应 [pages/agent/new.vue](../../app/pages/agent/new.vue)。

### 第二步：只读模板

模板由以下部分组成：

- `KnowledgePageHeader`：标题和操作按钮。
- `v-form`：表单容器。
- `v-text-field`、`v-textarea`、`v-select`：字段。
- 隐藏的原生 `input[type=file]`：选择头像文件。

从“创建智能体”按钮的 `@click="createAgent"` 进入 script。

### 第三步：追踪状态和提交

```text
agentForm (reactive 表单)
-> formRef.validate() 校验
-> 构造 payload 并 trim 字符串
-> agentService.create(payload)
-> axios POST /api/agent
-> $tip 提示成功
-> router.push('/chat?agentId=...')
```

继续阅读 [services/agent/index.ts](../../app/services/agent/index.ts)，观察 `Agent` 接口如何描述前后端数据，`create` 如何声明输入和返回类型。

### 第四步：理解文件上传支线

选择文件后先检查 MIME 类型和 5MB 大小限制，再用 FileReader 做本地预览，同时调用 [services/fileUpload/index.ts](../../app/services/fileUpload/index.ts) 上传。无论请求成功失败，`finally` 都恢复 loading 并清空 file input，保证再次选择同一文件也能触发 change。

### 你应该学到

- 表单双向绑定和校验。
- DOM ref 与文件事件类型转换。
- loading / try / catch / finally。
- service 分层和成功后的路由跳转。

## 主线二：业务知识 CRUD

入口是 [pages/knowledge/business.vue](../../app/pages/knowledge/business.vue)。这个页面适合学习“列表 + 搜索 + 表格 + 编辑弹窗”的后台管理模式。

### 数据加载

```text
onMounted
-> loadBusinessKnowledge（useCrudPage 返回的 loadItems）
-> businessKnowledgeService.list(agentId, keyword)
-> items 更新
-> v-data-table 自动重绘
```

`agentId` 是一个 computed，来自当前路由 query。搜索词是局部 ref。

### 新建和编辑为何共用一个弹窗

`isEdit` 决定标题、按钮文字和保存分支；`knowledgeForm` 保存当前表单。点击新增时创建空表单，点击编辑时浅拷贝当前行。

`useCrudPage<T, TCreate, TUpdate>` 的三个泛型分别代表：

- 列表项类型 `BusinessKnowledgeVO`。
- 创建请求类型 `CreateBusinessKnowledgeDTO`。
- 更新请求类型 `UpdateBusinessKnowledgeDTO`。

这样列表、创建和更新的字段不必强行完全一致。

### 删除为何先确认

页面调用 `useConfirm().showConfirm` 保存一个确认回调；布局中的全局 ConfirmDialog 执行确认后才调用 `deleteItem`。这是跨组件共享交互流程的例子。

### 你应该学到

- 表格 headers、item 插槽与 loading。
- 可复用 CRUD Composable。
- 泛型如何保持不同数据结构的类型安全。
- 确认框和提示消息如何成为全局能力。

## 主线三：数据问答与 SSE

这是项目最复杂的前端主线。第一次只追踪正常发送一条消息，不要同时研究报告渲染、人工反馈和所有错误分支。

### 页面组合

[pages/chat.vue](../../app/pages/chat.vue) 组合：

```text
ChatSidebar       会话列表
ChatMessageList   历史消息和流式结果
ChatInputArea     输入、模型、数据源、发送/停止
```

三个兄弟组件通过同一个 [stores/chat.ts](../../app/stores/chat.ts) 共享状态。

### 初始化

```text
从 route.query.agentId 得到 currentAgentId
-> init(agentId)
-> 加载智能体资料
-> 建立会话标题 SSE
-> store.loadSessions
-> 选择已有首个会话，或创建新会话
-> 加载数据源和聊天模型
```

当 query 中的 agentId 改变，watch 会清理当前视图状态后重新初始化。离开页面时关闭会话流。

### 发送

1. [ChatInputArea.vue](../../app/components/chat/ChatInputArea.vue) 从 `inputText` 取值并做空值、会话和 streaming 检查。
2. 调用 `store.sendMessage(query)`。
3. store 先通过 [services/chat/index.ts](../../app/services/chat/index.ts) 保存用户消息。
4. store 构造 `GraphRequest`。
5. [services/graph/index.ts](../../app/services/graph/index.ts) 用 EventSource 连接 `/api/stream/search?...`。
6. 每次收到节点事件，store 将内容按 step 放入 `nodeBlocks`。
7. [ChatMessageList.vue](../../app/components/chat/ChatMessageList.vue) 因响应式状态变化而更新。
8. complete 后保存时间线/最终回答，关闭流并重新加载消息。

### 消息为什么有多种渲染方式

ChatMessage 有 `role` 和 `messageType`。消息列表根据类型选择：

- 用户文本气泡。
- AI Markdown 文本。
- 结果集表格。
- 工作流时间线。
- Markdown 或 HTML 报告。
- warning / error 状态条。

这种写法的核心是“数据先带类型，视图按类型分派”。新增一种消息类型时，通常要同时检查服务类型、保存逻辑、历史消息渲染和流式渲染。

### 第二次阅读再看这些难点

- 人工反馈如何复用 `threadId` 恢复中断的图任务。
- sessionStateManager 如何为多个会话分别保存流状态。
- `requestAnimationFrame` 和 80ms timer 如何降低频繁重绘。
- Markdown 如何转换、清洗并渲染 ECharts。
- `stopStreaming` 如何同时关闭浏览器流并通知后端停止任务。

## 一张跨层定位表

| 你看到的问题 | 优先去哪里找 |
| --- | --- |
| URL 打开了哪个页面 | `app/pages`、`nuxt.config.ts` |
| 左侧菜单或顶部栏 | `app/layouts/default.vue` |
| 按钮长相、表格单元格 | 当前 page 或 component 的 template/style |
| 点击没反应 | `@click` 对应函数、浏览器 Console |
| 页面数据不更新 | ref/reactive/computed/watch、Pinia store |
| 请求地址或请求参数不对 | `app/services`、Network 面板 |
| 聊天流中途断开 | graph service、chat store、Network 的 EventStream |
| 提示框或确认框 | tips store/plugin、useConfirm、default layout |
| Markdown/图表显示异常 | utils/markdown、useEchartsRenderer、ChatMessageList |

## 源码阅读完成标准

当你能不看文档画出下面任意一条链路，就已经“看懂”了对应功能：

```text
用户动作 -> 组件事件 -> 页面/store action -> service -> 后端
后端数据 -> service -> ref/store -> template 分支 -> 用户看到的界面
```
