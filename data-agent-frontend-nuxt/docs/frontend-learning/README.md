# DataAgent 前端零基础学习手册

这套手册的目标不是让你一次记住所有前端知识，而是让你尽快形成一张“地图”：看到一个 `.vue` 文件时，知道页面长什么样、数据从哪里来、点击按钮后发生什么，以及出错时去哪里查。

## 你正在学习什么

这个前端是一个单页应用（SPA），核心技术如下：

| 技术 | 在项目中的职责 | 先掌握到什么程度 |
| --- | --- | --- |
| HTML | 描述页面结构 | 能看懂标签、表单和属性 |
| CSS | 控制布局和外观 | 能改间距、颜色、Flex 布局和响应式样式 |
| JavaScript | 处理交互和异步逻辑 | 能看懂变量、函数、数组、对象、事件、Promise |
| TypeScript | 给 JavaScript 增加类型约束 | 能看懂 `interface`、`type`、泛型、可选属性 |
| Vue 3 | 用组件和响应式数据构建界面 | 掌握模板语法、`ref`、`reactive`、`computed`、`watch` |
| Nuxt 4 | 组织 Vue 项目、生成路由和自动导入 | 能从 URL 找到页面，理解 layout 和配置 |
| Vuetify | 提供按钮、表格、弹窗等 UI 组件 | 会查组件属性、事件和插槽 |
| Pinia | 保存跨组件共享状态 | 能读懂 store 的 state 和 action |
| Axios / SSE | 与 Java 后端通信 | 能跟踪普通请求和流式响应 |

项目是前后端分离的：浏览器运行本目录中的前端，Java 后端运行在 `8065` 端口。开发时，Nuxt 把浏览器发出的 `/api/**` 和 `/nl2sql/**` 请求代理给 Java 后端，配置见 [nuxt.config.ts](../../nuxt.config.ts)。

## 推荐阅读顺序

1. [01：浏览器、HTML、CSS、JavaScript 与 TypeScript](./01-web-basics.md)
2. [02：用本项目学 Vue 3](./02-vue3.md)
3. [03：Nuxt 4 与项目地图](./03-nuxt-project-map.md)
4. [04：数据、状态、接口与流式响应](./04-data-flow.md)
5. [05：Vuetify、CSS 与页面布局](./05-ui-and-css.md)
6. [06：沿三条业务主线阅读源码](./06-source-walkthrough.md)
7. [07：调试方法与渐进式练习](./07-debug-and-practice.md)

如果时间很少，先读 02、03、06；遇到陌生概念再回到 01、04、05 查阅。

## 阅读源码的固定方法

每次只回答下面五个问题，不要从第一行硬读到最后一行：

1. **入口在哪里？** URL 对应 `app/pages` 下哪个文件？
2. **页面由什么组成？** 先只看 `<template>` 中使用的组件。
3. **数据在哪里？** 在 `<script setup>` 中找 `ref`、`reactive`、`computed` 或 Pinia store。
4. **谁改变数据？** 找 `@click`、`@change`、`watch` 和 `onMounted` 对应的函数。
5. **后端在哪里被调用？** 从函数继续追到 `app/services`。

例如 `/chat?agentId=1` 的最短阅读链路是：

```text
app/pages/chat.vue
  -> app/components/chat/ChatInputArea.vue
  -> app/stores/chat.ts
  -> app/services/chat/index.ts 或 app/services/graph/index.ts
  -> Java 后端 /api/**
```

## 7 天上手路线

| 天数 | 学习内容 | 当天成果 |
| --- | --- | --- |
| 第 1 天 | 01 的 HTML、CSS、JS 部分 | 能用浏览器开发者工具修改一个元素 |
| 第 2 天 | 01 的 TypeScript + 02 前半 | 能解释一个 `.vue` 文件的三部分 |
| 第 3 天 | 02 后半 + 03 | 能根据 URL 找到页面和布局 |
| 第 4 天 | 04 普通 HTTP 请求 | 能追踪“新建智能体”的请求链路 |
| 第 5 天 | 05 | 能安全修改一个页面的间距和响应式样式 |
| 第 6 天 | 06 | 能口头讲清聊天消息从输入到显示的流程 |
| 第 7 天 | 07 中的入门练习 | 独立完成一个小改动并验证 |

## 常见术语速查

- **DOM**：浏览器把 HTML 解析成的对象树，JavaScript 可以读取和修改它。
- **组件**：可复用的界面单元；本项目多数是 `.vue` 文件。
- **响应式**：数据变化时，依赖它的界面自动更新。
- **路由**：URL 与页面组件之间的映射。
- **状态**：界面当前需要记住的数据，例如加载中、当前会话、表单内容。
- **Props**：父组件传给子组件的数据。
- **Emit**：子组件通知父组件发生了某件事。
- **Slot**：父组件填入子组件预留位置的一段界面。
- **Composable**：封装可复用 Vue 状态和逻辑的函数，惯例以 `use` 开头。
- **Store**：跨多个组件共享状态和业务动作的容器。
- **CRUD**：Create、Read、Update、Delete，即增、查、改、删。
- **API**：前端与后端约定的通信接口。
- **SSE**：服务器持续向浏览器推送事件的一种单向流式通信方式。

> 学习原则：先能追踪，再能修改，最后才追求从零默写。真实开发中，会搜索、会定位、会验证比背 API 更重要。
