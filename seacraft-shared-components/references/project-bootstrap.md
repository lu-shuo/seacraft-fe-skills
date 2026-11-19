# 工程接入总览：依赖、初始化与打包

> 适用范围：本文面向 **Vite + `jy-player-ui` + `jy-digital-patrol-ui` 封装**（`MultiWindowPlayer` / `PlayerMask` / `SvgIcon`）。
> `jy-application-resourcemanage-ui` 的 `GlobalUploader` 是 **Vue CLI 5 / Webpack** 栈，依赖与初始化不同，见 [`global-uploader.md`](global-uploader.md)。

本文把 `jy-player-ui`、`MultiWindowPlayer`、`PlayerMask`、`SvgIcon` 的工程级接线集中到一处，目标是一次性跑通：装依赖 → 配 Vite → 配入口 → 配引擎/配置 → 构建验证。

组件 API 分别见 [`player-kmedia.md`](player-kmedia.md)、[`multi-window-player.md`](multi-window-player.md)、[`player-mask.md`](player-mask.md)、[`svg-icon.md`](svg-icon.md)。

## 1. 依赖矩阵

本项目（`jy-digital-patrol-ui`）实际版本：

| 依赖 | 版本 | 类型 | 被谁需要 |
| --- | --- | --- | --- |
| `vue` | `^3.5.18` | runtime | 全部组件 |
| `vue-router` | `^4.2.4` | runtime | 应用（非组件强依赖） |
| `pinia` | `^2.1.6` | runtime | 应用（非组件强依赖） |
| `element-plus` | `2.10.6` | runtime | `MultiWindowPlayer` 的 `el-popover`、`ElMessage` |
| `jy-player-ui` | `^1.7.8` | runtime | 播放器组件与引擎封装 |
| `@vueuse/core` | `^10.4.1` | runtime | `MultiWindowPlayer` 的 `useFullscreen` |
| `mitt` | `^3.0.1` | runtime | 播放器多屏同步 |
| `dayjs` | `^1.11.11` | runtime（可选） | 调用方的打点/时间格式化，组件本身不依赖 |
| `vite` | `^4.5.1` | dev | 构建 |
| `@vitejs/plugin-vue` | `^4.3.4` | dev | 构建 `.vue` |
| `typescript` | `5.2.2` | dev | 类型检查 |
| `vue-tsc` | `^1.8.11` | dev | `typecheck` 脚本 |
| `sass` | `1.90.0` | dev | SCSS 与 Element Plus sass 主题 |
| `vite-plugin-svg-icons` | `^2.0.1` | dev | `SvgIcon` 雪碧图 |
| `unplugin-vue-setup-extend-plus` | `^1.0.0` | dev | 支持 `<script setup name="...">` |
| `unplugin-auto-import` | `^0.16.6` | dev | 自动导入 `computed`、`ElMessage` 等 |
| `unplugin-vue-components` | `^0.25.2` | dev | 自动注册 `SvgIcon`、`el-popover` |

关键点：业务封装组件**不是自包含的**。

- `MultiWindowPlayer` 用了 `computed`、`onBeforeUnmount`、`onDeactivated` 但没 import（靠 auto-import），用了 `ElMessage` 但没 import（靠 Element Plus resolver），模板里用了全局 `SvgIcon` 与 `el-popover`。
- `PlayerMask` 多个组件用了 `computed` 但没 import，模板里用了全局 `SvgIcon`。
- `SvgIcon` 依赖 `virtual:svg-icons-register` 与 Vite 插件。

所以要么照本文配齐自动导入，要么在复制组件时补齐显式 import（见第 9 节）。

## 2. 安装

```bash
# .npmrc（jy-player-ui 在公司私有源）
# @jy-player-ui:registry=https://nexus.kedacom.com/repository/public-npm/
# registry=https://nexus.kedacom.com/repository/public-npm/

# runtime
pnpm add vue vue-router pinia element-plus @vueuse/core mitt jy-player-ui

# dev
pnpm add -D vite @vitejs/plugin-vue typescript vue-tsc sass \
  vite-plugin-svg-icons unplugin-vue-setup-extend-plus \
  unplugin-auto-import unplugin-vue-components
```

按需追加：`dayjs`（时间格式化）、`@element-plus/icons-vue`（Element 图标）、`@vitejs/plugin-vue-jsx`（需要 JSX 时）。

Node 建议 18+（本项目脚本按 Node 20 使用）。

## 3. 目录基线

```text
project/
├── index.html
├── vite.config.ts
├── tsconfig.json
├── static/                              # publicDir（项目约定，默认 public 亦可）
│   ├── config/global-development.json   # 运行时配置
│   └── lib/kmedia/kmedia-uni.js         # 播放引擎
└── src/
    ├── main.ts
    ├── vite-env.d.ts
    ├── components.d.ts                  # unplugin-vue-components 生成
    ├── auto-import.d.ts                 # unplugin-auto-import 生成
    ├── assets/icons/                    # SVG 源文件
    ├── components/
    │   ├── SvgIcon/index.vue
    │   ├── PlayerMask/
    │   └── MultiWindowPlayer/index.vue
    ├── config/playerErrorCode.ts        # PlayerErrorMask 错误码表
    ├── hooks/
    │   ├── useInlineCssVars.ts
    │   ├── useWatermark.ts
    │   └── useRecordPlayerState.ts
    ├── styles/
    └── utils/global.ts                  # getCustomConfig / Global.init
```

### 3.1 复制清单与可替换项

组件源码之间有隐式依赖，复制时按下表一起带走或替换：

| 文件 / 目录 | 被谁依赖 | 是否可直接替换 |
| --- | --- | --- |
| `components/SvgIcon/index.vue` | 全部遮罩、`MultiWindowPlayer` | 否，图标体系基础 |
| `assets/icons/*.svg` | `SvgIcon` | 否，至少要有用到的图标 |
| `components/PlayerMask/*.vue` | `MultiWindowPlayer`、页面 | 可替换为本地遮罩，但需保持插槽契约 |
| `components/MultiWindowPlayer/index.vue` | 页面 | 可整体替换为自研多画面 |
| `hooks/useInlineCssVars.ts` + `hooks/useDarkMode.ts` | `MultiWindowPlayer`、`PlayerMask` | 可替换为项目主题方案（需保留 `--window-prompt-*` / `--room-detail-player-*` 变量或改样式） |
| `hooks/useRecordPlayerState.ts` | `MultiWindowPlayer` | 可替换为页面本地状态，需保留 `loadState` 结构 |
| `hooks/useWatermark.ts` + `shared/watermark.ts` + `api/modules/showSetApi.ts` | `MultiWindowPlayer` | 可替换为固定水印参数 |
| `config/playerErrorCode.ts` | `PlayerErrorMask` | 可裁剪，但错误归类会退化 |
| `stores/modules/sysConfig.ts` 的 `playerExceptionMsgConfig` | `PlayerErrorMask` | 必须提供同名 store 或改掉取数方式，否则 import 失败 |
| `utils/global.ts` 的 `getCustomConfig()` | `MultiWindowPlayer`、`useWatermark` | 必须替换为项目配置读取 |
| `styles/var.scss` | Vite `additionalData` | 可删配置；删了就不需要该文件 |

最小图标集（缺一个对应 UI 就空白）：

| 使用方 | 图标 `name` |
| --- | --- |
| `SvgIcon` 示例 / 遮罩 | `loading`、`play-btn`、`replay-btn` |
| `MultiWindowPlayer` | `muted`、`unMuted`、`play`、`split-screen`、`screen-1`、`screen-2`、`screen-3`、`screen-4`、`popper-close` |
| 业务调用方（可选） | `progress-alarm`、`progress-alarm-new` 等打点/告警图标 |

## 4. Vite 配置

与业务仓一致的版本（推荐，可直接跑复制来的组件）：

```ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import vueSetupExtend from 'unplugin-vue-setup-extend-plus/vite';
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons';
import { resolve } from 'node:path';

const root = process.cwd();
const r = (p: string) => resolve(root, p);

export default defineConfig({
	root,
	publicDir: 'static', // 若用默认 public 目录，改为 'public'
	resolve: {
		alias: { '@': r('src') }
	},
	css: {
		preprocessorOptions: {
			scss: {
				api: 'modern',
				silenceDeprecations: ['legacy-js-api'],
				// 项目主题变量；没有该文件时删掉这一行
				additionalData: '@use "@/styles/var.scss" as *;'
			}
		}
	},
	plugins: [
		vue(),
		// 支持 <script setup name="SvgIcon">
		vueSetupExtend({}),
		// SVG 雪碧图
		createSvgIconsPlugin({
			iconDirs: [r('src/assets/icons')],
			symbolId: 'icon-[dir]-[name]'
		}),
		// 自动导入 Vue API / Element Plus API（ElMessage 等）
		AutoImport({
			imports: ['vue', 'vue-router', 'pinia', '@vueuse/core'],
			resolvers: [ElementPlusResolver()],
			dts: 'src/auto-import.d.ts'
		}),
		// 自动注册 src/components 下的组件 + Element Plus 组件
		Components({
			extensions: ['vue'],
			resolvers: [ElementPlusResolver({ importStyle: 'sass' })],
			dts: 'src/components.d.ts'
		})
	],
	build: {
		outDir: 'dist',
		target: 'es2015',
		minify: 'esbuild'
	},
	server: {
		host: '0.0.0.0',
		port: 3000
	}
});
```

插件作用与缺失后果：

| 插件 | 缺了会怎样 |
| --- | --- |
| `@vitejs/plugin-vue` | `.vue` 无法编译 |
| `unplugin-vue-setup-extend-plus` | `<script setup name="SvgIcon">` 的组件名可能丢失；改用 `defineOptions` 可替代 |
| `vite-plugin-svg-icons` | `virtual:svg-icons-register` 解析失败，图标全空 |
| `unplugin-auto-import` + `ElementPlusResolver` | `MultiWindowPlayer` 里 `computed`、`ElMessage` 未定义 |
| `unplugin-vue-components` + `ElementPlusResolver` | `SvgIcon`、`el-popover` 未注册 |

注意：

- `ElementPlusResolver({ importStyle: 'sass' })` 需要 `sass`。若首次跑通不想处理主题，可改成 `importStyle: 'css'`，或在入口全量 `import 'element-plus/dist/index.css'`。
- `additionalData` 指向的 `src/styles/var.scss` 必须存在，否则构建直接失败；没有就删掉该配置。
- 如果不用 `unplugin-vue-components`，就要显式 `app.component('SvgIcon', SvgIcon)` 并全量 `app.use(ElementPlus)`。

## 5. 入口初始化

`index.html`：

```html
<!doctype html>
<html lang="zh-CN">
	<head>
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
		<title>App</title>
		<!-- 播放引擎必须在播放器挂载前加载 -->
		<script src="/lib/kmedia/kmedia-uni.js" defer></script>
	</head>
	<body>
		<div id="app"></div>
		<script type="module" src="/src/main.ts"></script>
	</body>
</html>
```

引擎文件的位置跟随 `publicDir`：本项目 `publicDir: 'static'`，所以文件放在 `static/lib/kmedia/kmedia-uni.js`；若用默认 `public`，则放 `public/lib/kmedia/kmedia-uni.js`，URL 仍是 `/lib/kmedia/kmedia-uni.js`。

`src/main.ts`（顺序不要随意调整）：

```ts
import { createApp } from 'vue';
import App from './App.vue';
import router from '@/routers';
import pinia from '@/stores';
import JyPlayerUI from 'jy-player-ui';
import Global from '@/utils/global';

// 1. SVG 雪碧图（必须一次，且在组件渲染前）
import 'virtual:svg-icons-register';

// 2. 全局样式（Element Plus 暗色变量 + 项目主题）
import 'element-plus/theme-chalk/dark/css-vars.css';
import '@/styles/reset.scss';
import '@/styles/common.scss';
import '@/styles/element.scss';
import '@/styles/light.scss';
import '@/styles/dark.scss';

async function bootstrap() {
	// 3. 运行时配置（PLAYER_*、水印等），内部写入 localStorage
	await Global.init();

	const app = createApp(App);

	// 4. 插件与全局组件
	app.use(pinia).use(router).use(JyPlayerUI);

	// 如果未使用 unplugin-vue-components，手动注册：
	// import SvgIcon from '@/components/SvgIcon/index.vue';
	// app.component('SvgIcon', SvgIcon);

	app.mount('#app');
}

bootstrap();
```

初始化顺序要点：

1. `virtual:svg-icons-register` 必须早于任何 `SvgIcon` 渲染。
2. `Global.init()` 必须在播放器挂载前完成，`MultiWindowPlayer` 会读 `PLAYER_PAUSED_ON_REFRESH_PAGE`。
3. `app.use(JyPlayerUI)` 注册 `JyKmediaPlayer` / `JyPlayerControl`；`MultiWindowPlayer` 模板依赖它们。
4. `svg-icons-register` 不要放进异步路由 chunk。

## 6. 运行时配置与主题

本项目把配置放 `static/config/global-<mode>.json`，由 `Global.init()` 写入 `localStorage` 的 `STORAGE_KEY_CUSTOM_CONFIG`：

```json
{
	"API_BASE_URL": "/jy-digital-patrol",
	"AJAX": { "timeout": 30000, "withCredentials": true },
	"CUSTOM_CONFIG": {
		"PLAYER_PROTOCOL": "webrtc",
		"RECORD_PROTOCOL": "webrtc",
		"PLAYER_PAUSED_ON_REFRESH_PAGE": true,
		"PLAYER_FILL_TYPE": "horizontal",
		"PLAYER_RETRY_INTERVAL": 30000,
		"SHOW_PLAYER_WATER_MARK": "Y"
	}
}
```

文件位置同样跟随 `publicDir`：`static/config/global-development.json`（默认 `public` 时为 `public/config/...`）。`Global.init()` 内部用 `getContextPath() + '/config/global-<mode>.json'` 拼接，子路径部署时不要写死根路径。

组件直接读取的键：

| 键 | 读取方 | 作用 |
| --- | --- | --- |
| `PLAYER_PAUSED_ON_REFRESH_PAGE` | `MultiWindowPlayer` | 刷新后暂停 / 静音续播 |
| `SHOW_PLAYER_WATER_MARK`、`SPV_WATER_MARK_CONFIG` | `useWatermark` | 水印兜底配置 |
| `PLAYER_PROTOCOL`、`RECORD_PROTOCOL` | 调用方页面 | 决定 `src` 形态与 `transport`，组件不直接读 |

若新工程没有 `Global.init()`：

- 要么把 `CUSTOM_CONFIG` 写进 `localStorage` 同名 key；
- 要么把 `MultiWindowPlayer` / `useWatermark` 里的 `global.getCustomConfig()` 换成自己的配置源；
- 否则 `PLAYER_PAUSED_ON_REFRESH_PAGE` 会走默认值 `true`（不报错，但行为可能不符合预期）。

主题相关：

- `MultiWindowPlayer` 强制 `--jy-player-primary-color: var(--el-color-primary)`；没有 Element Plus 主题变量时按钮/边框颜色异常。
- 弹层背景使用 `--room-detail-player-*`，由组件内 `useInlineCssVars` 注入，不依赖外部主题文件。
- `PlayerMask` 的 `--window-prompt-*` 由组件自身注入；`followTheme=false` 时使用硬编码深色。
- 全局样式入口顺序：reset → common → element → light/dark。

## 7. TypeScript 配置

`tsconfig.json` 关键项：

```json
{
	"compilerOptions": {
		"target": "ESNext",
		"module": "ESNext",
		"moduleResolution": "Node",
		"strict": true,
		"jsx": "preserve",
		"baseUrl": "./",
		"paths": { "@/*": ["src/*"] },
		"types": ["node", "element-plus/global", "jy-player-ui/lib/src/components.d.ts"]
	},
	"include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.vue", "src/components.d.ts", "src/auto-import.d.ts"]
}
```

`src/vite-env.d.ts`：

```ts
/// <reference types="vite/client" />
/// <reference types="vite-plugin-svg-icons/client" />

// 若上面一行在 moduleResolution: bundler 下解析失败，改用：
// declare module 'virtual:svg-icons-register';

declare module '*.vue' {
	import type { DefineComponent } from 'vue';
	const component: DefineComponent<{}, {}, any>;
	export default component;
}
```

`jy-player-ui/lib/src/components.d.ts` 提供 `JyKmediaPlayer` 等全局组件类型；漏掉它时模板不报错但失去类型提示。

## 8. 构建与产物验证

```bash
pnpm dev        # 首次跑通
pnpm build      # 生产构建
pnpm preview    # 本地预览 dist
```

构建后逐项确认：

- [ ] `dist/index.html` 引用了引擎脚本，路径与部署 `base` 匹配（子路径部署时绝对路径 `/lib/...` 可能 404，改成相对路径或拼接 `base`）。
- [ ] `dist` 中没有独立的图标 SVG 文件；雪碧图已打进 JS，浏览器 DOM 中存在 `#__svg__icons__dom__`。
- [ ] `<symbol id="icon-loading">` 等 id 与代码里的 `name` 一致。
- [ ] 播放器页面无 `window.KMediaUni ... 未挂载` / `Aliplayer 未挂载` 报错。
- [ ] `MultiWindowPlayer` 主画面 `loadedmetadata` 后出现控制栏。
- [ ] 遮罩颜色、按钮主色与主题一致。
- [ ] 刷新页面后播放器行为符合 `PLAYER_PAUSED_ON_REFRESH_PAGE`。
- [ ] `pnpm typecheck` 通过（若项目有该脚本）。

打包注意：

- `vite-plugin-svg-icons` 是运行时注入雪碧图，不做静态资源拷贝；不要手动把 `src/assets/icons` 复制到 `public`。
- 引擎脚本是不打包的全局脚本，必须放 `publicDir`（本项目为 `static`）或由网关/CDN 提供。
- `jy-player-ui` 的 CSS 已随 ESM 产物自动引入，不需要额外 import 样式文件。
- 生产压缩（esbuild/terser）不影响 `virtual:svg-icons-register`，但不要把入口 import 摇掉（它被标记为副作用模块）。

## 9. 不配自动导入时的最小改动

如果目标工程不想引入 `unplugin-*`，保留复制来的组件时需要补：

| 文件 | 补齐内容 |
| --- | --- |
| `MultiWindowPlayer/index.vue` | `import { computed, onBeforeUnmount, onDeactivated } from 'vue'`、`import { ElMessage } from 'element-plus'`、`import SvgIcon from '@/components/SvgIcon/index.vue'` |
| `PlayerMask/PlayerErrorMask.vue` | `import { computed } from 'vue'` |
| `PlayerMask/PlayerLoadingMask.vue` | `import { computed } from 'vue'` |
| `SvgIcon/index.vue` | 已有显式 import；把 `setup name` 改 `defineOptions({ name: 'SvgIcon' })`（若不用 setup-extend） |
| 入口 | `app.use(ElementPlus)`（注册 `el-popover`）、`app.component('SvgIcon', SvgIcon)` |

这比配置插件改动更多，且容易漏，因此优先推荐第 4 节的自动导入方案。

## 10. 一次性跑通清单

1. `pnpm add` 第 2 节依赖（注意私有源）。
2. 把 `SvgIcon`、`PlayerMask`、`MultiWindowPlayer`、相关 `hooks/config/utils` 复制到 `src/` 对应目录。
3. 建 `src/assets/icons`，放入至少一个 `loading.svg`。
4. 按第 4 节写 `vite.config.ts`，确认 `additionalData` 指向的文件存在。
5. 按第 5 节写 `index.html` 与 `main.ts`，确认 `virtual:svg-icons-register`、`Global.init()`、`app.use(JyPlayerUI)` 都在。
6. 按第 6 节准备 `static/config/global-development.json`（或在 `localStorage` 写入 `CUSTOM_CONFIG`）。
7. 放好 KMedia 引擎到 `static/lib/kmedia/kmedia-uni.js`。
8. 按第 7 节补 `tsconfig` 与 `vite-env.d.ts`。
9. `pnpm dev`，先验证 `SvgIcon`（图标显示）→ 再验证单播放器 → 再验证遮罩 → 最后验证 `MultiWindowPlayer`。
10. `pnpm build && pnpm preview`，按第 8 节逐项验收。

## 11. 常见报错对照

| 报错 / 现象 | 原因 | 处理 |
| --- | --- | --- |
| `Failed to resolve import "virtual:svg-icons-register"` | 没装插件或 `iconDirs` 配置错误 | 加 `vite-plugin-svg-icons` 与插件配置 |
| TS 找不到 `virtual:svg-icons-register` | 缺 client 类型 | 引 `vite-plugin-svg-icons/client` |
| `ElMessage is not defined` | 没配 `unplugin-auto-import` 的 Element Plus resolver | 配 resolver 或显式 import |
| `Failed to resolve component: el-popover` | Element Plus 未注册 | 配 `unplugin-vue-components` 或 `app.use(ElementPlus)` |
| `Failed to resolve component: SvgIcon` | 组件不在 `src/components` 或未注册 | 移到扫描目录 / 全局注册 / 局部导入 |
| 图标全空白 | 雪碧图未注入或 `name` 不匹配 | 查 `#__svg__icons__dom__` 与 `symbol id` |
| 图标全是 100×100 | 默认内联尺寸未覆盖 | `:icon-style` 或 `!important` class |
| `window.KMediaUni/window.KMediaMobile 未挂载` | 引擎脚本没加载 / 加载晚于挂载 | 放 `index.html` 或加挂载门禁 |
| 播放器空白、无控制栏 | 容器无高度 / 控制栏未接 | 给父容器宽高比，接 `#control` |
| `MultiWindowPlayer` 无控制栏 | 主画面未 `loadedmetadata` 或引擎异常 | 查主画面加载与 error 事件 |
| 遮罩透明 / 颜色不对 | 主题变量缺失或 `followTheme` 不符合预期 | 补 `--el-color-primary`，显式传 `followTheme` |
| 构建报找不到 `@/styles/var.scss` | `additionalData` 指向了不存在的文件 | 删掉该配置或补文件 |

## 12. 最小依赖子集

| 只使用 | 必需依赖 | 必需插件 |
| --- | --- | --- |
| `SvgIcon` | `vue` | `@vitejs/plugin-vue`、`vite-plugin-svg-icons`、`unplugin-vue-setup-extend-plus`（或改 `defineOptions`） |
| `PlayerMask` | `vue` + `SvgIcon` | 同上 |
| `MultiWindowPlayer` | 上面全部 + `element-plus`、`jy-player-ui`、`@vueuse/core`、`mitt` | 上面全部 + `unplugin-auto-import`、`unplugin-vue-components` |
| `JyKmediaPlayer` 单播放器 | `vue` + `jy-player-ui` | `@vitejs/plugin-vue`；引擎脚本 |
