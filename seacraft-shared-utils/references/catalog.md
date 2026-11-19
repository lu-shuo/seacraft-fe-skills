# Utils 目录与签名

> 来源：公司多个 Vue 后台项目
> 收录范围：跨项目复用、无业务/UI 依赖、模块加载无副作用。
> 标注「推荐签名」的是从旧实现改造后的公共版本；「源码签名」是原项目实际签名。

## 1. 类型与值

### is / 类型守卫

来源：`src/utils/is/index.ts`（template、patrol、tqmp、supervision、space 基本一致）

| 函数 | 签名 | 说明 |
| --- | --- | --- |
| `is` | `is(val: unknown, type: string): boolean` | `Object.prototype.toString` 精确匹配，`type` 区分大小写 |
| `isFunction` | `isFunction<T = Function>(val: unknown): val is T` | 函数 |
| `isDef` | `isDef<T = unknown>(val?: T): val is T` | `typeof val !== 'undefined'` |
| `isUnDef` | `isUnDef<T = unknown>(val?: T): val is T` | 未定义 |
| `isObject` | `isObject(val: any): val is Record<any, any>` | 仅普通对象；数组、Date 为 false |
| `isDate` | `isDate(val: unknown): val is Date` | Date |
| `isNumber` | `isNumber(val: unknown): val is number` | Number |
| `isAsyncFunction` | `isAsyncFunction<T = any>(val: unknown): val is Promise<T>` | 实际判断 `AsyncFunction`，类型谓词不准确 |
| `isPromise` | `isPromise<T = any>(val: unknown): val is Promise<T>` | Promise + then/catch |
| `isString` | `isString(val: unknown): val is string` | 字符串 |
| `isBoolean` | `isBoolean(val: unknown): val is boolean` | 布尔 |
| `isArray` | `isArray(val: any): val is Array<any>` | `Array.isArray` |
| `isClient` | `isClient(): boolean` | `typeof window !== 'undefined'` |
| `isWindow` | `isWindow(val: any): val is Window` | Window |
| `isElement` | `isElement(val: unknown): val is Element` | 有 `tagName` 的对象 |
| `isNull` | `isNull(val: unknown): val is null` | `val === null` |
| `isNullOrUnDef` | `isNullOrUnDef(val: unknown): val is null \| undefined` | 两者 |
| `isHexColor` | `isHexColor(str: string): boolean` | 3 或 6 位 hex |

注意：

- `isUnDef` 的类型谓词 `val is T` 不严谨；公共版本建议返回 `boolean`。
- `isAsyncFunction` 判断的是 `AsyncFunction`，但返回 `val is Promise<T>`；不要依赖这个类型缩窄。
- `isObject` 不包含数组、Date；判断「对象或数组」要组合 `isObject || isArray`。

### getType

```ts
getType(val: any): string
```

- `null` → `'null'`
- 非对象 → `typeof`（`'string'`、`'number'`、`'boolean'`、`'function'`、`'undefined'`、`'symbol'`、`'bigint'`）
- 对象 → `Object.prototype.toString` 截取并小写：`'array'`、`'date'`、`'object'`、`'promise'`、`'map'`、`'set'`

与 `is` 的区别：`getType([])` 是 `'array'`，`is([], 'Array')` 才是 `true`。

## 2. 树、对象与表格显示

### getFlatArr

```ts
// 源码（template/patrol/supervision）
getFlatArr(menuList: Menu.MenuOptions[]): Menu.MenuOptions[]

// tqmp 版本多一个 parent 参数
getFlatArr(menuList, parent?: Menu.MenuOptions): Menu.MenuOptions[]
```

行为：先 `JSON.parse(JSON.stringify(menuList))` 深拷贝，再 `reduce` 递归拍平；`parent` 版本会把父节点信息带进结果时使用。

风险：

- JSON 克隆丢失 `undefined`、函数、`Date`、`Map/Set`，循环引用直接抛错。
- 公共版本建议用 `structuredClone`，或显式只读递归不克隆。

### findItemNested

```ts
findItemNested(enumData: any[], callValue: any, value: string, children: string): any
```

- 递归查找第一个 `current[value] === callValue` 的节点。
- 无循环保护、无「未找到」默认值（返回 `undefined`）。
- `value` / `children` 通常传 `'value'` / `'children'`。

### handleProp

```ts
handleProp(prop: string): string
```

- `'user.name'` → `'name'`；`'name'` → `'name'`。
- 只取最后一段，不校验路径是否存在。

### handleRowAccordingToProp

```ts
handleRowAccordingToProp(row: Record<string, any>, prop: string): any
```

- `'user.name'` 逐级取值，任一环节为空就回退 `'--'`。
- 不支持 `a[0].b` 这类表达式，只按 `.` 分割。
- `0` / `false` 会被 `??` 保留；但空字符串 `''` 不会回退。

### filterEnum

```ts
filterEnum(
  callValue: any,
  enumData?: any,
  fieldNames?: { label?: string; value?: string; children?: string },
  type?: 'tag'
): string
```

- 默认字段 `value` / `label` / `children`。
- `type === 'tag'` 时返回节点的 `tagType`，否则返回 `label`。
- 未命中返回 `'--'`（tag 模式返回 `''`）。
- `enumData` 必须是数组，否则返回 `'--'`。

### formatValue

```ts
formatValue(callValue: any): any
```

- 数组：有值 `join(' / ')`，空数组 `'--'`。
- 其它：`callValue ?? '--'`。
- 返回值可能是非字符串（对象/数字），模板里会隐式转字符串。

## 3. 时间与格式化

### getTimeState

```ts
getTimeState(): string
```

源码区间（template/tqmp/patrol）：

| 小时 | 文案 |
| --- | --- |
| 6–10 | 早上好 ⛅ |
| 10–14 | 中午好 🌞 |
| 14–18 | 下午好 🌞 |
| 18–24 | 晚上好 🌛 |
| 0–6 | 凌晨好 🌛 |

边界有重叠（10、14、18 同时命中两个条件），按 `if` 顺序先命中。supervision 版本是 6–11 / 11–14 / 14–18，项目间文案边界不一致，公共版本应显式定义区间（如 `[5,11)`、`[11,13)`、`[13,18)`、`[18,23)`）。

### getBrowserLang

```ts
getBrowserLang(): 'zh' | 'en'
```

- 读 `navigator.language`，兼容旧的 `navigator.browserLanguage`。
- `cn` / `zh` / `zh-cn` → `'zh'`，其余 `'en'`。
- 不支持繁体/多语言细分；需要完整语言标签用 `navigator.languages`。

### convertToChinaNum

```ts
convertToChinaNum(num: number): string
```

- 0 → `'零'`；`NaN` / 空值 → `'零'`。
- 支持到「亿」位，超过 16 位会错位。
- 不支持负数、小数。
- 连续零会合并，`一十` 开头会转 `十`。

### formatNumberUnit

```ts
formatNumberUnit(value: number): string
```

- `< 10000`：`n.toLocaleString()`（千分位随 locale 变化）。
- `< 1 亿`：`(n / 10000).toFixed(1) + '万'`。
- 其它：`(n / 1e8).toFixed(1) + '亿'`。
- 不做四舍五入策略配置，固定 1 位小数。

### formatFileSize

```ts
formatFileSize(size: number): string
```

| 范围 | 输出 |
| --- | --- |
| `< 1024` | `${size}B` |
| `< 1024²` | `(size/1024).toFixed(1) + 'KB'` |
| `< 1024³` | `(size/1024²).toFixed(1) + 'M'` |
| 其它 | `(size/1024³).toFixed(1) + 'G'` |

注意单位是 `M` / `G`，不是 `MB` / `GB`；`size` 为负数/NaN 没有处理。

### parseTimeToSeconds

```ts
parseTimeToSeconds(timeStr: string): number
```

- `hh:mm:ss` → `h*3600 + m*60 + s`
- `mm:ss` → `m*60 + s`
- `ss` → `s`
- 空串/非法段返回 `NaN`（源码没有校验）。

### parseSecondsToTimeString

```ts
parseSecondsToTimeString(seconds: number): string
```

- `< 1h` → `mm:ss`
- `>= 1h` → `hh:mm:ss`
- 使用 `Math.floor`，负数会得到奇怪结果，调用前夹到 0。

### formatDate

```ts
formatDate(fmt: string, date?: string | number | Date): string
```

源码来自 `app-core/comm.js`：

- 支持 `y+`、`M+`、`d+`、`h+`、`m+`、`s+`、`q+`、`S`。
- 字符串日期先 `replace(/-/g, '/')`，兼容旧 Safari。
- 空日期返回 `''`。
- 使用 `RegExp.$1`，多次调用/并发下可读性差；公共版本建议 dayjs。

### formatDuration

```ts
formatDuration(seconds: number): string
```

源码 `comm.formatTime`：

- `< 60` → `00:ss`
- `< 3600` → `mm:ss`
- `>= 3600` → `hh:mm:ss`
- `parseInt` 处理，负数未定义。

### timestampToDateString

```ts
timestampToDateString(timestamp: number): string
```

- 毫秒时间戳 → `YYYY-MM-DD`。
- 无效时间戳会输出 `NaN-NaN-NaN`，调用前校验。

## 4. 浏览器、DOM、设备与 URL

### addScript

```ts
addScript(url: string): Promise<boolean>
```

- 用 `encodeURIComponent(url)` 作为 script id。
- `loadedMap` 记录已加载；重复调用直接 `resolve(true)`。
- 加载失败 `reject(false)`，不是 `Error`。
- 没有移除 script、没有超时；刷新后 `loadedMap` 清空。

### copyToClipboard

源码签名：

```ts
copyToClipboard(text: string, showMessage = true): void
```

- 优先 `navigator.clipboard.writeText`，否则创建 textarea + `execCommand('copy')`。
- 成功直接 `ElMessage.success('复制成功')`；Clipboard API 的 Promise 没有 await，失败也会提示成功。

推荐签名：

```ts
copyToClipboard(text: string): Promise<boolean>
```

由调用方决定提示文案。

### changeDomPosition

```ts
changeDomPosition(a: Element, b: Element): void
```

- 交换两个节点的位置，处理相邻、包含、普通三种情况。
- 直接操作真实 DOM；框架管理的节点可能被后续 diff 覆盖。
- 参数为 `null` 或非节点会抛错。

### blurActiveElement

```ts
blurActiveElement(): void
```

- 让当前 `document.activeElement` 失焦，触发原生 `change`。
- 用于移动端输入框 + 分页按钮场景；`document` 不存在时直接返回。

### waitUntilSuccess

```ts
waitUntilSuccess(fn: () => boolean, intervalMs: number, tries: number): void
```

- `fn()` 为真立即返回；否则 `setTimeout` 递归，`tries` 减到 0 停止。
- 没有返回值、没有取消、没有超时错误；`tries` 为 0 时不执行 `fn`（源码先判断 `fn()` 再判断 `tries`，实际会执行一次）。

### submitHiddenForm

```ts
submitHiddenForm(action: string, fields: Record<string, string | number | null | undefined>): void
```

- 创建隐藏 `<form method="POST">`，逐字段创建 hidden input，提交。
- 提交后浏览器导航；表单节点不会自动移除。
- 适合 SSO/导出等需要浏览器原生 POST 的场景。

### getQueryParam / getHashQueryParam

源码：

```ts
getQueryParam(name: string): string
getHashQueryParam(key: string): string | null
```

- `getQueryParam` 只分割 `?`，不处理 hash。
- `getHashQueryParam` 只解析 `#/path?tab=x` 的 query。
- 都返回字符串/`null`，不自动解码（`getHashQueryParam` 用 `URLSearchParams` 会解码）。
- 推荐统一用 `new URLSearchParams()`，并支持传入 `search` 便于测试。

### 设备判断

源码有两套：

1. `space` / `jy-player-ui`：`ismobilejs` + 模块顶层 `navigator` / `window.outerWidth`。
2. `tqmp`：UA 正则 + `?mobile=1` + iPadOS 触控点判断，函数式。

推荐公共签名（无依赖、函数式）：

```ts
isMobile(): boolean
isTablet(): boolean
isPhone(): boolean
isIOS(): boolean
isAndroid(): boolean
isWeChat(): boolean
getDeviceType(): 'mobile' | 'tablet' | 'desktop'
```

注意：

- 不在模块顶层读 `navigator`，否则 SSR 报错。
- `ismobilejs` 是可选依赖，不装也能用 UA 版本。
- iPadOS 13+ 桌面 UA 用 `Macintosh + maxTouchPoints > 1` 判断。
- `?mobile=1` 是测试/强制开关，公共版本不要默认带。

### getLocalCover

```ts
getLocalCover(file: File, width = '120', height = '70'): Promise<string>
```

- 音频或 `.mp3` / `.m4a` 返回 `''`。
- 创建隐藏 `<video>`，`crossOrigin=anonymous`，`muted`，`playsInline`。
- `loadeddata` 后 seek 到 `min(2, duration)`，`seeked` 时 canvas 截图。
- 5 秒超时返回 `''`；会 revoke URL、移除事件、清空 src。
- 返回 JPEG data URL，可直接作为封面预览。

## 5. ID、文件、哈希

### generateUUID

```ts
generateUUID(): string
```

源码有两种：

- `Math.random()` 逐位生成（patrol/space）。
- `crypto.getRandomValues` 模板替换（tqmp）。

推荐：

```ts
crypto.randomUUID?.() ?? fallbackUUID()
```

`Math.random` 版本不保证密码学随机，不要用于安全场景。

### md5Hash / generateFileMD5

```ts
md5Hash(str: string): string
generateFileMD5(file: File, callbacks?: { onProgress?; onSuccess?; onError? }, chunkSize?: number): void
```

- `md5Hash` = `SparkMD5.hash(str)`。
- 文件 MD5 有两个变体：
  - patrol：全量分片，接收原生 `File`。
  - resourcemanage：最多采样 5 个分片，接收 uploader 的 `file.file` 包装对象。
- 采样版本速度快，但不是完整文件哈希；秒传/去重可接受，完整性校验不要用。
- 公共版本建议返回 `Promise<string>`，并加 `sample` 选项。

### base64ToFile

```ts
base64ToFile(base64: string, filename: string): File
```

- 支持 `data:image/png;base64,xxx` 或纯 base64。
- 从 data URL 解析 MIME，缺省 `application/octet-stream`。
- `atob` 对非 Latin1/超长字符串有坑；大文件建议用 `fetch(dataUrl).blob()`。

## 6. 颜色

来源：`patrol/src/utils/theme/tool.ts`、`resourcemanage/app-core/color.ts`

```ts
hexToRgb(str: string): number[] | void
rgbToHex(r: number, g: number, b: number): string | void
getLightColor(color: string, level: number): string | void
getDarkColor(color: string, level: number): string | void
```

- 只接受 6 位 hex（`#RRGGBB` 或 `RRGGBB`）。
- 旧实现校验失败时调用 `ElMessage.warning(...)` 并返回 `undefined`；util 不应依赖 UI 库，公共版本应 `throw new Error` 或返回 `null`。
- `getLightColor` 与白色混合：`round(255*level + c*(1-level))`。
- `getDarkColor` 与 `20.5` 混合：`round(20.5*level + c*(1-level))`，不是纯黑。
- `rgbToHex` 不校验 0–255 上限，只校验 1–3 位数字；`256` 会生成非法 hex。
