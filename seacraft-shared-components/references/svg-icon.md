# SvgIcon（SVG 雪碧图图标）

> Vue 位置：`jy-digital-patrol-ui/src/components/SvgIcon/index.vue`
> React 位置（mobile）：`jy-digital-patrol-ui/mobile/src/components/SvgIcon.tsx`
> 机制：`vite-plugin-svg-icons` 扫描 `src/assets/icons` 生成 symbol 雪碧图，入口 `import 'virtual:svg-icons-register'` 把雪碧图注入 DOM，`SvgIcon` 用 `<use href="#icon-<name>">` 引用。

## 1. 组件源码（Vue）

```vue
<template>
	<svg :style="iconStyle" :class="iconClass" aria-hidden="true">
		<use :xlink:href="symbolId" />
	</svg>
</template>

<script lang="ts" setup name="SvgIcon">
import { computed } from 'vue';
import type { CSSProperties } from 'vue';

interface SvgProps {
	name: string;
	prefix?: string;
	iconStyle?: CSSProperties;
	iconClass?: string;
}

const props = withDefaults(defineProps<SvgProps>(), {
	prefix: 'icon',
	iconStyle: () => ({ width: '100px', height: '100px' })
});

const symbolId = computed(() => `#${props.prefix}-${props.name}`);
</script>
```

两个细节必须保留：

- `symbolId` 的前缀默认是 `icon`，必须与 Vite 插件的 `symbolId: 'icon-[dir]-[name]'` 对齐。
- `<script setup name="SvgIcon">` 依赖 `unplugin-vue-setup-extend-plus`；没有该插件时改成 `defineOptions({ name: 'SvgIcon' })`，否则组件名可能丢失（影响自动注册与 devtools）。

## 2. 依赖安装

```bash
pnpm add -D vite-plugin-svg-icons
```

| 依赖 | 版本基线 | 作用 |
| --- | --- | --- |
| `vite-plugin-svg-icons` | `^2.0.1` | 扫描 SVG 目录、生成 symbol 雪碧图、提供 `virtual:svg-icons-register` |
| `vite` | `^4.5.1`（本项目） | 插件宿主；插件的 peer 要求是 `>=2.0.0` |
| `unplugin-vue-setup-extend-plus` | `^1.0.0` | 支持 `<script setup name="...">`；不装则改 `defineOptions` |

插件自身会带 `svgo`、`svg-baker` 等依赖，无需单独安装。

## 3. Vite 配置

```ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import vueSetupExtend from 'unplugin-vue-setup-extend-plus/vite';
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons';
import { resolve } from 'node:path';

export default defineConfig({
	plugins: [
		vue(),
		vueSetupExtend({}),
		createSvgIconsPlugin({
			iconDirs: [resolve(process.cwd(), 'src/assets/icons')],
			symbolId: 'icon-[dir]-[name]'
			// svgoOptions: true // 默认就是 true，需要保留动画/特殊属性时再显式配置
		})
	]
});
```

| 选项 | 本项目值 | 说明 |
| --- | --- | --- |
| `iconDirs` | `[src/assets/icons]` | 必须用绝对路径；可配多个目录 |
| `symbolId` | `'icon-[dir]-[name]'` | 必须含 `[name]`；不含 `[dir]` 时子目录不参与 id |
| `svgoOptions` | 默认 `true` | 默认开启压缩；如动画/渐变被破坏，传 `false` 或细粒度配置 |
| `inject` | 默认 `body-last` | 雪碧图节点注入位置 |
| `customDomId` | 默认 `__svg__icons__dom__` | 注入的 `<svg>` 容器 id，排查时可在 DOM 里搜它 |

## 4. 初始化与注册

### 4.1 入口引入雪碧图（必须且只需一次）

```ts
// src/main.ts
import 'virtual:svg-icons-register';
```

- 必须在应用挂载前执行；放在入口最靠前的 import 区域即可（ESM import 会被提升）。
- 不要放进懒加载路由或异步 chunk，否则图标组件先渲染时雪碧图还没注入，图标会是空白。

### 4.2 类型声明

最稳的写法是手写声明（不依赖包的 `exports` 解析）：

```ts
declare module 'virtual:svg-icons-register';
```

插件也自带 `client.d.ts`，支持时可直接引用：

```ts
/// <reference types="vite-plugin-svg-icons/client" />
```

若 TS 报找不到 `vite-plugin-svg-icons/client`（部分 `moduleResolution` 下被 `exports` 限制），回退到上面的手写声明。插件还提供 `virtual:svg-icons-names`（图标名数组），需要枚举图标时可用。

### 4.3 组件注册

三选一：

```ts
// A. 显式全局注册（最直接）
import SvgIcon from '@/components/SvgIcon/index.vue';
app.component('SvgIcon', SvgIcon);
```

```ts
// B. 组件内局部导入
import SvgIcon from '@/components/SvgIcon/index.vue';
```

```ts
// C. unplugin-vue-components 自动注册（本项目采用）
import Components from 'unplugin-vue-components/vite';

Components({
	extensions: ['vue'],
	dts: 'src/components.d.ts'
});
```

`unplugin-vue-components` 默认扫描 `src/components`，因此组件必须放在 `src/components/SvgIcon/index.vue` 才能被自动注册。

## 5. 目录与命名

```text
src/assets/icons/
├── loading.svg                    → #icon-loading             → <SvgIcon name="loading" />
├── muted.svg                      → #icon-muted               → <SvgIcon name="muted" />
├── split-screen.svg               → #icon-split-screen        → <SvgIcon name="split-screen" />
└── fa-pro/
    ├── grid-2.svg                 → #icon-fa-pro-grid-2       → <SvgIcon name="fa-pro-grid-2" />
    └── gallery-thumbnails.svg     → #icon-fa-pro-gallery-thumbnails → name="fa-pro-gallery-thumbnails"
```

规则：

- 文件名（不含 `.svg`）就是 `name`；子目录会以 `-` 拼进 id。
- 根目录文件不会产生双横线：插件会把 `icon--name` 归一成 `icon-name`。
- id 区分大小写。仓库里已有 `unMuted.svg`、`unFavorite.svg`、`tias-fullSeat.svg` 这类驼峰命名，新增图标建议统一 kebab-case，但**不要重命名存量文件**，否则所有引用会断。
- 不要用空格、中文、`#`、`.` 等字符；只用 `[a-z0-9-]` 最稳。
- 不同子目录可以有同名文件（id 会带目录前缀）。

## 6. Props

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `name` | `string` | 必填 | 图标名，对应 symbol id 去掉 `icon-` 前缀的部分 |
| `prefix` | `string` | `'icon'` | symbol id 前缀；只有改过插件 `symbolId` 时才需要动 |
| `iconStyle` | `CSSProperties` | `{ width: '100px', height: '100px' }` | 内联样式；**传了就会整体替换默认值**，要自己补宽高 |
| `iconClass` | `string` | — | 追加到根 `<svg>` 的 class |

模板里 `icon-class` 与 `iconClass` 等价（Vue 会做 kebab/camel 匹配）。

## 7. 用法

### 基础

```vue
<SvgIcon name="loading" />
```

默认渲染 100×100，绝大多数场景需要改尺寸。

### 指定尺寸与颜色（推荐）

```vue
<SvgIcon name="muted" :icon-style="{ width: '20px', height: '20px', color: 'var(--el-color-primary)' }" />
```

`iconStyle` 是内联样式，能覆盖默认的 100px；颜色通过 CSS `color` 传给 SVG 的 `currentColor`。

### 用 class 控制尺寸

```vue
<SvgIcon name="muted" icon-class="muted-icon" />
```

```scss
.muted-icon {
	width: 20px !important;
	height: 20px !important;
	color: var(--el-color-primary);
}
```

**必须带 `!important`**：默认的 `width/height: 100px` 是内联样式，普通 class 优先级不够。项目里大量 `iconClass="svg-icon"` + `!important` 的写法就是这个原因。

### 动态图标

```vue
<SvgIcon :name="isPlaying ? 'playing-light' : 'playing-dark'" :icon-style="{ width: '16px', height: '16px' }" />
```

### 子目录图标

```vue
<SvgIcon name="fa-pro-grid-2" :icon-style="{ width: '14px', height: '14px' }" />
```

### 动画图标

`loading.svg` 内置旋转动画，直接引用即可；雪碧图会保留 SVG 内的 `<style>` 与 animation。若开了 `svgoOptions` 后发现动画丢失，关闭压缩或配置 `removeUnknownsAndDefaults: false` 等保活选项。

### React（mobile）对应组件

```tsx
<SvgIcon name="fa-pro-grid-2" className="text-[14px]" />
```

React 版默认 `size="1em"`，支持 `size`、`className`、`style` 并透传其它 SVG 属性；与 Vue 版共用同一套图标目录和 symbol id。注意 Vue 版默认 100px、React 版默认 1em，不要互相套用尺寸经验。

## 8. 颜色、主题与可访问性

- 图标 SVG 应使用 `fill="currentColor"` 或 `stroke="currentColor"`，组件才能通过 `color` 换色。
- 若 SVG 内部写死 `fill="#xxx"` / `fill="rgb(...)"`，`color` 不会生效；要么改 SVG 为 `currentColor`，要么用 `:deep()`/设计变量覆盖。
- 组件固定 `aria-hidden="true"`，视为装饰性图标。需要语义时在外部补 `aria-label` 或可见文字，不要依赖图标本身被读屏识别。
- 暗色主题切换不需要换图标文件，只要图标用 `currentColor`，由外层 `color` 变量控制。

## 9. 打包行为

- 雪碧图会被编译进 JS（Vite 虚拟模块），运行时注入 `<svg id="__svg__icons__dom__">` 到 `body` 末尾；**不会**在 `dist` 生成独立 SVG 文件，也不需要配置 `public` 或 `assetsInclude`。
- 引用方式是 `#symbol-id` 片段，不是 URL，因此 `base` 子路径部署不影响图标。
- 图标随入口 chunk 走；入口 import 一次即可，不要按路由懒加载 `virtual:svg-icons-register`。
- `svgo` 默认开启；若图标依赖内部 id、渐变引用或动画 class，压缩可能导致渲染异常，先在 dev 验证再上生产。
- SSR / Nuxt 场景没有 `document`，需要把雪碧图注册放到客户端入口；本组件面向 SPA。
- 生产验证：打开页面后 DevTools 搜索 `__svg__icons__dom__`，确认雪碧图存在且 `<symbol id="icon-xxx">` 与使用的 `name` 匹配。

## 10. 新增图标流程

1. 从设计/图标库导出 SVG，保留 `viewBox`，去掉固定 `width/height`（或在组件里统一控制）。
2. 把填充色改成 `currentColor`（需要换色时）。
3. 按用途命名文件，放入 `src/assets/icons/`（或子目录）。
4. 组件里 `<SvgIcon name="文件名（含子目录前缀）" />`。
5. 若新图标在 dev 下不生效：重启 dev server、硬刷新；确认文件名与 `name` 完全一致（含大小写）。
6. 提交前跑一次 `pnpm build && pnpm preview`，确认生产环境图标正常。

## 11. 故障排查

| 现象 | 常见原因 | 处理 |
| --- | --- | --- |
| 图标空白，DOM 里没有雪碧图 | 忘记 `import 'virtual:svg-icons-register'`，或写进了懒加载 chunk | 移到入口 `main.ts` |
| 图标空白，但有 `__svg__icons__dom__` | `name` 与 symbol id 不匹配（大小写、子目录前缀、拼写） | 在 DOM 里搜 `symbol id` 对照 |
| 所有图标都是 100×100 | 没传 `iconStyle`，class 又被内联样式压住 | 传 `iconStyle`，或 class 用 `!important` |
| 传了 `:icon-style="{ color }"` 后尺寸异常 | `iconStyle` 整体替换默认值，没写宽高 | 同时给 `width` / `height` |
| 颜色改不动 | SVG 内部写死 `fill`，不是 `currentColor` | 改 SVG 或覆盖 fill |
| 子目录图标找不到 | `name` 少了目录前缀，或插件 `symbolId` 没配 `[dir]` | `name="fa-pro-grid-2"`，检查 `symbolId` |
| 新增图标 dev 不生效 | 开发服务器缓存 / 文件名大小写不一致 | 重启 dev server，核对文件名 |
| 生产环境个别 SVG 变形/动画丢失 | SVGO 压缩移除了属性 | 关掉 `svgoOptions` 或细化配置 |
| TS 报找不到 `virtual:svg-icons-register` | 没引 client 类型 | `/// <reference types="vite-plugin-svg-icons/client" />` |

## 12. 接入检查清单

- [ ] 已安装 `vite-plugin-svg-icons`，Vite 配置了 `iconDirs` 与 `symbolId`。
- [ ] 入口已 `import 'virtual:svg-icons-register'`，且不在懒加载 chunk。
- [ ] `vite-env.d.ts` 引了 `vite-plugin-svg-icons/client`。
- [ ] `SvgIcon` 已全局注册、局部导入或被 `unplugin-vue-components` 自动注册。
- [ ] 图标目录、文件名与 `name` 完全对应，子目录用 `[dir]` 前缀。
- [ ] 尺寸通过 `iconStyle` 或 `!important` class 控制，没有沿用默认 100px。
- [ ] 需要换色的图标使用 `currentColor`。
- [ ] `pnpm build && pnpm preview` 后图标正常。
