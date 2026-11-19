# GlobalUploader（全局分片上传）

> 位置：`jy-application-resourcemanage-ui/app-index/components/GlobalUploader/`
> 性质：**业务仓全局上传器**，不随 npm 发布。基于 `vue-simple-uploader`（`simple-uploader.js`）封装分片上传、MD5 校验、秒传/断点续传、全局悬浮面板。
> 技术栈：Vue 3.2 + Element Plus 2.10 + Vue CLI 5 / Webpack；不是 Vite 工程。

## 1. 什么时候用它

- 大文件（视频、课件、文档）分片上传，需要暂停/恢复/重试/取消。
- 需要「秒传 + 断点续传」：上传前用文件指纹探测服务端已存在的分片。
- 页面不关心上传 UI，只想「选文件 → 自动传 → 成功后拿 `fileId` 落库」。
- 全站只挂一个上传面板，任意页面通过事件触发。

不适用：

- 小文件、图片直传、只需要 `el-upload` 的简单场景。
- 需要每个页面独立上传队列、独立 UI 的多实例场景（当前组件是单例，硬编码 id）。
- 非 Vue 3 工程。

## 2. 依赖与初始化

### 2.1 依赖

本项目实际依赖：

| 依赖 | 版本 | 作用 |
| --- | --- | --- |
| `vue` | `3.2.47` | 组件运行 |
| `element-plus` | `2.10.4` | 按钮、图标、`ElMessageBox`、`ElNotification` |
| `@element-plus/icons-vue` | 随 `element-plus` 安装 | `FullScreen`、`Minus`、`Close`、`SuccessFilled`、`CircleCloseFilled` |
| `vue-simple-uploader` | `^1.0.1`（实际解析 `1.0.3`） | 上传根组件与分片引擎（peer `vue >=3.1`） |
| `spark-md5` | `^3.0.2` | 文件指纹 |
| `uuid` | `^11.0.3` | 为每次上传生成唯一 ID |
| `mitt` | `^3.0.0` | `mittBus` 事件总线 |
| `sass` + `sass-loader` | `^1.32.7` / `^10.2.0` | 组件 SCSS 与 Element Plus 主题 |

安装：

```bash
pnpm add vue-simple-uploader spark-md5 uuid mitt element-plus
# pnpm 严格 node_modules 下，直接 import 传递依赖会解析失败；显式安装图标包
pnpm add @element-plus/icons-vue
pnpm add -D sass sass-loader
```

`@element-plus/icons-vue` 通常由 `element-plus` 间接安装并随其升级；用 pnpm 的严格依赖布局时仍建议显式安装，避免 `import '@element-plus/icons-vue'` 解析失败。

### 2.2 入口注册（Vue CLI / Webpack）

```js
// app-index/main.js
import uploader from 'vue-simple-uploader';
import 'vue-simple-uploader/dist/style.css';
import * as ElementPlusIconsVue from '@element-plus/icons-vue';

const app = createApp(App);

// 上传组件（uploader / uploader-btn / uploader-list / uploader-file）
app.use(uploader);

// 上传面板用的 Element Plus 图标
for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
	app.component(key, component);
}
```

要点：

- `vue-simple-uploader/dist/style.css` **必须全局引入**，否则上传列表网格、文件图标、进度条全部错位。
- 如果工程没有全量注册 Element Plus 图标，需要在 `GlobalUploader` 里显式注册/导入 `FullScreen`、`Minus`、`Close`、`SuccessFilled`、`CircleCloseFilled`。
- Element Plus 的按钮/弹窗组件可用全局注册、自动导入或局部导入；本项目用 `unplugin-vue-components` + `ElementPlusResolver`。

### 2.3 挂载全局实例

```vue
<!-- app-index/App.vue -->
<template>
	<el-config-provider :locale="zhCn">
		<router-view />
	</el-config-provider>
	<GlobalUploader />
</template>

<script setup>
import GlobalUploader from '@/components/GlobalUploader/index.vue';
</script>
```

- 只挂一次，放在 `App.vue` 根部、路由视图之外；面板才能跨页面持续上传。
- 不要放到某个页面内再配合 `global` 模式，否则切路由会卸载上传器、中断任务。

### 2.4 配套文件

| 文件 | 作用 |
| --- | --- |
| `app-index/utils/mittBus.ts` | 全局事件总线，定义 `OPEN_GLOBAL_UPLOADER` 等事件 |
| `app-index/utils/md5.js` | `generateMD5(file, callbacks)` 分片采样计算 MD5 |
| `app-core/auth.js` | `Auth.getJwtToken()` 取 JWT |
| `app-core/global.js` | `Global.getBaseUrl()` 拼接上传地址 |
| `GlobalUploader/useFileStatus.js` | 自定义状态文案（校验 MD5 / 合并中 / 转码中 / 失败） |
| `GlobalUploader/CustomFileListHeader.vue` | 列表表头 |
| `GlobalUploader/CustomFileListItem.vue` | 列表行（进度、状态、暂停/恢复/重试/取消） |
| `GlobalUploader/index.scss` | 面板样式、文件类型图标、按钮隐藏 |

`mittBus` 类型：

```ts
// app-index/utils/mittBus.ts
import mitt from 'mitt';

type Event = {
	OPEN_GLOBAL_UPLOADER: any;
	GLOBAL_UPLOADER_ADDED_FILE: void;
	GLOBAL_UPLOADER_SUCCESS: any;
	GLOBAL_UPLOADER_FAIL: void;
};

export default mitt<Event>();
```

## 3. 打开上传器

全局模式通过事件总线触发，**不要直接操作组件 ref**：

```ts
import mittBus from '@/utils/mittBus';
import Global from '@core/global';

mittBus.emit('OPEN_GLOBAL_UPLOADER', {
	// 每次分片请求都会带的业务参数
	params: {
		resourceId: 123,
		fileName: '第一讲.mp4',
		viewNum: 1
	},
	options: {
		// 必传；initOptions 里的 localhost 只是占位
		target: `${Global.getBaseUrl()}/v1/vod_upload/chunk`,
		// 可选：是否开启服务端分片校验（秒传/断点续传）
		testChunks: true,
		// 可选：单个文件上传成功后的回调，可 async
		successCb: async (file, res) => {
			console.log(file.params, res.data);
		},
		// 可选：文件选择框 accept，必须是数组
		accept: ['video/*']
	},
	// 可选：直接上传已选好的 File；为空则打开系统文件选择框
	files: []
});
```

### 3.1 `params`

- 会作为 `query` 合并进**每个分片 POST / 校验 GET** 请求。
- 同时以快照形式写入 `file.params`，`successCb` 里可读，用于把 `fileId` 与业务记录关联。
- 常用字段：`resourceId`、`fileName`、`viewNum`、`courseId` 等。
- 是「本次打开」的共享参数；一次 `OPEN` 里的多个文件共用同一份 `params`。

### 3.2 `options`

`customizeOptions` 实际只处理 4 个键：

| 键 | 类型 | 说明 |
| --- | --- | --- |
| `target` | `string` | 上传地址，直接改写 `uploaderIns.opts.target`；**必须传** |
| `testChunks` | `boolean` | 是否开启分片校验；关闭后退化为普通分片上传，失去秒传/续传 |
| `successCb` | `(file, res) => void \| Promise<void>` | 单文件成功回调，`res` 是服务端响应解析后的对象 |
| `accept` | `string[]` | 写入文件选择框的 `accept`；内部执行 `accept.join()`，**不能传字符串** |

其他上传器选项（`chunkSize`、`headers`、`maxChunkRetries` 等）在组件内 `initOptions` 固定，当前不通过 `options` 暴露；需要改就改组件源码。

### 3.3 `files`

- `files: []`：调用隐藏的 `uploader-btn` 打开文件选择框。
- `files: [File]`：直接 `uploaderIns.addFile(file)`，跳过文件选择，适合业务表格里已经选好文件、点「保存」时再统一上传。
- 传多个文件时它们共用同一份 `params`；需要每个文件不同参数（如 `viewNum`）时，按文件循环 `emit`，每次带自己的 `params` 和一个 `files: [file]`。

## 4. 事件与回调

组件内部 `trigger` 会同时做两件事：`mittBus.emit(name)` 和 `emits(name)`。全局挂载时父级就是 `App.vue`，业务页面通过 `mittBus.on` 监听。

| 事件 | 负载 | 触发时机 |
| --- | --- | --- |
| `GLOBAL_UPLOADER_ADDED_FILE` | 无 | 文件加入队列（MD5 校验前） |
| `GLOBAL_UPLOADER_SUCCESS` | 无 | 单个文件上传成功、`successCb` 执行完之后 |
| `GLOBAL_UPLOADER_FAIL` | 无 | 分片上传失败（`file-error`） |

注意：

- **事件不带业务数据**。需要 `fileId`、`identifier` 时用 `options.successCb`，不要依赖事件参数。
- `successCb` 在成功事件之前执行；如果它抛错，后续事件与 `successNum` 自增不会执行，且组件没有 catch。回调内部自行 `try/catch`。
- 成功事件按文件触发，多文件会触发多次；聚合落库请在 `successCb` 里做计数，或监听事件后自行去重。
- 页面卸载时记得 `mittBus.off('GLOBAL_UPLOADER_SUCCESS', handler)`。

## 5. Props（直接使用组件时）

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `global` | `boolean` | `true` | `true` 为右下角悬浮面板并隐藏选择按钮；`false` 为页面内模式（显示按钮、面板不固定） |
| `params` | `object` | — | 业务参数；**初始值不生效**（watch 非 immediate），需挂载后变更或走 `OPEN_GLOBAL_UPLOADER` |
| `options` | `object` | — | 同上；初始值不生效，且只支持第 3.2 节的 4 个键 |

结论：**统一走 `mittBus.emit('OPEN_GLOBAL_UPLOADER', ...)`**，不要依赖 props 初始值。

## 6. 上传流程

```text
OPEN_GLOBAL_UPLOADER
  → （files 为空）打开文件选择框 / （files 有值）addFile
  → file-added
  → panelShow = true，file.params = 当前 customParams
  → 暂停文件，计算 MD5（分片采样）
  → uniqueIdentifier = `${md5}@${uuid}`
  → resume() 开始分片上传
  → 每个分片：POST target（multipart，字段名 file）
  → testChunks=true 时：先 GET 探测已上传分片
  → fileSuccess：解析 JSON，data 为空则失败
  → successCb(file, res) → GLOBAL_UPLOADER_SUCCESS
```

细节：

- `auto-start="false"`：选完文件不会自动传，等 MD5 完成后 `file.resume()` 启动。
- `chunkSize` 10MB、`forceChunkSize: true`、`maxChunkRetries: 3`、`fileParameterName: 'file'`、`allowDuplicateUploads: true`。
- `testChunks: true` + `checkChunkUploadedByResponse` 实现秒传/续传（见第 7 节）。
- 暂停/恢复/重试/取消由 `uploader-file` 行内按钮调用 `file` 实例方法；「取消」会从列表移除。
- 关闭面板：`close()` 会 `uploaderIns.cancel()` 取消全部任务；存在未完成任务时先 `ElMessageBox.confirm`。
- `successNum` 只在面板关闭时清零；跨批次打开时标题的 `(成功数/文件数)` 可能包含上一批残留，直到手动关闭。

## 7. 后端协议契约

前端按以下结构解析，后端必须匹配（源码位置：`initOptions.checkChunkUploadedByResponse`、`onFileSuccess`）。

### 7.1 请求

- **地址**：`options.target`，本项目为 `${Global.getBaseUrl()}/v1/vod_upload/chunk`。
- **鉴权头**：`jwt-token: <Auth.getJwtToken()>`。
- **分片上传**：`POST`，`multipart/form-data`，文件字段名 `file`。
- **分片校验**：`testChunks: true` 时对每个分片发 `GET`（同 target），带相同 query 与鉴权头。
- **query 标准字段**（由 simple-uploader 附加）：`chunkNumber`（从 1 开始）、`chunkSize`、`currentChunkSize`、`totalSize`、`identifier`、`filename`、`relativePath`、`totalChunks`。
- **业务字段**：`params` 里的键值直接合并进 query。
- **`identifier`**：`${md5}@${uuid}`，服务端可用 `@` 前的 MD5 做内容级秒传判断。

### 7.2 分片校验响应

```json
{
	"data": {
		"skipUpload": false,
		"uploaded": [1, 2, 3]
	}
}
```

前端逻辑：

```js
const result = JSON.parse(message);
if (result.data.skipUpload) return true; // 整体已存在，所有分片视为已上传
return (result.data.uploaded || []).indexOf(chunk.offset + 1) >= 0;
```

- `uploaded` 是 **1-based 分片编号**（与 `chunk.offset + 1` 比对）；返回 0-based 会漏传/重传。
- `skipUpload: true` 表示秒传：所有分片都被标记为已上传，不再发分片 POST；此时 `fileSuccess` 的 `message` 来自触发完成的那次测试响应，**测试响应也要返回与上传成功一致的 `data` 结构**（至少 `fileId`、`identifier`）。
- 本组件显式定义了 `checkChunkUploadedByResponse`，因此 2xx 只代表请求成功，**是否跳过由响应体决定**；响应解析失败会返回 `undefined`，当前分片会被重传。
- 响应必须是合法 JSON；解析异常被组件吞掉，表现为「所有分片都重新上传」。

### 7.3 上传成功响应

最后一个分片的响应作为 `fileSuccess` 的 `message`：

```json
{
	"data": {
		"fileId": "xxx",
		"identifier": "md5@uuid"
	},
	"message": "success"
}
```

- `data` 必须为真值，否则组件弹「错误」通知并把文件标记为失败，**不触发**成功事件。
- 业务侧常用字段：`res.data.fileId`（资源 ID）、`res.data.identifier`。
- 响应必须是 JSON；网关返回 HTML 错误页会直接 `JSON.parse` 抛错。

### 7.4 跨域

- 自定义头 `jwt-token` 会触发 CORS 预检；后端需允许该 header、`GET/POST` 与对应 `Origin`。
- 组件未设置 `withCredentials`，跨域时不带 Cookie；鉴权依赖 `jwt-token`。
- 上传地址与页面不同源时，确保网关允许大文件分片请求与足够 body 限制。

## 8. 业务接入配方

### 8.1 选文件上传（无预选文件）

```ts
mittBus.emit('OPEN_GLOBAL_UPLOADER', {
	params: { resourceId: courseId },
	options: {
		target: `${Global.getBaseUrl()}/v1/vod_upload/chunk`,
		accept: ['video/*'],
		successCb: async (file, res) => {
			await relateVideoApi({ fileId: res.data.fileId, viewNum: file.params.viewNum });
		}
	},
	files: []
});
```

### 8.2 多视角视频：按文件循环打开，聚合后统一落库

来自 `CourseManage/UploadDialog.vue` 的真实模式：

```ts
const addVideoList = tableData.filter(v => typeof v.id !== 'number' && v.file);
const uploaded = new Set();
let saved = false;

const successCb = async (file, res) => {
	const key = file.uniqueIdentifier || file.id || res.data.fileId;
	if (saved || uploaded.has(key)) return;
	uploaded.add(key);

	vodMessages.push({ voreId: res.data.fileId, viewNum: file.params.viewNum });
	if (uploaded.size !== addVideoList.length) return;

	saved = true;
	await saveCourseResourceVideoApi({ id: file.params.resourceId, vodMessages });
};

addVideoList.forEach(video => {
	mittBus.emit('OPEN_GLOBAL_UPLOADER', {
		params: { resourceId: courseId, fileName: video.name, viewNum: video.viewNum },
		options: { target: `${Global.getBaseUrl()}/v1/vod_upload/chunk`, successCb },
		files: [video.file]
	});
});
```

要点：

- 每个文件单独 `emit`，保证 `file.params.viewNum` 正确。
- `successCb` 是共享闭包，用 `Set` 去重（同一文件可能重复回调）。
- 计数达到预期再调保存接口，避免每个文件都落库一次。

### 8.3 单文档上传

```ts
mittBus.emit('OPEN_GLOBAL_UPLOADER', {
	params: { resourceId },
	options: {
		target: `${Global.getBaseUrl()}/v1/vod_upload/chunk`,
		accept: ['.pdf', '.doc', '.docx', '.ppt', '.pptx'],
		successCb: async (file, res) => {
			await relateResourceTeclApi({
				fileId: res.data.fileId,
				resourceId: file.params.resourceId,
				identifier: res.data.identifier
			});
		}
	},
	files: []
});
```

### 8.4 页面内使用（`global=false`）

```vue
<GlobalUploader :global="false" />
```

- 按钮可见，面板不再固定在右下角，由父容器布局控制。
- 仍然通过 `mittBus` 传入 `params` / `options` 再点按钮；不要依赖 props 初始值。
- 同一页面不能再挂另一个 `GlobalUploader`（见第 9 节）。

## 9. 已知限制与坑

1. **单例、硬编码 id**：`#global-uploader`、`#global-uploader-btn` 固定；同页挂两个会 id 冲突、事件重复。全局只挂 `App.vue` 一处。
2. **`target` 默认是 `http://localhost:3000/upload`**：只是占位，任何真实调用都必须传 `options.target`。
3. **JWT 是启动时快照**：`headers` 在组件 `setup` 时读取 `Auth.getJwtToken()`，登录/刷新 token 后不会更新。确保组件在拿到 token 后挂载，或把 `headers` 改为函数（需要改源码）。
4. **`params` / `options` 的初始 props 不生效**：两个 `watch` 都不是 `immediate`；必须走 `OPEN_GLOBAL_UPLOADER` 或挂载后变更。
5. **`options` 只认 4 个键**：`target`、`testChunks`、`successCb`、`accept`；`mergeFn` 已注释，其它键不会生效。
6. **`accept` 必须是数组**：内部 `accept.join()`，传字符串会运行时报错。
7. **`customParams` / `successCb` 是模块级单变量**：并发打开多批不同参数会互相覆盖；一次只处理一批。
8. **`successCb` 异常未捕获**：回调抛错会导致成功事件、计数都不执行；回调内部自行 try/catch。
9. **成功响应必须是 JSON 且 `data` 为真值**：`data` 为空会走失败分支，且不触发失败事件。
10. **事件无负载**：`GLOBAL_UPLOADER_SUCCESS` 拿不到文件信息，必须用 `successCb`。
11. **`successNum` 跨批次残留**：只有关闭面板才清零；频繁打开时标题统计可能不准。
12. **自定义状态用 DOM 注入**：`useFileStatus` 往 `.file_<id> .uploader-file-status` 里插 `<p>`；`customStatus` 又是所有行共享的 class，多文件时状态样式会互相影响。
13. **MD5 是采样计算**：文件大于 5 个分片时只取 5 片，不是全量哈希；不同内容可能产生相同采样 MD5 的概率极低但非零，且 `md5@uuid` 让每次上传的 `identifier` 都不同，服务端要按 MD5 前缀做内容级秒传。
14. **`testChunks` 依赖后端 GET**：后端未实现校验接口时，每个分片都认为未上传；关闭 `testChunks` 可绕过，但失去秒传/续传。
15. **面板固定右下角、`z-index: 1000000`**：可能遮挡弹窗；`global=false` 可内嵌。
16. **全局样式非 scoped**：`#global-uploader` 与 `.uploader-file-info` 等是全局样式，可能影响页面里同名类。
17. **没有 `destroy`**：只暴露内部 `uploaderIns.cancel()`；组件常驻，不主动销毁。
18. **`onMounted` 注册的 `mittBus.on` 没有 `off`**：组件被重挂载会重复注册；因此只挂一次。
19. **不拦截页面关闭**：提示文案写了「不要关闭或刷新」，但没有 `beforeunload` 拦截。
20. **大文件内存/读盘成本**：MD5 与分片都会读文件，超大视频在低端机上有卡顿风险。

## 10. 复制与适配

### 10.1 目录清单

```text
app-index/
├── components/GlobalUploader/
│   ├── index.vue
│   ├── CustomFileListHeader.vue
│   ├── CustomFileListItem.vue
│   ├── useFileStatus.js
│   ├── index.scss
│   └── images/ (audio/image/text/video/zip 图标)
├── utils/mittBus.ts
├── utils/md5.js
└── App.vue (挂载 <GlobalUploader />)
```

### 10.2 必须替换的工程依赖

| 原依赖 | 替换建议 |
| --- | --- |
| `@core/auth` 的 `Auth.getJwtToken()` | 项目自己的 token 读取；或改为 `options.headers` 函数 |
| `@core/global` 的 `Global.getBaseUrl()` | 项目 API base URL |
| `@/utils/mittBus` | 项目事件总线；或改成 props / expose 调用 |
| `@/utils/md5` 的 `generateMD5` | 保留 `spark-md5` 实现；或换全量哈希 |
| `element-plus` 图标 | 显式导入 `FullScreen/Minus/Close/SuccessFilled/CircleCloseFilled` |
| `vue-simple-uploader/dist/style.css` | 必须保留全局引入 |

### 10.3 构建配置

本项目是 Vue CLI 5 / Webpack：

- `app.use(uploader)` + 全局 CSS。
- 组件 SCSS 里的 `url('./images/xxx.png')` 由 Webpack 的 `url-loader/file-loader` 处理并输出为静态资源。
- `vue.config.js` 配置了 `@ -> app-index`、`@core -> app-core` 别名；复制到其他工程时对齐 alias。
- `css.extract: false`（本项目），上传器样式随 JS 打包；生产构建仍会输出 CSS 资源，验证面板样式是否加载。
- Element Plus 通过 `unplugin-vue-components` + `ElementPlusResolver` 自动注册；若不用自动导入，需要显式注册 `ElButton`、`ElIcon`、`ElMessageBox`、`ElNotification`。

Vite 工程接入注意：

- `import 'vue-simple-uploader/dist/style.css'` 同样需要；`url()` 图片由 Vite 作为 asset 处理。
- `vue-simple-uploader` 同时提供 UMD/ESM 入口，Vite 会走 `module` 字段；首次启动如遇 CJS 预构建问题，加 `optimizeDeps.include: ['vue-simple-uploader']` 后重启。
- alias 需同时配 `@` 与 `@core`（或替换 import 路径）。
- 本项目验证环境是 Vue CLI 5 + Vue 3.2；Vite + Vue 3.5 下需回归 CSS、图片和 Element Plus 图标注册。

### 10.4 构建后验证

- [ ] 页面加载无 `Failed to resolve component: uploader`。
- [ ] 上传面板样式正常（CSS 已加载），文件类型图标显示。
- [ ] 选文件后出现「校验MD5」状态，随后进度开始。
- [ ] 暂停/恢复/重试/取消可用。
- [ ] 断点续传：刷新后重新选同一文件，已传分片被跳过。
- [ ] 秒传：服务端返回 `skipUpload: true` 时不再发分片。
- [ ] 成功回调拿到 `fileId`；网络面板中请求带 `jwt-token`。
- [ ] 失败响应（`data` 为空、HTTP 500）有错误提示。

## 11. 排错

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 页面报 `Failed to resolve component: uploader` | 没 `app.use(uploader)` | 入口注册插件 |
| 上传列表样式错乱 | 没引 `vue-simple-uploader/dist/style.css` | 全局引入 |
| 选文件没反应 | 全局模式按钮被隐藏且 `files` 为空 | 通过 `OPEN_GLOBAL_UPLOADER` 触发，或 `global=false` |
| 上传 404 | 没传 `options.target`，走了 localhost 占位 | 传 `${baseUrl}/v1/vod_upload/chunk` |
| 401 | `jwt-token` 为空或过期 | 确认挂载时 token 已存在；刷新页面 |
| 一直「校验MD5」 | 文件读取出错 / 超大文件 | 看控制台错误；检查 `generateMD5` |
| 所有分片重传 | 校验响应不是预期 JSON，或 `uploaded` 用 0-based | 按第 7.2 节返回 1-based |
| 成功后没有事件 | `res.data` 为空 | 后端返回非空 `data` |
| 成功回调没执行 | `successCb` 被后续 `OPEN` 覆盖 | 一次只打开一批 |
| 面板遮挡弹窗 | `z-index: 1000000` | 用 `global=false` 或调低层级 |
| 同页出现两个上传面板 | 组件被挂载两次 | 只保留 `App.vue` 一处 |

## 12. 接入检查清单

- [ ] `vue-simple-uploader` 插件与 CSS 已在入口注册。
- [ ] Element Plus 图标已全局注册或局部导入。
- [ ] `GlobalUploader` 只在 `App.vue` 挂载一次。
- [ ] `mittBus` 事件类型与 `OPEN_GLOBAL_UPLOADER` 负载已对齐。
- [ ] 每次打开都传 `options.target`，URL 与网关/上下文路径一致。
- [ ] `params` 包含业务关联字段，并在 `successCb` 中正确读取 `file.params`。
- [ ] 后端实现 GET 校验与 POST 分片，响应符合第 7 节。
- [ ] `successCb` 内部有 try/catch，聚合落库做了去重。
- [ ] 页面卸载时 `mittBus.off` 掉监听。
- [ ] 大文件场景验证暂停/恢复、断点续传、秒传。
