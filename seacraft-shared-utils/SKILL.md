---
name: seacraft-shared-utils
description: "Company shared frontend utility library abstracted from company Vue admin projects (jy-template-ui, jy-digital-patrol-ui, jy-application-space-ui, jy-application-supervision-ui-new, jy-application-tqmp-ui, jy-application-resourcemanage-ui). Covers type guards, tree/table helpers, time/date/number/file formatting, browser/DOM, device/URL detection, UUID/MD5, base64/file, and color conversion. Use when adding or extracting a plain utility or deduplicating repeated helpers. 中文触发：公共 utils、工具函数、类型判断、树扁平化、时间格式化、日期格式化、文件大小、浏览器工具、设备判断、UUID、MD5、base64、颜色转换。"
license: MIT
metadata:
  author: seacraft
  version: "0.1.0"
---

# Seacraft 公共 Utils 使用规范

公共 utils 是**无框架状态、无业务单例**的纯函数集合，放在 L4 基础层（`src/utils/` 或 `packages/utils/src/`）。页面、组件、store、hook 都可以依赖，但 utils 不得 import 页面、组件、路由、store 或 API model。

本 skill 从公司多个 Vue 后台项目提取：

| 项目 | 主要贡献 |
| --- | --- |
| `jy-template-ui` | 12 个基础函数（类型、树、表格显示、数字） |
| `jy-digital-patrol-ui` | 基础函数、颜色、脚本加载、UUID、`deepEqual` |
| `jy-application-supervision-ui-new` | 基础函数、颜色、脚本加载 |
| `jy-application-tqmp-ui` | 基础函数、时间换算、base64、hash query、设备检测 |
| `jy-application-space-ui` | 基础函数、时间换算、文件大小、剪贴板、DOM 交换、视频封面 |
| `jy-application-resourcemanage-ui` | 类型守卫、日期格式化、颜色、MD5、隐藏表单提交 |

## 收录标准

同时满足才收录：

1. **跨项目复用**：在 ≥2 个前端项目出现，或明显与业务无关且被多处使用。
2. **无业务依赖**：不 import API model、store、router、权限、主题或具体业务常量。
3. **无 UI 依赖**：不直接 import Element Plus 等组件库；需要提示时通过回调或返回值交给调用方。
4. **无 import 副作用**：模块加载时不访问 `window` / `navigator` / `document`；确实需要时用函数包起来。
5. **可独立测试**：不启动完整应用即可覆盖主要分支。

## 已排除的业务 utils

| 排除项 | 原因 |
| --- | --- |
| `getShowMenuList`、`getAllBreadcrumbList`、`filterMenuByType/Signs`、`collectSigns`、`getAuthMenu*`、`getResourceField` | 依赖菜单/权限模型与 `Menu.MenuOptions` |
| `saveColumnConfig` / `loadColumnConfig` / `applyColumnConfig` | 依赖 ProTable 列配置协议 |
| `formatViewCount` | 「单视角/双视角」业务语义 |
| `imageProxy`、`appInfo`、`global`、`errorHandler`、`logger` | 项目环境、配置、日志体系 |
| `mittBus`、`stompSocket`、`eventBus` | 项目事件/长连接实现 |
| `query`（加密 query）、`url-auth`、`deviceRedirect` | 项目鉴权/跳转协议 |
| `elementPopperZoomPatch` | 绑定 Element Plus 内部 props 与版本 |
| `debounce`（带全局 key 的定时器 Map） | 不是通用 debounce；优先 `@vueuse/core` / `lodash-es` |
| `deepEqual`（patrol/supervision 版本） | 不处理数组、Date、Map/Set、循环引用，见第 6 节 |

## Utils 总表

### 类型与值

| 函数 | 作用 | 来源 |
| --- | --- | --- |
| `is(val, type)` | `Object.prototype.toString` 类型判断 | 5 个项目 |
| `isFunction` / `isDef` / `isUnDef` / `isObject` / `isDate` / `isNumber` | 常用类型守卫 | 5 个项目 |
| `isAsyncFunction` / `isPromise` / `isString` / `isBoolean` / `isArray` | 常用类型守卫 | 5 个项目 |
| `isClient` / `isWindow` / `isElement` / `isNull` / `isNullOrUnDef` / `isHexColor` | 环境与值判断 | 5 个项目 |
| `getType(val)` | 返回小写类型名（`null`、`array`、`date`…） | 5 个项目 |

### 树、对象与表格显示

| 函数 | 作用 | 来源 |
| --- | --- | --- |
| `getFlatArr(tree, parent?)` | 树扁平化为一维数组 | 5 个项目 |
| `findItemNested(list, value, valueKey?, childrenKey?)` | 递归查找第一个匹配节点 | 4 个项目 |
| `handleProp(prop)` | 多级 prop 取最后一段 | 4 个项目 |
| `handleRowAccordingToProp(row, prop)` | 按 `a.b.c` 取行值，空值回退 `'--'` | 4 个项目 |
| `filterEnum(value, enumData, fieldNames?, type?)` | 枚举/字典回显 label 或 tagType | 4 个项目 |
| `formatValue(value)` | 数组用 ` / ` 拼接，空值回退 `'--'` | 4 个项目 |

### 时间与格式化

| 函数 | 作用 | 来源 |
| --- | --- | --- |
| `getTimeState()` | 按小时返回「早上好/中午好…」 | 5 个项目 |
| `getBrowserLang()` | `zh` / `en` | 5 个项目 |
| `convertToChinaNum(num)` | 数字转中文（支持到亿） | 4 个项目 |
| `formatNumberUnit(value)` | 万/亿单位 | patrol |
| `formatFileSize(bytes)` | B / KB / M / G | space |
| `parseTimeToSeconds(str)` | `hh:mm:ss` / `mm:ss` / 秒 → 秒 | space、resourcemanage |
| `parseSecondsToTimeString(seconds)` | 秒 → `mm:ss` 或 `hh:mm:ss` | space、resourcemanage、tqmp |
| `formatDate(fmt, date)` | 日期格式化（`yyyy-MM-dd HH:mm:ss`） | resourcemanage |
| `formatDuration(seconds)` | 秒 → `mm:ss` / `hh:mm:ss` | resourcemanage |
| `timestampToDateString(ms)` | 毫秒时间戳 → `YYYY-MM-DD` | resourcemanage |

### 浏览器、DOM、设备与 URL

| 函数 | 作用 | 来源 |
| --- | --- | --- |
| `addScript(url)` | 动态加载脚本并去重 | 3 个项目 |
| `copyToClipboard(text, onSuccess?, onError?)` | Clipboard API + textarea 兜底 | space |
| `changeDomPosition(a, b)` | 交换两个 DOM 节点位置 | space、resourcemanage |
| `blurActiveElement()` | 让当前焦点元素失焦，触发 change | tqmp |
| `waitUntilSuccess(fn, interval, tries)` | 轮询直到 `fn()` 为真 | space、tqmp |
| `submitHiddenForm(action, fields)` | 动态隐藏表单 POST 跳转 | space |
| `getQueryParam(name)` / `getHashQueryParam(key)` | 普通 query / hash query | resourcemanage、tqmp |
| `isMobile` / `isTablet` / `isIOS` / `isAndroid` / `isWeChat` / `getDeviceType` | 设备判断 | space、tqmp、jy-player-ui |
| `getLocalCover(file, width?, height?)` | 视频首帧截图 | space |

### ID、文件、哈希与颜色

| 函数 | 作用 | 来源 |
| --- | --- | --- |
| `generateUUID()` | UUID v4 | 3 个项目 |
| `md5Hash(str)` | 字符串 MD5 | resourcemanage |
| `generateFileMD5(file, callbacks?, chunkSize?)` | 文件分段 MD5 | patrol、resourcemanage |
| `base64ToFile(base64, filename)` | base64 → `File` | tqmp |
| `hexToRgb` / `rgbToHex` | 颜色格式互转 | patrol、resourcemanage |
| `getLightColor(color, level)` / `getDarkColor(color, level)` | 调浅/调深 | patrol、resourcemanage |

## 使用原则

1. **纯函数优先**。输入决定输出，不读全局状态；需要环境信息时显式传参。
2. **不依赖 UI 库**。`copyToClipboard` 返回成功/失败或接受回调，不在 util 里 `ElMessage.success`。
3. **浏览器 API 要防御**。SSR/Node 下 `navigator`、`window`、`document`、`FileReader`、`URL` 都可能不存在。
4. **模块加载无副作用**。设备判断写成函数，不在模块顶层读 `navigator.userAgent`。
5. **类型明确**。输入 `unknown` + 类型守卫，或具体类型；不要到处 `any`。
6. **不可变优先**。树扁平化、`changeDomPosition` 这类会改 DOM/对象的函数要在文档写明副作用。
7. **能原生就原生**。UUID 用 `crypto.randomUUID()`，深拷贝用 `structuredClone()`，相等比较用 `lodash-es/isEqual`。
8. **按需导出**。全部 named exports，保持 tree-shaking；不要 `export default { ... }` 大对象。
9. **不复制有缺陷的实现**。`deepEqual`、`formatDate` 的旧实现有边界问题，按本文建议修。
10. **有测试**。类型判断、时间换算、树查找、颜色转换、设备判断至少各一组用例。

## 依赖速查

| 范围 | 依赖 | 说明 |
| --- | --- | --- |
| 核心类型/树/格式化 | 无 | 纯 TS/JS，可直接复制 |
| 文件 MD5 | `spark-md5`（可选） | 不装就不导出 `md5Hash` / `generateFileMD5`，或放子路径 `./hash` |
| 日期格式化 | 无 / `dayjs`（推荐） | 旧 `formatDate` 无依赖；复杂时区/本地化用 dayjs |
| 设备判断 | 无（推荐 UA 函数）/ `ismobilejs`（可选） | 不建议在模块顶层调用 |
| 颜色 | 无 | 去掉 Element Plus 提示 |
| 剪贴板提示 | 无 | 通过 `onSuccess`/`onError` 或返回值 |

打包与测试配置见 [`extraction-guide.md`](references/extraction-guide.md)。

## 按场景读取参考文档

- 全部函数签名、来源、行为与注意事项：[`catalog.md`](references/catalog.md)
- 类型守卫、树/对象/表格显示：[`type-tree-table.md`](references/type-tree-table.md)
- 时间、日期、数字、文件大小、中文数字：[`format.md`](references/format.md)
- 浏览器、DOM、设备、URL、文件、UUID/MD5：[`browser-file.md`](references/browser-file.md)
- 颜色转换与调色：[`color.md`](references/color.md)
- 去重、依赖治理、测试与 Vite 打包：[`extraction-guide.md`](references/extraction-guide.md)

## 高频陷阱

1. **`getType` 与 `is` 的返回值不同**：`getType([])` 是 `'array'`，`is([], 'Array')` 区分大小写，传 `'array'` 会失败。
2. **`getFlatArr` 用 `JSON.parse(JSON.stringify())` 深拷贝**：会丢 `undefined`、函数、`Date`、`Map/Set`，也会把循环引用直接抛错。
3. **`findItemNested` 没有循环保护**：树存在环或重复引用时会栈溢出；返回第一个匹配节点，不保证唯一。
4. **`handleRowAccordingToProp` 返回 `'--'` 且会覆盖局部变量**：只适合表格展示，不适合需要区分 `0` / `false` 的业务判断。
5. **`filterEnum` 依赖 `fieldNames` 和 `tagType` 约定**：字典结构不同要显式传 `value/label/children`。
6. **`deepEqual` 不要直接复用**：旧实现不比较数组内容、`Date`、`Map/Set`，也不处理循环引用；用 `lodash-es/isEqual` 或完整实现。
7. **`formatDate` 的 `-` 在旧 Safari 可能解析失败**：字符串日期先 `replace(/-/g, '/')`；新项目优先 dayjs。
8. **`formatNumberUnit` 的 `<1 万` 用 `toLocaleString()`**：不同 locale 的分隔符不同，测试断言不要写死逗号。
9. **`convertToChinaNum` 上限到亿**：超过 16 位会截断/错位；负数、小数没有定义。
10. **`formatFileSize` 单位是 `M/G` 不是 `MB/GB`**：UI 文案要求 MB 时自己补 B。
11. **`addScript` reject 的是 `false` 不是 `Error`**：调用方无法拿到具体原因；重复加载依赖 `loadedMap`，刷新即失效。
12. **`copyToClipboard` 的 Clipboard API 是异步的**：旧实现没有 await，失败时仍提示成功；公共版本应返回 `Promise<boolean>`。
13. **`changeDomPosition` 会移动真实 DOM**：Vue/React 管理的节点被移动后，框架的 diff 可能出错；只用于纯展示节点。
14. **`waitUntilSuccess` 没有返回值、没有取消**：无法 await 成功/失败；公共版本改为返回 `Promise<boolean>` + `AbortSignal`。
15. **`getQueryParam` 只解析 `?` 后的 query**：hash 路由的 `#/path?tab=x` 用 `getHashQueryParam`。
16. **设备判断在模块顶层读 `navigator`**：SSR 直接报错；必须包成函数。
17. **`generateUUID` 用 `Math.random`**：不保证密码学随机；用 `crypto.randomUUID()`，并为旧浏览器兜底。
18. **MD5 采样 vs 全量**：resourcemanage 版本只取最多 5 个分片，速度快但非全文件哈希；需要完整性校验时用全量。
19. **`hexToRgb` / `rgbToHex` 旧实现用 `ElMessage.warning`**：util 不应依赖组件库；应抛错或返回 `null`。
20. **`getLocalCover` 有 5 秒超时**：超时返回空字符串；记得 `URL.revokeObjectURL` 和移除 video 事件。

## 文件落位

- 公共 utils：L4，`src/utils/<domain>.ts`；跨项目复用抽到 `packages/utils/src/`。
- 按域拆文件：`is.ts`、`tree.ts`、`format.ts`、`browser.ts`、`device.ts`、`file.ts`、`color.ts`。
- 一个文件一个职责；不要继续把所有函数堆在 `index.ts`，也不要新增 `common.ts` / `misc.ts`。
- 测试：根级 `tests/`，命名 `<util>.test.ts`。

## 完成前检查

- [ ] 只收录跨项目通用、无业务/UI 依赖的函数。
- [ ] 浏览器 API 有 `typeof window !== 'undefined'` 等防御。
- [ ] 模块加载无副作用。
- [ ] 输入输出有类型，没有新增 `any`。
- [ ] 会改 DOM/入参的函数已注明副作用。
- [ ] 旧实现缺陷已修复（`deepEqual`、`copyToClipboard`、设备判断、颜色错误处理）。
- [ ] 有单元测试。
- [ ] 打包为 named exports，`sideEffects: false`。

## 组件实现后格式化检查

- 组件/utils 实现完成后，检查项目中是否已有 `Prettier`、`ESLint` 和 `Stylelint` 配置。
- 如果存在，按项目现有规则格式化本次修改的文件，并修复格式化、lint 和 stylelint 相关错误。
