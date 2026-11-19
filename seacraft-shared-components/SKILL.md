---
name: seacraft-shared-components
description: Company public frontend component integration guide for Vue 3 business apps, covering the jy-player-ui player suite (JyKmediaPlayer, JyKmediaMobilePlayer, JyAliPlayer, JyPlayerControl, JyMobilePlayerControl, JyControlProgress, JySlider), the jy-digital-patrol-ui business wrappers (MultiWindowPlayer multi-view player, PlayerMask loading/error/pause/end masks, SvgIcon SVG sprite icon), the jy-application-resourcemanage-ui GlobalUploader (chunked upload, MD5, instant upload, resume, global panel), and the full project wiring (dependencies, Vite/Webpack config, entry registration, build verification). Use when selecting a player, loading the KMedia/Aliplayer engine, wiring props/events/slots/exposes, building a custom control bar, syncing multiple screens, embedding a 1-4 view player, choosing player masks, adding SVG icons, implementing chunked upload, or bootstrapping a project so the whole flow runs end-to-end.
license: MIT
metadata:
  author: seacraft
  version: "0.4.0"
---

# Seacraft 前端公共组件使用规范

公司公共组件分三类：

- `jy-player-ui` 以 npm 包发布，业务项目只做**消费**（props / events / slots / exposes，不改 `node_modules`）。
- `jy-digital-patrol-ui` 的 `MultiWindowPlayer` / `PlayerMask` / `SvgIcon` 是 **Vite** 业务仓封装，按需复制并按其依赖清单接线。
- `jy-application-resourcemanage-ui` 的 `GlobalUploader` 是 **Vue CLI 5 / Webpack** 业务仓全局上传器，按需复制并接入 `vue-simple-uploader`。

当前覆盖 [`jy-player-ui`](references/player-overview.md) 播放器套件（版本基线 `1.11.0`）、[`jy-digital-patrol-ui`](references/multi-window-player.md) 的播放器封装与图标体系、[`jy-application-resourcemanage-ui`](references/global-uploader.md) 的分片上传，以及 [Vite 工程级接线](references/project-bootstrap.md)。后续公共组件按同一结构追加参考文档。

## 适用范围

- 适用：Vue 3 业务项目接入 `jy-player-ui`，含 PC 端 kmedia、H5 端 kmedia-mobile、阿里播放器，以及单页 / 多屏 / 双屏对比场景。
- 适用：在 `jy-digital-patrol-ui`（或复制其封装的工程）中使用 1–4 画面播放器、播放器遮罩、SVG 图标。该工程依赖 `jy-player-ui@^1.7.8`，升级依赖后需回归封装层用到的 props / 方法。
- 适用：在 `jy-application-resourcemanage-ui`（或复制其封装的工程）中接入大文件分片上传、秒传、断点续传、全局上传面板。该工程是 Vue CLI 5 + Vue 3.2 + Element Plus 2.10，不是 Vite。
- 适用：新 Vite 工程从零接线——安装依赖、配置 Vite 插件、入口注册、引擎脚本、运行时配置与构建验证，按 [`project-bootstrap.md`](references/project-bootstrap.md) 一次跑通。
- 不适用：修改播放器/上传器库本身、调整其构建与发布流程。遇到库缺陷按文末「边界」处理。
- 分工：目录落位遵循 `seacraft-frontend-spa-conventions`，Vue 写法遵循 `seacraft-vue-skills`。三者冲突时，落位以目录规范为准，代码写法以 Vue 规范为准。

## 使用原则

1. **先定引擎，再选组件**。`JyKmediaPlayer` / `JyKmediaMobilePlayer` / `JyAliPlayer` 都依赖全局引擎脚本，引擎不存在时组件只打印错误日志并渲染空壳。
2. **只用发布版**。业务仓库 `package.json` 依赖 `jy-player-ui`；示例代码里的 `@jy-player-ui/components` 是库内 workspace 包，业务项目不可引用。
3. **能力优先走公开 API**。行为差异先查 props / slots / exposes；不要用内部 DOM 类名（`.jkp-*`、`.jap-*`）驱动业务逻辑。唯一例外是「多屏交换画面」这类官方示例级 DOM 操作，按 [`player-multi-screen.md`](references/player-multi-screen.md) 的约束使用。
4. **一个实例一个 `domId`**。`domId` 会参与 DOM id 与 `sessionStorage` key，同页重复会导致多播放器互相覆盖音量与刷新状态。
5. **父容器必须有尺寸**。播放器自身按 `100%` 填充，容器无高度就是「空白播放器」。
6. **`src` 不是响应式源**。改 `src` 后必须显式调用 `reloadVideo()`（`JyAliPlayer` 无此方法，见其文档）。
7. **多屏只用一个 emitter**。在同一 feature 容器创建 `mitt()` 实例向下传递，不要每个播放器各建一个。
8. **不提前抽象**。单页只封装一层播放器组件；出现第二个真实使用方再提升为跨页组件。
9. **多画面优先复用 `MultiWindowPlayer`**。巡课/教室详情这类 1–4 画面场景不要重新拼装多个 `JyKmediaPlayer`；只有每路需要独立控制栏时才自行组合。
10. **遮罩统一用 `PlayerMask`**。加载、错误、暂停、结束、空态分别用 `PlayerLoadingMask` / `PlayerErrorMask` / `PlayerPauseMask` / `PlayerEndMask` / `PlayerPromptIllustration`，不要每个页面重写一套。
11. **图标统一用 `SvgIcon`**。不要新增图标字体、内联 SVG 或图片代替；图标源文件放 `src/assets/icons`，通过 `vite-plugin-svg-icons` 生成雪碧图。
12. **工程接线一次配齐**。依赖、Vite 插件、入口注册、引擎脚本、运行时配置、构建验证按 [`project-bootstrap.md`](references/project-bootstrap.md) 一起处理，不要只复制组件源码就运行。
13. **大文件上传统一用 `GlobalUploader`**。不要自己拼 `FormData` 分片或另引上传库；通过 `mittBus.emit('OPEN_GLOBAL_UPLOADER', { params, options, files })` 复用分片、MD5、秒传、断点续传与全局面板。
14. **上传器是单例**。只在 `App.vue` 挂载一次；`#global-uploader` / `#global-uploader-btn` 是硬编码 id，重复挂载会冲突、事件重复。

## 选择组件

| 场景 | 组件 | 配套 | 必需全局依赖 |
| --- | --- | --- | --- |
| PC / 桌面浏览器，kmedia 协议（WebRTC / RTSP / FLV / HLS / 文件） | `JyKmediaPlayer` | `JyPlayerControl` | `window.KMediaUni`（缺失时回退 `window.KMediaMobile`） |
| H5 / 移动端浏览器，kmedia-mobile 协议 | `JyKmediaMobilePlayer` | `JyMobilePlayerControl` | `window.KMediaMobile` |
| 标准 HLS / FLV / MP4（阿里云或第三方 CDN） | `JyAliPlayer` | `JyPlayerControl` | `window.Aliplayer` |
| 巡课 / 教室详情，1–4 画面共用一套控制栏 | `MultiWindowPlayer`（业务仓封装） | 内嵌 `JyPlayerControl` | `window.KMediaUni` |
| 加载 / 错误 / 暂停 / 结束 / 空态遮罩 | `PlayerMask/*`（业务仓通用） | `SvgIcon` | 雪碧图已注入 |
| SVG 图标 | `SvgIcon` | `vite-plugin-svg-icons` | `virtual:svg-icons-register` |
| 大文件分片、秒传、断点续传、全局上传面板 | `GlobalUploader`（业务仓全局上传器） | `vue-simple-uploader` | `app.use(uploader)` + 上传器 CSS + Element Plus 图标 |
| 只需要进度条、音量条、打点能力 | `JyControlProgress` / `JySlider` | — | 无（API 见 [`player-kmedia.md`](references/player-kmedia.md) 第 9 节） |

同一页面同时出现 PC 与 H5 播放器时，两个引擎脚本都要引入；`JyKmediaPlayer` 优先使用 `window.KMediaUni`，只有它不存在时才回退到 `window.KMediaMobile`。

## 依赖速查

完整版本、Vite 配置、入口顺序与构建验证见 [`project-bootstrap.md`](references/project-bootstrap.md)。

| 使用范围 | 运行时/构建依赖 | 初始化关键点 |
| --- | --- | --- |
| 单播放器 | `vue`、`jy-player-ui` | `app.use(JyPlayerUI)`；`index.html` 引 `kmedia-uni.js` |
| 多画面 | 单播放器全部 + `element-plus`、`@vueuse/core`、`mitt` | `ElMessage` / `el-popover` / `SvgIcon` / `computed` 需自动导入或显式 import |
| 遮罩 | `vue` + `SvgIcon` | 雪碧图已注入；父级 `position: relative` 且有高度 |
| 图标 | `vue` + `vite-plugin-svg-icons` | Vite 插件 + `import 'virtual:svg-icons-register'` + 组件注册 |
| 分片上传（Vue CLI/Webpack） | `vue-simple-uploader`、`spark-md5`、`uuid`、`mitt`、`element-plus` | `app.use(uploader)` + `vue-simple-uploader/dist/style.css` + 图标注册；详见 [`global-uploader.md`](references/global-uploader.md) |
| 业务封装脚本 | `unplugin-vue-setup-extend-plus` | 支持 `<script setup name="...">`，或用 `defineOptions` 替代 |

## 最小接入流程

完整工程接线（依赖版本、Vite 插件、入口初始化、构建验证）见 [`project-bootstrap.md`](references/project-bootstrap.md)；下面是最小步骤。

1. **配置私有源并安装**：`npm i jy-player-ui`（registry 见 [`player-overview.md`](references/player-overview.md)）。
2. **引入引擎脚本**：放在 `index.html` 或入口 HTML，位于应用挂载之前；异步加载时用 `window.KMediaUni` 等全局变量做挂载门禁。
3. **注册组件**：`app.use(JyPlayerUI)` 全局注册，或按需 `import { JyKmediaPlayer } from 'jy-player-ui'`；类型从同一入口导入。用 `SvgIcon` / `PlayerMask` 时同时配好雪碧图与图标组件注册。
4. **给容器定尺寸**：`aspect-ratio: 16 / 9` 或显式高度，父级 `position: relative` 以便蒙层定位。
5. **组装播放器与控制栏**：用 `#control` 插槽把插槽 props `v-bind` 给 `JyPlayerControl`，不要手工拼装状态。
6. **按需叠加业务封装**：多画面用 `MultiWindowPlayer`，状态遮罩用 `PlayerMask`，图标用 `SvgIcon`。

```vue
<template>
	<div class="player-box">
		<JyKmediaPlayer
			ref="playerRef"
			dom-id="lesson-1"
			:src="src"
			:is-live="false"
			:paused-on-refresh-page="false"
			show-mute-btn
			@loadedmetadata="onLoaded"
			@error="onError"
		>
			<template #loading>加载中…</template>
			<template #mask="{ KMEDIA_STATUS, play, errorMessage }">
				<div v-if="KMEDIA_STATUS === 'PAUSED'" class="mask" @click.stop="play">继续播放</div>
				<div v-else-if="KMEDIA_STATUS === 'ENDED'" class="mask" @click.stop="play">重播</div>
				<div v-else-if="KMEDIA_STATUS === 'ERROR'" class="mask">{{ errorMessage }}</div>
			</template>
			<template #control="controlProps">
				<JyPlayerControl v-bind="controlProps" :is-live="false" />
			</template>
		</JyKmediaPlayer>
	</div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { JyKmediaPlayer, JyPlayerControl, type JyKmediaPlayerInstance } from 'jy-player-ui';

const playerRef = ref<JyKmediaPlayerInstance>();
const src = ref('https://example.com/lesson.mp4');
const onLoaded = () => console.info('loaded');
const onError = (err: { code: string; message: string }) => console.error(err.code, err.message);
</script>

<style scoped>
.player-box {
	position: relative;
	width: 100%;
	aspect-ratio: 16 / 9;
	background: #000;
}
.mask {
	position: absolute;
	inset: 0;
	display: flex;
	align-items: center;
	justify-content: center;
	background: rgb(0 0 0 / 70%);
}
</style>
```

## 按场景读取参考文档

- 工程级依赖、Vite 插件、入口初始化、构建验证（从零跑通先读这篇）：[`project-bootstrap.md`](references/project-bootstrap.md)
- 安装、引擎脚本、注册、样式定制、数据源格式、故障排查：[`player-overview.md`](references/player-overview.md)
- PC 端 `JyKmediaPlayer` + `JyPlayerControl` 完整 API 与配方（含独立使用 `JyControlProgress` / `JySlider`）：[`player-kmedia.md`](references/player-kmedia.md)
- H5 端 `JyKmediaMobilePlayer` + `JyMobilePlayerControl`：[`player-kmedia-mobile.md`](references/player-kmedia-mobile.md)
- 阿里播放器 `JyAliPlayer`：[`player-ali.md`](references/player-ali.md)
- 多屏同步、双屏对比、画中画、stage 分屏：[`player-multi-screen.md`](references/player-multi-screen.md)
- 业务内嵌 1–4 画面播放器（分屏布局、视角切换、拖拽换位、反向语音）：[`multi-window-player.md`](references/multi-window-player.md)
- 播放器遮罩（加载、错误归类、暂停、结束、空态插图）：[`player-mask.md`](references/player-mask.md)
- SVG 图标（雪碧图接入、命名规则、尺寸/换色、新增图标、打包行为）：[`svg-icon.md`](references/svg-icon.md)
- 大文件分片上传（`vue-simple-uploader`、MD5、秒传/断点续传、全局面板、后端契约）：[`global-uploader.md`](references/global-uploader.md)

## 高频陷阱

1. **引擎未挂载**：控制台出现 `window.KMediaUni/window.KMediaMobile 未挂载` 或 `Aliplayer 未挂载`，播放器空白。先修脚本加载，再查组件。
2. **容器无高度**：宽度 100% 但高度为 0，表现为「什么都没渲染」。给容器 `aspect-ratio` 或固定高度。
3. **`src` 改了没反应**：组件不监听 `src`。`JyKmediaPlayer` / `JyKmediaMobilePlayer` 调用 `reloadVideo()`；`JyAliPlayer` 无 `reloadVideo`，先 `destroy()` 再 `loadVideo()`。
4. **`volumeChange` 单位不统一**：`JyKmediaPlayer` 首次（`loadedmetadata` 后）回调 0-100，后续音量变化回调 0-1。不要用回调值直接渲染百分比，统一用 `getVolume()` / `setVolume(0-100)`。
5. **`showMuteBtn` 默认是 `false`**（README 写的是 `true`，以源码 `withDefaults` 为准）。需要左上角静音按钮必须显式 `show-mute-btn`。
6. **插槽名以源码为准**：静音按钮插槽是 `mute-btn`，不是 README 里的 `muteBtn`；控制栏插槽是 `control`，必须把插槽 props `v-bind` 给 `JyPlayerControl`，否则播放/进度/全屏全部失效。
7. **`MODE` 不能运行时导入**：`MODE` 是 `declare enum`，包产物只导出 `FILL_TYPE`、`VIDEO_RESOLUTION`。设置 `transport` 时用 `window.KMediaUni.MODE.RTSP` 或数字（`WEBRTC = 1`、`RTSP = 2`）。
8. **多屏漏传 `emitter` / `playerNum`**：不传 `emitter` 时控制栏只操作本实例；传了 `emitter` 但不传 `playerNum`，控制栏的静音判断会错（`playerNum` 用于「全部静音才算静音」）。
9. **`isPrimary` 必须唯一**：双屏只能有一个 `:is-primary="true"`，否则音量互斥与播放同步角色混乱。
10. **多播放器热键冲突**：`JyPlayerControl` 默认绑定全局快捷键。非主屏或非激活屏传 `:support-hotkey="false"`，避免空格同时控制多个播放器。
11. **刷新页面自动播放策略**：`pausedOnRefreshPage`（默认 `true`）决定刷新后暂停或静音播放；直播场景会先静音拉首帧再暂停。需要刷新后直接续播时显式设为 `false`。
12. **不要依赖 `kmediaIns`**：它是 setup 阶段的快照（`null`），不会随后续初始化更新；需要底层能力时用已暴露方法，或向上游提需求。
13. **`MultiWindowPlayer` 的隐藏窗口仍在拉流**：分屏切换只改 CSS 布局，已挂载的播放器（最多 4 路）不会卸载；切到单屏后这些路仍持续拉流/解码。
14. **`MultiWindowPlayer` 的 `height` 不是播放器高度**：只限制分屏弹层 `max-height`，播放器高度由父容器决定。
15. **`viewSrcList` 变化要重挂载**：组件内 `innerPlayerSrcMap` 只增不减，切换教室或视角集合时用 `:key` / `v-if`，否则残留旧窗口。
16. **`MultiWindowPlayer` 语音呼叫从屏索引有偏移**：`subPlayerRefs[index]` 应为 `[index - 1]`，接入前核对版本是否修复。
17. **遮罩依赖父级定位与尺寸**：`PlayerErrorMask` / `PlayerPauseMask` / `PlayerEndMask` 是绝对定位，父级需 `position: relative` 且有高度；`PlayerLoadingMask` 是流内 100%，兄弟节点用法容易高度为 0。
18. **已知错误码下 `errorMessage` 不生效**：`PlayerErrorMask` 对 network / device / browser 三类使用系统配置文案，只有 `other` 或无错误码时才展示 `error-message`。
19. **`SvgIcon` 图标空白**：先查入口是否 `import 'virtual:svg-icons-register'`，再查 `name` 与雪碧图 `symbol id` 是否一致（含子目录前缀与大小写）。
20. **`SvgIcon` 默认 100×100**：`iconStyle` 默认内联宽高 100px；传 `:icon-style` 会整体替换默认值（要自己写宽高），用 class 改尺寸必须带 `!important`。
21. **业务封装依赖自动导入**：`MultiWindowPlayer` / `PlayerMask` 源码里有未显式 import 的 `computed`、`ElMessage`、`SvgIcon`。要么按 [`project-bootstrap.md`](references/project-bootstrap.md) 配 `unplugin-auto-import` + `unplugin-vue-components`，要么在复制件里补显式 import。
22. **`<script setup name="...">` 需要额外插件**：`SvgIcon` / `MultiWindowPlayer` 用了该写法；没有 `unplugin-vue-setup-extend-plus` 时改成 `defineOptions({ name })`。
23. **`GlobalUploader` 的 `target` 默认是 localhost 占位**：每次 `OPEN_GLOBAL_UPLOADER` 必须传 `options.target`，否则上传打到 `http://localhost:3000/upload`。
24. **`GlobalUploader` 的 `params` / `options` 初始 props 不生效**：两个 watch 都不是 `immediate`；统一走 `mittBus.emit('OPEN_GLOBAL_UPLOADER', ...)`。
25. **上传成功事件不带数据**：`GLOBAL_UPLOADER_SUCCESS` 无负载，拿 `fileId` 必须用 `options.successCb(file, res)`。
26. **上传器 JWT 是启动时快照**：`headers` 在组件 setup 时读取 `Auth.getJwtToken()`；登录/刷新 token 后要重新加载页面，或把 headers 改成函数（需改源码）。

## 文件落位

- 页面内播放器封装：L2，`views/<模块>/components/<PlayerName>/index.vue`（或页面内 `components/`）。
- 多页复用的播放器容器：L3，`src/components/<PlayerName>/index.vue`。
- 多画面播放器、遮罩、图标已属 L3：`src/components/MultiWindowPlayer/`、`src/components/PlayerMask/`、`src/components/SvgIcon/`。新页面直接导入复用，不要复制第二份；业务差异用 props / 插槽表达，确需扩展时改原组件并回归既有调用方。
- 图标源文件：`src/assets/icons/`（子目录会带目录前缀），不要放 `public/`，也不要新增图标字体。
- 全局上传器已属 L3：`src/components/GlobalUploader/`，只在 `App.vue` 挂一次；业务参数通过 `params`、成功后通过 `successCb` 关联业务数据。
- 多屏 emitter 与同步逻辑：页面内 `composables/usePlayerSync.ts`；跨页复用再提升到 L3 `shared/`。
- 引擎脚本：不打包的放 `public/` 或 `index.html` 外链；需要随构建产物发布的按项目静态资源规范处理。
- 播放器相关类型：跟随组件目录；接口出入参类型回到 `api/models`。

## 边界：什么时候改库

- 先确认已升级到包含该能力的版本，并核对 props / slots / exposes；很多「缺能力」其实是没有读源码 API。
- 确认能力确实缺失时，在 `jy-player-ui` 仓库实现（补 changeset、补单测、更新 README），业务侧不长期保留 `patch-package` 或 fork。
- 业务封装层（`MultiWindowPlayer` / `PlayerMask` / `GlobalUploader`）的缺陷优先在原工程修复并回归全部调用方，不要在各页面复制分叉版本。
- 业务侧临时绕过必须在代码注释中写明原因、上游 issue 与预计删除版本，并纳入技术债清单。

## 完成前检查

- [ ] 引擎全局变量在播放器挂载前已存在，且与所选组件匹配。
- [ ] 每个播放器 `domId` 唯一，父容器有明确尺寸。
- [ ] `src` 变更通过 `reloadVideo()` / 重新挂载生效，没有只改 prop。
- [ ] 控制栏通过 `#control` + `v-bind="controlProps"` 接入，未手写状态。
- [ ] 音量、进度、倍速单位与 API 文档一致（音量 0-100，进度为秒）。
- [ ] 多屏场景 `emitter`、`isPrimary`、`playerNum`、`volumeMutex` 组合正确。
- [ ] 1–4 画面场景复用 `MultiWindowPlayer`，`viewSrcList` 变化有 `:key`，并发路数在预算内。
- [ ] 加载 / 错误 / 暂停 / 结束遮罩使用 `PlayerMask`，父级已定位且有高度；已知错误码的文案走系统配置。
- [ ] 图标使用 `SvgIcon`，入口已注册雪碧图，`name` 与 `symbol id` 对齐，尺寸/颜色已显式设置。
- [ ] `unplugin-auto-import` / `unplugin-vue-components` / `unplugin-vue-setup-extend-plus` / `vite-plugin-svg-icons` 已按工程接入文档配置。
- [ ] 上传场景复用 `GlobalUploader`，只在 `App.vue` 挂一次，`target` / `params` / `successCb` 均已传。
- [ ] 后端分片接口实现 GET 校验与 POST 分片，响应结构与 `global-uploader.md` 第 7 节一致。
- [ ] `pnpm build && pnpm preview` 通过，产物中雪碧图、引擎脚本、播放器 chunk 均正常。
- [ ] 错误与 loading 有明确 UI，不依赖 kmedia 原生控制栏。
- [ ] 卸载时组件已销毁，页面切换没有残留实例或全局快捷键。

## 组件实现后格式化检查

- 组件实现完成后，检查项目中是否已有 `Prettier`、`ESLint` 和 `Stylelint` 配置。
- 如果存在，按项目现有规则格式化本次修改的文件，并修复格式化、lint 和 stylelint 相关错误。
