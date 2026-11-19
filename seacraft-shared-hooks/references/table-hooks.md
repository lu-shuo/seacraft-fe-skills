# 表格查询与多选

本文覆盖 `useTable`、`useSelection`、`useSelectionOrigin` 以及 `src/hooks/interface/index.ts` 里的表格类型。项目里 `ProTable` 是这三个 hook 的主要消费者。

## 1. useTable

### 典型用法

```ts
import { useTable } from '@/hooks/useTable';
import { getWarningListApi } from '@/api/modules/warningData';

const {
	tableData,
	loading,
	pageParams,
	searchParams,
	initSearchParams,
	finalParams,
	search,
	reset,
	getTableList,
	updateFinalParams,
	handleCurrentChange,
	handleSizeChange
} = useTable({
	api: getWarningListApi,
	initParams: { warningType: 'teaching' },
	apiDataKeyMap: {
		tableData: 'list',
		pageNo: 'pageNum',
		pageSize: 'pageSize',
		total: 'total'
	},
	filterEmptyParams: true,
	handleSearchParams: params => {
		if (params.range) {
			params.startTime = params.range[0];
			params.endTime = params.range[1];
			delete params.range;
		}
	},
	initOnMount: true,
	initOnActivated: false
});
```

### 选项

| 选项 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `api` | `(params) => Promise<any>` | 必填 | 列表接口 |
| `apiDataKeyMap` | `Table.ApiDataKeyMap` | `list / currPage / pageSize / totalCount` | 返回数据字段映射 |
| `initParams` | `object` | `{}` | 固定查询参数；每次 `updateFinalParams` 都会合并 |
| `pageable` | `boolean` | `true` | 是否分页 |
| `pageParamsKeyMap` | `Table.PageParamsKeyMap` | `pageSize / pageNum` | 分页请求字段映射 |
| `filterEmptyParams` | `boolean` | `false` | 过滤 `undefined` / `null` / `NaN`；保留 `false` / `0` / `''` |
| `handleSearchParams` | `(params) => void` | — | 请求前改写查询参数（原地修改） |
| `handlePageParams` | `(pageParam) => void` | — | 请求前改写分页参数（原地修改） |
| `dataCallback` | `(data) => any` | — | 加工 `tableData` |
| `initOnMount` | `boolean` | `true` | 挂载时 `reset()` |
| `initOnActivated` | `boolean` | `true` | KeepAlive 激活时 `reset(false)` |

### 返回

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `loading` | `Ref<boolean>` | 请求中 |
| `tableData` | `Ref<T[]>` | 表格数据 |
| `pageParams` | `Ref<{ pageNo; pageSize; total }>` | 分页状态 |
| `initSearchParams` | `Ref<object>` | 查询默认值；`reset` 回填它 |
| `searchParams` | `Ref<object>` | 查询表单绑定值 |
| `finalParams` | `Ref<object>` | 最终请求参数 |
| `getTableList` | `() => Promise<void>` | 按当前参数请求 |
| `search` | `() => void` | 页码重置为 1 后请求 |
| `reset` | `(resetPageNo = true) => void` | 重置查询参数后请求 |
| `updateFinalParams` | `(newInitParams?) => void` | 重新拼装 `finalParams`，不发请求 |
| `handleCurrentChange` | `(pageNo) => void` | 切页并请求 |
| `handleSizeChange` | `(pageSize) => void` | 改每页条数并请求 |

### 请求体

```ts
{
  ...searchParams,
  ...initParams,
  ...(newInitParams ?? {}),
  pageParam: {
    pageNum: 1,
    pageSize: 10
  }
}
```

- 分页是**嵌套**在 `pageParam` 下的；后端不是这个结构时，用 `handlePageParams` 改写，或改 hook。
- `getTableList` 每次都会重新 `Object.assign(state.finalParams, pageable ? pageParam : {})`，重复调用会保留旧键。

### 成功判定

```ts
if (res.status === ResponseStatus.SUCCESS) { ... } else { 清空数据 + 重置分页 }
```

- 成功：`tableData = pageable ? data[apiDataKeyMap.tableData] : data`，再走 `dataCallback`。
- 失败：`tableData = []`，分页重置为 `1 / 10 / 0`。
- 异常：`console.error`，`loading` 归位，数据保持原样。

### 生命周期

```ts
onMounted(() => { if (initOnMount) reset(); });
onActivated(() => { if (initOnActivated) reset(false); });
```

KeepAlive 首次挂载时 `onMounted` 与 `onActivated` 可能都触发，默认配置会请求两次。页面应按需关闭其中一个：

```ts
useTable({ api, initOnActivated: false }); // 只挂载时请求
```

### 常见坑

1. **不会自动重查**：修改 `initParams` / `searchParams` 后必须手动 `search()`；只调 `getTableList()` 不会重新合并 `searchParams`，需要先 `updateFinalParams()`。
2. **`reset` 不等于恢复 `initParams`**：它清空 `searchParams` 并回填 `initSearchParams`；`initParams` 在下一次 `updateFinalParams` 时才合并。
3. **`initSearchParams` 默认为空**：需要查询表单默认值时，挂载后手动 `Object.assign(initSearchParams.value, defaults)`。
4. **改每页条数不重置页码**：`handleSizeChange` 只改 `pageSize`；产品要求回到第一页时在调用处补 `pageParams.value.pageNo = 1`。
5. **并发无取消**：快速切页/搜索时旧请求可能后返回覆盖新数据；需要时加 `AbortController` 或请求序号。
6. **错误被吞**：失败只清空数据，不弹提示；提示依赖请求层拦截器或页面自己处理。
7. **`finalParams` 是响应式对象**：直接传给接口即可，但不要在请求后再改它。

### 提取为公共 hook

参数化点：

```ts
interface UseTableOptions<T, R> {
	api: (params: any) => Promise<R>;
	isSuccess?: (res: R) => boolean;
	getTableData?: (res: R) => T[];
	getPageInfo?: (res: R) => { pageNo: number; pageSize: number; total: number };
	pageParamKey?: string; // 默认 'pageParam'
	...
}
```

至少把 `ResponseStatus` 和字段映射改为可配置；否则换项目就得改源码。

## 2. useSelection

### 适用

Element Plus `el-table` 多选，含「跨页全选 + 排除项」能力。

### 用法（非跨页）

```ts
import { ref } from 'vue';
import type { ElTable } from 'element-plus';
import { useSelection } from '@/hooks/useSelection';

const tableRef = ref<InstanceType<typeof ElTable>>();
const {
	isSelected, selectedList, selectedIds,
	getRowKey, handleSelectionChange,
	toggleRowsSelection, toggleSelectAllCurrentPage, clearSelection
} = useSelection('id', false, tableRef);
```

```vue
<el-table ref="tableRef" :data="tableData" row-key="id" @selection-change="handleSelectionChange">
	<el-table-column type="selection" width="48" />
	...
</el-table>
```

### 用法（跨页）

```ts
const {
	selectedIds, excludeSelectedIds, crossPageSelectActive,
	toggleSelectAllCrossPage, toggleSelectAllCurrentPage, clearSelection
} = useSelection('id', true);

const getRequestSelection = () =>
	crossPageSelectActive.value
		? { all: true, excludeIds: excludeSelectedIds.value }
		: { ids: selectedIds.value };
```

### API

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `isSelected` | `ComputedRef<boolean>` | `selectedList.length > 0` |
| `selectedList` | `Ref<any[]>` | 已选行 |
| `selectedIds` | `ComputedRef<any[]>` | 已选行 key |
| `crossPageSelectActive` | `Ref<boolean>` | 是否跨页全选 |
| `excludeSelectedList` | `Ref<any[]>` | 跨页全选下排除的行 |
| `excludeSelectedIds` | `ComputedRef<any[]>` | 排除行的 key |
| `getRowKey` | `(row) => any` | 行 key |
| `handleSelectionChange` | `(val) => void` | 绑定表格事件；跨页模式下忽略 |
| `toggleRowsSelection` | `(rows, selected?) => void` | 切换行选中；非跨页模式需 `tableRef` |
| `toggleSelectAllCurrentPage` | `(tableData) => void` | 当前页全选/取消 |
| `selectAllCurrentPage` | `(tableData) => void` | 清空跨页状态后选当前页 |
| `toggleSelectAllCrossPage` | `(tableData) => void` | 进入/退出跨页全选 |
| `clearSelection` | `() => void` | 清空选择；非跨页模式需 `tableRef` |

### 跨页语义

- `crossPageSelectActive = true` 表示「全部数据都选中，除了 excludeSelectedList」。
- 此时 `selectedList` 只保存用户主动选中的行，不能直接当全量结果用。
- 请求后端时传 `{ all: true, excludeIds }`；非跨页传 `{ ids }`。
- `toggleSelectAllCrossPage` 第二次调用会 `clearSelection()` 退出全选。

### 常见坑

1. **`enableCrossPageSelect` 没有默认值**：源码必传，漏传相当于 `false`。
2. **`tableRef` 可选但非跨页方法需要**：`clearSelection` / `toggleRowsSelection` 在非跨页模式直接访问 `tableRef.value!`，不传会报错。
3. **`handleSelectionChange` 在跨页模式直接 return**：不要指望它维护 `selectedList`。
4. **`isSelected` 只看 `selectedList.length`**：跨页全选且 `selectedList` 为空时可能为 `false`，判断「是否有选择」应结合 `crossPageSelectActive` / `excludeSelectedIds`。
5. **行必须有稳定 key**：`rowKey` 与 `el-table` 的 `row-key` 保持一致。
6. **切换数据源要清空**：换查询条件/换页后，按业务决定是否 `clearSelection()`。

### 提取为公共 hook

- 把 `ElTable` 实例类型改为最小接口：

```ts
interface TableSelectionApi {
	clearSelection(): void;
	toggleRowSelection(row: any, selected?: boolean): void;
	toggleAllSelection(): void;
}
```

- 把「全选协议」与 UI 状态分离，返回 `getSelectionPayload()`，避免每个页面自己拼 `{ all, excludeIds }`。

## 3. useSelectionOrigin

旧版多选，功能更简单：

```ts
import { useSelection } from '@/hooks/useSelectionOrigin';

const { isSelected, selectedList, selectedIds, handleSelectionChange, toggleRowsSelection, toggleAllSelection } =
	useSelection('id', tableRef);
```

| 差异 | `useSelection` | `useSelectionOrigin` |
| --- | --- | --- |
| 导出名 | `useSelection` | `useSelection`（同名） |
| 跨页 | 支持 | 不支持 |
| `isSelected` | `ComputedRef<boolean>` | `Ref<boolean>` |
| 清空 | `clearSelection()` | `toggleRowsSelection()` 不传 rows 时清空 |
| 全选 | 当前页/跨页两套 | `toggleAllSelection()` 调表格原生 |

新代码优先 `useSelection`；维护旧页面时注意 import 路径，不要因为同名拿错实现。

## 4. 类型定义

```ts
// src/hooks/interface/index.ts
export namespace Table {
	export interface ApiDataKeyMap {
		tableData: string;
		pageNo: string;
		pageSize: string;
		total: string;
	}
	export interface PageParamsKeyMap {
		pageNo: string;
		pageSize: string;
	}
	export interface UseTableParams { ... }
	export interface PageParams {
		pageNo: number;
		pageSize: number;
		total: number;
	}
	export interface TableStateProps<T = any> { ... }
}
```

提取到公共包时，把这些类型一并导出，避免调用方用 `any`。

## 5. 与 ProTable 的关系

项目里的 `ProTable` 已经封装了：

- `useTable` 的查询/分页/重置；
- `useSelection` 的多选与跨页；
- 查询表单、列设置、分页组件；
- `selectedIds` / `excludeSelectedIds` / `crossPageSelectActive` 通过组件 ref 暴露。

页面优先用 `ProTable`，不要重复接 `useTable` + `useSelection`；只有在 `ProTable` 不满足布局或交互时才直接使用 hook，并参考它的消费方式（`src/components/ProTable/index.vue`）。
