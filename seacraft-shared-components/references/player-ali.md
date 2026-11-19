# JyAliPlayer（阿里播放器）

`JyAliPlayer` 封装阿里云 Web 播放器，适合标准 HLS / FLV / MP4 播放（课程回放、活动直播、第三方 CDN 流），不依赖 KMedia 协议。控制栏复用 `JyPlayerControl`。

引擎要求：`window.Aliplayer` 必须存在，同时引入阿里播放器样式。详见 [`player-overview.md`](player-overview.md)；工程级依赖、入口注册与打包见 [`project-bootstrap.md`](project-bootstrap.md)。

## 1. 完整骨架

```vue
<template>
	<div class="player-box">
		<JyAliPlayer
			ref="playerRef"
			dom-id="ali-1"
			:src="src"
			:is-live="isLive"
			:autoplay="true"
			:muted="false"
			show-mute-btn
			@created="onCreated"
			@loadedmetadata="onLoaded"
			@duration-change="duration = $event"
			@playing="playing = true"
			@paused="playing = false"
			@error="onError"
		>
			<template #loading>加载中…</template>

			<template #mask="{ ALI_PLAYER_STATUS, play, errorMessage }">
				<div v-if="ALI_PLAYER_STATUS === 'PAUSED'" class="player-mask" @click.stop="play">继续播放</div>
				<div v-else-if="ALI_PLAYER_STATUS === 'ENDED'" class="player-mask" @click.stop="play">重播</div>
				<div v-else-if="ALI_PLAYER_STATUS === 'ERROR'" class="player-mask">{{ errorMessage }}</div>
			</template>

			<template #control="controlProps">
				<JyPlayerControl v-bind="controlProps" :is-live="isLive" :duration="duration" />
			</template>
		</JyAliPlayer>
	</div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { JyAliPlayer, JyPlayerControl, type JyAliPlayerInstance } from 'jy-player-ui';

const playerRef = ref<JyAliPlayerInstance>();
const src = ref('https://example.com/lesson.m3u8');
const isLive = ref(false);
const duration = ref(0);
const playing = ref(false);
const onCreated = () => console.info('created');
const onLoaded = () => console.info('loadedmetadata');
const onError = (err: { message: string; code: string }) => console.error(err.code, err.message);
</script>
```

## 2. Props

| 属性 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `domId` | `number \| string` | 必填 | 实例标识，参与 DOM id `jap-<domId>` 与 `sessionStorage` key |
| `src` | `string \| Partial<ISource>` | 必填 | 单地址，或多清晰度对象 `{ OD, FD, LD, SD, HD, '2K', '4K' }` |
| `format` | `'mp4' \| 'flv' \| 'hls' \| 'm3u8' \| 'mp3' \| ''` | 自动检测 | **声明了但源码未使用**，不要依赖 |
| `autoplay` | `boolean` | `true` | 仅首次加载生效 |
| `autoplayPolicy` | `{ fallbackToMute: boolean; showUnmuteBtn: boolean }` | `{ fallbackToMute: false, showUnmuteBtn: false }` | **声明了但源码未使用**，自动播放降级由浏览器策略决定 |
| `muted` | `boolean` | `false` | 仅首次加载生效 |
| `isLive` | `boolean` | `false` | 直播语义，影响刷新策略 |
| `rePlay` | `boolean` | `false` | 结束后自动重播 |
| `playsinline` | `boolean` | `true` | H5 内联播放；部分 Android 浏览器不生效 |
| `controlHideTime` | `number` | `3000` | 控制栏隐藏时间 |
| `volumeMutex` | `boolean` | `true` | 多播放器音量互斥 |
| `pausedOnRefreshPage` | `boolean` | `true` | 刷新后暂停 / 静音播放 |
| `emitter` | `Emitter<JyAliPlayerSyncEvents>` | — | 多屏总线 |
| `isPrimary` | `boolean` | `false` | 多屏主屏 |
| `pauseOnClick` | `boolean` | `true` | 点击画面暂停/播放 |
| `fullscreenOnDblclick` | `boolean` | `true` | 双击全屏 |
| `showMuteBtn` | `boolean` | `false` | 左上角静音按钮 |
| `defaultVolume` | `number` | `75` | 默认音量 0-100 |

`ISource` 多清晰度写法：

```ts
const src = ref({
	HD: 'https://cdn.example.com/lesson_hd.m3u8',
	SD: 'https://cdn.example.com/lesson_sd.m3u8',
	LD: 'https://cdn.example.com/lesson_ld.m3u8'
});
```

键名固定为 `OD` / `FD` / `LD` / `SD` / `HD` / `2K` / `4K`，缺省清晰度不展示。

## 3. Events

| 事件 | 参数 | 说明 |
| --- | --- | --- |
| `created` | — | 播放器实例创建完成 |
| `loadedmetadata` | — | 元数据加载完成（注意：只有首次触发，重载后提前 return） |
| `currentTimeChange` | `time: number` | 播放时间变化 |
| `durationChange` | `duration: number` | 时长变化 |
| `volumeChange` | `vol: number` | 音量 0-100，单位一致 |
| `speedRateChange` | `speed: number` | 倍速变化 |
| `fullscreenChange` | `isFullscreen: boolean` | 全屏状态 |
| `playing` / `paused` / `stop` / `ended` | — | 播放状态 |
| `waiting` | `Event` | 缓冲等待 |
| `error` | `{ message: string; code: string }` | 播放错误 |
| `warning` | `{ message: string; code: string }` | 声明了但源码未触发 |
| `playerContainerEvent` | `Event` | 播放画面容器鼠标事件 |
| `content-click` | — | 内容区单击；模板监听写 `@content-click` |
| `refresh-page` | — | 刷新后的恢复流程；模板监听写 `@refresh-page` |
| `progress` | — | 声明了但源码未触发 |
| `muted-on-refresh` | — | 声明了但源码未触发 |
| `audioFrequencyData` | — | 声明了但源码未触发 |

## 4. Slots

| 插槽 | 插槽 props | 用途 |
| --- | --- | --- |
| `mute-btn` | `{ volume }` | 自定义静音按钮 |
| `hover-content` | — | hover 时内容 |
| `default` | — | 默认插槽 |
| `loading` | `{ loadType: 'loading' \| 'waiting' }` | 加载 / 缓冲 UI |
| `mask` | `{ ALI_PLAYER_STATUS, play, pause, errorMessage, errorCode }` | 蒙层，状态为 `UNCREATED` / `CREATED` / `LOAD_META_DATA` / `PLAYING` / `PAUSED` / `ENDED` / `STOP` / `ERROR` |
| `content` | `currentTime`、`volume`、`isMuted`、`speedRate`、`duration`、`playing`、`paused`、`isFullscreen`、`play`、`pause`、`setMuted`、`getVolume`、`setVolume`、`setCurrentTime`、`getCurrentTime`、`setPlaybackRate`、`displayVolumeHint`、`toggleFullscreen` | 字幕、弹幕、贴片 |
| `control` | `visible` + `ControlProps` 子集，`v-bind` 给 `JyPlayerControl` | 控制栏 |

## 5. Exposes

| 方法 | 参数 / 返回 | 说明 |
| --- | --- | --- |
| `loadVideo` | `(loading?: boolean) => void` | **有实例时只 dispose 并 return**，不会重新创建，见第 7 节 |
| `play` / `pause` / `stop` | `() => void` | 播放控制 |
| `getVolume` | `() => number \| undefined` | 0-100 |
| `setVolume` | `(vol: number, showHint?: boolean) => void` | 0-100 |
| `getMuted` | `() => boolean \| undefined` | 以音量为 0 判定 |
| `setMuted` | `(mute: boolean, volume?: number) => void` | 取消静音时恢复缓存音量或指定音量 |
| `toggleMuted` | `(volume?: number) => void` | 切换静音 |
| `getCurrentTime` | `() => number \| undefined` | 秒 |
| `setCurrentTime` | `(second: number) => void` | 秒，超过时长为 `Math.floor(duration)` |
| `getDuration` | `() => number \| undefined` | 秒 |
| `setPlaybackRate` | `(rate: number, showHint?: boolean) => void` | 倍速 |
| `getPlaybackRate` | `() => number \| undefined` | 当前倍速 |
| `requestFullscreen` / `exitFullscreen` | `() => void` | 全屏 |
| `displayVolumeHint` | `() => void` | 音量提示 |
| `destroy` | `() => void` | `dispose()` 并置空实例 |

没有 `reloadVideo`、`getResolution`、`loop`、画中画、截图、水印、`toggleControlVisible`。需要这些能力请用 `JyKmediaPlayer`。

## 6. 多屏同步

与 kmedia 播放器共用 `JyPlayerControl` 与 mitt 约定，但事件负载不同：

```ts
const emitter: Emitter<JyAliPlayerSyncEvents> = mitt();
```

`JyAliPlayerSyncEvents` 包含 `currentTimeChange`、`playing`、`paused`、`stop`、`mutedChange`、`playerMutedChange`、`controlReady`、`volumeChange`、`playBackRateChange`，与 kmedia 一致。不建议 kmedia 与 Ali 播放器共用同一个 emitter：事件负载接近，但两端底层能力和状态机不同（进度事件、`ENDED` 恢复、静音恢复策略），混用容易出现「只同步一半」的问题。

## 7. 换源与重载

`loadVideo` 的实现是：已有实例则 `dispose()` 后直接返回，不会重建。因此换源有两种可靠做法：

```ts
// 方案 A：destroy 后重新 loadVideo（需要 nextTick 等 props 刷新）
async function switchSource(next: string) {
	src.value = next;
	await nextTick();
	playerRef.value?.destroy();
	await nextTick();
	playerRef.value?.loadVideo();
}
```

```vue
<!-- 方案 B：用 key 强制重挂载，最稳 -->
<JyAliPlayer :key="src" ... />
```

不要只改 `src` 后调用 `loadVideo()`，结果是播放器被销毁且不再创建。

## 8. 注意事项

1. **必须引阿里播放器 CSS**：只引 JS 时控制栏/图标错位，自定义控制栏也可能被原生样式干扰。
2. **`format`、`autoplayPolicy` 不生效**：源码只声明未使用。格式由地址与 `src` 对象决定；自动播放失败请走「点击播放」兜底。
3. **没有原生进度事件**：需要缓冲进度用 kmedia 播放器，或自行轮询 `getCurrentTime()` 做近似。
4. **`loadedmetadata` 只触发一次**：`handleLoadedmetadata` 里 `if (!playerState.firstInit) return`，重载后不会再次触发，业务侧不要把它当「每次加载完成」。
5. **`isLive` 时刷新策略**：直播刷新后先静音拉流，`playing` 后暂停并恢复音量，同时触发 `refresh-page`。
6. **截图 / 水印 / 画中画 / 音波**：本组件不支持，需要这些能力选 `JyKmediaPlayer`。
7. **全屏没有 CSS 兜底**：`requestFullscreen()` 直接调用 `useFullscreen` 的 `enter()`，不支持 Fullscreen API 的设备（如部分 iOS）不会像 kmedia 播放器那样降级为假全屏，需要业务侧自行处理。
8. **多实例音画同步**：`volumeMutex` 的语义与 kmedia 相同，具体组合见 [`player-multi-screen.md`](player-multi-screen.md)。
