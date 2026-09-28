# 03：Nuxt 4 与项目地图

Vue 解决“组件怎么写”，Nuxt 解决“整个应用怎么组织”。这个项目最重要的 Nuxt 能力是文件路由、布局、自动导入、插件和开发代理。

## 1. 从启动入口到页面

应用最外层是 [app/app.vue](../../app/app.vue)：

```vue
<NuxtLayout>
  <NuxtPage />
</NuxtLayout>
```

- `NuxtPage` 根据当前 URL 渲染 `app/pages` 中对应页面。
- `NuxtLayout` 用 layout 包住页面；默认使用 [layouts/default.vue](../../app/layouts/default.vue)。
- 默认布局提供左侧导航、顶部栏、全局确认框和全局提示。

可以把整体想成：

```text
app.vue
└─ default layout
   ├─ 左侧导航
   ├─ 顶部栏
   ├─ 当前 page
   ├─ ConfirmDialog
   └─ Tip
```

## 2. 文件路由

`app/pages` 下的文件路径会生成 URL：

| URL | 页面文件 | 用途 |
| --- | --- | --- |
| `/agent/new` | `pages/agent/new.vue` | 新建智能体 |
| `/agent/12` | `pages/agent/[id].vue` | id 为 12 的智能体详情 |
| `/chat?agentId=12` | `pages/chat.vue` | 数据问答，query 中携带智能体 id |
| `/system/agents` | `pages/system/agents.vue` | 智能体管理 |
| `/system/data-sources` | `pages/system/data-sources/index.vue` | 数据连接 |
| `/system/model-config` | `pages/system/model-config.vue` | 模型配置 |
| `/knowledge/business` | `pages/knowledge/business.vue` | 业务知识 |
| `/knowledge/agents` | `pages/knowledge/agents.vue` | 智能体知识库 |
| `/knowledge/semantic-models` | `pages/knowledge/semantic-models.vue` | 语义模型 |
| `/prompt-config` | `pages/prompt-config/index.vue` | 提示词配置 |

`[id].vue` 中的方括号表示动态路由参数，通过 `route.params.id` 获取。问号后的 `agentId` 是 query，通过 `route.query.agentId` 获取。

注意：`pages/system/data-sources` 中还放了几个对话框/管理器组件。按 Nuxt 文件路由约定，`pages` 下的 `.vue` 文件也可能被扫描成路由。阅读业务入口时优先从上表和默认布局中的导航开始；可复用组件通常更适合放在 `app/components`。

## 3. 路由读取与跳转

```ts
const route = useRoute();   // 读取当前路由
const router = useRouter(); // 主动跳转

const id = Number(route.params.id);
const agentId = Number(route.query.agentId);

router.push({ path: '/chat', query: { agentId: 12 } });
router.replace({ path: '/chat', query: { agentId: 12 } });
```

`push` 增加浏览器历史，用户可以点返回；`replace` 替换当前历史。默认布局首次自动选择智能体时使用 replace，避免用户点返回又回到缺少 `agentId` 的同一路由。

## 4. 目录职责

```text
data-agent-frontend-nuxt/
├─ app.vue / app/
│  ├─ assets/css/      全局样式
│  ├─ components/      可复用界面组件
│  ├─ composables/     可复用 Vue 状态与流程
│  ├─ layouts/         页面外壳
│  ├─ pages/           URL 对应的页面入口
│  ├─ plugins/         Nuxt 启动时注册的扩展
│  ├─ services/        HTTP/SSE 接口与数据类型
│  ├─ stores/          Pinia 全局状态
│  └─ utils/           尽量无状态的工具函数
├─ public/             原样公开的静态文件
├─ docs/               项目文档
├─ nuxt.config.ts      Nuxt 总配置
└─ package.json        依赖和命令
```

判断代码应该放哪里：

- 只负责显示：component。
- 对应一个 URL，组织多个组件：page。
- 多个组件共享且带响应式状态：store 或 composable。
- 负责请求后端：service。
- 输入决定输出、不依赖组件生命周期：util。

## 5. 读懂 `nuxt.config.ts`

当前配置中的关键项：

- `modules`：启用 Vuetify、Pinia、ESLint。
- `components.pathPrefix: false`：自动导入组件时不强制带目录前缀。
- `imports.dirs`：让 composables 和 services 的部分导出可自动导入。
- `ssr: false`：关闭服务端渲染，按客户端 SPA 运行。
- `routeRules['/']`：根路径重定向到 `/agent/new`。
- `routeRules['/api/**']`：开发请求代理到 `http://localhost:8065`。
- `app.pageTransition`：配置页面切换动画。
- `css`：加载全局样式 [main.css](../../app/assets/css/main.css)。
- `manualChunks`：把 ECharts、zrender、highlight.js 拆成独立构建包，改善缓存和主包体积。

## 6. 插件与全局能力

[plugins/tipPlugin.ts](../../app/plugins/tipPlugin.ts) 注册 `$tip`：

```ts
const { $tip } = useNuxtApp();
$tip('保存成功');
```

插件适合注册应用级能力。这里真正的提示状态由 [stores/tips/index.ts](../../app/stores/tips/index.ts) 管理，布局中的 `<Tip />` 负责显示。调用者不需要知道提示组件位于哪里。

## 7. 项目地图的阅读顺序

第一次浏览仓库时按下面顺序，不要先打开最大的文件：

1. `package.json`：项目用什么技术、有哪些命令。
2. `nuxt.config.ts`：运行方式、代理、全局模块。
3. `app/app.vue`：最外层入口。
4. `app/layouts/default.vue`：全局导航和页面壳。
5. `app/pages/*.vue`：业务入口。
6. 页面引用的 components、stores、services。
7. 最后才读 utils 和复杂流式处理细节。

这叫从“应用骨架”逐步下钻到“实现细节”。
