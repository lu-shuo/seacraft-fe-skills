# JyKmediaPlayer（PC / kmedia 引擎）

`JyKmediaPlayer` 是 PC 端播放器，封装 KMediaUni。控制栏与播放器解耦：不传 `#control` 插槽时只渲染画面和可选蒙层，需要自己接 `JyPlayerControl`。

引擎要求：`window.KMediaUni` 必须存在（缺失时回退 `window.KMediaMobile`，按移动端引擎处理）。详见 [`player-overview.md`](player-overview.md)；工程级依赖、入口注册与打包见 [`project-bootstrap.md`](project-bootstrap.md)。

## 1. 完整骨架

```vue
<template>
	<div class="player-box">
		<JyKmediaPlayer
			ref="playerRef"
			dom-id="lesson-1"
			:src="src"
			:autoplay="true"
			:muted="false"
			:paused-on-refresh-page="false"
			show-mute-btn
			:is-live="isLive"
			:default-volume="70"
			@created="onCreated"
			@loadedmetadata="onLoaded"
			@duration-change="duration = $event"
			@playing="playing = true"
			@paused="playing = false"
			@ended="playing = false"
			@error="onError"
		>
			<template #loading="{ loadType }">
				<div class="player-loading">{{ loadType === 'waiting' ? '缓冲中…' : '加载中…' }}</div>
			</template>

			<template #mask="{ KMEDIA_STATUS, play, errorMessage, errorCode }">
				<div v-if="KMEDIA_STATUS === 'PAUSED'" class="player-mask" @click.stop="play">继续播放</div>
				<div v-else-if="KMEDIA_STATUS === 'ENDED'" class="player-mask" @click.stop="play">重播</div>
				<div v-else-if="KMEDIA_STATUS === 'ERROR'" class="player-mask">
					{{ errorMessage }}<small v-if="errorCode">（{{ errorCode }}）</small>
				</div>
			</template>

			<template #control="controlProps">
				<JyPlayerControl v-bind="controlProps" :is-live="isLive" :duration="duration" />
			</template>
		</JyKmediaPlayer>
	</div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { JyKmediaPlayer, JyPlayerControl, type JyKmediaPlayerInstance } from 'jy-player-ui';

const playerRef = ref<JyKmediaPlayerInstance>();
const src = ref('https://example.com/lesson.mp4');
const isLive = ref(false);
const playing = ref(false);
const duration = ref(0);

const onCreated = () => console.info('kmedia created');
const onLoaded = () => console.info('loadedmetadata');
const onError = (err: { message: string; code: string }) => console.error(err.code, err.message);
</script>

<style scoped>
.player-box {
	position: relative;
	width: 100%;
	aspect-ratio: 16 / 9;
	background: #000;
}
.player-mask,
.player-loading {
	position: absolute;
	inset: 0;
	display: flex;
	align-items: center;
	justify-content: center;
	color: #fff;
	background: rgb(0 0 0 / 70%);
}
</style>
```

## 2. Props

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `domId` | `number \| string` | 必填 | 实例标识，参与 DOM id `jkp-<domId>` 与 `sessionStorage` key，同页唯一 |
| `src` | `ImodeSrc` | 必填 | 播放源，形态见总览第 6 节 |
| `autoplay` | `boolean` | `true` | 仅首次加载生效 |
| `muted` | `boolean` | `false` | 仅首次加载生效 |
| `pausedOnRefreshPage` | `boolean` | `true` | 刷新后是否暂停；`false` 为静音自动播放并提示开启声音 |
| `showMuteTipOnRefreshPage` | `boolean` | `true` | 刷新静音时是否展示「开启声音」提示 |
| `transport` | `MODE` | 自动探测 | 播放方案，`window.KMediaUni.MODE.WEBRTC = 1`、`RTSP = 2` |
| `decode` | `'hard' \| 'soft'` | 引擎默认 | 解码方式，仅 RTSP 有效 |
| `showMuteBtn` | `boolean` | **`false`** | 左上角静音按钮；需要时显式传 `true` |
| `showKmediaLoading` | `boolean` | `false` | kmedia 原生 loading |
| `showKmediaMessage` | `boolean` | `false` | kmedia 原生消息 |
| `useKmediaControl` | `boolean` | `false` | 使用 kmedia 原生控制栏；为 `true` 时 `#control` 插槽不渲染 |
| `kmediaControlTools` | `string[] \| object[]` | 全部 `TOOLS` | 原生控制栏按钮，仅 `useKmediaControl` 时有效 |
| `watermarkParams` | `IWatermarkParams` | — | 水印参数，`playing` 时下发 |
| `watermarkPreview` | `boolean` | — | 水印预览 |
| `controlHideTime` | `number` | `3000` | 鼠标静止后控制栏隐藏时间（ms） |
| `emitter` | `Emitter<JyKmediaPlayerSyncEvents>` | — | 多屏同步事件总线 |
| `isPrimary` | `boolean` | `false` | 多屏主屏，同组唯一 |
| `pauseOnClick` | `boolean` | `true` | 点击画面暂停/播放 |
| `fullscreenOnDblclick` | `boolean` | `true` | 双击全屏 |
| `fillType` | `FILL_TYPE` | `AUTO` | 填充方式，`FULLUP` / `HORIZONTAL` / `VERTICAL` / `RATIO_16_9` / `RATIO_4_3` / `AUTO` |
| `crossOrigin` | `boolean` | `true` | 视频跨域（`anonymous`） |
| `isLive` | `boolean` | `false` | 直播语义 |
| `volumeMutex` | `boolean` | `true` | 多播放器音量互斥（同时只有一个有声） |
| `defaultVolume` | `number` | `75` | 默认音量 0-100 |
| `restore` | `boolean` | `true` | 直播中断自动重连 |
| `decoderMaxProbeFrameNum` | `number` | `20` | 音频时间阈值 |
| `mseCacheNumber` | `number` | 引擎默认 `50` | Fmp4Cache 首次放行前缓存分片数 |
| `mseFragSegmentLength` | `number` | 引擎默认 `10` | MseRender 首次写入 SourceBuffer 前队列阈值 |
| `backgroundPlay` | `boolean` | `true` | 后台播放 |
| `disabled` | `boolean` | `false` | 禁用 `emitter` 事件响应（从屏冻结同步时用） |
| `mediaOapUrl` | `string` | — | 媒体上报地址 |
| `stageRatio` | `number` | `50` | stage 分屏主区比例，实际 clamp 到 30-70 |
| `stageVisible` | `boolean` | `false` | 是否展示 stage 分屏（需提供 `#stage` 插槽） |

## 3. Events

模板监听写 kebab-case（`@current-time-change`），脚本里声明是 camelCase。

| 事件 | 参数 | 触发时机 |
| --- | --- | --- |
| `created` | — | kmedia 实例创建完成 |
| `loadedmetadata` | — | 元数据加载完成，此时才能 `play` / `duration` |
| `currentTimeChange` | `time: number` | 播放时间变化 |
| `seeking` / `seeked` | `flag: boolean` | 跳转开始 / 结束 |
| `progress` | `Event` | 原生 `progress`，用于缓冲进度 |
| `waiting` | `Event` | 缓冲等待，3 秒内只触发一次 |
| `durationChange` | `duration: number` | 时长变化 |
| `volumeChange` | `vol: number` | 音量变化，**单位不统一**，见陷阱 |
| `speedRateChange` | `speed: number` | 倍速变化 |
| `fullscreenChange` | `isFullscreen: boolean` | 全屏状态变化 |
| `playing` / `paused` / `stop` / `ended` | — | 播放状态 |
| `contentClick` | — | 点击内容区（单击） |
| `controlVisibleChange` | `visible: boolean` | 控制栏显隐 |
| `containerMouseMove` | — | 容器内鼠标/触摸移动 |
| `playerContainerEvent` | `Event` | 播放画面容器的鼠标事件；全覆盖面板（PTZ、区域绘制等）内的鼠标事件不返回 |
| `warning` | `{ message: string; code: string }` | kmedia 警告 |
| `error` | `{ message: string; code: string }` | kmedia 错误，`code === '30207'` 表示语音呼叫错误 |
| `refreshPage` | — | 刷新页面后的恢复流程触发 |
| `audioFrequencyData` | `any`（频域数组） | 音频数据，需 `bindAudioFrequencyData()` 或默认绑定 |
| `enterPiP` / `exitPiP` | — | 进入 / 退出画中画 |
| `stageResizeStart` | `MouseEvent \| TouchEvent` | 拖动 stage 分隔条开始 |
| `mutedOnRefresh` | — | 声明了但源码未触发，不要依赖 |

## 4. Slots

| 插槽 | 插槽 props | 用途 |
| --- | --- | --- |
| `mute-btn` | `{ volume }` | 自定义左上角静音按钮。名字是 `mute-btn`，不是 README 的 `muteBtn` |
| `hover-content` | — | hover 时展示的额外内容（标题、返回按钮等） |
| `default` | — | 默认插槽，渲染在 `jkp-default-slot-wrap` |
| `loading` | `{ loadType: 'loading' \| 'waiting' }` | 自定义 loading / 缓冲 UI |
| `mask` | `{ KMEDIA_STATUS, play, pause, errorMessage, errorCode }` | 暂停 / 结束 / 错误蒙层，`KMEDIA_STATUS` 为 `UNCREATED` / `CREATED` / `LOAD_META_DATA` / `PLAYING` / `PAUSED` / `ENDED` / `STOP` / `ERROR` |
| `content` | 见下 | 字幕、弹幕、贴片等覆盖层 |
| `control` | 见下 | 控制栏，通常直接 `v-bind` 给 `JyPlayerControl` |
| `stage` | 见下 | 右侧副画面区域（双屏对比 / 讲稿 / 资料区） |

`content` 插槽 props：`currentTime`、`volume`、`isMuted`、`speedRate`、`duration`、`playing`、`paused`、`isFullscreen`、`play`、`pause`、`stop`、`setMuted`、`getVolume`、`setVolume`、`setCurrentTime`、`getCurrentTime`、`setPlaybackRate`、`displayVolumeHint`、`toggleFullscreen`。

`control` 插槽 props：`visible` + 上表全部状态与方法 + `preload`、`showTurnOnVoiceTip`、`setTurnOnVoiceTipStatus`、`defaultVolume`、`isMobileEngine`，与 `ControlProps` 对齐，直接 `v-bind="controlProps"`。

`stage` 插槽 props 与 `content` 一致，另加 `visible`、`preload`；分屏比例用 `stageRatio`（30-70），宽度通过 CSS 变量 `--jkp-stage-primary-ratio` / `--jkp-stage-secondary-ratio` 下发。

## 5. Exposes

通过模板 ref 调用。音量、进度单位都已注明，不要按 kmedia 原生单位猜。

| 方法 | 参数 / 返回 | 说明 |
| --- | --- | --- |
| `loadVideo` | `(loading?: boolean) => void` | 用当前 props 重新加载（内部会初始化） |
| `reloadVideo` | `(loading?: boolean) => void` | `stop()` 后 `nextTick` 重新加载，**换源后必须调用** |
| `play` / `pause` / `stop` | `() => void` | 仅 `loadedmetadata` 之后调用；`stop` 不触发 `ended` |
| `getVolume` | `() => number \| undefined` | 0-100 |
| `setVolume` | `(vol: number, showHint?: boolean) => void` | 0-100；`showHint` 展示音量提示 |
| `getMuted` | `() => boolean \| undefined` | 是否静音 |
| `setMuted` | `(mute: boolean, volume?: number) => void` | 取消静音时可同时指定音量 0-100 |
| `toggleMuted` | `(volume?: number) => void` | 切换静音并同步多屏状态 |
| `getCurrentTime` | `() => number \| undefined` | 秒 |
| `setCurrentTime` | `(second: number) => void` | 秒；超过时长会落到 `duration - 0.1`，`ENDED` 时忽略 |
| `getDuration` | `() => number \| undefined` | 秒，不支持实时流 |
| `getResolution` | `() => VIDEO_RESOLUTION \| undefined` | 实时 / 历史视频可用 |
| `changeResolution` | `(resolution: VIDEO_RESOLUTION) => void` | 切换分辨率 |
| `setPlaybackRate` | `(rate: number, showHint?: boolean, frames?: 'intra' \| 'predicted') => number \| undefined` | 设置倍速 |
| `getPlaybackRate` | `() => number \| undefined` | 当前倍速 |
| `loop` | `(loop?: boolean) => boolean \| undefined` | 不传参为读取 |
| `requestFullscreen` / `exitFullscreen` | `() => void` | 原生全屏或 CSS 假全屏 |
| `displayVolumeHint` | `() => void` | 展示音量提示 2 秒 |
| `destroy` | `() => void` | 销毁 kmedia 实例；组件卸载时也会自动调用 |
| `startVoiceCall` / `stopVoiceCall` | `(params?: IStartVoiceCallParams)` / `() => void` | 反向语音 |
| `imgCapture` | `(params?: ISnapshotParams) => Promise<string> \| false \| undefined` | 截图，返回 base64 / URL |
| `exitImgCapture` | `() => void` | 退出截图态 |
| `imgControl` | `(type: 'BRIGHTNESS' \| 'SATURATE' \| 'CONTRAST' \| 'HUEROTATE', val: number) => void` | 图像增强 |
| `getImgControlState` | `() => { type, value }[]` | 当前图像增强参数 |
| `setVideoFillType` | `(fillType: FILL_TYPE) => void` | 仅 `loadedmetadata` 后调用 |
| `toggleControlVisible` | `(visible: boolean) => void` | 控制栏显隐（`1.2.0`） |
| `addGraphics` | `(info: IGraphics) => void` | 区域绘制（`1.5.0`） |
| `bindAudioFrequencyData` / `unBindAudioFrequencyData` | `() => void` | 手动绑定/解绑音频数据事件 |
| `setTurnOnVoiceTipStatus` | `(show: boolean) => void` | 控制「开启声音」提示 |
| `enterPiP` / `exitPiP` | `() => void` | 画中画，需浏览器支持且有 `<video>` |
| `KMEDIA_STATUS` | `Ref<string>` | 当前状态（模板 ref 上已解包） |
| `isFullscreen` | `Ref<boolean>` | 全屏状态 |
| `isPiP` | `Ref<boolean>` | 画中画状态 |
| `showTurnOnVoiceTip` | `Ref<boolean>` | 提示是否展示 |
| `kmediaIns` | `null` | setup 阶段快照，**不可用**，不要依赖 |

## 6. JyPlayerControl

### Props

必填：`currentTime`、`volume`（0-100）、`isMuted`、`playing`、`paused`、`isFullscreen`。使用 `#control` 插槽时这些都由播放器透传，直接用 `v-bind`。

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `speedRate` | `number` | `1` | 当前倍速 |
| `duration` | `number` | `0` | 总时长（秒） |
| `preload` | `number` | `0` | 缓冲比例 0-1 |
| `play` / `pause` / `stop` | `() => void` | — | 未传 `emitter` 时直接调用 |
| `getVolume` / `setVolume` | `() => number` / `(vol, showHint?) => void` | — | 音量 0-100 |
| `setMuted` | `(isMuted: boolean, volume?: number) => void` | — | 静音 |
| `setCurrentTime` / `getCurrentTime` | `(time: number) => void` / `() => number` | — | 秒 |
| `setPlaybackRate` | `(rate, showHint?, frames?) => number` | — | 倍速 |
| `toggleFullscreen` | `() => void` | — | 全屏切换 |
| `displayVolumeHint` | `() => void` | — | 音量提示 |
| `supportHotkey` | `boolean` | `true` | 是否绑定全局快捷键 |
| `skipDuration` | `number` | `5` | 快进退步长（秒） |
| `isLive` | `boolean` | `false` | 直播模式：隐藏进度条、禁用左右快进 |
| `liveTime` | `number` | `0` | 已直播时长，叠加在 `currentTime` 上显示 |
| `controlBtns` | `('stop' \| 'playTime' \| 'volume' \| 'fullscreen')[]` | `['playTime', 'volume', 'fullscreen']` | 显示哪些按钮 |
| `showProgress` | `boolean` | `true` | 点播是否显示进度条 |
| `progressDisabled` | `boolean` | `false` | 禁止拖动进度 |
| `progressSize` | `number \| string` | `10` | 进度条高度 |
| `progressMarkMode` | `boolean` | `false` | 打点模式，点击不改变进度 |
| `playerNum` | `number` | — | 多屏时播放器数量，用于「全部静音才算静音」 |
| `emitter` | `Emitter` | — | 多屏总线，有值时按钮改为发事件 |
| `isPad` | `boolean` | `false` | Pad 判定，业务侧传入 |
| `isMobileEngine` | `boolean` | `false` | 是否移动端引擎（影响音量条形态） |
| `hideAllBtn` | `boolean` | `false` | 隐藏所有按钮 |
| `showTurnOnVoiceTip` | `boolean` | — | 展示开启声音提示 |
| `defaultVolume` | `number` | — | 开启声音时的默认音量 |
| `setTurnOnVoiceTipStatus` | `(show: boolean) => void` | — | 更新提示状态 |

### Events

| 事件 | 参数 | 说明 |
| --- | --- | --- |
| `mount` | — | 控制栏挂载完成 |
| `progress-change-start` | `(progress, currentTime)` | 点击进度条开始 |
| `progress-change-end` | `(progress, currentTime)` | 点击进度条结束（nextTick） |
| `progress-drag-start` | `(currentTime, progress)` | 拖拽开始，**注意第一个参数是 currentTime** |
| `progress-drag-end` | `(currentTime)` | 拖拽结束 |
| `progress-skip-start` | `(currentTime, skipDuration)` | 快捷键/按钮快进开始 |
| `progress-skip-end` | `(currentTime, skipDuration)` | 快进结束 |
| `progress-cursor-move` | `(progress, duration, event)` | 进度条 hover/触摸移动，用于缩略图 |
| `muted-change` | `(isMuted: boolean)` | 静音切换 |

### Slots

| 插槽 | props | 用途 |
| --- | --- | --- |
| `progress` | `{ duration, currentTime, progress, cursorProgress }` | 进度条背景层（高亮、章节） |
| `progress-dot` | `{ duration, currentTime, setCurrentTime }` | 进度点 / 打点标记 |
| `progress-hover` | `{ duration, currentTime, cursorProgress }` | hover 缩略图 |
| `control-left` | 全部 props + `handlePlaybackRateChange` | 左侧自定义按钮 |
| `control-right` | 全部 props + `innerMuted`、`handlePlaybackRateChange` | 右侧自定义按钮 |
| `control-end` | 全部 props + `handlePlaybackRateChange` | 最右侧插槽（`1.3.0`） |

### Exposes

| 方法 | 说明 |
| --- | --- |
| `openHotkey` | 绑定快捷键（默认 `onMounted` 已绑定） |
| `offHotkey` | 解绑全部 `hotkeys-js` 快捷键 |

默认快捷键：`space` 播放/暂停、`f` 全屏、`esc` 退出全屏、`↑` / `↓` 音量 ±10、`←` / `→` 快退/快进 `skipDuration`，长按 `→` 临时 3 倍速。

## 7. 常用配方

### 7.1 倍速 + 画中画（右侧插槽）

```vue
<template #control="controlProps">
	<JyPlayerControl v-bind="controlProps" :is-live="isLive">
		<template #control-right="{ speedRate, handlePlaybackRateChange }">
			<div class="rate-menu">
				<button v-for="rate in [2, 1.5, 1.25, 1, 0.75, 0.5]" :key="rate" @click="handlePlaybackRateChange(rate)">
					{{ rate }}X
				</button>
			</div>
			<button @click="togglePiP">{{ playerRef?.isPiP ? '退出画中画' : '画中画' }}</button>
		</template>
	</JyPlayerControl>
</template>
```

```ts
function togglePiP() {
	playerRef.value?.isPiP ? playerRef.value.exitPiP() : playerRef.value.enterPiP();
}
```

### 7.2 外部按钮控制

```ts
const seekTo = (second: number) => playerRef.value?.setCurrentTime(second);
const mute = () => playerRef.value?.toggleMuted();
const snapshot = async () => {
	const url = await playerRef.value?.imgCapture({ download: true, name: 'snapshot' });
	// url 可能是 base64，按需转 Blob 下载
};
```

### 7.3 换源

```ts
async function switchLesson(next: string) {
	src.value = next;
	await nextTick(); // 等 props 刷新
	playerRef.value?.reloadVideo();
}
```

### 7.4 打点 / 片段标记

```vue
<template #control="controlProps">
	<JyPlayerControl v-bind="controlProps" :progress-mark-mode="markMode" @progress-drag-start="onDragStart">
		<template #progress-dot="{ duration }">
			<span
				v-for="point in marks"
				:key="point"
				class="mark-dot"
				:style="{ left: `${(point * 100) / duration}%` }"
			/>
		</template>
	</JyPlayerControl>
</template>
```

- `progressMarkMode` 为 `true` 时点击进度条不改变播放进度，适合打点。
- `progress-dot` 的 `left` 用百分比；`duration` 为 0 时不要计算。
- 拖拽标记需要自己监听 `mousedown` + `document.mousemove`，库不提供拖拽标记能力。

### 7.5 音频波形

```ts
onMounted(() => {
	playerRef.value?.bindAudioFrequencyData();
});
// 事件
const onAudio = (data: number[]) => drawWave(data);
```

移动端引擎（`isMobileEngine`）不触发 `audioFrequencyData`。

### 7.6 控制栏常显 / 隐藏

```ts
playerRef.value?.toggleControlVisible(true); // 常显（仍需鼠标移动来维持）
```

配合 `#control="{ visible }"` 可让自定义按钮跟随控制栏显隐（`v-show="visible"`）。

## 8. 陷阱与源码差异

1. **音量单位不统一**：`volumeChange` 在 `loadedmetadata` 后的首次上报是 0-100，之后音量变化上报 0-1。UI 里不要直接用它算百分比，统一 `getVolume()` / `setVolume()`（0-100）。
2. **`showMuteBtn` 默认 `false`**：README 表格写 `true`，以源码 `withDefaults` 为准。
3. **插槽名 `mute-btn`**：README 写 `muteBtn`，实际模板是 `<slot name="mute-btn">`。
4. **`progress-drag-start` 参数顺序**：实际是 `(currentTime, progress)`，README 写的是 `(progress, currentTime)`。
5. **`MODE` 运行时不存在**：`transport` 用 `window.KMediaUni.MODE` 或数字。
6. **`src` 不响应**：必须 `reloadVideo()`；`reloadVideo` 内部先 `stop()` 再 `nextTick` 加载。
7. **`play` / `pause` / `getDuration` 需在 `loadedmetadata` 之后**：过早调用只会打印「KMediaUni 实例不存在」或返回空。
8. **`stop` 不触发 `ended`**：需要在结束时展示重播，用 `ended` 事件而不是 `stop`。
9. **`setCurrentTime` 边界**：超过 `duration` 落到 `duration - 0.1`；`ENDED` 状态下调用被忽略，重播需先 `play()`。
10. **快捷键是全局的**：同页多个 `JyPlayerControl` 都会绑定 `hotkeys-js`，非主屏传 `:support-hotkey="false"`，卸载时调 `offHotkey()` 或依赖组件 `onUnmounted` 的 `hotkeys.unbind()`。
11. **`kmediaIns` 不可用**：暴露的是 `null` 快照；需要底层实例能力时用封装方法或提上游需求。
12. **画中画只支持单屏**：同一时刻只能有一个播放器进入 PiP，切换前先退出另一个（见 [`player-multi-screen.md`](player-multi-screen.md)）。
13. **`disabled` 只禁用 emitter 响应**：它不阻止用户操作，也不停止播放，只用于冻结从屏同步。

## 9. 独立使用 JyControlProgress 与 JySlider

两个底层组件也可以脱离播放器单独使用（自定义播放器、音频波形、任意滑杆）。

### JyControlProgress

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `currentTime` | `number` | 必填 | 当前时间（秒） |
| `duration` | `number` | 必填 | 总时长（秒） |
| `preload` | `number` | 必填 | 缓冲比例 0-1 |
| `size` | `number \| string` | `'10px'` | 进度条高度 |
| `disabled` | `boolean` | `false` | 禁止拖动 |
| `isPad` | `boolean` | `false` | Pad 判定（影响 hover 提示方式） |

事件：`progress-change(progress)`、`cursor-move(cursorProgress, event)`、`drag-start(progress, currentTime)`、`drag-end(currentTime)`。

插槽：`default`（`{ progress, cursorProgress }`）、`progress-dot`（无 props）、`hover-content`（`{ cursorProgress }`）。

`defineOptions name` 是 `ProgressBar`，全局注册后模板名也是 `ProgressBar`；推荐局部导入。

### JySlider

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `modelValue` | `number` | `0` | 当前值，范围 0-1 |
| `preloadValue` | `number` | `0` | 预加载值，0-1 |
| `size` | `number \| string` | `'10px'` | 轨道高度 / 宽度 |
| `vertical` | `boolean` | `false` | 垂直模式（音量条） |
| `disabled` | `boolean` | `false` | 禁用 |
| `showHoverTip` | `boolean` | `true` | 是否显示 hover 时间提示 |
| `hoverText` | `string` | `''` | hover 提示文本 |
| `btnHoverText` | `string` | — | 拖拽按钮提示 |
| `isPad` | `boolean` | `false` | Pad 判定 |

事件：`update:modelValue(value)`、`change(value)`（均 0-1）、`mousemove(value, event)`、`touchmove(value, event)`、`dragStart(value)`、`dragEnd()`。

插槽：`slider-dot`（无 props）、`hover-content`。

```vue
<JySlider
	:model-value="volume / 100"
	size="5px"
	vertical
	@change="val => (volume = Math.round(val * 100))"
/>
```

音量条建议配 `vertical`，进度条建议配 `show-hover-tip` + `hoverText`。
