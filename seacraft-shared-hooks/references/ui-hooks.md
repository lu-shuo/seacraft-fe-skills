# 弹窗、确认与下载

本文覆盖 `useDialog`、`useConfirm`、`useDownload`。三者都不依赖具体业务 store / API / 路由，属于通用 UI 能力；`useConfirm` / `useDownload` 只需要替换成功判定。

## 1. useDialog

### 用法

```vue
<template>
	<el-dialog v-model="dialogVisible" title="编辑" @closed="handleClose">
		<EditForm :id="currentId" @submit="handleClose" />
	</el-dialog>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { useDialog } from '@/hooks/useDialog';

const currentId = ref<number>();

const openCb = (row: { id: number }) => {
	currentId.value = row.id;
};

const { dialogVisible, showDialog, handleClose } = useDialog<{ id: number }>(openCb, () => {
	currentId.value = undefined;
});

// 父组件通过 ref 调 showDialog(row)
defineExpose({ showDialog });
</script>
```

### API

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `dialogVisible` | `Ref<boolean>` | 绑定 `el-dialog` 的 `v-model` |
| `showDialog` | `(...args: T[]) => void` | 先打开弹窗，再执行 `openCb(...args)` |
| `handleClose` | `() => void` | 先执行 `closeCb()`，再把 `dialogVisible` 置 `false` |

### 注意

- `useDialog<T>` 的 `T` 是单个参数类型；需要多参时用 `useDialog<[A, B]>` 不行（源码是 `T[]`），可改传对象。
- `openCb` 在 `dialogVisible = true` 之后执行；如果它依赖弹窗内 DOM，需要 `nextTick`。
- `closeCb` 抛错会导致弹窗不关闭。
- 弹窗关闭分「确定」和「取消」时，建议在 `closeCb` 里统一重置表单，不要只监听 `el-dialog` 的 `closed`。

### 提取为公共 hook

`useDialog` 已经是通用实现，复制时只需：

- 保留 `Ref` 返回。
- 显式 `import { ref } from 'vue'`（源码依赖自动导入）。
- 可选增加 `beforeOpen` / `afterClose` 异步钩子，但不要引入 UI 库。

## 2. useConfirm

### 用法

```ts
import { useConfirm } from '@/hooks/useConfirm';
import { deletePlanApi } from '@/api/modules/patrolPlan';

const handleDelete = async (id: number) => {
	try {
		const ok = await useConfirm(deletePlanApi, { id }, '删除后不可恢复，是否确定删除', '删除成功', '删除失败');
		if (ok) await refresh();
	} catch {
		// 业务失败已经弹过 ElMessage.error
	}
};
```

### 行为与返回

- 成功：`resolve(true)`，并弹 `ElMessage.success`。
- 业务失败（`res` 为空或 `status !== 200`）：弹 `ElMessage.error`，`reject(false)`。
- 用户取消：只执行 `onCancel`，**Promise 保持 pending**。
- `api` 抛异常：被 `.catch` 捕获，只执行 `onCancel`，同样 pending。

源码返回类型推断为 `Promise<unknown>`，不是 `Promise<boolean>`。

这意味着 `await useConfirm(...)` 在取消/异常时不会继续执行；如果后面有 `finally` 或 loading 复位，不会被触发。

### 提取为公共 hook 的推荐签名

把成功判定、提示实现、取消行为参数化：

```ts
export interface ConfirmOptions {
	message: string;
	title?: string;
	type?: 'success' | 'warning' | 'info' | 'error';
	successMessage?: string;
	errorMessage?: string;
	confirmButtonText?: string;
	cancelButtonText?: string;
	isSuccess?: (res: any) => boolean;
	onCancel?: () => void;
}

export function useConfirm<T>(
	api: (params?: any) => Promise<T>,
	params: any,
	options: ConfirmOptions
): Promise<boolean> {
	const {
		message, title = '提示', type = 'warning',
		successMessage = '操作成功', errorMessage = '操作失败',
		confirmButtonText = '确定', cancelButtonText = '取消',
		isSuccess = res => res?.status === 200,
		onCancel
	} = options;

	return new Promise<boolean>(resolve => {
		ElMessageBox.confirm(`${message}?`, title, { confirmButtonText, cancelButtonText, type })
			.then(async () => {
				try {
					const res = await api(params);
					if (isSuccess(res)) {
						ElMessage.success(successMessage);
						resolve(true);
					} else {
						ElMessage.error((res as any)?.message || errorMessage);
						resolve(false);
					}
				} catch (error) {
					ElMessage.error(errorMessage);
					resolve(false);
				}
			})
			.catch(() => {
				onCancel?.();
				resolve(false);
			});
	});
}
```

关键改进：

- 取消和异常都 `resolve(false)`，调用方可以可靠地 `if (ok)`。
- `isSuccess` 可替换，适配不同后端。
- 提示文案与按钮文案可配置。
- 不再用 `reject(false)` 表达业务失败，避免未捕获 rejection。

## 3. useDownload

### 用法

```ts
import { useDownload } from '@/hooks/useDownload';
import { exportTeacherFaceApi } from '@/api/modules/teachFace';

await useDownload(exportTeacherFaceApi, { taskId }, '.zip', '教师人脸批量导出数据', false);
```

### API

```ts
useDownload(
  api: (params: any) => Promise<any>,
  params: any = {},
  fileType: string = '.xlsx',
  fileName: string = '缺省文件名',
  isNotify: boolean = true,
  prompt: string = '如果数据庞大会导致下载缓慢哦，请您耐心等待'
): Promise<void>
```

### 实现要点

1. 可选弹 `ElNotification` 提示「数据量大，耐心等待」。
2. 调接口拿到二进制。
3. 用 `FileReader.readAsText` 尝试按文本读：
   - 能 `JSON.parse` 且 `status !== 200` → 说明后端返回错误 JSON，`ElMessage.error`，1 秒后关闭通知。
   - 能 `JSON.parse` 且 `status === 200` → 什么都不做（不会下载）。
   - 解析失败 → 按 Blob 下载。
4. 下载文件名 `${fileName}${fileType}`。

### 接入要求

- 接口必须返回二进制：Axios 用 `responseType: 'blob'`。
- 不要给 `fileName` 带扩展名，否则会变成 `a.xlsx.xlsx`。
- 下载成功后没有成功提示（源码里已注释），需要提示就在调用后自己弹。
- `ElNotification` 在非错误分支不会手动关闭，依赖 3 秒 `duration`。

### 提取为公共 hook 的推荐签名

```ts
interface DownloadOptions {
	fileType?: string;
	fileName?: string;
	notify?: boolean;
	prompt?: string;
	isSuccess?: (json: any) => boolean;
	getFileName?: (res: any) => string | undefined;
}

export async function useDownload(
	api: (params?: any) => Promise<Blob>,
	params: any = {},
	options: DownloadOptions = {}
): Promise<boolean>
```

- 返回值改为 `boolean`，调用方知道是否真的触发了下载。
- `getFileName` 允许从 `Content-Disposition` 解析文件名。
- `isSuccess` 适配不同后端。
- 用 `URL.createObjectURL` + `a.click()` 的现代实现即可，不必保留 `msSaveOrOpenBlob`。

## 4. 选择建议

| 需求 | 选择 |
| --- | --- |
| 普通弹窗显隐 | `useDialog` |
| 删除/提交前确认 + 调接口 + 提示 | `useConfirm`（建议按第 2 节改造） |
| 导出 Excel/ZIP 等二进制 | `useDownload`（接口需 `responseType: 'blob'`） |
