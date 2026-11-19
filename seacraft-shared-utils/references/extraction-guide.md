# 从多个项目提取公共 utils

本文说明如何从公司多个前端项目的 `utils/` 里把重复函数收敛成公共包：盘点、去重、修缺陷、设计 API、测试、打包、迁移。

## 1. 盘点

### 1.1 找出重复函数

按项目导出名求交集，比逐个读文件快：

```powershell
# 存放各前端项目的目录，按本机情况设置
$workspace = "<workspace-root>"

$projects = @(
  'jy-template-ui',
  'jy-application-supervision-ui-new',
  'jy-application-tqmp-ui',
  'jy-digital-patrol-ui',
  'jy-application-space-ui'
)

$sets = @{}
foreach ($p in $projects) {
  $file = Join-Path $workspace "$p\src\utils\index.ts"
  if (!(Test-Path $file)) { continue }
  $sets[$p] = Select-String -Path $file -Pattern '^\s*export\s+(const|function|async function)\s+([A-Za-z0-9_]+)' |
    ForEach-Object { $_.Matches[0].Groups[2].Value }
}

$all = $sets.Values | ForEach-Object { $_ } | Sort-Object -Unique
foreach ($name in $all) {
  $found = @($sets.Keys | Where-Object { $sets[$_] -contains $name })
  if ($found.Count -ge 2) { "{0}`t{1}" -f $name, ($found -join ',') }
}
```

本次结果的核心交集：

```text
getTimeState, getBrowserLang, getShowMenuList, getFlatArr, getType,
getAllBreadcrumbList, handleProp, handleRowAccordingToProp, filterEnum,
findItemNested, formatValue, convertToChinaNum
```

再按功能决定哪些保留（菜单/权限类排除）。

### 1.2 看单文件 utils

`index.ts` 之外，重点看：

- `utils/is/index.ts`（5 个项目几乎一致，space 多了设备判断）
- `utils/time.ts` / `utils/md5.ts` / `utils/dom.ts` / `utils/device.ts`
- `app-core/color.ts`、`app-core/comm.js`、`app-core/util.js`
- `app-index/utils/time.js`、`dom.js`

### 1.3 判断是否公共

| 问题 | 保留 | 排除 |
| --- | --- | --- |
| 是否 import 业务 model/store/router/权限？ | 否 | 是 |
| 是否依赖 Element Plus 等 UI 库？ | 可通过回调/返回值解耦 | 直接 `ElMessage` |
| 是否模块加载时读 `window/navigator`？ | 包成函数后可保留 | 顶层读取 |
| 是否只对某个业务字段有意义？ | 否 | 「单视角/双视角」等 |
| 是否 ≥2 个项目出现？ | 是 | 单项目私有 |

## 2. 去重与修复

### 2.1 选 canonical 版本

同名函数在项目间常有细微差异，先列差异再选：

| 函数 | 差异 | 选择 |
| --- | --- | --- |
| `getTimeState` | patrol/template 6–10/10–14；supervision 6–11/11–14 | 统一显式区间，写测试锁边界 |
| `getFlatArr` | tqmp 多 `parent` 参数 | 用 tqmp 版本，或拆 `flattenTree` / `flattenTreeWithParent` |
| `parseSecondsToTimeString` | space/resourcemanage 省略小时；tqmp 固定 `HH:mm:ss` | 保留「<1h 省略小时」；固定格式另设 `formatClock` |
| `generateUUID` | Math.random vs crypto | 用 `crypto.randomUUID` + fallback |
| `generateFileMD5` | 全量 vs 采样 5 片 | 默认全量，提供 `sample` 选项 |
| 设备判断 | ismobilejs + 顶层常量 vs UA 函数 | 用函数 + 可选 ismobilejs |
| `copyToClipboard` | 直接 ElMessage | 返回 `Promise<boolean>` |

### 2.2 修掉已知缺陷

公共化不是原样搬运，至少修：

- `isArray` 返回非布尔 → `Array.isArray`。
- `isUnDef` / `isAsyncFunction` 类型谓词不准确 → 返回 `boolean` 或修谓词。
- `deepEqual` 不支持数组/Date/Map/Set/循环引用 → 不收录，改用 `lodash-es/isEqual`。
- `getFlatArr` 的 JSON 克隆 → 改为只读递归或 `structuredClone`。
- `copyToClipboard` 未 await Clipboard API → 返回 Promise。
- `addScript` reject `false` → reject `Error`。
- `waitUntilSuccess` 不可取消 → `retryUntil` + `AbortSignal`。
- 颜色函数 `ElMessage.warning` → 返回 `null` / 抛错。
- 设备判断顶层 `navigator` → 函数内读取。
- `formatDate` 的 `replaceAll` → `replace(/-/g, '/')`，或直接用 dayjs。

### 2.3 统一口径

- 时间单位：秒。
- 文件大小：B/KB/M/G 还是 B/KB/MB/GB，产品先确认。
- 数字单位：万/亿、保留几位小数。
- 颜色：6 位 hex，不带 alpha。
- 空值：`'--'` 是表格展示专用，不要污染通用函数默认值。

## 3. 设计公共 API

### 3.1 纯函数

```ts
// good
export function formatFileSize(bytes: number): string;

// bad：读全局、依赖 UI
export function formatFileSize() {
	const bytes = globalStore.file.size;
	ElMessage.success('ok');
}
```

### 3.2 可选参数用 options object

```ts
export interface FileMD5Options {
	chunkSize?: number;
	sample?: boolean;
	onProgress?: (current: number, total: number) => void;
}

export function generateFileMD5(file: File, options: FileMD5Options = {}): Promise<string>;
```

### 3.3 错误处理

- 可预期失败返回 `null` / `false`。
- 编程错误抛 `TypeError` / `RangeError`。
- 不要 `console.error` 后返回 `undefined` 让调用方猜。

```ts
export function hexToRgb(hex: string): RGB | null {
	if (!/^#?[0-9a-f]{6}$/i.test(hex)) return null;
	// ...
}
```

### 3.4 浏览器防御

```ts
export const isClient = () => typeof window !== 'undefined';

export function copyToClipboard(text: string): Promise<boolean> {
	if (!isClient()) return Promise.resolve(false);
	// ...
}
```

### 3.5 命名

- 动词开头：`get` / `is` / `format` / `parse` / `create` / `load`。
- 不用 `utils1`、`common`、`misc`。
- 一个文件一个领域：`is.ts`、`tree.ts`、`format.ts`、`browser.ts`、`device.ts`、`file.ts`、`color.ts`。

## 4. 依赖治理

| 能力 | 依赖 | 建议 |
| --- | --- | --- |
| 类型/树/格式化/颜色 | 无 | 直接打包 |
| 文件 MD5 | `spark-md5` | optional peer，或放 `./hash` 子路径 |
| 日期 | `dayjs` | optional peer；简单格式化可零依赖 |
| 设备 | `ismobilejs` | 可选；优先 UA 函数 |
| 剪贴板提示 | 无 | 返回 Promise，调用方提示 |
| 深拷贝/相等 | `structuredClone` / `lodash-es` | 不要把 lodash 整包打进 utils |

`package.json` 示例：

```json
{
	"name": "@seacraft/utils",
	"version": "0.1.0",
	"type": "module",
	"main": "./dist/index.cjs",
	"module": "./dist/index.mjs",
	"types": "./dist/index.d.ts",
	"sideEffects": false,
	"exports": {
		".": {
			"types": "./dist/index.d.ts",
			"import": "./dist/index.mjs",
			"require": "./dist/index.cjs"
		},
		"./is": {
			"types": "./dist/is.d.ts",
			"import": "./dist/is.mjs",
			"require": "./dist/is.cjs"
		},
		"./hash": {
			"types": "./dist/hash.d.ts",
			"import": "./dist/hash.mjs",
			"require": "./dist/hash.cjs"
		},
		"./color": {
			"types": "./dist/color.d.ts",
			"import": "./dist/color.mjs",
			"require": "./dist/color.cjs"
		}
	},
	"files": ["dist"],
	"peerDependencies": {
		"spark-md5": "^3.0.2",
		"dayjs": "^1.11.0"
	},
	"peerDependenciesMeta": {
		"spark-md5": { "optional": true },
		"dayjs": { "optional": true }
	}
}
```

## 5. Vite 打包

```ts
import { defineConfig } from 'vite';
import dts from 'vite-plugin-dts';

export default defineConfig({
	build: {
		lib: {
			entry: {
				index: 'src/index.ts',
				is: 'src/is.ts',
				hash: 'src/hash.ts',
				color: 'src/color.ts'
			},
			formats: ['es', 'cjs'],
			fileName: (format, entry) => `${entry}.${format === 'es' ? 'mjs' : 'cjs'}`
		},
		rollupOptions: {
			external: ['spark-md5', 'dayjs']
		}
	},
	plugins: [dts({ insertTypesEntry: true })]
});
```

要点：

- `sideEffects: false`，所有导出可 tree-shake。
- 可选依赖 external，不打进 bundle。
- 子路径入口让不使用 hash 的项目不解析 `spark-md5`。
- 类型用 `vite-plugin-dts` 生成；`exports` 同时给 `types`。
- 无 CSS、无 Vue 依赖，框架无关。

## 6. 测试

### 6.1 工具

- `vitest` + `jsdom`（浏览器函数）。
- `@vue/test-utils` 不需要（utils 无组件）。
- 设备/剪贴板用 `vi.stubGlobal`。

### 6.2 类型判断

```ts
import { describe, expect, it } from 'vitest';
import { getType, is, isObject } from '../src/is';

describe('type', () => {
	it('getType', () => {
		expect(getType(null)).toBe('null');
		expect(getType([])).toBe('array');
		expect(getType(new Date())).toBe('date');
	});
	it('is', () => {
		expect(is([], 'Array')).toBe(true);
		expect(is([], 'array')).toBe(false);
		expect(isObject([])).toBe(false);
	});
});
```

### 6.3 树

```ts
it('flattens tree', () => {
	const tree = [{ id: 1, children: [{ id: 2 }] }];
	expect(flattenTree(tree).map(n => n.id)).toEqual([1, 2]);
});

it('finds nested item', () => {
	const tree = [{ value: 1, children: [{ value: 2 }] }];
	expect(findItemNested(tree, 2, 'value', 'children')?.value).toBe(2);
});
```

### 6.4 时间

```ts
it('parses and formats duration', () => {
	expect(parseTimeToSeconds('01:02:03')).toBe(3723);
	expect(parseSecondsToTimeString(3723)).toBe('01:02:03');
	expect(parseSecondsToTimeString(59)).toBe('00:59');
});
```

### 6.5 颜色

```ts
it('converts color', () => {
	expect(hexToRgb('#027bf5')).toEqual({ r: 2, g: 123, b: 245 });
	expect(rgbToHex({ r: 2, g: 123, b: 245 })).toBe('#027bf5');
	expect(hexToRgb('#fff')).toBeNull();
});
```

### 6.6 设备（mock UA）

```ts
import { afterEach, vi } from 'vitest';

afterEach(() => vi.unstubAllGlobals());

it('detects iOS', async () => {
	vi.stubGlobal('navigator', { userAgent: 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)', maxTouchPoints: 1 });
	const { isIOS, getDeviceType } = await import('../src/device');
	expect(isIOS()).toBe(true);
	expect(getDeviceType()).toBe('mobile');
});
```

### 6.7 剪贴板

```ts
it('copies text', async () => {
	const writeText = vi.fn().mockResolvedValue(undefined);
	vi.stubGlobal('navigator', { clipboard: { writeText } });
	const { copyToClipboard } = await import('../src/browser');
	await expect(copyToClipboard('abc')).resolves.toBe(true);
	expect(writeText).toHaveBeenCalledWith('abc');
});
```

## 7. 迁移策略

1. 在业务仓新建 `packages/utils` 或独立包，先只放 `is` / `format` / `color` 等低风险模块。
2. 旧 `src/utils/index.ts` 保留 re-export：

```ts
export { getType, is, isObject } from '@seacraft/utils';
export { getFlatArr, findItemNested } from '@seacraft/utils';
// 逐步替换 import
```

3. 按页面/模块迁移，避免一次全量替换。
4. 每迁移一个域，删除旧实现并跑测试。
5. 菜单/权限/主题/播放器等业务 utils 留在业务仓，不迁。
6. 更新本文档和 `seacraft-shared-utils` 的目录表。

## 8. 交付检查

- [ ] 只收录跨项目通用函数，业务 utils 未混入。
- [ ] 所有浏览器 API 有 SSR 防御。
- [ ] 模块加载无顶层 `window/navigator` 读取。
- [ ] 已知缺陷已修（`deepEqual`、剪贴板、设备、颜色、脚本加载）。
- [ ] 类型完整，错误处理明确。
- [ ] `sideEffects: false`，可选依赖 external。
- [ ] 子路径 `./is` / `./hash` / `./color` 可独立使用。
- [ ] Vitest 覆盖类型、树、时间、颜色、设备、剪贴板。
- [ ] 旧项目有 re-export 过渡，未一次性大改。
