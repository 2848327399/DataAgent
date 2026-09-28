# 07：调试方法与渐进式练习

## 1. 启动项目

环境要求以仓库 README 为准：Java 17+、Node.js 22+、pnpm 11+。

在仓库根目录启动后端：

```powershell
.\mvnw.cmd -pl data-agent-management spring-boot:run
```

另开一个终端启动前端：

```powershell
cd data-agent-frontend-nuxt
pnpm install
pnpm dev
```

依赖已经安装时不必重复执行 `pnpm install`。修改 `.vue`、`.ts`、`.css` 后，开发服务器通常会热更新浏览器。

常用命令：

```powershell
pnpm dev       # 开发服务器
pnpm build     # 生产构建和类型相关检查
pnpm test:unit # Vitest 单元测试
pnpm gen:ctx   # 更新项目自动生成的模块说明
```

## 2. 浏览器开发者工具四个核心面板

### Elements

检查真实 DOM、class、尺寸和最终生效的 CSS。适合解决“样式为什么没生效”“是谁撑宽了页面”。临时修改只在浏览器中有效，确认方案后再改源码。

### Console

查看 JavaScript 错误、`console.log` 和 Promise 未处理错误。点击报错中的文件与行号，结合 Source Map 跳回 `.vue`/`.ts` 源码。

### Network

查看请求 URL、方法、query、request payload、HTTP 状态码和 response。遇到页面没有数据时先判断：

1. 根本没发送请求：多半是事件、条件或生命周期问题。
2. 请求 404：URL 或后端路由不对。
3. 请求 400：参数或校验不对。
4. 请求 500：后端处理失败，但仍要检查前端传参。
5. 请求成功但页面为空：响应结构、赋值或模板条件可能不对。

SSE 请求可在 Network 中观察持续到达的 EventStream 消息。

### Sources

在事件函数或 store action 中加断点，逐步执行并观察变量。比到处添加 `console.log` 更适合理解复杂流程。

## 3. Vue DevTools 应该看什么

- Components：组件树、props、本地 state。
- Pinia：chat store 当前状态和 action 后的变化。
- Timeline：组件更新和事件。

调试聊天时同时打开 Network 与 Pinia：前者回答“服务器发了什么”，后者回答“前端保存成什么”。

## 4. 一套通用排错流程

```text
稳定复现
-> 缩小到具体页面和操作
-> 看 Console 是否有首个错误
-> 看 Network 请求是否正确
-> 从 template 事件追到函数
-> 从函数追到 store/service
-> 验证响应是否真的写入响应式状态
-> 验证 template 的 v-if/v-for 是否允许显示
-> 做最小修改
-> 重测正常、空数据、失败和移动端场景
```

不要一次改很多处。最小改动更容易判断你的假设是否正确。

## 5. 常见新手问题

### 忘记 `.value`

`ref` 在 script 中需要 `.value`，template 中不需要。

### 把 Promise 当成结果

调用异步 service 通常需要 `await`，调用方函数也要标记 `async`。

### 修改 props

props 是父组件的数据。需要修改时，创建本地状态或 emit 更新事件。

### 请求成功但列表不刷新

保存/删除后可能需要重新 `loadItems()`，或正确更新响应式数组。

### `v-if` 与空值判断错误

`0`、空字符串、`false` 都是假值。id 的业务规则若允许 0，就不能简单写 `if (!id)`。

### 页面离开后仍重复执行

检查是否在 `onUnmounted` 清理了 document 监听、timer、EventSource。

### 样式只在开发者工具有效

浏览器 Elements 中的修改不会写回文件，需要找到对应组件 style 或全局 CSS。

## 6. 渐进式练习

每次练习都先创建自己的 Git 分支，并保持一次只改一个目标。

### 入门：只改展示

1. 修改 `/agent/new` 的副标题和输入框 placeholder。
2. 给创建按钮换一个 MDI 图标。
3. 调整 `.page-shell` 的间距，分别检查桌面和窄屏。
4. 在聊天空状态增加一条示例问题。

验收：页面无 Console 错误，刷新后修改仍存在，移动端不溢出。

### 初级：增加局部状态和交互

1. 在新建智能体页显示描述已输入字数。
2. 增加“重置表单”按钮，并在重置前弹确认框。
3. 给业务知识搜索增加“正在搜索”提示。
4. 让聊天输入框在按 `Escape` 时清空内容。

验收：状态变化会立即反映到界面，空值和重复点击不会报错。

### 中级：贯穿页面与 service

1. 给智能体列表增加一个前端状态筛选。
2. 为某个列表补齐明确的错误提示和重试按钮。
3. 新增一个简单字段：同步修改 TypeScript 类型、表单、payload、展示。
4. 为一个纯工具函数补 Vitest 单元测试，参考 `app/utils/*.test.ts`。

验收：`pnpm test:unit` 和 `pnpm build` 通过，请求 payload 与后端约定一致。

### 进阶：理解聊天

1. 在输入区显示当前会话 id（仅开发环境）。
2. 为 SSE 连接增加明确的“连接中/已断开”状态。
3. 分析连续快速 SSE 消息为何需要批量渲染。
4. 为新增 messageType 写一个独立渲染组件并接入消息列表。

验收：切换智能体、切换会话、停止生成、离开再返回都不会重复连接或串数据。

## 7. 每次改动前后的检查清单

改动前：

- 我修改的是 page、component、store、service 还是 util？
- 数据的来源和唯一负责人是谁？
- 是否已有可复用组件或 composable？
- 这个操作是否可能失败或被重复点击？

改动后：

- 正常路径是否工作？
- loading、空数据、错误路径是否合理？
- TypeScript 是否出现红线？
- Console 是否干净？Network 请求是否符合预期？
- 窄屏是否溢出？
- 事件、timer、SSE 是否清理？
- 相关测试和 `pnpm build` 是否通过？

## 8. 如何向 AI 提问更容易学会

不要只说“解释这个文件”。可以这样问：

```text
请按 template -> state -> event -> API 的顺序解释这个组件，
并指出每个响应式变量由谁修改。先讲主流程，暂时忽略 CSS。
```

```text
请带我追踪点击这个按钮后的完整调用链。
每一步给出文件和函数名，并区分同步代码、异步请求和界面更新。
```

```text
不要直接替我写答案。先给我一个小任务和验收标准，
我完成后再帮我审查代码并解释问题。
```

最有效的学习循环是：先预测代码会做什么，再运行验证，然后解释预测与实际为什么不同。
