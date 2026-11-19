# 播放器接入总览

本文覆盖 `jy-player-ui` 的安装、引擎脚本、注册方式、样式定制、数据源写法与故障排查。组件级 API 见 `player-kmedia.md`、`player-kmedia-mobile.md`、`player-ali.md`。工程级依赖、初始化与打包配置见 [`project-bootstrap.md`](project-bootstrap.md)。

版本基线：`jy-player-ui@1.11.0`。升级前先读仓库 `packages/jy-player-ui/CHANGELOG.md`，接入相关变更集中在：`1.2.0` 控制栏显隐方法、`1.3.0` 控制栏最右插槽、`1.7.0` stage 插槽、`1.9.0` 引擎切换、`1.10.0` 移动端控制栏、`1.11.0` 移动端竖屏插槽。

## 1. 包信息

| 项 | 值 |
| --- | --- |
| 包名 | `jy-player-ui` |
| 发布源 | 公司 Nexus 私有 npm |
| 入口 | `main: lib/index.js`、`module: es/index.mjs`、`typings: lib/index.d.ts` |
| 产物 | ESM + CJS，`preserveModules`，支持 tree-shaking |
| 运行时依赖 | `@jy-player-ui/utils`、`@vueuse/core`、`hammerjs`、`hotkeys-js` |
| Vue | 不随包提供，由业务项目提供（当前依赖树解析为 Vue 3.5，低版本项目需自行回归） |
| 样式 | 组件内 `import './style/index.css'`，构建工具会自动收集；`sideEffects` 仅声明 `**/*.css` |

安装：

```powershell
npm config set registry https://nexus.kedacom.com/repository/public-npm/
npm i jy-player-ui
```

```bash
# .npmrc 方式（推荐写进项目，避免污染全局）
# @jy-player-ui:registry=https://nexus.kedacom.com/repository/public-npm/
# registry=https://nexus.kedacom.com/repository/public-npm/
pnpm add jy-player-ui
```

`@jy-player-ui/utils` 是包的运行时依赖，会从同一私有源解析；公司内网环境不要混用公网 registry。

## 2. 引擎脚本（接入失败的第一原因）

三个播放器都不内置播放引擎，只调用全局构造函数。引擎脚本必须在播放器 `onMounted` 之前执行完毕。

| 组件 | 使用的全局变量 | 脚本 | 说明 |
| --- | --- | --- | --- |
| `JyKmediaPlayer` | `window.KMediaUni`，缺失时回退 `window.KMediaMobile` | `kmedia-uni.js` | 回退仅在 `KMediaUni` 不存在时发生，且会按移动端引擎处理 |
| `JyKmediaMobilePlayer` | `window.KMediaMobile` | `kmedia-mobile.js` | 只认 `KMediaMobile` |
| `JyAliPlayer` | `window.Aliplayer` | `aliplayer-min.js` + `aliplayer-min.css` | 需要同时引样式 |

`index.html` 参考写法（顺序在应用入口之前）：

```html
<!-- PC / kmedia 引擎 -->
<script src="/vendor/kmedia-uni.js"></script>
<!-- H5 / kmedia-mobile 引擎，仅移动端页面需要 -->
<script src="/vendor/kmedia-mobile.js"></script>

<!-- 阿里播放器，仅使用 JyAliPlayer 时需要 -->
<link rel="stylesheet" href="https://g.alicdn.com/apsara-media-box/imp-web-player/2.26.1/skins/default/aliplayer-min.css" />
<script src="https://g.alicdn.com/apsara-media-box/imp-web-player/2.26.1/aliplayer-min.js"></script>
```

异步加载或按需加载时，用全局变量做挂载门禁，不要让播放器先挂载：

```ts
const engineReady = ref(false);

function loadEngine() {
	const script = document.createElement('script');
	script.src = '/vendor/kmedia-uni.js';
	script.onload = () => (engineReady.value = true);
	script.onerror = () => ElMessage.error('播放器引擎加载失败');
	document.head.appendChild(script);
}
```

```vue
<JyKmediaPlayer v-if="engineReady" ... />
<div v-else class="player-placeholder">播放器加载中…</div>
```

引擎缺失时的控制台特征：

- `[JyKmediaPlayerError]-<domId>: window.KMediaUni/window.KMediaMobile 未挂载`
- `[JyKmediaMobilePlayerError]: window.KMediaMobile 未挂载`
- `[JyAliPlayerError]-<domId>: Aliplayer 未挂载`

## 3. 注册与类型

全局注册（适合页面大量使用、不关心包体积）：

```ts
import { createApp } from 'vue';
import JyPlayerUI from 'jy-player-ui';

const app = createApp(App);
app.use(JyPlayerUI);
```

按需引入（推荐，类型完整、tree-shaking 友好）：

```ts
import {
	JyKmediaPlayer,
	JyKmediaMobilePlayer,
	JyAliPlayer,
	JyPlayerControl,
	JyMobilePlayerControl,
	JyControlProgress,
	JySlider,
	FILL_TYPE,
	VIDEO_RESOLUTION
} from 'jy-player-ui';

import type {
	JyKmediaPlayerInstance,
	JyKmediaPlayerProps,
	JyKmediaPlayerSyncEvents,
	JyKmediaMobilePlayerInstance,
	JyKmediaMobilePlayerProps,
	JyAliPlayerInstance,
	JyAliPlayerProps,
	JyPlayerControlInstance,
	ControlProps,
	MobileControlProps,
	MODE
} from 'jy-player-ui';
```

注意：

- `MODE` 只是类型声明（`declare enum`），**运行时不存在**，不能 `import { MODE }` 后使用。设置 `transport` 用 `window.KMediaUni.MODE.RTSP` 或数字 `1` / `2`。
- 全局注册的组件名依次是 `JyKmediaPlayer`、`JyKmediaMobilePlayer`、`JyAliPlayer`、`JyPlayerControl`、`JyMobilePlayerControl`、`JySlider`；`JyControlProgress` 的 `defineOptions name` 是 `ProgressBar`，全局注册后模板里应写 `<ProgressBar>`。为避免这个歧义，统一局部导入 `JyControlProgress` 使用。
- 业务项目不要从 `@jy-player-ui/components` 导入，那是库内 workspace 包，发布产物里不存在。

## 4. 样式与主题

- 组件样式随 ESM 产物自动引入，业务侧**不需要**手动 `import 'jy-player-ui/style.css'`（包内没有该入口）。
- 主色通过 CSS 变量覆盖，默认 `#31c27c`：

```vue
<style scoped>
.player-box :deep(.jy-kmedia-player) {
	--jy-player-primary-color: var(--el-color-primary);
}
</style>
```

三个播放器都要各自设置该变量：`.jy-kmedia-player`、`.jy-kmedia-mobile-player`、`.jy-ali-player`。

常用内部类前缀（仅用于样式微调，不用于业务逻辑）：

| 播放器 | 根类名 | 内部前缀 |
| --- | --- | --- |
| `JyKmediaPlayer` | `.jy-kmedia-player` | `.jkp-*` |
| `JyKmediaMobilePlayer` | `.jy-kmedia-mobile-player` | `.jkmp-*` |
| `JyAliPlayer` | `.jy-ali-player` | `.jap-*` |
| `JyPlayerControl` | `.jy-player-control-wrap` | `.control-*` |
| `JyControlProgress` | `.jy-control-progress` | `.jy-slider` |
| `JyMobilePlayerControl` | `.jy-mobile-player-control-wrap` | `.control-wrap__*` |

插槽内容默认不带定位和层级，蒙层 / 字幕 / 弹幕需要自己写 `position: absolute; inset: 0;` 与 `z-index`。

## 5. 容器与布局

播放器根节点是 `width/height: 100%` 的块，父容器不给尺寸就不可见。

```scss
.player-box {
	position: relative; // 蒙层、字幕定位基准
	width: 100%;
	aspect-ratio: 16 / 9; // 或显式 height
	background: #000; // 首帧前不露白
}
```

- 双屏左右/上下布局由外层 flex/grid 决定，两个播放器容器各自保持比例。
- 全屏在支持 `requestFullscreen` 的设备走原生全屏；不支持时（如 iOS）组件用 `position: fixed` 实现假全屏，此时父级 `overflow: hidden` 可能影响表现，接入移动端时注意。
- 不要在播放器容器上使用 `transform` 做缩放，kmedia 的坐标/全屏计算会受影响。

## 6. 数据源写法

`JyKmediaPlayer` 的 `src` 类型是 `ImodeSrc`，按协议选择形态：

| 场景 | `src` 形态 | 关键字段 |
| --- | --- | --- |
| 普通文件 / MP4 / HLS 地址 | `string` | 完整 URL |
| 多段文件合并播放 | `{ url: string; duration?: number }[]` | 顺序即播放顺序，`duration` 用于时间轴 |
| 实时视频（WebRTC） | `ILive` | `websocketUrl`、`devId`、`accountToken?`、`stunService?`、`turnService?` |
| 历史录像（RTSP / WebRTC 回放） | `IPlayBack` | `ILive` + `startTime`、`endTime`、`supportReverse?` |
| RTSP 直连（RTW） | `IRTW` | `websocketUrl`、`rtspUrl`、`username?`、`password?` |
| HTTP-FLV / HLS 对象 | `{ url: string; type: 'flv' \| 'hls'; useWebAudio?: boolean }` | `type` 必填 |
| 本地流 | `MediaStream` | 摄像头 / 屏幕共享流 |

`JyKmediaMobilePlayer` 的 `src` 类型是 `any`，形态以 kmedia-mobile 官方文档为准，常用同上（`string`、`ILive`、`{ url, type }`）。

`JyAliPlayer` 的 `src` 是 `string` 或清晰度对象：`{ OD, FD, LD, SD, HD, '2K', '4K' }`（传 `Partial` 即可，缺省清晰度不展示）。

传输方式与解码：

```ts
// 直播 RTSP：必须显式指定 transport
playerState.transport = window.KMediaUni.MODE.RTSP;
playerState.decode = 'hard'; // 仅 RTSP 下有效，可选 'soft'
```

不传 `transport` 时组件按媒体能力探测选择播放方案，一般不要手动指定。

## 7. 刷新与自动播放策略

浏览器自动播放策略会拦截有声自动播放，组件内置了降级逻辑：

| props | 默认 | 行为 |
| --- | --- | --- |
| `autoplay` | `true` | 首次加载生效；`reloadVideo` 后固定按 `true` 处理 |
| `muted` | `false` | 仅首次加载生效；后续重载按 `volumeMutex` 与 `sessionStorage` 恢复静音状态 |
| `pausedOnRefreshPage` | `true` | 刷新后：点播暂停并触发 `refreshPage`；直播先静音拉首帧再暂停、恢复音量 |
| `showMuteTipOnRefreshPage` | `true` | 刷新后静音播放时展示「请点击开启声音」提示 |
| `isLive` | `false` | 直播语义，影响刷新暂停策略、进度条显示与重连 |

相关 `sessionStorage` key 以 `domId` 区分（`JY_KMEDIA_PLAYER_*` / `JY_ALI_PLAYER_*`），因此**同一浏览器标签页内 `domId` 必须唯一**。这也是多播放器场景必须传 `emitter` 的原因之一。

## 8. 故障排查

| 现象 | 常见原因 | 处理 |
| --- | --- | --- |
| 播放器区域全黑且无控制栏 | 引擎脚本未加载 / 加载晚于挂载 | 检查全局变量，加挂载门禁 |
| 区域高度为 0，页面看不到播放器 | 父容器无高度 | 给 `aspect-ratio` 或固定高度 |
| 有画面但无法播放/暂停 | 没有接 `#control` 插槽，或未 `v-bind="controlProps"` | 按骨架接入 `JyPlayerControl` |
| 有画面但没有控制栏 | `useKmediaControl` 默认 `false`，且未提供 `#control` 插槽 | 接入自定义控制栏，或显式开启原生控制栏 |
| 首次有声音、刷新后静音 | 浏览器策略 + `pausedOnRefreshPage` / `volumeMutex` | 按第 7 节调整，或提供「开启声音」入口 |
| 切源后仍播旧视频 | 只改了 `src` | 调 `reloadVideo()`；阿里播放器先 `destroy()` 再 `loadVideo()` |
| 两个播放器音量互相干扰 | `domId` 重复，或未传 `emitter` | 唯一 `domId`；多屏场景统一传 `emitter` |
| 控制台 `kmedia 错误 code/message` | 流地址、鉴权、跨域或引擎能力问题 | 监听 `error` / `warning`，把 `code` 反馈给流媒体/后端 |
| 移动端无法全屏 | 浏览器不支持 Fullscreen API | 组件已降级为 CSS 假全屏；如仍异常检查父级 `overflow` |
| Vue 警告 Unknown custom element | 只装了一个组件却使用了另一个 | 检查局部导入或全局注册 |

## 9. 交付清单

- [ ] 私有源、包版本、lockfile 已提交。
- [ ] 引擎脚本位置与页面类型匹配（PC / H5 / 阿里），并有加载失败提示。
- [ ] 播放器容器有尺寸和背景色，蒙层有定位。
- [ ] 控制栏、loading、错误态、结束态四类 UI 均已处理。
- [ ] 刷新策略与产品预期一致（暂停 / 静音续播 / 提示开启声音）。
- [ ] 多屏场景 `domId`、`emitter`、`isPrimary`、`playerNum`、`volumeMutex` 已核对。
- [ ] 页面卸载或路由切换后实例销毁，无残留快捷键与音频。
