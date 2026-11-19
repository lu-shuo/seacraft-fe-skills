---
name: seacraft-shared-hooks
description: "Company shared Vue 3 hooks limited to genuinely reusable composables abstracted from jy-digital-patrol-ui/src/hooks: dialog, confirm, download, table pagination, table selection, inline CSS variables, and ECharts. Use when adding or reusing a generic hook, wiring dependencies (Element Plus, ECharts, VueUse), or extracting a project hook into a reusable package. 中文触发：公共 hooks、组合式函数、弹窗、确认、下载、表格查询、分页、表格多选、跨页选择、CSS 变量、ECharts、useDialog、useConfirm、useTable、useSelection、useEcharts。"
license: MIT
metadata:
  author: seacraft
  version: "0.2.0"
---

# Seacraft 公共 Hooks 使用规范

公共 hooks 是 Vue 3 composables，统一放在 **L4 基础层**（`src/hooks/` 或 `packages/hooks/`）。本 skill 只收录**真正通用**的 hook：不依赖某个业务仓库的 store、API、路由、socket 实现、全局配置或业务 id 约定。

依赖仅限 Vue 与通用库：Element Plus、ECharts、VueUse。业务耦合通过参数或适配器注入，不写死在 hook 里。

## 保留标准

一个 hook 要进入本 skill，必须同时满足：

1. **无业务单例**：不 import `@/stores/*`、`@/api/*`、`@/utils/global`、业务 socket 工具或具体路由。
2. **无业务协议**：成功判定、接口字段、分页结构、暗色来源都可以通过参数替换。
3. **无业务 id 约定**：不假设 `'1'` / `'2'` 之类的窗口编号、角色或固定路径。
4. **可独立测试**：不启动完整业务应用也能覆盖主要分支。

按此标准，以下 6 个 hook **已从本 skill 移除**，只在原业务仓使用：

| 移除的 hook | 原因 |
| --- | --- |
| `useDarkMode` | 依赖 `useRoute()` 的 `/dashboard` 规则与 app store |
| `useTheme` | 依赖项目主题 token、app store、`@/styles/theme/*` |
| `useWatermark` | 依赖项目水印 API、user store、全局配置 |
| `useStompSocket` | 依赖项目 socket 工具、user store、全局 `window.socketMap` |
| `useRecordPlayerState` | 依赖播放器 domId 约定，`'1'` / `'2'` 写死 |
| `useGlobalVar` | 依赖运行时 `globalProperties`，setup 外不可用，且不是可复用状态 |

需要这些能力时，在业务仓按 [`extraction-guide.md`](references/extraction-guide.md) 的适配思路重写，不要把业务实现伪装成公共 hook。

## 保留的 Hook

| Hook | 作用 | 主要返回 | 通用依赖 | 复用级别 | 业务引用 |
| --- | --- | --- | --- | --- | --- |
| `useDialog` | 弹窗显隐 + 打开/关闭回调 | `dialogVisible`、`showDialog`、`handleClose` | 无 | A 直接复用 | 19 处 |
| `useConfirm` | 确认框 + 调接口 + 成功/失败提示 | `Promise<unknown>`（成功 `true`） | Element Plus | B 需适配成功判定 | 2 处 |
| `useDownload` | 调导出接口、识别错误 JSON、Blob 下载 | `Promise<void>` | Element Plus、Blob/FileReader | B 需适配成功判定 | 5 处 |
| `useTable` | 表格查询、分页、重置、加载态 | 状态 refs + 查询方法 | 接口返回 `{ status, data }` | B 需适配成功判定/字段 | 1 处（ProTable） |
| `useSelection` | 表格多选，含跨页全选 | 选中列表/ID、操作方法 | Element Plus Table ref | B 需确认跨页协议 | 1 处（ProTable） |
| `useSelectionOrigin` | 旧版表格多选 | 选中状态/列表/ID | Element Plus Table ref | A 直接复用（legacy） | 1 处 |
| `useInlineCssVars` | 主题对象 → CSS 变量 | `cssVars` | 无（需注入 `isDark`） | B 需注入暗色来源 | 55 处 |
| `useEcharts` | ECharts 初始化 + resize 监听 | 无 | ECharts、VueUse | A 直接复用 | 4 处 |

「业务引用」是 `src/` 内除 `hooks/` 自身外的 import 数量，仅用于判断真实使用度。

## 使用原则

1. **只复用通用 hook**。业务专用的 `useTheme` / `useWatermark` / `useStompSocket` / `useRecordPlayerState` 不要跨项目复制，按需重写。
2. **成功判定参数化**。`useConfirm` / `useDownload` / `useTable` 源码写死了 `status === 200`；跨项目使用时提供 `isSuccess`。
3. **业务字段参数化**。`useTable` 的 `apiDataKeyMap`、`pageParamsKeyMap`、`pageParam` 结构都可通过选项或适配器替换。
4. **依赖注入代替全局读取**。暗色来源、接口、用户标识、路由规则都从参数传入。
5. **输入可响应**。可能变化的输入用 `MaybeRefOrGetter<T>` + `toValue()`；不要强制调用方先转 ref。
6. **输出用 ref / computed**。返回普通对象会丢失响应性；`reactive` 返回值避免解构。
7. **副作用可清理**。`window` 监听、定时器、`ResizeObserver`、ECharts 实例都要在卸载时释放。
8. **生命周期只触发一次**。`onMounted` + `onActivated` 在 KeepAlive 首挂载时可能都执行；用开关或去重。
9. **UI 依赖显式化**。`ElMessage` / `ElMessageBox` / `ElNotification` 要么显式 import，要么明确要求自动导入。
10. **公共 hook 要测试**。覆盖成功、失败、取消、清理分支；见 [`extraction-guide.md`](references/extraction-guide.md)。

## 依赖速查

| 分组 | 运行依赖 | 初始化 / 构建要点 |
| --- | --- | --- |
| 弹窗 / 确认 / 下载 | `vue`、`element-plus` | `ElMessage` / `ElMessageBox` / `ElNotification` 显式 import 或 Element Plus 自动导入 |
| 表格 / 多选 | `vue`、`element-plus` | `ElTable` 实例 ref；行需要稳定 `rowKey` |
| CSS 变量 | `vue` | 注入 `isDark`；`cssVars` 绑定到容器 `:style` |
| 图表 | `vue`、`echarts`、`@vueuse/core` | 显式 import `useDebounceFn`；按需引入 ECharts 可减小包体 |

完整依赖、测试与打包配置见 [`extraction-guide.md`](references/extraction-guide.md)。

## 按场景读取参考文档

- 8 个 hook 的签名、返回、依赖、行为与用法：[`hook-catalog.md`](references/hook-catalog.md)
- 弹窗、确认、下载：[`ui-hooks.md`](references/ui-hooks.md)
- 表格查询、分页、多选、跨页选择：[`table-hooks.md`](references/table-hooks.md)
- 主题对象转 CSS 变量：[`inline-css-vars.md`](references/inline-css-vars.md)
- ECharts 初始化与 resize：[`echarts-hook.md`](references/echarts-hook.md)
- 抽象为公共 hook（依赖注入、类型、测试、打包）：[`extraction-guide.md`](references/extraction-guide.md)

## 高频陷阱

1. **`useConfirm` 取消或接口异常后 Promise 永远 pending**：内部 `.catch` 只执行 `onCancel`，没有 `resolve/reject`；公共版本应 `resolve(false)`。
2. **`useTable` 首挂载可能请求两次**：`onMounted` 与 `onActivated` 默认都开启，KeepAlive 首次激活会同时触发。
3. **`useTable` 不会自动重查**：改 `searchParams` 后要先 `updateFinalParams()` 再 `getTableList()`，或直接 `search()`。
4. **`useTable` 分页参数是嵌套的**：请求体是 `{ ...查询参数, pageParam: { pageNum, pageSize } }`，不是扁平字段。
5. **`useTable` 失败会重置分页**：`status !== 200` 时清空数据并重置为 `1 / 10 / 0`。
6. **`useSelection` 的 `tableRef` 可为空但某些方法必填**：非跨页模式 `clearSelection` / `toggleRowsSelection` 直接访问 `tableRef.value`。
7. **两个文件都导出 `useSelection`**：`useSelection.ts` 与 `useSelectionOrigin.ts` 同名导出，import 路径必须核对。
8. **两个 `isSelected` 语义不同**：新版是 `ComputedRef<boolean>`（选中数量 > 0），旧版是 `Ref<boolean>`（由表格事件维护）。
9. **`useInlineCssVars` 变量名保留 camelCase**：`bgColor` → `--scope-bgColor`，不是 `bg-color`。
10. **`useInlineCssVars` 数组不展开**：数组作为值写入变量，CSS 通常无效；只传标量。
11. **`useDownload` 依赖 `responseType: 'blob'`**：返回 JSON 会被当错误提示；文件名会追加 `fileType`。
12. **`useEcharts` 不负责 `dispose`**：只做 `setOption` 与 resize 监听；卸载时要自己 `chart.dispose()`。
13. **`useEcharts` 只监听 `window.resize`**：侧边栏折叠等容器尺寸变化不会触发，需要 `ResizeObserver`。
14. **自动导入依赖**：`ElMessage`、`ElNotification`、`computed`、`useDebounceFn` 在源码里靠自动导入；抽成包必须显式 import。
15. **`useDialog` 的 `closeCb` 抛错会导致弹窗不关闭**：`handleClose` 先执行回调再置 `false`。

## 文件落位

- 公共 hook：L4，`src/hooks/useXxx.ts`；跨项目复用抽到 `packages/hooks/src/useXxx.ts`。
- 类型：与 hook 同目录 `interface/index.ts`，或 `packages/hooks/src/types.ts`。
- 测试：按仓库约定放根级 `tests/`，命名 `<hook>.test.ts`。
- 页面/组件只调用 hook，不复制 hook 内部逻辑。

## 完成前检查

- [ ] 只使用了本 skill 保留的 8 个通用 hook；业务 hook 没有跨项目复制。
- [ ] 成功判定、字段映射、暗色来源已通过参数替换。
- [ ] 输入输出有类型，响应式输入支持 `MaybeRefOrGetter`。
- [ ] 取消、异常、空数据分支都有确定结果，不会让 Promise 永久 pending。
- [ ] `onMounted` / `onActivated` 不会重复初始化。
- [ ] 监听、定时器、observer、ECharts 实例已清理。
- [ ] 关键分支有单元测试。
- [ ] 文档写清依赖、自动导入要求与适配点。

## 组件实现后格式化检查

- 组件/hook 实现完成后，检查项目中是否已有 `Prettier`、`ESLint` 和 `Stylelint` 配置。
- 如果存在，按项目现有规则格式化本次修改的文件，并修复格式化、lint 和 stylelint 相关错误。
