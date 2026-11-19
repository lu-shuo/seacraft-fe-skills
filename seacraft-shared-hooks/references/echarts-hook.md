# useEcharts（ECharts 初始化与响应式）

`useEcharts` 负责两件事：调用 `setOption` 和监听 `window.resize` 做自适应。它不创建实例、不销毁实例、不监听 option 变化，属于通用但职责很窄的 hook。

## 1. 用法

```vue
<template>
	<div ref="chartRef" class="chart"></div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue';
import * as echarts from 'echarts';
import { useEcharts } from '@/hooks/useEcharts';

const chartRef = ref<HTMLDivElement>();
let chart: echarts.ECharts | null = null;

onMounted(() => {
	if (!chartRef.value) return;

	chart = echarts.init(chartRef.value);
	useEcharts(chart, {
		grid: { top: 20, right: 16, bottom: 24, left: 32 },
		xAxis: { type: 'category', data: ['一', '二', '三'] },
		yAxis: { type: 'value' },
		series: [{ type: 'bar', data: [12, 20, 15] }]
	});
});

onBeforeUnmount(() => {
	chart?.dispose();
	chart = null;
});
</script>

<style scoped>
.chart {
	width: 100%;
	height: 320px;
}
</style>
```

## 2. API

```ts
useEcharts(myChart: echarts.ECharts, options: echarts.EChartsCoreOption): void
```

| 行为 | 说明 |
| --- | --- |
| 初始化 | `options` 是对象时立即 `myChart.setOption(options)` |
| resize | `useDebounceFn(() => myChart.resize(), 100)` 监听 `window.resize` |
| 卸载 | `onBeforeUnmount` 移除监听 |
| KeepAlive | `onActivated` 添加监听，`onDeactivated` 移除 |

## 3. 依赖

- `echarts`（项目 `^5.5.1`）。
- `useDebounceFn` 来自 `@vueuse/core`，源码**没有显式 import**，依赖项目自动导入。抽成公共包时必须补：

```ts
import { useDebounceFn } from '@vueuse/core';
```

## 4. 职责边界

| 事情 | 谁负责 |
| --- | --- |
| 创建实例 `echarts.init` | 调用方 |
| 首次 `setOption` | `useEcharts` |
| `window.resize` → `chart.resize()` | `useEcharts` |
| 容器尺寸变化（侧边栏、弹窗） | 调用方（`ResizeObserver`） |
| option 更新 | 调用方（`chart.setOption(next)`） |
| `chart.dispose()` | 调用方 |

## 5. 常见坑

1. **不负责 `dispose`**：组件卸载只移除监听；实例不销毁会泄漏 DOM 与 canvas。
2. **只监听 `window.resize`**：侧边栏折叠、弹窗打开、flex 布局变化不会触发；需要 `ResizeObserver`。
3. **不监听 option**：动态数据要自己 `setOption`；不要重复调用 `useEcharts`，否则重复加监听。
4. **KeepAlive 首挂载**：`onActivated` 可能再添加一次监听；浏览器对同一函数会去重，但 hook 本身没有状态保护。
5. **ECharts 全量引入**：`import * as echarts from 'echarts'` 体积大；按需引入：

```ts
import * as echarts from 'echarts/core';
import { BarChart } from 'echarts/charts';
import { GridComponent, TooltipComponent } from 'echarts/components';
import { CanvasRenderer } from 'echarts/renderers';

echarts.use([BarChart, GridComponent, TooltipComponent, CanvasRenderer]);
```

6. **容器无高度**：ECharts 初始化时容器宽高为 0，图表不显示；容器必须有明确高度。

## 6. 提取为公共 hook

推荐签名：

```ts
interface UseEchartsOptions {
	chart: MaybeRefOrGetter<echarts.ECharts | null | undefined>;
	option?: MaybeRefOrGetter<echarts.EChartsCoreOption>;
	autoResize?: boolean;
	resizeTarget?: MaybeRefOrGetter<HTMLElement | Window | null>;
	debounce?: number;
}

export function useEcharts(options: UseEchartsOptions): void;
```

改进点：

- `chart` 用 ref，支持实例创建晚于 hook 调用。
- `option` 响应式，内部 `watch` 调 `setOption`。
- 用 `ResizeObserver` 监听容器，`window.resize` 作为兜底。
- `onScopeDispose` 断开 observer / 监听，不负责 dispose。
- `echarts` 作为 optional peer，调用方传入实例，避免包内绑定 ECharts 版本。

## 7. 检查清单

- [ ] 调用方在 `onBeforeUnmount` 调 `chart.dispose()`。
- [ ] 容器有明确高度。
- [ ] 动态数据通过 `setOption` 更新，没有重复调用 hook。
- [ ] 容器尺寸变化场景有 `ResizeObserver`。
- [ ] 公共包显式 import `useDebounceFn`，并把 ECharts 设为 optional peer。
