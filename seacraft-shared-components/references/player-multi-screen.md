# 多屏同步、双屏对比与画中画

适用：主讲 + 课件双画面、教师 + 学生双视角、主备流切换、同页多个播放器需要联动。

核心机制：所有播放器与控制栏共享**同一个 `mitt` 实例**。播放器根据 `isPrimary` 与 `volumeMutex` 决定自己是发布者还是订阅者，控制栏在有 `emitter` 时把按钮操作改为发事件。

## 1. 事件契约

```ts
import mitt, { type Emitter } from 'mitt';
import type { JyKmediaPlayerSyncEvents } from 'jy-player-ui';

const emitter: Emitter<JyKmediaPlayerSyncEvents> = mitt();
```

| 事件 | 负载 | 发布方 | 订阅行为 |
| --- | --- | --- | --- |
| `playing` | `domId`（控制栏触发时为空） | 播放器 / 控制栏 | 非自身时 `play()`；从屏在 `ENDED` 状态不恢复 |
| `paused` | `domId` | 播放器 / 控制栏 | 非自身时 `pause()` |
| `stop` | — | 播放器 / 控制栏 | 全部 `stop()` |
| `currentTimeChange` | `time: number` | 控制栏进度操作 | 全部 `setCurrentTime(time)` |
| `playBackRateChange` | `{ rate, showHint? }` | 控制栏倍速 | 全部 `setPlaybackRate()` |
| `volumeChange` | `{ vol: number; showHint: boolean }` | 控制栏音量 | 按 `volumeMutex` 规则应用到有声窗口 |
| `mutedChange` | `isMuted: boolean` | 控制栏静音 | 按缓存与非静音窗口决定是否静音 |
| `playerMutedChange` | `{ isMute, domId }` | 播放器 | `volumeMutex` 下保证同时只有一个窗口有声 |
| `controlReady` | — | 控制栏挂载 | 播放器回报当前静音状态 |

要点：

- **进度与倍速是双向同步**：任一侧拖动进度或改倍速，另一侧跟随。没有「只跟随主屏」的开关，需要单向控制时用 `disabled` 或拆成两个 emitter 组。
- **播放/暂停也是双向**：`playing` 处理器只排除自身 `domId`，主屏与从屏都互相跟随。
- **音量**：`volumeMutex: true` 为互斥（同时只有一个有声），`false` 允许多个有声（独立声音）。
- **`disabled`**：只屏蔽该实例对 emitter 事件的响应，不阻止用户操作。切布局、换源期间可临时冻结从屏。

## 2. 双屏骨架

```vue
<template>
	<div class="double-screen">
		<div class="pane primary">
			<JyKmediaPlayer
				ref="primaryRef"
				dom-id="primary"
				:src="primarySrc"
				:emitter="emitter"
				:is-primary="true"
				:volume-mutex="false"
				:is-live="isLive"
				:paused-on-refresh-page="false"
				show-mute-btn
			>
				<template #mask="{ KMEDIA_STATUS, play }">
					<div v-if="KMEDIA_STATUS === 'PAUSED'" class="mask" @click.stop="play">继续播放</div>
					<div v-else-if="KMEDIA_STATUS === 'ENDED'" class="mask" @click.stop="play">重播</div>
				</template>
				<template #control="controlProps">
					<JyPlayerControl
						v-bind="controlProps"
						:emitter="emitter"
						:player-num="playerCount"
						:is-live="isLive"
					/>
				</template>
			</JyKmediaPlayer>
		</div>

		<div v-if="doubleScreen" class="pane secondary">
			<JyKmediaPlayer
				ref="secondaryRef"
				dom-id="secondary"
				:src="secondarySrc"
				:emitter="emitter"
				:is-primary="false"
				:volume-mutex="false"
				:muted="true"
				:is-live="isLive"
				:paused-on-refresh-page="false"
				show-mute-btn
			>
				<template #mask="{ KMEDIA_STATUS, play }">
					<div v-if="KMEDIA_STATUS === 'PAUSED'" class="mask" @click.stop="play">继续播放</div>
				</template>
				<template #control="controlProps">
					<JyPlayerControl
						v-bind="controlProps"
						:emitter="emitter"
						:player-num="playerCount"
						:is-live="isLive"
						:support-hotkey="false"
					/>
				</template>
			</JyKmediaPlayer>
		</div>
	</div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';
import mitt, { type Emitter } from 'mitt';
import { JyKmediaPlayer, JyPlayerControl, type JyKmediaPlayerInstance, type JyKmediaPlayerSyncEvents } from 'jy-player-ui';

// 不要用 reactive()/ref() 包 emitter，mitt 实例按原对象透传
const emitter: Emitter<JyKmediaPlayerSyncEvents> = mitt();

const primaryRef = ref<JyKmediaPlayerInstance>();
const secondaryRef = ref<JyKmediaPlayerInstance>();
const primarySrc = ref('https://example.com/teacher.m3u8');
const secondarySrc = ref('https://example.com/slides.m3u8');
const isLive = ref(false);
const doubleScreen = ref(true);

// 必须与实际参与同步的播放器数量一致
const playerCount = computed(() => (doubleScreen.value ? 2 : 1));
</script>
```

## 3. 音量模式

| 配置 | 效果 | 适用 |
| --- | --- | --- |
| `volumeMutex: true`（默认） | 同一时刻只有一个窗口有声，取消静音会静音其他窗口 | 主讲 + 课件，用户只关心一个声音 |
| `volumeMutex: false` | 允许多窗口同时有声，各自独立控制 | 双声道对比、需要同时听两个源 |

配套注意：

- 控制栏的「静音」图标在 `emitter` 模式下按 `mutedPlayerDomIds.size === playerNum` 判定，`playerNum` 传错就会出现「明明有声音却显示静音」。
- 单屏/双屏切换时 `playerNum` 要跟着变。库内示例固定传 `2`，隐藏副屏后静音判定就会错误；业务侧应像上面的骨架一样用实际数量。
- `volumeMutex: false` 时，取消静音的窗口会加入 `JY_*_UNMUTED_DOMID_MULTI` 列表，重载时按列表恢复。
- 默认音量用 `default-volume`（0-100），首次加载生效。

## 4. 画中画

只有 `JyKmediaPlayer` 支持 `enterPiP()` / `exitPiP()` / `isPiP`，且**同一时刻只允许一个播放器进入画中画**。

```ts
import { nextTick } from 'vue';

async function togglePiP(target: 'primary' | 'secondary') {
	const current = target === 'primary' ? primaryRef.value : secondaryRef.value;
	const other = target === 'primary' ? secondaryRef.value : primaryRef.value;
	if (other?.isPiP) other.exitPiP();
	await nextTick();
	current?.isPiP ? current.exitPiP() : current.enterPiP();
}

function closeAllPiP() {
	primaryRef.value?.exitPiP();
	secondaryRef.value?.exitPiP();
}
```

- 浏览器需支持 `document.pictureInPictureEnabled`，且 kmedia 容器内存在 `<video>`。
- 进入/退出会触发 `enterPiP` / `exitPiP` 事件；也可监听 `document` 的 `enterpictureinpicture` / `leavepictureinpicture` 做兜底。
- 页面切后台或播放器销毁前先退出画中画，避免残留小窗。

## 5. stage 分屏（单实例双画面）

`JyKmediaPlayer` 提供 `#stage` 插槽：在一个播放器内把画面分成主区 + 右侧副区，适合「视频 + 讲稿/资料/白板」同容器展示，不需要第二个播放器实例。

```vue
<JyKmediaPlayer
	dom-id="lesson-stage"
	:src="videoSrc"
	:stage-visible="stageVisible"
	:stage-ratio="60"
	@stage-resize-start="onResizeStart"
>
	<template #stage="{ currentTime, playing, play, pause, toggleFullscreen }">
		<div class="stage-content">
			<!-- 讲稿、PPT 翻页、随堂练习等，按 currentTime 联动 -->
			<button @click="playing ? pause() : play()">{{ playing ? '暂停' : '播放' }}</button>
			<button @click="toggleFullscreen()">全屏</button>
		</div>
	</template>
</JyKmediaPlayer>
```

- `stageVisible` 为 `true` 且存在 `#stage` 插槽时才渲染分屏；loading 或错误态下自动退回单画面。
- `stageRatio` 是主区百分比，源码 clamp 到 `30`-`70`。
- 分屏宽度通过 CSS 变量 `--jkp-stage-primary-ratio` / `--jkp-stage-secondary-ratio` 下发，可覆盖但不要改比例计算。
- `stageResizeStart` 只通知拖拽开始，拖拽过程由业务实现（如果需要自定义把手）。
- `stage` 插槽的 props 与 `content` 基本一致，可直接用 `currentTime` 驱动内容同步。

## 6. 切换画面 / 换源

优先级从高到低：

1. **换 `src` + `reloadVideo()`**：最稳，保留实例与 emitter 关系。
2. **两路播放器各自换源**：主副独立 `reloadVideo()`，进度通过 emitter 自动对齐。
3. **DOM 级交换画面**：库的示例用 `switchView()` 交换 `.kmd-app-container`、`.jkp-hover-wrap`、`.player-loading-mask` 节点位置。这依赖内部类名，跨版本可能失效，仅作为最后手段，并加注释与回归用例。

```ts
async function switchPrimary(next: string) {
	primarySrc.value = next;
	await nextTick();
	primaryRef.value?.reloadVideo();
}
```

`JyAliPlayer` 没有 `reloadVideo()`，参考 [`player-ali.md`](player-ali.md) 第 7 节。

## 7. 陷阱

1. **只给播放器传 `emitter`，没给控制栏传**：控制栏会直接调用本实例方法，另一侧不跟随。
2. **每个播放器各建一个 `emitter`**：等于没有同步。emitter 必须在共同父级创建并向下传。
3. **`isPrimary` 多个为 `true`**：音量互斥与 `ENDED` 恢复逻辑会异常。
4. **`playerNum` 与实际数量不一致**：静音图标判定错误（见第 3 节）。
5. **用 `reactive()` 包 emitter**：`mitt` 实例被代理后 `on/emit` 的 `this` 可能异常，保持原始对象。
6. **在会重挂载的子组件里创建 emitter**：路由切换或 `v-if` 重建后新旧实例不再共享，必须提升到稳定的 feature 容器。
7. **kmedia 与 Ali 播放器共用 emitter**：事件名相同但底层能力不同，不要混组。
8. **多控制栏都绑定全局快捷键**：非主屏 `:support-hotkey="false"`，否则空格/方向键会同时作用于多个实例。
9. **`disabled` 当成禁用交互**：它只屏蔽 emitter 响应，用户仍可操作本实例。
10. **切布局时音频叠音**：切换前后显式 `mute()` / `exitPiP()`，并在布局稳定后再恢复声音，避免两个实例短暂同时出声。
