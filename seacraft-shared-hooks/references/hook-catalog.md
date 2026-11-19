# Hook 目录与签名

> 来源：`jy-digital-patrol-ui/src/hooks`
> 收录范围：只保留不依赖业务 store / API / 路由 / socket / 业务 id 约定的通用 hook。
> 类型定义：`src/hooks/interface/index.ts`（`Confirm` / `Table`）

## 1. useDialog

```ts
import { useDialog } from '@/hooks/useDialog';

const { dialogVisible, showDialog, handleClose } = useDialog(openCb, closeCb);
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useDialog<T = any>(openCb?: (...args: any[]) => any, closeCb?: (...args: any[]) => any)` |
| 返回 | `dialogVisible: Ref<boolean>`、`showDialog: (...args: T[]) => void`、`handleClose: () => void` |
| 依赖 | 仅 `vue` |
| 行为 | `showDialog` 先置 `true` 再执行 `openCb(...args)`；`handleClose` 先执行 `closeCb()` 再置 `false` |
| 级别 | A，可直接复用 |

注意：`T` 是参数元组的元素类型，不是整个参数对象类型；`useDialog<{ id: number }>()` 之后 `showDialog({ id })` 合法。若 `closeCb` 抛错，`dialogVisible` 不会置 `false`。

## 2. useConfirm

```ts
import { useConfirm } from '@/hooks/useConfirm';

await useConfirm(deleteApi, { id }, '删除后不可恢复，是否确定删除', '删除成功', '删除失败');
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useConfirm<P = any, R = any>(api, params, message, successMessage?, errorMessage?, title?, type?, onCancel?)` |
| 返回 | `Promise<unknown>`：成功 `resolve(true)`；业务失败 `reject(false)`；取消/异常时 Promise 保持 pending |
| 依赖 | `element-plus`、`@/api/interface` 的 `ResponseStatus` |
| 级别 | B，需替换成功判定与错误提示策略 |

默认文案：`successMessage = '删除成功'`、`errorMessage = '删除失败'`、`title = '提示'`、`type = 'warning'`。`message` 会自动补 `?`。

行为：

1. `ElMessageBox.confirm` 确认。
2. 调 `api(params)`，返回空值 `reject(false)`。
3. `res.status === ResponseStatus.SUCCESS(200)` → `ElMessage.success` + `resolve(true)`。
4. 其它状态 → `ElMessage.error(res.message || res.result || errorMessage)` + `reject(false)`。
5. 取消或 `api` 抛错 → `.catch` 里只执行 `onCancel`，**不 resolve 也不 reject**。

业务用法：`if (await useConfirm(...)) { refresh(); }`；取消时不会继续，但 Promise 永久挂起，不要在需要确定性结果的流程里依赖它。

## 3. useDownload

```ts
import { useDownload } from '@/hooks/useDownload';

await useDownload(exportApi, { id }, '.xlsx', '导出结果', true);
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useDownload(api, params = {}, fileType = '.xlsx', fileName = '缺省文件名', isNotify = true, prompt = '如果数据庞大会导致下载缓慢哦，请您耐心等待')` |
| 返回 | `Promise<void>` |
| 依赖 | `element-plus`、`ResponseStatus`、浏览器 `Blob` / `FileReader` / `URL` |
| 级别 | B，需替换成功判定与下载实现 |

行为：

- `isNotify` 为真时先弹 `ElNotification`（3 秒）。
- `api(params)` 的返回值按 `Blob` 处理，用 `FileReader.readAsText` 尝试当 JSON 读。
- 能解析成 JSON 且 `status !== 200` → `ElMessage.error`，1 秒后关闭通知。
- 能解析成 JSON 且 `status === 200` → **不做任何事**（不下载）。
- 解析 JSON 失败 → `new Blob([res])` + `<a download>` 下载，文件名 `${fileName}${fileType}`。
- 兼容 IE/Edge 的 `navigator.msSaveOrOpenBlob`。

注意：

- 接口必须返回二进制（Axios 需 `responseType: 'blob'`）；返回 `ArrayBuffer` 时 `readAsText` 会失败。
- `fileName` 不要再带扩展名，否则会重复追加。
- 下载成功没有 toast（源码已注释）。

## 4. useTable

```ts
import { useTable } from '@/hooks/useTable';

const {
  tableData,
  pageParams,
  searchParams,
  initSearchParams,
  loading,
  finalParams,
  search,
  reset,
  getTableList,
  updateFinalParams,
  handleCurrentChange,
  handleSizeChange
} = useTable({
  api: getListApi,
  initParams: { clroId },
  filterEmptyParams: true
});
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useTable<T>(options: Table.UseTableParams)` |
| 返回 | `toRefs(state)` + `getTableList`、`search`、`reset`、`updateFinalParams`、`handleCurrentChange`、`handleSizeChange` |
| 依赖 | `vue`、接口返回 `{ status, data }`、`ResponseStatus` |
| 级别 | B，需参数化成功判定与接口字段 |

选项：

| 选项 | 默认 | 说明 |
| --- | --- | --- |
| `api` | 必填 | `(params) => Promise<{ status; data; message? }>` |
| `apiDataKeyMap` | `{ tableData:'list', pageNo:'currPage', pageSize:'pageSize', total:'totalCount' }` | 接口返回字段映射 |
| `initParams` | `{}` | 固定查询参数，每次查询都会合并 |
| `pageable` | `true` | 是否分页 |
| `pageParamsKeyMap` | `{ pageSize:'pageSize', pageNo:'pageNum' }` | 分页请求字段映射 |
| `filterEmptyParams` | `false` | 过滤 `undefined` / `null` / `NaN`；保留 `false` / `0` / `''` |
| `handleSearchParams` | — | 查询前改写查询参数 |
| `handlePageParams` | — | 查询前改写分页参数 |
| `dataCallback` | — | 加工表格数据 |
| `initOnMount` | `true` | `onMounted` 是否自动 `reset()` |
| `initOnActivated` | `true` | `onActivated` 是否自动 `reset(false)` |

返回状态：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `loading` | `Ref<boolean>` | 请求中 |
| `tableData` | `Ref<T[]>` | 表格数据 |
| `pageParams` | `Ref<{ pageNo; pageSize; total }>` | 分页状态 |
| `initSearchParams` | `Ref<object>` | 查询表单默认值；`reset` 会回填它 |
| `searchParams` | `Ref<object>` | 查询表单绑定值 |
| `finalParams` | `Ref<object>` | 最终请求参数（含 `pageParam`） |

请求体形状：

```ts
{
  ...queryParams,
  pageParam: {
    [pageParamsKeyMap.pageNo]: pageNo,
    [pageParamsKeyMap.pageSize]: pageSize
  }
}
```

失败分支：`status !== 200` 时清空 `tableData`，并把分页重置为 `{ pageNo: 1, pageSize: 10, total: 0 }`。

## 5. useSelection

```ts
import { useSelection } from '@/hooks/useSelection';

const {
  isSelected, selectedList, selectedIds,
  crossPageSelectActive, excludeSelectedList, excludeSelectedIds,
  getRowKey, handleSelectionChange,
  toggleRowsSelection, toggleSelectAllCurrentPage,
  selectAllCurrentPage, toggleSelectAllCrossPage, clearSelection
} = useSelection('id', true, tableRef);
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useSelection(rowKey: string \| ((row) => string), enableCrossPageSelect: boolean, tableRef?: any)` |
| 依赖 | `element-plus` 的 `ElTable` 实例方法（非跨页模式） |
| 级别 | B，跨页协议需与后端对齐 |

返回值：

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `isSelected` | `ComputedRef<boolean>` | `selectedList.length > 0` |
| `selectedList` | `Ref<any[]>` | 已选行 |
| `selectedIds` | `ComputedRef<any[]>` | 已选行 key |
| `crossPageSelectActive` | `Ref<boolean>` | 是否处于「跨页全选」模式 |
| `excludeSelectedList` | `Ref<any[]>` | 跨页全选下被排除的行 |
| `excludeSelectedIds` | `ComputedRef<any[]>` | 被排除行的 key |
| `getRowKey` | `(row) => any` | 取行 key |
| `handleSelectionChange` | `(val) => void` | 表格 `selection-change` 回调；跨页模式下直接 return |
| `toggleRowsSelection` | `(rows, selected?) => void` | 切换指定行；非跨页模式调用 `tableRef.toggleRowSelection` |
| `toggleSelectAllCurrentPage` | `(tableData) => void` | 当前页全选/取消 |
| `selectAllCurrentPage` | `(tableData) => void` | 清空跨页状态后选中当前页 |
| `toggleSelectAllCrossPage` | `(tableData) => void` | 进入/退出跨页全选 |
| `clearSelection` | `() => void` | 清空选择；非跨页模式调用 `tableRef.clearSelection()` |

跨页状态机：

```text
普通选择
  → toggleSelectAllCrossPage(tableData)
  → crossPageSelectActive = true, excludeSelectedList = []
  → 用户取消某行 → 加入 excludeSelectedList
  → 再次 toggleSelectAllCrossPage → clearSelection()
```

后端请求约定通常为：

```ts
crossPageSelectActive
  ? { all: true, excludeIds: excludeSelectedIds }
  : { ids: selectedIds }
```

本项目 `ProTable` 就是这么消费的，见 [`table-hooks.md`](table-hooks.md)。

## 6. useSelectionOrigin

```ts
import { useSelection } from '@/hooks/useSelectionOrigin';

const { isSelected, selectedList, selectedIds, handleSelectionChange } = useSelection('id', tableRef);
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useSelection(rowKey, tableRef?)` |
| 返回 | `isSelected: Ref<boolean>`、`selectedList: Ref<any[]>`、`selectedIds: ComputedRef`、`getRowKey`、`handleSelectionChange`、`toggleRowsSelection`、`toggleAllSelection` |
| 依赖 | Element Plus Table ref |
| 级别 | A，旧版；新代码优先 `useSelection` |

注意：该文件导出的函数名也是 `useSelection`，与新版同名；import 路径写错会拿到旧版（没有跨页能力）。

## 7. useInlineCssVars

```ts
import { useInlineCssVars } from '@/hooks/useInlineCssVars';

const { cssVars } = useInlineCssVars({
	scope: 'room-detail',
	light: { player: { bgColor: '#fff' } },
	dark: { player: { bgColor: '#000' } }
});
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useInlineCssVars({ light, dark, scope? })` |
| 返回 | `cssVars: ComputedRef<Record<string, string>>` |
| 依赖 | 源码依赖 `useDarkMode`；公共版本注入 `isDark` |
| 级别 | B，需注入暗色来源 |

规则：

- 嵌套对象用 `-` 展平：`player.bgColor` → `player-bgColor`（保留 camelCase）。
- `scope` 会加前缀：`--room-detail-player-bgColor`。
- 数组不展开，作为值写入变量（通常无效）。
- 只接受字符串（`ThemeObject` 类型），数字需要自己转字符串。

## 8. useEcharts

```ts
import { useEcharts } from '@/hooks/useEcharts';

useEcharts(chart, option);
```

| 项 | 内容 |
| --- | --- |
| 签名 | `useEcharts(myChart: echarts.ECharts, options: echarts.EChartsCoreOption)` |
| 返回 | `void` |
| 依赖 | `echarts`、`@vueuse/core` 的 `useDebounceFn`（源码未显式 import，靠自动导入） |
| 级别 | A，注意依赖与清理 |

行为：立即 `setOption(options)`；注册 `window.resize`（100ms debounce）；`onBeforeUnmount` 移除；`onActivated` 重新添加；`onDeactivated` 移除。

注意：不调用 `dispose`；KeepAlive 首挂载可能同时走 `onMounted` 与 `onActivated`，重复添加监听（浏览器对同一函数重复 `addEventListener` 会去重，影响较小）。
