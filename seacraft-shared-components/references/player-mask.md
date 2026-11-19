# PlayerMask（通用播放器遮罩）

> 位置：`jy-digital-patrol-ui/src/components/PlayerMask/`
> 性质：业务仓通用组件，不随 `jy-player-ui` 发布。可独立使用，也可直接塞进 `JyKmediaPlayer` 的 `#loading` / `#mask` 插槽。

## 1. 组件组成

| 组件 | 对应状态 | 默认定位 | 是否绝对定位 |
| --- | --- | --- | --- |
| `PlayerLoadingMask` | 加载中 / 缓冲 | 填充父容器 | 否（`100% × 100%` 流内元素） |
| `PlayerErrorMask` | 播放错误 / 无资源 | 覆盖父容器 | 是（`inset: 0`，`z-index: 3`） |
| `PlayerPauseMask` | 暂停 / 手势门禁 | 覆盖父容器 | 是（`inset`，`z-index: 3`） |
| `PlayerEndMask` | 播放结束 | 覆盖父容器 | 是（`inset`，`z-index: 10`） |
| `PlayerPromptIllustration` | 空态插图 | 跟随内容 | 否（SVG 插图） |

项目中这些组件已由 `unplugin-vue-components` 自动注册；`MultiWindowPlayer` 采用显式导入。两种方式都可，团队内保持一致即可。

### 1.1 依赖与初始化

- 运行依赖：`vue`；图标依赖 `SvgIcon` 与 `vite-plugin-svg-icons` 注入的雪碧图。
- `PlayerErrorMask` / `PlayerLoadingMask` 使用了 `computed` 但源码未 import，依赖 `unplugin-auto-import` 的 `vue` 预设；不配就补 `import { computed } from 'vue'`。
- `SvgIcon` 需要全局注册、局部导入或被 `unplugin-vue-components` 自动注册；否则遮罩里的 loading / play / replay 图标不渲染。
- 入口必须 `import 'virtual:svg-icons-register'`，并准备图标：`loading`、`play-btn`、`replay-btn`。
- 完整依赖、Vite 配置、入口顺序与构建验证见 [`project-bootstrap.md`](project-bootstrap.md)；图标体系见 [`svg-icon.md`](svg-icon.md)。

## 2. 与播放器插槽的对应关系

| 播放器状态 | 组件 | 接入位置 |
| --- | --- | --- |
| 加载 / 缓冲 | `PlayerLoadingMask` | `#loading` 插槽（插槽 props `loadType` 可区分 `loading` / `waiting`） |
| 播放错误 | `PlayerErrorMask` | `#mask` 插槽，`v-show` 或 `v-if` 配合 `KMEDIA_STATUS === 'ERROR'` |
| 暂停 | `PlayerPauseMask` | `#mask` 插槽 `PAUSED` 分支，或业务层自己做的手势门禁 |
| 结束 | `PlayerEndMask` | `#mask` 插槽 `ENDED` 分支 |
| 无资源 / 空态 | `PlayerPromptIllustration` + 文案 | 播放器外层，与播放器互斥渲染 |

完整组合：

```vue
<JyKmediaPlayer
	:dom-id="domId"
	:src="src"
	@paused="paused = true"
	@playing="paused = false"
>
	<template #loading>
		<PlayerLoadingMask :follow-theme="maskFollowTheme" />
	</template>

	<template #mask="{ KMEDIA_STATUS, play, pause, errorCode, errorMessage }">
		<PlayerPauseMask v-if="KMEDIA_STATUS === 'PAUSED'" @play="play" />
		<PlayerEndMask v-else-if="KMEDIA_STATUS === 'ENDED'" @replay="play">
			<!-- 可选：追加「下一节」「返回列表」等按钮 -->
		</PlayerEndMask>
		<PlayerErrorMask
			v-else-if="KMEDIA_STATUS === 'ERROR'"
			:error-code="errorCode"
			:error-message="errorMessage"
			:follow-theme="maskFollowTheme"
		/>
	</template>
</JyKmediaPlayer>
```

注意：kmedia / ali 播放器的 `#mask` 插槽只在「非 loading 且非 waiting」时渲染。缓冲期间暂停/结束/错误遮罩会暂时消失，回到播放状态后再出现；这是播放器行为，不是遮罩组件的问题。

## 3. PlayerLoadingMask

```vue
<PlayerLoadingMask :follow-theme="false" />
```

| 属性 / 事件 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `followTheme` | `boolean` | `false` | `true` 时按亮/暗主题取色；否则强制黑底白字 |

- 无事件、无插槽。
- 根节点是 `width/height: 100%` 的流内元素，**不是绝对定位**：
  - 放进播放器 `#loading` 插槽时，插槽容器本身是绝对定位覆盖层，表现正常。
  - 作为播放器兄弟节点使用时，父级必须有明确高度（如 `aspect-ratio` 或固定高度），否则高度为 0。
- 图标依赖全局 `SvgIcon` 的 `loading`，颜色取 `--el-color-primary`。

## 4. PlayerErrorMask

```vue
<PlayerErrorMask
	:error-code="errorCode"
	:error-message="errorMessage"
	:follow-theme="false"
/>
```

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `errorCode` | `string` | — | kmedia / ali 错误码，用于归类异常类型并选插图 |
| `errorMessage` | `string` | — | 兜底文案，**仅在 other 类型生效** |
| `followTheme` | `boolean` | `false` | `true` 按主题取色；否则黑底 |

### 错误归类

组件按固定顺序匹配错误码，命中即停：

1. `BROWSER_ERROR_CODE_MAP` → `browser`（解码器、MSE、自动播放等）
2. `DEVICE_ERROR_CODE_MAP` → `device`（设备/信令/录像/鉴权等）
3. `NETWORK_ERROR_CODE_MAP` → `network`（连接、超时、码流中断等）
4. 未命中或无 `errorCode` → `other`

同一错误码同时存在于多张表时，按上述顺序优先。例如 `30010` 同时在网络表与设备表，最终归类为 `device`。

### 文案来源

| 类型 | 文案来源 | 为空时的兜底 |
| --- | --- | --- |
| `network` | `sysConfigStore.playerExceptionMsgConfig.networkExceptionMsg` | 「当前网络可能不稳定，请稍后重试」 |
| `device` | `...deviceExceptionMsg` | 「设备连接可能存在波动，请稍后重试」 |
| `browser` | `...chromeExceptionMsg` | 「建议刷新页面，或更换浏览器后重试」 |
| `other` | `errorMessage` → `...otherExceptionMsg` | 「请稍后重试，如持续无法播放可联系管理员协助处理」 |

因此：

- 传了已知的 `errorCode` 时，`errorMessage` **不会覆盖** network/device/browser 文案；要改文案请改系统配置（系统管理 → 加载提示）。
- 不传 `errorCode`（或无资源、接口失败等业务态）时，`errorMessage` 才会直接展示。项目里的 `<PlayerErrorMask v-else error-message="暂无可播放录像" />` 就是这种用法。

### 插图

内部使用 `PlayerPromptIllustration`，按归类渲染 `network` / `device` / `browser` / `other` 四种 SVG。

## 5. PlayerPauseMask

```vue
<PlayerPauseMask @play="handlePlay" />
```

- 无 props、无插槽；只有 `play` 事件。
- 点击整个遮罩触发 `play`，按钮文案固定为「播放」。
- 深色 70% 蒙层 + 白字按钮，不跟随主题。
- 常用于：移动端手势门禁、业务层自定义暂停态（例如 `v-if="paused"` 时覆盖在播放器上）。

## 6. PlayerEndMask

```vue
<PlayerEndMask @replay="handleReplay">
	<button class="next-btn" @click.stop="goNext">下一节</button>
</PlayerEndMask>
```

| 属性 / 事件 / 插槽 | 说明 |
| --- | --- |
| `replay` 事件 | 点击「再看一遍」触发 |
| 默认插槽 | 追加业务按钮，渲染在「再看一遍」下方 |
| props | 无 |

- 绝对定位、`z-index: 10`，与部分控制栏层级相同，注意 DOM 顺序。
- **样式不是 `scoped`**：`.player-end-mask`、`.opt-btn` 会全局生效。复制或二次封装时建议改成 `scoped` 或加命名空间。
- 不跟随主题，固定深色蒙层。

## 7. PlayerPromptIllustration

```vue
<PlayerPromptIllustration type="no-view" />
```

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `type` | `'network' \| 'device' \| 'browser' \| 'other' \| 'no-view'` | 选择插图；未命中时按 `other` 渲染 |

- 纯展示 SVG，无事件、无插槽。
- 颜色取 `--window-prompt-illustrationColor`，默认 `#8facdf`。
- 常用于「当前教室未提供此视角画面」这类空态；配合自己的文案使用，组件不带文字。

## 8. 主题变量

四个遮罩共用一套 `--window-prompt-*` 变量（由 `useInlineCssVars({ scope: 'window-prompt' })` 生成）：

| 变量 | 用途 | 亮色 | 暗色 |
| --- | --- | --- | --- |
| `--window-prompt-bgColor` | 遮罩背景 | `#f8faff` | `transparent` |
| `--window-prompt-illustrationColor` | 错误插图颜色 | `#9db2e1` | `#8facdf` |
| `--window-prompt-descColor` | 错误描述 / 加载文案 | `#74839a` | Error：`rgba(255,255,255,0.58)`；Loading：`#ffffff` |
| `--window-prompt-titleColor` | 预留标题色（当前样式未消费） | `#1f2937` | `rgba(255,255,255,0.92)` |

`followTheme` 的行为：

- `false` / 不传：不使用上表，组件内联写死深色值（Error / Loading 均为 `#000` 背景，Loading 文案 `#fff`），适合播放器黑底场景。
- `true`：使用当前亮/暗主题值；亮色主题下是浅色背景，需要确认与播放器画面是否协调。

也可以像 `SingleWindow.vue` 那样在外层统一挂 `maskCssVars`，再给遮罩传 `followTheme`；变量会从父级继承。

## 9. 使用位置与布局

| 用法 | 父级要求 |
| --- | --- |
| 播放器 `#loading` 插槽 | 无额外要求，插槽容器已绝对定位 |
| 播放器 `#mask` 插槽 | 无额外要求，插槽默认渲染在播放器根节点内 |
| 作为播放器兄弟节点覆盖 | 父子容器 `position: relative`，且有宽高 |
| 替换整个播放区域（加载/无资源） | 容器有明确宽高，`PlayerLoadingMask` 走流内填充，`PlayerErrorMask` 走绝对覆盖 |

## 10. 陷阱

1. **父级没有 `position: relative` + 高度**：`PlayerErrorMask` / `PlayerPauseMask` / `PlayerEndMask` 会按最近的定位祖先铺满，可能盖住整页。
2. **`PlayerLoadingMask` 不是绝对定位**：当兄弟节点用时容易高度为 0；需要覆盖效果请放进播放器 `#loading` 插槽。
3. **`followTheme` 默认是深色**：浅色页面直接复用会得到黑底遮罩，需要显式传 `true` 或在外层覆盖变量。
4. **`errorMessage` 对已知错误码无效**：要自定义网络/设备/浏览器错误文案，改 `sysConfigStore.playerExceptionMsgConfig`，不要在调用处传 `error-message`。
5. **移动端播放器没有 `errorCode`**：`JyKmediaMobilePlayer` 的 `#mask` 只提供 `errorMessage`，`PlayerErrorMask` 会落到 `other`，无法按网络/设备/浏览器分类。
6. **`PlayerEndMask` 样式全局泄漏**：非 scoped，同页其他同名类会被影响。
7. **`#mask` 在缓冲时会卸载**：暂停/结束遮罩在 `waiting` 期间不可见，属播放器插槽渲染条件。
8. **`z-index` 冲突**：EndMask 的 `10` 与控制栏 `10` 同级，DOM 顺序决定谁在上；需要按钮可点时调高层级或缩小遮罩范围。
9. **暂停/结束遮罩不跟随主题**：只有 Loading / Error 支持 `followTheme`。
10. **插图依赖全局 `SvgIcon`**（Loading / Pause / End 内的图标），脱离项目环境时要替换图标实现。

## 11. 复用到其他项目

- 复制 `PlayerMask/` 目录；按需保留 5 个组件。
- 替换或提供：
  - 全局 `SvgIcon` 及图标 `loading`、`play-btn`、`replay-btn`；
  - `useInlineCssVars` / `useDarkMode`（或改成项目主题方案）；
  - `useSysConfigStore.playerExceptionMsgConfig`、`@/config/playerErrorCode`（错误归类与文案）。
- 若不需要错误分类，可去掉 `PlayerErrorMask` 的 code 逻辑，退化为「插图 + 文案」的通用遮罩。
- 建议把 `PlayerEndMask` 的样式改为 scoped，或在类名前加业务前缀。
