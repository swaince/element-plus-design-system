---
name: element-plus-design-system
description: >
  将任意设计系统（DESIGN.md）完整适配到 Element Plus 组件库。基于 Tailwind CSS v4 @theme 定义 ds-* 设计令牌，
  通过 CSS 变量桥接层将设计令牌注入 Element Plus 的 --el-* 变量体系，确保组件视觉效果与设计系统完全一致。
  必须在使用 Element Plus + Tailwind CSS 的项目中统一设计系统时触发，
  包括但不限于：适配设计系统到 Element Plus、为 Element Plus 创建主题、转换设计令牌到 CSS 变量、
  统一 Element Plus 设计风格、将 DESIGN.md 应用到 Element Plus 等场景。
---

# Element Plus 设计系统适配

将任意设计系统转换为 Tailwind CSS v4 `@theme` 令牌，并桥接到 Element Plus `--el-*` CSS 变量，确保所有组件自动继承设计系统样式。

## 核心架构

```
DESIGN.md → Tailwind @theme (ds-* 令牌) → element-bridge (--el-* 变量) → Element Plus 组件
                                                                          → component-overrides (变量无法覆盖的部分)
```

三层文件，单一职责：

| 文件 | 职责 | 内容 |
|------|------|------|
| `tokens.css` | 设计令牌定义 | Tailwind v4 `@theme` 块，使用 `ds-*` 命名空间 |
| `element-bridge.css` | CSS 变量桥接 | `--el-*` 变量引用 `ds-*` 令牌，分 `:root` 和 `.dark` |
| `component-overrides.css` | 组件细节覆盖 | CSS 变量无法控制的样式（如按钮 `translate(6px)` 动画） |

## 关键原则

### 1. 从 Element Plus 源码提取变量，而非文档

**ELPT-SOURCE-DRIVEN**：Element Plus 官方文档的 CSS 变量列表可能滞后。必须直接从依赖包源码提取：

```
node_modules/element-plus/theme-chalk/src/common/var.scss
```

该文件通过 SASS 循环生成约 **350+ 个**全局 CSS 变量。文档只列出一小部分。
使用 `grep` 搜索 `--el-` 或在 `@each $type in $types` 位置找到语义色生成逻辑，
确认每个语义色有 **light-1 到 light-9** 共 9 个变体 + **dark-2**。

在 bridge 文件中必须覆盖全部 9 个 light 变体和 dark-2，
否则未覆盖的变体回退到 Element Plus 默认的 `#409eff` 衍生色，
而非设计系统的指定颜色。使用 `color-mix()` 自动计算变体：

```css
/* 浅色模式：用 white 提亮 */
--el-color-primary-light-3: color-mix(in srgb, var(--color-ds-primary), white 30%);
--el-color-primary-light-5: color-mix(in srgb, var(--color-ds-primary), white 50%);
/* ... light-1 到 light-9，每 2 级一个 */
--el-color-primary-dark-2: var(--color-ds-primary-dark);

/* 暗色模式：用 black 加深（对于 light-*），用 white 提亮（对于 dark-2） */
--el-color-primary-light-3: color-mix(in srgb, var(--color-ds-primary), black 30%);
--el-color-primary-dark-2: color-mix(in srgb, var(--color-ds-primary), white 20%);
```

### 2. Tailwind v4 @theme 命名空间规范

**ELPT-NAMING**：`@theme` 变量名决定生成的 Tailwind 工具类，必须使用正确命名空间：

| 用途 | 正确命名空间 | 错误示例 | 生成工具类 |
|------|-------------|---------|-----------|
| 颜色 | `--color-*` | — | `text-ds-primary`, `bg-ds-primary` |
| 字体系列 | `--font-*` | `--font-family-*` ❌ | `font-ds-sans` |
| 字号 | `--text-*` | `--font-size-*` ❌ | `text-ds-hero` |
| 行高 | `--leading-*` | `--line-height-*` ❌ | `leading-ds-hero` |
| 字重 | `--font-weight-*` | — | `font-ds-hero` |
| 字间距 | `--tracking-*` | — | `tracking-ds-hero` |
| 圆角 | `--radius-*` | — | `rounded-ds-sm` |
| 阴影 | `--shadow-*` | — | `shadow-ds-card` |
| 过渡时长 | `--duration-*` | — | `duration-ds-normal` |
| 缓动函数 | `--ease-*` | — | `ease-ds-in-out` |

使用 `ds-*` 前缀避免污染 Tailwind 原生令牌（如 `--color-red-500`）。

### 3. 暗色模式通过 .dark class 控制

**ELPT-DARK-MODE**：不使用 `prefers-color-scheme`，通过 `.dark` class 切换。
暗色块必须覆盖 `:root` 中**所有**语义色及其变体，但字体、圆角、阴影、过渡、组件尺寸等
主题无关的变量不需要暗色版本。

```css
:root {
  --el-color-primary: var(--color-ds-primary);
  --el-color-primary-light-3: ...;
}
.dark {
  --el-color-primary: var(--color-ds-primary-light); /* 暗色模式用较亮的主色 */
  --el-color-primary-light-3: ...; /* black 替代 white */
}
```

## 实施流程

### 第一步：确保字体可用

如果设计系统指定了不可用的字体（如商业字体），选择替代方案：

```bash
pnpm add @fontsource-variable/inter  # 可变字体
```

创建 `src/styles/fonts/inter.css`：
```css
@import "@fontsource-variable/inter";
```

在 `main.ts` 中最先导入。

### 第二步：提取 Element Plus 全部 CSS 变量

```bash
# 查找 var.scss
ls node_modules/element-plus/theme-chalk/src/common/var.scss

# 提取所有 --el- 变量
cat node_modules/element-plus/theme-chalk/src/common/var.scss
```

关注：
- 语义色循环（`@each $type in $types`）→ 每种色有 9 个 light 变体 + 1 个 dark-2
- 边框色彩系统（含 `extra-light`, `dark`, `darker`）
- 填充色彩系统（含 `extra-light`, `dark`, `darker`）
- 文字颜色（含 `disabled`）
- 全部字号变体（`extra-large`, `large`, `medium`, `base`, `small`, `extra-small`）

### 第三步：编写 tokens.css

将 DESIGN.md 的值填入 `@theme` 块。必须覆盖：

1. **颜色**：每个色值一个 `--color-ds-*` 令牌
2. **字体**：`--font-ds-sans`（主字体）、`--font-ds-mono`（等宽）
3. **圆角**：设计系统定义的所有级别（如 2px/4px/8px/50%）
4. **阴影**：主阴影 + 轻/暗变体（至少 2-3 级）
5. **字号/行高/字重/字间距**：逐项注册为令牌
6. **间距比例尺**：完整的设计系统间距值
7. **过渡**：`--duration-ds-*`、`--ease-ds-*`
8. **组件尺寸**：large/default/small

### 第四步：编写 element-bridge.css

将 Element Plus 源码中的所有全局 `--el-*` 变量桥接到 `ds-*` 令牌。
必须覆盖的类别：

| 类别 | 变量数 | 说明 |
|------|--------|------|
| 语义色主体 | 6 | primary/success/warning/danger/error/info |
| 语义色变体 | 60 (6×10) | light-1 到 light-9 + dark-2 |
| 颜色 RGB 分量 | 6 | 手动计算（CSS 无法 hex→RGB），如 `#146ef5` = `20, 110, 245` |
| 文字颜色 | 5 | primary/regular/secondary/placeholder/disabled |
| 边框颜色 | 6 | base/light/lighter/extra-light/dark/darker |
| 填充颜色 | 7 | base/light/lighter/extra-light/dark/darker/blank |
| 字体排版 | 8 | family + 6 个字号 + weight + line-height |
| 圆角 | 4 | base/small/round/circle |
| 阴影 | 4 | base/light/lighter/dark |
| 过渡 | 11 | duration/function/all/fade/color/md-fade 等 |
| 组件尺寸 | 3 | large/default/small |
| 遮罩 | 3 | base/light/lighter |
| 禁用 | 3 | bg/text/border |
| z-index | 3 | normal/top/popper |
| border 属性 | 2 | width/style |

**允许的硬编码**：RGB 分量（技术限制）、`border-width`、`border-style`（设计系统通常不定义这些）。

### 第五步：编写 component-overrides.css

CSS 变量无法覆盖的细节：按钮 hover `translate(6px)` 动画、标签 `text-transform: uppercase`、
表格表头 uppercase、特定组件的阴影应用范围等。

### 第六步：串联导入

**ELPT-IMPORT-ORDER**：`main.ts` 中全局样式的 `import` 必须放在 `import App from "./App.vue"` 之后，否则 Element Plus 默认样式会覆盖桥接变量。

```
main.ts:
  import App from "./App.vue"       ← 1. 先导入，触发 Element Plus 默认样式
  import "@/styles/fonts/inter.css" ← 2. 字体加载
  import "@/styles/index.css"       ← 3. 全局样式（bridge + overrides），覆盖默认值
  import { createApp } from "vue"
  createApp(App).mount("#app")

index.css → tailwind.css → "tailwindcss" + tokens.css
          → element-bridge.css
          → component-overrides.css
```

### 第七步：验证

```bash
pnpm type-check  # 验证类型
pnpm build-only  # 验证构建
pnpm dev         # 浏览器验证视觉效果
```

在演示页面中覆盖所有常用组件，提供暗色模式切换按钮，
逐个组件检查颜色、字体、圆角、阴影是否与设计系统一致。

## 常见陷阱

- **永远不要在 @theme 外定义设计令牌**：只有 @theme 内的变量会生成 Tailwind 工具类
- **Vite 配置需要 @tailwindcss/vite 插件**：Tailwind v4 不通过 PostCSS，必须在 vite.config.ts 中注册
- **auto-imports.d.ts 和 components.d.ts 是自动生成的**：无需手动编辑，`pnpm dev` 会自动更新
- **bridge 文件中三个边框变量可能都映射到同一个 ds 令牌**：Element Plus 的 `light/lighter/extra-light` 是不同颜色，全映射到同一值会丧失层次感。如果设计系统只定义了一个边框色，用 `color-mix()` 派生层次
- **light-1 到 light-9 不是每级都同样重要**：最常用的是 light-3/5/7/8/9，但必须全部定义以避免未覆盖的回退
- **.dark 块中遗漏语义色**：常见错误是只在 :root 中定义了 success/warning/danger/error/info 的全部变体，暗色块却只写了 base 值
- **样式入口文件必须在 App.vue 之后导入**：`main.ts` 中全局样式（`index.css`）的 `import` 必须放在 `import App from "./App.vue"` 之后，否则 Element Plus 组件的样式覆盖不会生效。原因是 Vite 处理 `.vue` 文件时，先导入的组件会触发 Element Plus 的默认样式注入，如果全局桥接样式在组件之后加载，默认样式会覆盖桥接变量

```typescript
// main.ts — 正确顺序
import App from "./App.vue";        // 1. 先导入 App，触发 Element Plus 默认样式加载
import "@/styles/fonts/inter.css";  // 2. 字体
import "@/styles/index.css";        // 3. 全局样式（含 bridge + overrides），覆盖默认值
import { createApp } from "vue";    // 4. Vue

createApp(App).mount("#app");
```

## 文件模板

### tokens.css 骨架

```css
@theme {
  --color-ds-primary: /* 主色 */;
  --color-ds-primary-light: /* 浅主色 */;
  --color-ds-primary-dark: /* 深主色 */;
  --color-ds-primary-hover: /* hover 主色 */;
  --color-ds-text-primary: /* 主文字色 */;
  --color-ds-text-regular: /* 常规文字色 */;
  --color-ds-text-secondary: /* 次要文字色 */;
  --color-ds-text-placeholder: /* 占位文字色 */;
  --color-ds-text-link: /* 链接色 */;
  --color-ds-border: /* 边框色 */;
  --color-ds-border-hover: /* hover 边框色 */;
  --color-ds-bg-primary: /* 主背景色 */;
  --font-ds-sans: /* 无衬线字体栈 */;
  --font-ds-mono: /* 等宽字体栈 */;
  --radius-ds-sm: /* 小圆角 */;
  --radius-ds-md: /* 中圆角 */;
  --radius-ds-round: 50%;
  --shadow-ds-card: /* 卡片阴影 */;
  --shadow-ds-light: /* 轻阴影 */;
  --shadow-ds-dark: /* 重阴影 */;
  --text-ds-hero: /* hero */;
  --text-ds-section: /* 章节标题 */;
  --text-ds-body-standard: /* 正文 */;
  --text-ds-caption: /* 注释 */;
  --leading-ds-hero: /* 对应行高 */;
  --font-weight-ds-hero: /* 对应字重 */;
  --tracking-ds-hero: /* 对应字间距 */;
  --duration-ds-normal: 0.2s;
  --duration-ds-fast: 0.15s;
  --ease-ds-in-out: cubic-bezier(0.645, 0.045, 0.355, 1);
  --ease-ds-fast: cubic-bezier(0.23, 1, 0.32, 1);
  --size-ds-large: 40px;
  --size-ds-default: 32px;
  --size-ds-small: 24px;
  --overlay-ds: rgba(/* 基于主文字色的 RGB */);
}
```

### element-bridge.css 骨架

```css
:root {
  --el-color-primary: var(--color-ds-primary);
  --el-color-primary-light-3: color-mix(in srgb, var(--color-ds-primary), white 30%);
  /* ... 其他 light 变体和 dark-2 */
  --el-color-primary-dark-2: var(--color-ds-primary-dark);

  --el-font-family: var(--font-ds-sans);
  --el-font-size-base: var(--text-ds-body-standard);
  --el-border-radius-base: var(--radius-ds-sm);
  --el-text-color-primary: var(--color-ds-text-primary);
  --el-border-color: var(--color-ds-border);
  --el-bg-color: var(--color-ds-bg-primary);
  --el-box-shadow: var(--shadow-ds-card);
  --el-transition-duration: var(--duration-ds-normal);
  --el-component-size: var(--size-ds-default);
  --el-overlay-color-lighter: var(--overlay-ds);
  /* ... 全部 --el-* 变量 */
}

.dark {
  --el-color-primary: var(--color-ds-primary-light);
  --el-color-primary-light-3: color-mix(in srgb, var(--color-ds-primary-light), black 30%);
  /* ... 暗色版全部语义色变体 */
  --el-bg-color: var(--color-ds-text-primary);
  --el-text-color-primary: var(--color-ds-bg-primary);
  /* ... 暗色版文字/边框/背景/填充 */
}
```

## 审计检查清单

完成 bridge 后，运行以下检查：

- [ ] Element Plus 语义色 light-1 到 light-9 是否全部桥接（共 6×9=54 个）？
- [ ] .dark 块是否包含所有语义色和它们的变体？
- [ ] RGB 分量值是否与 hex 色值匹配？
- [ ] 所有 `var(--*-ds-*)` 引用在 tokens.css 中是否存在？
- [ ] tokens.css 中的字号/行高/字重/字间距是否与 DESIGN.md 逐行对应？
- [ ] 设计系统的间距比例尺是否全部注册为令牌？
- [ ] pnpm type-check 和 pnpm build-only 是否通过？
