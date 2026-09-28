# 01：浏览器、HTML、CSS、JavaScript 与 TypeScript

## 1. 一次页面访问发生了什么

在本项目执行 `pnpm dev` 后，Nuxt 启动开发服务器。浏览器访问页面时大致经历：

1. 浏览器下载 HTML、JavaScript 和 CSS。
2. HTML 形成 DOM，CSS 计算每个元素的样式。
3. JavaScript 启动 Vue，创建组件并把数据渲染到 DOM。
4. 用户点击或输入时，事件处理函数修改响应式数据。
5. Vue 根据新数据更新必要的 DOM。
6. 需要后端数据时，JavaScript 发送 HTTP 请求或建立 SSE 连接。

本项目在 [nuxt.config.ts](../../nuxt.config.ts) 中设置了 `ssr: false`，所以它主要按客户端 SPA 工作：切换页面通常由前端路由完成，不需要整个网页刷新。

## 2. HTML：先看结构，不看样式

HTML 标签表达“这是什么”。Vue 的 `<template>` 写的就是带有额外指令的 HTML。

```html
<section>
  <h1>智能体列表</h1>
  <button>新建智能体</button>
</section>
```

需要认识的概念：

- 标签：`section`、`button`、`input` 等。
- 属性：`type="file"`、`disabled`、`placeholder` 等。
- 嵌套：父元素包住子元素，形成树形结构。
- 语义：按钮应使用按钮组件，输入应使用表单控件。

项目大量使用 Vuetify，所以你会看到 `<v-btn>`、`<v-card>`、`<v-text-field>`。它们最终仍会渲染为浏览器认识的 HTML。

## 3. CSS：盒子、布局和选择器

浏览器把大多数元素看成一个盒子：内容外是 `padding`，再外是 `border`，最外是 `margin`。

```css
.card {
  width: 320px;
  padding: 16px;
  border: 1px solid #e2e8f0;
  margin-bottom: 12px;
}
```

项目最常见的是 Flex 布局。阅读 [BaseDrawer](../../app/components/BaseDrawer/index.vue) 时重点看：

```css
.base-drawer { display: flex; }
.base-drawer__left { width: var(--drawer-width); }
.base-drawer__right { flex: 1; min-width: 0; }
```

意思是父容器横向排列；左边固定宽度；右边占满剩余空间。`min-width: 0` 很重要，它允许 Flex 子项缩小，避免内容把页面撑出横向滚动条。

你还需要认识：

- `.name` 选择 class；`#name` 选择 id；`button` 选择标签。
- `:hover` 表示鼠标悬停状态。
- `@media (max-width: 768px)` 表示小屏幕时覆盖样式。
- `position: absolute` 脱离普通布局并相对定位祖先定位。
- `overflow: auto` 在内容超出时出现滚动。
- `transition` 和 `animation` 负责过渡与动画。

## 4. JavaScript：项目中最高频的语法

### 变量、对象和数组

```ts
const name = 'Data Agent';
let loading = false;

const agent = { id: 1, name: '销售助手' };
const agents = [agent];
```

优先使用 `const`。`const` 只表示变量不能重新指向别的值，对象内部属性仍可改变。

项目常见数组操作：

```ts
const active = agents.filter((item) => item.status === 'published');
const names = agents.map((item) => item.name);
const target = agents.find((item) => item.id === 1);
```

- `filter`：保留满足条件的多个元素。
- `map`：把每个元素转换成新元素。
- `find`：找到第一个满足条件的元素。

### 函数、条件与提前返回

```ts
function openAgent(id: number) {
  if (id <= 0) return;
  router.push(`/agent/${id}`);
}
```

`return` 不只用于返回结果，也常用于结束函数。项目里大量 `if (!id) return` 是“条件不满足就不继续”的保护写法。

### 解构、展开与可选链

```ts
const { id, name } = agent;        // 解构
const edited = { ...agent, name: '新名称' }; // 浅拷贝并覆盖字段
const avatar = agent?.avatar || '';          // 可选链 + 默认值
```

`?.` 表示左边为空时不要继续访问，避免报错。`??` 只在左边是 `null` 或 `undefined` 时取默认值；`||` 还会把空字符串、`0`、`false` 当成需要默认值。

### 模块导入导出

```ts
export interface Agent { id?: number; name?: string }
export default new AgentService();

import agentService, { type Agent } from '~/services/agent/index';
```

`default` 导出导入时可以自定义名字；具名导出必须使用花括号中的名字。`type` 表示只导入类型，构建后不会成为运行时代码。`~/` 在本项目中指向 `app/`。

## 5. 异步：等待后端响应

网络请求不会立刻完成。`async/await` 让异步流程看起来像顺序代码：

```ts
async function loadAgents() {
  loading.value = true;
  try {
    agents.value = await agentService.list();
  } catch (error) {
    console.error(error);
  } finally {
    loading.value = false;
  }
}
```

- `await`：等待 Promise 完成，只能直接写在 `async` 函数内。
- `try`：执行可能失败的代码。
- `catch`：处理错误。
- `finally`：无论成功失败都执行，适合关闭 loading。

这是本项目最值得模仿的异步骨架。

## 6. TypeScript：给数据形状加说明和检查

JavaScript 在运行时才暴露许多错误；TypeScript 在编辑器和构建阶段提前检查。

```ts
interface Agent {
  id?: number;
  name?: string;
  status?: string;
}
```

`id?: number` 表示 `id` 可以不存在；`string | null` 表示值可能是字符串或空值。

### `interface`、`type`、枚举和泛型

```ts
type ReportFormat = 'markdown' | 'html';

enum TextType {
  SQL = 'SQL',
  TEXT = 'TEXT',
}

interface ApiResponse<T> {
  success: boolean;
  data?: T;
}
```

`ApiResponse<Agent[]>` 的意思是：响应中的 `data` 如果存在，就是 Agent 数组。`T` 是暂时不知道具体类型的“类型参数”。项目的通用响应定义见 [services/common/index.ts](../../app/services/common/index.ts)。

### 常见类型工具

- `Agent[]`：Agent 数组。
- `Record<string, string>`：键和值都是字符串的对象。
- `Partial<Agent>`：Agent 的所有字段都变成可选。
- `Omit<Agent, 'id'>`：从 Agent 中排除 `id`。
- `HTMLElement | null`：DOM 元素可能尚未创建。
- `as SomeType`：告诉编译器按某类型理解；它不会在运行时转换数据。

不要为了消除红线随意使用 `as` 或 `any`。先检查数据真实形状是否与类型一致。

## 7. 读完后的自测

你应该能回答：

1. `filter`、`map`、`find` 各返回什么？
2. 为什么 loading 通常在 `finally` 中关闭？
3. `Agent | null` 与 `Agent | undefined` 表达了什么？
4. Flex 容器中 `flex: 1` 有什么作用？
5. 为什么网络请求必须考虑失败情况？

不需要背答案；不能回答时回到对应小节并在项目中搜索一个真实例子。
