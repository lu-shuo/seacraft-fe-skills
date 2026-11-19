# 从业务 hook 抽象公共 hook

本文只讨论**真正通用**的 hook：不依赖具体业务 store / API / 路由 / socket / 业务 id 约定。业务专用的 `useTheme`、`useWatermark`、`useStompSocket`、`useRecordPlayerState`、`useDarkMode`、`useGlobalVar` 不进入公共包，按业务实现保留。

## 1. 保留标准

一个 hook 要公共化，必须同时满足：

1. 不 import `@/stores/*`、`@/api/*`、`@/utils/global`、业务 socket 工具或具体路由。
2. 成功判定、接口字段、分页结构、暗色来源可通过参数替换。
3. 不假设 `'1'` / `'2'` 之类的窗口编号、角色或固定路径。
4. 不依赖运行时 `globalProperties` 之类的隐式全局。
5. 能在不启动业务应用的情况下测试主要分支。

## 2. 依赖替换

| 源码依赖 | 出现位置 | 替换方式 |
| --- | --- | --- |
| `ResponseStatus`（`@/api/interface`） | `useConfirm`、`useDownload`、`useTable` | 注入 `isSuccess?: (res) => boolean` |
| 接口返回字段 `list / currPage / totalCount` | `useTable` | 保留 `apiDataKeyMap`，或改为 `getTableData` / `getPageInfo` |
| 分页嵌套 `pageParam` | `useTable` | 参数化 `pageParamKey`，或改为 `buildPageParams` |
| `ElTable` 实例 | `useSelection` / `useSelectionOrigin` | 收窄成最小接口 `clearSelection / toggleRowSelection` |
| `useDarkMode()`（route + app store） | `useInlineCssVars` | 注入 `isDark: MaybeRefOrGetter<boolean>` |
| `useDebounceFn`（自动导入） | `useEcharts` | 显式 `import { useDebounceFn } from '@vueuse/core'` |
| `ElMessage` / `ElMessageBox` / `ElNotification` | `useConfirm` / `useDownload` | 显式 import，或定义可替换的 `notify` 适配器 |
| `getCurrentInstance()` | `useGlobalVar` | 不公共化 |

## 3. 设计公共 API

### 3.1 options object

```ts
// before
useConfirm(api, params, message, successMessage, errorMessage, title, type, onCancel);

// after
useConfirm(api, params, {
	message,
	title,
	type,
	successMessage,
	errorMessage,
	isSuccess,
	onCancel
});
```

### 3.2 响应式输入

```ts
import { computed, toValue, type MaybeRefOrGetter } from 'vue';

export function useInlineCssVars(options: {
	light: ThemeObject;
	dark: ThemeObject;
	scope?: string;
	isDark: MaybeRefOrGetter<boolean>;
}) {
	const cssVars = computed(() => {
		const vars = toValue(options.isDark) ? options.dark : options.light;
		return flattenToCssVars(vars, options.scope);
	});
	return { cssVars };
}
```

### 3.3 输出 ref

```ts
const visible = ref(false);
return { visible }; // 不要 return { visible: visible.value }
```

### 3.4 取消/错误必须 settle

`useConfirm` 的反例：取消/异常只执行 `onCancel`，Promise 永远 pending。公共版本统一 `resolve(false)`：

```ts
return new Promise<boolean>(resolve => {
	ElMessageBox.confirm(...)
		.then(async () => {
			try {
				const res = await api(params);
				resolve(isSuccess(res));
			} catch {
				resolve(false);
			}
		})
		.catch(() => {
			onCancel?.();
			resolve(false);
		});
});
```

### 3.5 清理

| 资源 | 清理方式 |
| --- | --- |
| `window` / `document` 监听 | `onBeforeUnmount` / `onScopeDispose` remove |
| 定时器 | `clearTimeout` / `clearInterval` |
| `ResizeObserver` / `IntersectionObserver` | `disconnect()` |
| ECharts | 调用方 `dispose()`，hook 断开监听 |
| `FileReader` | `reader.abort()` 或忽略回调 |

## 4. 自动导入显式化

公共包不能依赖调用方的 `unplugin-auto-import`。以下 hook 需要补 import：

| Hook | 需要补的 import |
| --- | --- |
| `useDialog` | `ref` from `vue` |
| `useConfirm` | `ElMessage`, `ElMessageBox` from `element-plus` |
| `useDownload` | `ElMessage`, `ElNotification` from `element-plus` |
| `useTable` | 已显式 import，无需补 |
| `useSelection` | 已显式 import，无需补 |
| `useSelectionOrigin` | 已显式 import，无需补 |
| `useInlineCssVars` | `computed` from `vue`；`useDarkMode` 改为参数 |
| `useEcharts` | `useDebounceFn` from `@vueuse/core` |

## 5. 依赖与打包

### 5.1 package.json

```json
{
	"name": "@seacraft/hooks",
	"version": "0.1.0",
	"type": "module",
	"main": "./dist/index.cjs",
	"module": "./dist/index.mjs",
	"types": "./dist/index.d.ts",
	"exports": {
		".": {
			"types": "./dist/index.d.ts",
			"import": "./dist/index.mjs",
			"require": "./dist/index.cjs"
		},
		"./echarts": {
			"types": "./dist/echarts.d.ts",
			"import": "./dist/echarts.mjs",
			"require": "./dist/echarts.cjs"
		}
	},
	"files": ["dist"],
	"peerDependencies": {
		"vue": "^3.2.0",
		"element-plus": ">=2.10.0",
		"echarts": ">=5.0.0",
		"@vueuse/core": ">=10.0.0"
	},
	"peerDependenciesMeta": {
		"element-plus": { "optional": true },
		"echarts": { "optional": true },
		"@vueuse/core": { "optional": true }
	}
}
```

- `vue` 必选 peer。
- `element-plus` / `echarts` / `@vueuse/core` 设为 optional；调用哪个 hook 就装哪个。
- 不要把这些依赖打进 bundle，避免重复实例。

### 5.2 Vite library 配置

```ts
import { defineConfig } from 'vite';
import dts from 'vite-plugin-dts';

export default defineConfig({
	build: {
		lib: {
			entry: { index: 'src/index.ts', echarts: 'src/echarts.ts' },
			formats: ['es', 'cjs'],
			fileName: (format, entryName) => `${entryName}.${format === 'es' ? 'mjs' : 'cjs'}`
		},
		rollupOptions: {
			external: ['vue', 'element-plus', 'echarts', '@vueuse/core']
		}
	},
	plugins: [dts({ insertTypesEntry: true })]
});
```

- `echarts` 单独入口，避免不使用图表时也解析 ECharts。
- 没有 CSS，不需要处理样式产物。

### 5.3 统一导出

```ts
// src/index.ts
export * from './useDialog';
export * from './useConfirm';
export * from './useDownload';
export * from './useTable';
export * from './useSelection';
export * from './useSelectionOrigin';
export * from './useInlineCssVars';

// src/echarts.ts
export * from './useEcharts';
```

## 6. 测试

### 6.1 工具

- `vitest` + `jsdom`（或 `happy-dom`）。
- `@vue/test-utils` 用于需要组件上下文的 hook（`onMounted`、`onActivated`）。
- Element Plus 用 `vi.mock` 替换。

### 6.2 useDialog

```ts
import { describe, expect, it, vi } from 'vitest';
import { useDialog } from '../src/useDialog';

describe('useDialog', () => {
	it('opens, runs openCb with args, and closes', () => {
		const openCb = vi.fn();
		const closeCb = vi.fn();
		const { dialogVisible, showDialog, handleClose } = useDialog<{ id: number }>(openCb, closeCb);

		showDialog({ id: 1 });
		expect(dialogVisible.value).toBe(true);
		expect(openCb).toHaveBeenCalledWith({ id: 1 });

		handleClose();
		expect(closeCb).toHaveBeenCalled();
		expect(dialogVisible.value).toBe(false);
	});
});
```

### 6.3 useInlineCssVars

```ts
import { ref } from 'vue';
import { useInlineCssVars } from '../src/useInlineCssVars';

it('flattens and switches by isDark', () => {
	const isDark = ref(false);
	const { cssVars } = useInlineCssVars({
		isDark,
		scope: 'page',
		light: { panel: { bgColor: '#fff' } },
		dark: { panel: { bgColor: '#000' } }
	});
	expect(cssVars.value['--page-panel-bgColor']).toBe('#fff');
	isDark.value = true;
	expect(cssVars.value['--page-panel-bgColor']).toBe('#000');
});
```

### 6.4 useTable

```ts
import { mount } from '@vue/test-utils';
import { expect, it, vi } from 'vitest';
import { useTable } from '../src/useTable';

it('loads on mount and maps page fields', async () => {
	const api = vi.fn().mockResolvedValue({
		status: 200,
		data: { list: [{ id: 1 }], pageNum: 1, pageSize: 10, total: 1 }
	});

	const wrapper = mount({
		setup() {
			return useTable({ api, initOnActivated: false });
		},
		template: '<div />'
	});

	await Promise.resolve();
	await Promise.resolve();
	expect(api).toHaveBeenCalled();
	expect(wrapper.vm.tableData).toEqual([{ id: 1 }]);
});
```

### 6.5 mock Element Plus

```ts
vi.mock('element-plus', () => ({
	ElMessage: { success: vi.fn(), error: vi.fn() },
	ElMessageBox: { confirm: vi.fn(() => Promise.resolve()) },
	ElNotification: vi.fn(() => ({ close: vi.fn() }))
}));
```

## 7. 适配矩阵

| Hook | 必须替换 | 可选增强 |
| --- | --- | --- |
| `useDialog` | 补 `ref` import | `beforeOpen` / `afterClose` |
| `useConfirm` | `ResponseStatus` → `isSuccess`；取消/异常 `resolve(false)` | 按钮文案、确认框组件适配 |
| `useDownload` | 成功判定、`responseType`、文件名解析 | 返回下载结果、失败重试 |
| `useTable` | 成功判定、字段映射、`pageParam` 结构 | 请求取消、缓存、错误回调 |
| `useSelection` | Element Plus Table 最小接口 | 统一 `getSelectionPayload()` |
| `useSelectionOrigin` | 补 `tableRef` 类型；建议合并进新版 | 直接废弃 |
| `useInlineCssVars` | 注入 `isDark`；导出 flatten 工具 | 支持 number/boolean 值转换 |
| `useEcharts` | 补 `useDebounceFn`、图表 ref、ResizeObserver | option 响应式、主题切换 |

## 8. 迁移步骤

1. 复制保留的 8 个 hook 到 `packages/hooks/src/`，保持 `useXxx.ts` 命名。
2. 按第 2 节替换依赖，删除所有 `@/` 业务 import。
3. 按第 4 节补显式 import。
4. 长参数改 options object，成功判定改 `isSuccess`。
5. 加 `onScopeDispose` / `onBeforeUnmount` 清理。
6. 写入口导出（`index.ts` + `echarts.ts`）。
7. 配置 `package.json` peerDependencies 与 Vite `external`。
8. 写 Vitest 测试，覆盖成功、失败、取消、清理。
9. 业务项目用 workspace / `pnpm link` 验证，再发布。
10. 更新 `seacraft-shared-hooks` 的 hook 列表。

## 9. 交付检查

- [ ] 只包含通用 hook；业务专用 hook 没有混入。
- [ ] 没有 `@/` 业务 import。
- [ ] 所有 Promise 在成功/失败/取消分支都能 settle。
- [ ] 生命周期与监听有清理。
- [ ] 可选 peer 依赖没有被打进 bundle。
- [ ] `exports` 同时提供 ESM/CJS/types。
- [ ] 测试覆盖关键分支。
- [ ] README 写清依赖、适配点与示例。
