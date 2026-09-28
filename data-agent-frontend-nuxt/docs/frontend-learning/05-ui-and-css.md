# 05：Vuetify、CSS 与页面布局

## 1. Vuetify 是什么

Vuetify 提供已经实现交互和样式的 Vue 组件。常见组件：

- `v-btn`：按钮。
- `v-text-field` / `v-textarea` / `v-select`：输入控件。
- `v-form`：表单校验。
- `v-card`：卡片容器。
- `v-dialog`：弹窗。
- `v-data-table`：表格。
- `v-row` / `v-col`：栅格布局。
- `v-list`：列表与导航。
- `v-icon`：MDI 图标。

遇到陌生 `v-*` 标签时，先分辨四类内容：

```vue
<v-btn
  color="blue-darken-3"
  :loading="loading"
  @click="save"
>
  <template #prepend>...</template>
  保存
</v-btn>
```

其中 `color` 是固定 prop，`:loading` 是动态 prop，`@click` 是事件，`#prepend` 是插槽。

## 2. 工具 class 与自定义 CSS

项目模板中常见 `d-flex`、`align-center`、`pa-4`、`mb-2`，它们是 Vuetify 工具 class：

- `d-flex`：`display: flex`。
- `align-center`：交叉轴居中。
- `justify-end`：主轴靠末端。
- `pa-4`：四周 padding。
- `px-6`：左右 padding。
- `mt-2` / `mb-4`：上/下 margin。
- `ga-2`：子项 gap。
- `text-body-2`、`font-weight-bold`：文字样式。

简单布局优先复用工具 class；复杂、需要伪类、媒体查询或多处规则时写语义化 class 和 CSS。

## 3. `scoped` 与 `:deep`

```vue
<style scoped>
.page { padding: 32px; }
</style>
```

`scoped` 让选择器主要作用于当前组件，避免同名 class 污染其他页面。但 Vuetify 子组件内部 DOM 不一定能被普通 scoped 选择器命中，因此项目使用：

```css
.agent-switcher :deep(.v-field) {
  border-radius: 10px;
}
```

`:deep` 会穿透 scoped 边界。它依赖组件库内部 class，升级 Vuetify 后更容易失效，所以只在组件 props、主题和工具 class 不能满足需求时使用。

## 4. 全局样式与局部样式

[assets/css/main.css](../../app/assets/css/main.css) 由 Nuxt 全局加载，适合：

- 页面级通用容器，如 `.page-shell`。
- 全局过渡动画。
- 多个页面一致使用的设计规则。

组件 `<style scoped>` 适合只服务当前组件的样式。判断标准不是“写在哪里方便”，而是“这个规则应该影响多大范围”。

## 5. 用 BaseDrawer 学 Flex

[BaseDrawer](../../app/components/BaseDrawer/index.vue) 是很好的布局教材：

```text
.base-drawer (横向 flex)
├─ aside 左栏（固定宽度、不可收缩）
└─ section 右栏（flex: 1）
   ├─ header（固定高度）
   └─ content（flex: 1，可滚动）
```

关键规则：

- 外层 `height: 100vh; overflow: hidden`：占满视口，不让整个 body 滚动。
- 左栏 `flex-shrink: 0`：空间不足时仍保持宽度。
- 右栏 `min-width: 0`：允许内容区收缩。
- 内容 `overflow: auto`：只让内容区滚动。
- 关闭时左栏宽度变成 0，通过 `transition` 产生动画。

## 6. 响应式布局

BaseDrawer 在小于 768px 时把左栏改成绝对定位：

```css
@media (max-width: 768px) {
  .base-drawer__left {
    position: absolute;
    z-index: 100;
  }
}
```

同时 [layouts/default.vue](../../app/layouts/default.vue) 使用 Vuetify 的 `useDisplay()` 监听 `mobile`，移动端首次进入时关闭抽屉。CSS 负责“怎么显示”，JavaScript 状态负责“是否打开”。

Vuetify 栅格也用于响应式：`<v-col cols="12" md="6">` 表示小屏占 12 列（整行），中等及以上屏占 6 列（半行）。

## 7. 表单校验

新建智能体页面使用：

```vue
<v-text-field
  v-model="agentForm.name"
  :rules="[(v) => !!v?.trim() || '智能体名称不能为空']"
/>
```

规则函数返回 true 表示通过，返回字符串表示失败信息。提交时：

```ts
const result = await formRef.value?.validate();
if (!result?.valid) return;
```

前端校验改善体验，但不能替代后端校验，因为用户可以绕过浏览器直接调用 API。

## 8. 改样式时的排查顺序

1. 浏览器 Elements 面板确认目标 DOM 和实际 class。
2. 在 Styles 面板看规则是否被划掉，即是否被更高优先级覆盖。
3. 先尝试 Vuetify prop 或工具 class。
4. 再写组件的 scoped CSS。
5. 必须改子组件内部时才用 `:deep`。
6. 用桌面宽度和小屏宽度各检查一次。

尽量避免连续叠加 `!important`。它会让后续覆盖越来越困难；先弄清楚选择器范围和优先级。
