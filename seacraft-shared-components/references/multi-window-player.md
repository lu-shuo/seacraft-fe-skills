# MultiWindowPlayer（业务内嵌多画面播放器）

> 位置：`jy-digital-patrol-ui/src/components/MultiWindowPlayer/index.vue`
> 性质：**业务仓封装，不随 `jy-player-ui` 发布**。它把 `JyKmediaPlayer` + `JyPlayerControl` + `mitt` + `PlayerMask` + 项目 hooks 组合成 1–4 画面播放器，其他项目按需复制并替换依赖。
> 基线：`jy-digital-patrol-ui` 当前依赖 `jy-player-ui@^1.7.8`。`JyKmediaPlayer` 自身 API 见 [`player-kmedia.md`](player-kmedia.md)。

## 1. 什么时候用它

适用：

- 一个容器内展示 1–4 路 kmedia 画面，共用一套播放/进度/音量/倍速/全屏控制。
- 教室详情、在线巡课、预警详情/处理、消息中心回放等「多视角同看」页面。
- 需要「分屏布局切换 + 视角切换 + 拖拽换位 + 反向语音」这类巡课场景能力。

不适用：

- 每路需要独立控制栏或独立时间轴（用多个 `JyKmediaPlayer` 自行组合，见 [`player-multi-screen.md`](player-multi-screen.md)）。
- 需要画中画、截图、分辨率切换、stage 单实例分屏（直接用 `JyKmediaPlayer`）。
- 只想播放单个视频（用 `JyKmediaPlayer` + `PlayerMask` 更轻）。

## 2. 能力清单

| 能力 | 实现方式 |
| --- | --- |
| 单屏 / 双屏 / 三屏 / 四屏 | 根节点 `count-N` class + 每个窗口绝对定位，`windowCount` 控制 |
| 视角切换 | 控制栏「分屏」弹层内按窗口选择 `viewSrcList` 中任一路 |
| 拖拽换位 | HTML5 `draggable` + `playerPositionMap` 交换位置（不交换数据源） |
| 统一控制栏 | 单个 `JyPlayerControl`，通过 `emitter` 驱动全部播放器 |
| 全屏 | `useFullscreen` 作用于整个多画面容器，控制栏按钮调用该 toggle |
| 反向语音 | 暴露 `startVoiceCall(viewNum)` / `stopVoiceCall(viewNum)`，配合 `PtzControl` |
| 刷新恢复 | 读取 `PLAYER_PAUSED_ON_REFRESH_PAGE`，展示「播放」或「请重新打开声音」提示 |
| 水印 | 未传 props 时使用 `useWatermark()` 的全局配置 |
| 加载 / 错误 | 内部使用 `PlayerLoadingMask`、`PlayerErrorMask`，不向外暴露遮罩插槽 |
| 扩展插槽 | `control-right`、`progress-dot`、`content`（字幕等） |

## 3. 前置条件

1. **KMedia 引擎**：`index.html` 必须已加载 `/lib/kmedia/kmedia-uni.js`，且 `window.KMediaUni` 存在。
2. **注册播放器**：入口 `app.use(JyPlayerUI)` 全局注册 `JyKmediaPlayer` / `JyPlayerControl`。组件模板直接使用这两个全局组件，脚本里只导入了类型；复制到其他工程时若没有全局注册，需要自行补局部导入。
3. **全局组件**：`SvgIcon` 需全局可用，且存在图标 `muted`、`unMuted`、`play`、`split-screen`、`screen-1` ~ `screen-4`、`popper-close`。
4. **Element Plus**：使用 `el-popover`、`ElMessage`。
5. **项目 hooks / 配置**：`useInlineCssVars`、`useRecordPlayerState`、`useWatermark`、`@/utils/global` 的 `getCustomConfig()`。
6. **父容器尺寸**：根节点是 `width/height: 100%`，父级必须给高度或宽高比，否则整块不可见。
7. **主题色**：组件把 `--jy-player-primary-color` 固定为 `var(--el-color-primary)`，并依赖 `--room-detail-player-*` 主题变量（由 `useInlineCssVars` 注入）。

### 3.1 依赖与初始化（最短路径）

```bash
pnpm add element-plus jy-player-ui @vueuse/core mitt
pnpm add -D vite-plugin-svg-icons unplugin-auto-import unplugin-vue-components unplugin-vue-setup-extend-plus
```

```ts
// vite.config.ts
import vue from '@vitejs/plugin-vue';
import vueSetupExtend from 'unplugin-vue-setup-extend-plus/vite';
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons';
import { resolve } from 'node:path';

plugins: [
	vue(),
	vueSetupExtend({}),
	createSvgIconsPlugin({ iconDirs: [resolve(process.cwd(), 'src/assets/icons')], symbolId: 'icon-[dir]-[name]' }),
	AutoImport({ imports: ['vue'], resolvers: [ElementPlusResolver()], dts: 'src/auto-import.d.ts' }),
	Components({ extensions: ['vue'], resolvers: [ElementPlusResolver({ importStyle: 'sass' })], dts: 'src/components.d.ts' })
]
```

```ts
// main.ts
import 'virtual:svg-icons-register';
import JyPlayerUI from 'jy-player-ui';
app.use(JyPlayerUI);
```

- 没有 `unplugin-auto-import` 时，组件里的 `computed`、`onBeforeUnmount`、`onDeactivated`、`ElMessage` 会未定义；需要在复制件里补显式 import。
- 没有 `unplugin-vue-components` 时，`SvgIcon` 与 `el-popover` 需要手动注册。
- 完整依赖版本、`index.html` 引擎脚本、构建验证见 [`project-bootstrap.md`](project-bootstrap.md)。

## 4. Props

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `viewSrcList` | `{ viewName: string; viewNum: number; src: any }[]` | 必填 | 可切换的视角集合；实际最多挂载 4 路（见第 10 节） |
| `viewVoiceCallStatus` | `Map<number, 'CALLING' \| 'CALL_ON' \| 'CALL_OFF'>` | — | 反向语音状态，仅用于错误处理时判断是否需要挂断 |
| `watermarkParams` | `IWatermarkParams` | `useWatermark()` 的值 | 水印参数，未传时回落到全局配置 |
| `watermarkPreview` | `boolean` | `useWatermark()` 的值 | 是否展示水印 |
| `isLive` | `boolean` | `true` | 直播/点播语义，影响控制栏进度与刷新策略 |
| `height` | `number` | — | **只用于限制分屏弹层高度**（`height - 50` px），不控制播放器高度 |
| `transport` | `MODE` | — | 传入每个 `JyKmediaPlayer`；RTSP 用 `window.KMediaUni.MODE.RTSP` |
| `errorMessage` | `string` | — | 内部 `PlayerErrorMask` 的自定义错误文案（仅无错误码或 other 类生效，见 [`player-mask.md`](player-mask.md)） |
| `defaultUnmutedViewNum` | `number` | 第一个视角（索引 0） | 首次加载时保持有声的 `viewNum`，其余窗口静音 |

`height` 的常见误用：传 `:height="378"` 以为能控制播放器高度。实际播放器高度由外层容器决定，`height` 只是分屏弹层的 `max-height`。

## 5. Events

组件 `defineEmits` 里声明了很多事件，但**源码实际只触发两个**：

| 事件 | 参数 | 说明 |
| --- | --- | --- |
| `fullscreenChange` | `(isFullscreen: boolean)` | 整个多画面容器进入/退出全屏 |
| `voice-call-status-change` | `(viewNum: number, status: 'CALLING' \| 'CALL_ON' \| 'CALL_OFF')` | 反向语音状态变化 |

声明但未触发的列表：`created`、`loadedmetadata`、`currentTimeChange`、`playerContainerEvent`、`ended`、`playing`、`paused`、`stop`、`progress`、`waiting`、`durationChange`、`volumeChange`、`speedRateChange`、`warning`、`error`、`content-click`。**不要监听这些事件**；需要播放状态时自行扩展组件（把内部播放器事件向上抛出）。

## 6. Slots

| 插槽 | 插槽 props | 位置 / 用途 |
| --- | --- | --- |
| `control-right` | — | 控制栏右侧，排在「分屏」按钮与倍速菜单之前；用于字幕设置等业务按钮 |
| `progress-dot` | `{ duration: number; setCurrentTime: (t: number) => void }` | 进度条打点，透传给 `JyPlayerControl` 的 `progress-dot` |
| `content` | `{ isFullscreen: boolean }` | 根节点内容层，用于字幕、弹幕、贴片；需要自行定位与 `z-index` |

组件没有默认插槽、没有 `loading` / `mask` 插槽；加载与错误由内部遮罩处理。

## 7. Exposes

通过模板 ref 调用：

| 方法 | 参数 | 说明 |
| --- | --- | --- |
| `startVoiceCall` | `(viewNum: number) => Promise<void>` | 对指定视角发起反向语音 |
| `stopVoiceCall` | `(viewNum: number) => Promise<void>` | 挂断指定视角 |
| `setMultiCurrentTime` | `(second: number) => void` | 向 emitter 发 `currentTimeChange`，全部窗口跳到该秒 |

已知问题：`startVoiceCall` / `stopVoiceCall` 内部用 `viewSrcList` 的下标直接访问 `subPlayerRefs[index]`，而从屏 ref 数组是 `subPlayerRefs[index - 1]`；视角切换或拖拽后下标还会与实际窗口错位。稳妥做法是按当前 `innerPlayerSrcMap` 反查播放器索引再取 `subPlayerRefs[index - 1]`，并把修复提回上游；接入前先核对所用版本是否已修复。

## 8. 最小用法

```vue
<template>
	<div class="video-wrap">
		<PlayerLoadingMask v-if="fetchVideoLoading" />

		<MultiWindowPlayer
			v-else-if="courseViewSrcList.length"
			:key="courseId"
			ref="playerRef"
			:height="378"
			:view-src-list="courseViewSrcList"
			:is-live="false"
			:transport="transport"
			:default-unmuted-view-num="defaultAudioChannel"
			error-message="原始录像不可用，可能已被覆盖或删除"
			@voice-call-status-change="handleVoiceCallStatusChange"
		>
			<template #progress-dot="{ duration, setCurrentTime }">
				<SvgIcon
					v-for="dot in progressDots"
					:key="dot.time"
					name="progress-alarm"
					icon-class="alarm-icon"
					:style="{ left: `${(dot.time * 100) / duration}%` }"
					@click="setCurrentTime(dot.time)"
				/>
			</template>
		</MultiWindowPlayer>

		<PlayerErrorMask v-else error-message="原始录像不可用，可能已被覆盖或删除" />
	</div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';
import MultiWindowPlayer from '@/components/MultiWindowPlayer/index.vue';
import PlayerLoadingMask from '@/components/PlayerMask/PlayerLoadingMask.vue';
import PlayerErrorMask from '@/components/PlayerMask/PlayerErrorMask.vue';
import Global from '@/utils/global';

const playerRef = ref<InstanceType<typeof MultiWindowPlayer> | null>();
const fetchVideoLoading = ref(false);
const courseViewSrcList = ref<{ viewName: string; viewNum: number; src: any }[]>([]);
const courseId = ref<number>();
const defaultAudioChannel = ref<number>();

const transport = computed(() =>
	Global.getCustomConfig().RECORD_PROTOCOL === 'rtsp' ? window.KMediaUni.MODE.RTSP : undefined
);

const voiceCallStatus = ref<Map<number, 'CALLING' | 'CALL_ON' | 'CALL_OFF'>>(new Map());
function handleVoiceCallStatusChange(viewNum: number, status: 'CALLING' | 'CALL_ON' | 'CALL_OFF') {
	voiceCallStatus.value.set(viewNum, status);
}

const progressDots = ref<{ time: number }[]>([]);
</script>

<style scoped>
.video-wrap {
	position: relative;
	width: 100%;
	aspect-ratio: 16 / 9;
	background: #000;
}
</style>
```

加载/错误状态由父级 `v-if` 控制很重要：组件内部根节点是 `v-if="innerPlayerSrcMap.size"`，`viewSrcList` 为空时它什么都不渲染，也不会给出空态。

## 9. `viewSrcList` 数据准备

```ts
interface ViewSrcItem {
	viewName: string; // 分屏弹层里的视角按钮文案
	viewNum: number; // 业务视角编号，唯一；也是 defaultUnmutedViewNum / 语音呼叫的入参
	src: any; // 直接透传给 JyKmediaPlayer 的 src
}
```

`src` 按协议构造（与 [`player-overview.md`](player-overview.md) 第 6 节一致）：

```ts
// 直播 WebRTC
{ websocketUrl, devId, accountToken, stunService }

// 录像回放（WebRTC / RTSP）
{ websocketUrl, devId, accountToken, startTime: 'YYYY-MM-DD HH:mm:ss', endTime: '...' }

// HTTP-FLV / HLS
{ url: 'https://.../live.flv', type: 'flv' }
{ url: 'https://.../live.m3u8', type: 'hls' }
```

约定：

- `src` 里可以附带业务标记字段（如 `viewNum`、`chanNameMain`），播放器会忽略。
- `viewNum` 必须稳定且唯一；`defaultUnmutedViewNum` 匹配的是它，不是数组下标。
- `viewSrcList` 超过 4 条时，分屏弹层最多列 4 个布局按钮、只挂载前 4 路；其余视角仍可通过「视角切换」替换进已挂载窗口。

## 10. 分屏、拖拽与切换行为

- 初始 `windowCount = min(viewSrcList.length, 4)`，之后可在弹层切 1/2/3/4 屏。
- 布局是**纯 CSS**：`count-1` 全屏；`count-2` 主画面铺满 + 右下角 1/3 小窗；`count-3` 左 2/3 + 右侧上下两个 1/3；`count-4` 2×2。
- **隐藏的窗口不会卸载**：`v-for` 始终渲染 `playerCount`（≤4）个 `JyKmediaPlayer`，`windowCount` 只改 class。切到单屏后，其余已挂载的路（最多 3 路）仍在拉流/解码，注意并发与性能。
- 拖拽只交换 `playerPositionMap`（位置 ↔ 播放器索引），不换 `src`、不重新拉流。
- 视角切换 `changePlayerView` 会更新 `innerPlayerSrcMap` 并调用对应播放器 `reloadVideo()`。点播场景下源码尝试在窗口重新加载后补发一次 `currentTimeChange` 对齐进度，但 `loadState` 中已有记录且 `loaded` 仍为 `true` 时 watcher 不会触发；切换视角后建议业务侧显式调用 `setMultiCurrentTime(currentTime)` 对齐。
- `innerPlayerSrcMap` **只增不减**：`viewSrcList` 变短时旧条目仍留在 Map 里，`playerCount` 不会缩小。切换教室/视角集合时请用 `:key` 或 `v-if` 重新挂载组件。
- 控制栏要等**主画面（索引 0）`loadedmetadata`** 后才出现；主画面错误或未加载时没有控制栏属正常现象。

## 11. 控制栏、音量与全屏

- 控制栏是组件自绘的 `.jkp-control-wrap`，3 秒无鼠标移动后通过 `data-control-visible="false"` 下移隐藏；hover 控制栏时保持显示。
- 控制栏固定 `:support-hot-key="false"`，多画面场景不绑定空格/方向键。
- `volumeMutex` 使用 `JyKmediaPlayer` 默认值 `true`：同一时刻只有一个窗口有声；首次由 `defaultUnmutedViewNum` 决定。
- 控制栏的「全屏」调用组件自己的 `toggleFullscreen`，全屏的是**整个多画面容器**。
- `JyKmediaPlayer` 的 `fullscreen-on-dblclick` 为 `true`，双击某个画面会全屏**该单个播放器**，与控制栏的全屏语义不同；产品若要求统一，显式关闭双击全屏或改造该行为。
- 刷新页面后的提示：
  - `PLAYER_PAUSED_ON_REFRESH_PAGE` 为 `true`（默认）：显示居中的「播放」提示，点击 `emitter.emit('playing')`。
  - 为 `false`：显示「请重新打开声音」，点击 `emitter.emit('mutedChange', false)`，提示 60 秒后自动消失。

## 12. 反向语音

典型接线（来自 `roomDetail`）：

```vue
<PtzControl
	:view-src-list="ptzEnableViewSrcList"
	:view-voice-call-status="viewVoiceCallStatusMap"
	@start-voice-call="viewNum => playerRef?.startVoiceCall(viewNum)"
	@stop-voice-call="viewNum => playerRef?.stopVoiceCall(viewNum)"
/>
```

- `viewVoiceCallStatusMap` 建议在 `viewSrcList` 变化时重建：`new Map(list.map(i => [i.viewNum, 'CALL_OFF']))`。
- 父级收到 `voice-call-status-change` 后更新 Map，`PtzControl` 以 Map 状态渲染按钮。
- 播放器错误码 `30207` 表示语音呼叫被占用，组件会尝试挂断并 `ElMessage.error('当前设备被占用')`。
- 组件卸载或 `keep-alive` 失活时会遍历 `viewSrcList` 逐个挂断。

## 13. 已知限制与坑

1. **`height` 不是播放器高度**：只影响分屏弹层 `max-height`。播放器高度由父容器决定。
2. **隐藏窗口仍占流**：`viewSrcList` ≥ 4 时最多同时 4 路在拉流，单屏布局也一样。页面上同时存在多个 `MultiWindowPlayer` 时并发会叠加。
3. **视角集合变化必须重挂载**：`innerPlayerSrcMap` 只增不减，直接改 `viewSrcList` 可能残留旧窗口，用 `:key` 或 `v-if`。
4. **语音呼叫从屏索引偏移**：`subPlayerRefs[index]` 应为 `[index - 1]`，且视角切换后下标不再对应窗口；详见第 7 节，接入前验证或修复。
5. **控制栏依赖主画面加载**：主画面 `ERROR` 后控制栏消失，只剩错误遮罩。
6. **事件不透传**：只暴露 `fullscreenChange`、`voice-call-status-change`；播放/错误等事件需自行扩展组件。
7. **全屏语义不统一**：控制栏全屏容器，双击全屏单画面。
8. **`playerNum` 用源数量**：控制栏传的是 `innerPlayerSrcMap.size`，视角超过 4 条时与已挂载窗口数不一致，静音图标可能永远不显示「全部静音」。
9. **强依赖工程环境**：`@/` 别名、`SvgIcon`、Element Plus、项目 hooks/stores、`global.getCustomConfig()`。复制到其他项目需要替换这些依赖。
10. **主题变量**：依赖 `--el-color-primary` 与 `--room-detail-player-*`；脱离该主题体系时需补默认值，否则弹层背景/边框透明。
11. **空数据无 UI**：`viewSrcList` 为空时不渲染任何内容，父级必须提供 loading / empty / error 三态。

## 14. 复用到其他项目的检查清单

- [ ] 替换 `@/` 别名与组件导入路径。
- [ ] 提供 `SvgIcon` 及所需图标，或替换为项目图标组件。
- [ ] 提供 Element Plus（`el-popover`、`ElMessage`）或替换分屏弹层。
- [ ] 替换 `useInlineCssVars`（主题变量）与 `useWatermark`（水印）为本地实现。
- [ ] 替换 `global.getCustomConfig().PLAYER_PAUSED_ON_REFRESH_PAGE` 为项目配置来源。
- [ ] 保留 `PlayerMask` 或替换为本地遮罩，注意父级定位。
- [ ] 确认 `jy-player-ui` 版本支持所用 props（`watermarkParams`、`decoderMaxProbeFrameNum`、`startVoiceCall` 等）。
- [ ] 确认最多同时挂载 4 路流的并发预算。
- [ ] 为 `viewSrcList` 变化补 `:key`，或在组件内修正 Map 的清理逻辑。
- [ ] 回归：分屏切换、拖拽换位、视角切换、点播进度同步、刷新提示、全屏、语音呼叫。
