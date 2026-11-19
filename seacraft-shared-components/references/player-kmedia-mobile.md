# JyKmediaMobilePlayer（H5 / kmedia-mobile 引擎）

`JyKmediaMobilePlayer` 面向 H5 与移动端浏览器，封装 `window.KMediaMobile`。与 PC 版的关键差异：弱化音量能力、控制栏按横竖屏切换布局、默认不自动播放、支持右侧独立区域（`player-right`）。

引擎要求：`window.KMediaMobile` 必须存在。只加载 `kmedia-uni.js` 时本组件不可用（PC 播放器与它不共用引擎）。详见 [`player-overview.md`](player-overview.md)；工程级依赖、入口注册与打包见 [`project-bootstrap.md`](project-bootstrap.md)。

## 1. 完整骨架

```vue
<template>
	<div class="mobile-player-box">
		<JyKmediaMobilePlayer
			ref="playerRef"
			dom-id="mobile-1"
			:src="src"
			:autoplay="false"
			:muted="false"
			@created="onCreated"
			@loadedmetadata="onLoaded"
			@duration-change="duration = $event"
			@playing="playing = true"
			@paused="playing = false"
			@error="onError"
		>
			<template #hover-content>
				<div class="player-top-bar" @click.stop="playerRef?.exitFullscreen()">返回</div>
			</template>

			<template #loading>加载中…</template>

			<template #mask="{ KMEDIA_MOBILE_STATUS, play, errorMessage }">
				<div v-if="KMEDIA_MOBILE_STATUS === 'PAUSED'" class="player-mask" @click.stop="play">继续播放</div>
				<div v-else-if="KMEDIA_MOBILE_STATUS === 'ENDED'" class="player-mask" @click.stop="play">重播</div>
				<div v-else-if="KMEDIA_MOBILE_STATUS === 'ERROR'" class="player-mask">{{ errorMessage }}</div>
			</template>

			<template #control="controlProps">
				<JyMobilePlayerControl v-bind="controlProps" :is-live="isLive" :duration="duration" />
			</template>
		</JyKmediaMobilePlayer>
	</div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { JyKmediaMobilePlayer, JyMobilePlayerControl, type JyKmediaMobilePlayerInstance } from 'jy-player-ui';

const playerRef = ref<JyKmediaMobilePlayerInstance>();
const src = ref('https://example.com/lesson.mp4');
const isLive = ref(false);
const playing = ref(false);
const duration = ref(0);
const onCreated = () => console.info('created');
const onLoaded = () => console.info('loadedmetadata');
const onError = (err: { message: string; code: string }) => console.error(err.code, err.message);
</script>
```

## 2. Props

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `domId` | `number \| string` | 必填 | 实例标识，参与 DOM id `jkmp-<domId>` 与 `sessionStorage` key |
| `src` | `any` | 必填 | kmedia-mobile 播放源，常用 `string`、`ILive`、`{ url, type: 'flv' \| 'hls' }` |
| `autoplay` | `boolean` | **`false`** | 移动端浏览器默认拦截自动播放，不要默认打开 |
| `muted` | `boolean` | `false` | 仅首次加载生效 |
| `decode` | `'hard' \| 'soft'` | 引擎默认 | 仅 HTTP-FLV / HLS 有效 |
| `showKmediaLoading` | `boolean` | `false` | kmedia 原生 loading |
| `showKmediaMessage` | `boolean` | `false` | kmedia 原生消息 |
| `useKmediaControl` | `boolean` | `false` | 使用 kmedia 原生控制栏；为 `true` 时 `#control` 插槽不渲染 |
| `kmediaControlTools` | `string[] \| object[]` | 全部 `TOOLS` | 原生控制栏按钮 |
| `controlHideTime` | `number` | `3000` | 触摸后控制栏自动隐藏时间（ms） |
| `watermarkParams` | `IWatermarkParams` | — | 水印参数，`playing` 时下发 |
| `watermarkPreview` | `boolean` | — | 水印预览 |
| `emitter` | `Emitter<JyKmediaMobilePlayerSyncEvents>` | — | 多屏同步总线 |
| `isPrimary` | `boolean` | `false` | 多屏主屏 |
| `loadOnInit` | `boolean` | `true` | 初始化时加载视频；iOS 微信等需要用户手势后再加载时设为 `false` |
| `disabled` | `boolean` | `false` | 禁用 `emitter` 事件响应 |

## 3. Events

| 事件 | 参数 | 说明 |
| --- | --- | --- |
| `created` | — | kmedia-mobile 实例创建完成 |
| `loadedmetadata` | — | 元数据加载完成 |
| `currentTimeChange` | `time: number` | 播放时间变化 |
| `seeking` / `seeked` | `flag: boolean` | 跳转开始 / 结束 |
| `progress` | `Event` | 原生缓冲进度 |
| `waiting` | `Event` | 缓冲等待 |
| `durationChange` | `duration: number` | 时长变化 |
| `speedRateChange` | `speed: number` | 倍速变化 |
| `fullscreenChange` | `isFullscreen: boolean` | 全屏状态变化 |
| `mutedChange` | `isMuted: boolean` | 静音状态变化（`setMuted` 时触发） |
| `playing` / `paused` / `stop` / `ended` | — | 播放状态 |
| `warning` / `error` | `{ message: string; code: string }` | 引擎警告 / 错误 |
| `playerContainerEvent` | — | 声明了但源码未触发 |

注意：移动端播放器**没有** `volumeChange`、`audioFrequencyData`、`enterPiP` 等事件，不要照搬 PC 端代码。

## 4. Slots

| 插槽 | 插槽 props | 用途 |
| --- | --- | --- |
| `hover-content` | — | 顶部栏、返回按钮、标题（控制栏显示时展示） |
| `default` | — | 默认插槽 |
| `loading` | `{ loadType: 'loading' \| 'waiting' }` | 自定义加载 / 缓冲 UI |
| `mask` | `{ KMEDIA_MOBILE_STATUS, play, pause, errorMessage }` | 暂停 / 结束 / 错误蒙层，状态为 `UNCREATED` / `CREATED` / `LOAD_META_DATA` / `PLAYING` / `PAUSED` / `ENDED` / `STOP` / `ERROR` |
| `content` | 状态 + 方法（无 volume 相关） | 字幕、弹幕、贴片 |
| `control` | `visible` + `ControlProps` 子集 | 控制栏，`v-bind` 给 `JyMobilePlayerControl` |
| `player-right` | — | 播放器右侧独立区域（竖屏课程信息、章节列表等） |

`content` / `control` 插槽包含：`currentTime`、`isMuted`、`speedRate`、`duration`、`preload`（仅 control）、`playing`、`paused`、`isFullscreen`、`play`、`pause`、`stop`、`setMuted`、`setCurrentTime`、`getCurrentTime`、`setPlaybackRate`、`toggleFullscreen`。

移动端没有 `getVolume` / `setVolume` 透传到插槽，需要调音量用组件 expose。

## 5. Exposes

| 方法 | 参数 / 返回 | 说明 |
| --- | --- | --- |
| `loadVideo` | `() => void` | 用当前 props 加载 |
| `reloadVideo` | `() => void` | `nextTick` 后重新加载，换源后调用 |
| `play` / `pause` / `stop` | `() => void` | 建议在 `loadedmetadata` 之后 |
| `getVolume` | `() => number \| undefined` | 0-100 |
| `setVolume` | `(vol: number) => void` | 0-100，无 `showHint` |
| `getMuted` | `() => boolean \| undefined` | 是否静音 |
| `setMuted` | `(mute: boolean) => void` | 不接收音量参数 |
| `toggleMuted` | `() => void` | 切换静音 |
| `getCurrentTime` | `() => number \| undefined` | 秒 |
| `setCurrentTime` | `(second: number) => void` | 秒，超界落到 `duration - 0.1` |
| `getDuration` | `() => number \| undefined` | 秒 |
| `getResolution` | `() => VIDEO_RESOLUTION \| undefined` | 当前分辨率 |
| `setPlaybackRate` | `(rate, frames?) => number \| undefined` | 倍速 |
| `getPlaybackRate` | `() => number \| undefined` | 当前倍速 |
| `requestFullscreen` / `exitFullscreen` / `toggleFullscreen` | `() => void` | 支持时原生全屏，否则 CSS 假全屏（iOS） |
| `destroy` | `() => void` | 销毁实例 |
| `KMEDIA_MOBILE_STATUS` | `Ref<string>` | 当前状态 |
| `isFullscreen` | `Ref<boolean>` | 全屏状态 |
| `kmediaMobileIns` | 快照 | 同 PC 端，不要依赖 |

## 6. JyMobilePlayerControl

### Props

必填：`currentTime`、`isMuted`、`speedRate`、`duration`、`preload`、`playing`、`paused`、`isFullscreen` 与方法集合，均由播放器 `#control` 插槽透传。

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `isLive` | `boolean` | `false` | 直播模式：隐藏进度条，横屏底部只保留播放按钮与插槽 |
| `liveTime` | `number` | `0` | 已直播时长 |
| `disabledProgress` | `boolean` | `false` | 兼容旧 API，等同 `progressDisabled` |
| `progressDisabled` | `boolean` | `false` | 禁止拖动进度 |
| `controlBtns` | `('playTime' \| 'fullscreen')[]` | `['playTime', 'fullscreen']` | 显示哪些按钮 |
| `playerNum` | `number` | — | 多屏播放器数量 |
| `emitter` | `Emitter` | — | 多屏总线 |
| `hideAllBtn` | `boolean` | `false` | 隐藏所有按钮 |

### Events

| 事件 | 参数 | 说明 |
| --- | --- | --- |
| `mount` | — | 控制栏挂载 |
| `progress-change` | `(progress)` | 点击进度条 |
| `progress-drag-start` | `(currentTime)` | 拖拽开始 |
| `progress-drag-end` | `(currentTime)` | 拖拽结束 |
| `progress-cursor-move` | `(cursorProgress, duration, event)` | 进度条移动，用于缩略图 |
| `orientation-change` | `(isPortrait: boolean)` | 横竖屏变化 |

### Slots

| 插槽 | props | 位置 |
| --- | --- | --- |
| `progress` | `{ duration, currentTime, progress, cursorProgress }` | 进度条背景 |
| `progress-dot` | `{ duration, currentTime, setCurrentTime }` | 进度点 |
| `progress-hover` | `{ duration, currentTime, cursorProgress }` | 拖拽缩略图 |
| `portrait` | 全部 props + `handlePlaybackRateChange`、`handleMutedChange` | 竖屏顶部右侧 |
| `landscape` | 同上 | 横屏底部 |

### Exposes

没有 `defineExpose`，不要通过 ref 调用控制栏方法。

## 7. 移动端注意事项

1. **默认不自动播放**：`autoplay` 默认 `false`。确实需要自动播放时，同时 `:muted="true"` 才可能成功。
2. **首帧手势门禁**：iOS / 微信建议先渲染「点击播放」遮罩，用户点击后再挂载播放器或调用 `play()`。这在库的示例中就是用 `PlayerPauseMask` 做的。
3. **`loadOnInit: false`**：iOS 微信下若必须在用户手势后拉流，先设 `false`，手势回调里再调 `loadVideo()`。
4. **控制栏按方向切换**：`JyMobilePlayerControl` 通过 `matchMedia('(orientation: portrait)')` 判定；竖屏顶部显示播放按钮/时间/全屏，横屏底部显示播放按钮与 `landscape` 插槽。横屏时进度条位于顶部，不是底部。
5. **触摸显隐**：播放器容器 `touchstart` / `touchmove` 会重置隐藏定时器；控制栏内部触摸会阻止自动隐藏（`controlTouchFlag`）。自定义控制栏应放在 `#control` 插槽内，才能复用这套显隐状态与触摸保持逻辑。
6. **右侧区域**：`player-right` 插槽会把容器改为左右布局，左侧固定播放器区域。只在需要「播放器 + 课程信息」并排时使用，纯播放页不要用。
7. **横屏全屏**：优先调用 `requestFullscreen()`，组件内部已处理 iOS 的 CSS 假全屏；不要自己再监听 `orientationchange` 改样式。
8. **无音量滑块**：移动端控制栏不提供音量按钮，音量通过 `setVolume(0-100)` 或系统音量控制。
9. **进度条只在点播显示**：直播必须把控制栏 `:is-live="true"`，否则会出现无意义的进度条。
10. **多屏同步同样用 `mitt`**：见 [`player-multi-screen.md`](player-multi-screen.md)，但移动端同步事件里没有 `volumeChange`。

## 8. 常见组合

### 8.1 竖屏顶部返回栏

```vue
<template #hover-content>
	<div class="player-top-bar" @click.stop="onBack">返回</div>
</template>
```

### 8.2 竖屏右侧自定义按钮

```vue
<template #control="controlProps">
	<JyMobilePlayerControl v-bind="controlProps" :is-live="isLive">
		<template #portrait="{ handlePlaybackRateChange }">
			<button @click="handlePlaybackRateChange(2)">2X</button>
		</template>
	</JyMobilePlayerControl>
</template>
```

### 8.3 手势门禁（自动播放兜底）

```vue
<div v-if="!hasInteractive" class="tap-to-play" @click="hasInteractive = true">点击开始学习</div>
<JyKmediaMobilePlayer v-else ... />
```

`hasInteractive` 变为 `true` 后播放器才挂载，`loadedmetadata` 后执行 `play()`，避免移动端自动播放被静默拦截。
