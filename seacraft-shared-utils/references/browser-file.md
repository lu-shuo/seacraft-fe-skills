# 浏览器、DOM、设备、URL 与文件

本文覆盖 `addScript`、`copyToClipboard`、`changeDomPosition`、`blurActiveElement`、`waitUntilSuccess`、`submitHiddenForm`、`getQueryParam` / `getHashQueryParam`、设备判断、`getLocalCover`、`generateUUID`、`md5Hash` / `generateFileMD5`、`base64ToFile`。

这些函数直接操作浏览器 API，公共化时最重要的是：**SSR 防御、副作用说明、错误处理、可测试性**。

## 1. addScript

```ts
import { addScript } from '@/utils';

await addScript('https://cdn.example.com/sdk.js');
```

源码：

```ts
const loadedMap: Record<string, boolean> = {};

export function addScript(url: string) {
	return new Promise((resolve, reject) => {
		const scriptId = encodeURIComponent(url);
		if (document.getElementById(scriptId) && loadedMap[scriptId]) {
			resolve(true);
			return;
		}
		const script = document.createElement('script');
		script.type = 'text/javascript';
		script.src = url;
		script.id = scriptId;
		document.head.appendChild(script);
		script.onload = () => {
			loadedMap[scriptId] = true;
			resolve(true);
		};
		script.onerror = () => reject(false);
	});
}
```

注意：

- `reject(false)` 不是 `Error`；调用方 `catch` 拿不到原因。
- 没有超时；CDN 挂起会一直 pending。
- `loadedMap` 只在当前页面有效，刷新后丢失。
- 并发加载同一 URL 会创建多个 script；公共版本应缓存 Promise。
- SSR 下 `document` 不存在，调用前判断 `isClient()`。

推荐版本：

```ts
const scriptPromiseMap = new Map<string, Promise<void>>();

export function loadScript(url: string, attrs: Record<string, string> = {}): Promise<void> {
	if (typeof document === 'undefined') return Promise.reject(new Error('loadScript requires DOM'));
	const cached = scriptPromiseMap.get(url);
	if (cached) return cached;

	const promise = new Promise<void>((resolve, reject) => {
		const existing = document.querySelector<HTMLScriptElement>(`script[data-src="${CSS.escape(url)}"]`);
		if (existing?.dataset.loaded === 'true') return resolve();
		const script = existing ?? document.createElement('script');
		script.src = url;
		script.async = true;
		script.dataset.src = url;
		Object.entries(attrs).forEach(([k, v]) => script.setAttribute(k, v));
		script.onload = () => {
			script.dataset.loaded = 'true';
			resolve();
		};
		script.onerror = () => reject(new Error(`Failed to load script: ${url}`));
		if (!existing) document.head.appendChild(script);
	});

	scriptPromiseMap.set(url, promise);
	return promise;
}
```

## 2. copyToClipboard

源码：

```ts
export function copyToClipboard(text: string, showMessage = true) {
	if (navigator.clipboard) {
		navigator.clipboard.writeText(text);
	} else {
		const textarea = document.createElement('textarea');
		document.body.appendChild(textarea);
		textarea.style.position = 'fixed';
		textarea.style.clip = 'rect(0 0 0 0)';
		textarea.style.top = '10px';
		textarea.value = text;
		textarea.select();
		document.execCommand('copy', true);
		document.body.removeChild(textarea);
	}
	showMessage && ElMessage.success('复制成功');
}
```

问题：

- `writeText` 是异步的，没有 `await`；权限拒绝时仍会走到成功提示。
- `showMessage` 直接依赖 `ElMessage`，util 不应依赖 UI 库。
- `execCommand('copy')` 已废弃，但仍是旧浏览器兜底。
- `textarea` 样式缺少 `opacity: 0` / `left: -9999px`，部分浏览器会闪一下。

推荐版本：

```ts
export async function copyToClipboard(text: string): Promise<boolean> {
	if (typeof navigator !== 'undefined' && navigator.clipboard?.writeText) {
		try {
			await navigator.clipboard.writeText(text);
			return true;
		} catch {
			// fall through to legacy path
		}
	}
	if (typeof document === 'undefined') return false;
	const textarea = document.createElement('textarea');
	textarea.value = text;
	textarea.setAttribute('readonly', '');
	textarea.style.position = 'fixed';
	textarea.style.top = '-9999px';
	textarea.style.opacity = '0';
	document.body.appendChild(textarea);
	textarea.select();
	try {
		return document.execCommand('copy');
	} catch {
		return false;
	} finally {
		document.body.removeChild(textarea);
	}
}
```

调用方：

```ts
if (await copyToClipboard(text)) ElMessage.success('复制成功');
else ElMessage.error('复制失败');
```

## 3. changeDomPosition

```ts
import { changeDomPosition } from '@/utils';

changeDomPosition(document.querySelector('#a')!, document.querySelector('#b')!);
```

源码：

```ts
export function changeDomPosition(a: any, b: any) {
	if (a === b) return;
	const aParent = a.parentNode;
	const bParent = b.parentNode;
	const aNext = a.nextElementSibling;
	const bNext = b.nextElementSibling;
	if (aNext === b) return bParent.insertBefore(b, a);
	if (bNext === a) return aParent.insertBefore(a, b);
	if (a.contains(b)) return aParent.insertBefore(b, a), bParent.insertBefore(a, bNext);
	return bParent.insertBefore(a, b), aParent.insertBefore(b, aNext);
}
```

注意：

- 直接移动真实 DOM；Vue/React 的虚拟 DOM 不知道节点被移动，后续 diff 可能还原或错位。只用于纯展示节点，或移动后立即更新框架状态。
- 没有校验 `a` / `b` 是否为元素、是否有 `parentNode`。
- 返回值是 `insertBefore` 的结果（节点），类型未声明。
- 被移动节点上的事件监听会随节点保留，但组件状态不会跟着走。

## 4. blurActiveElement

```ts
import { blurActiveElement } from '@/utils';

blurActiveElement();
```

源码：

```ts
export function blurActiveElement() {
	if (typeof document === 'undefined') return;
	const active = document.activeElement as HTMLElement | null;
	if (!active || active === document.body) return;
	if (typeof active.blur === 'function') active.blur();
}
```

用途：移动端点击分页/提交按钮时，输入框不会自动失焦，表单模型滞后；先 `blur()` 触发 `change` 再提交。

注意：会打断用户输入焦点；只在确实需要同步表单模型时调用。

## 5. waitUntilSuccess

```ts
import { waitUntilSuccess } from '@/utils';

waitUntilSuccess(() => Boolean(window.someSDK), 100, 50);
```

源码：

```ts
export function waitUntilSuccess(fn: any, intervalMs: number, tries: number) {
	if (fn()) return;
	if (tries > 0) {
		setTimeout(() => waitUntilSuccess(fn, intervalMs, tries - 1), intervalMs);
	}
}
```

问题：

- 没有返回值，无法 `await` 成功/失败。
- 没有取消机制；组件卸载后定时器仍可能执行。
- `tries` 语义是「剩余重试次数」，不是总次数。
- `fn` 抛错会中断且无提示。

推荐版本：

```ts
export interface RetryOptions {
	intervalMs?: number;
	tries?: number;
	signal?: AbortSignal;
}

export function retryUntil(
	fn: () => boolean,
	{ intervalMs = 100, tries = 50, signal }: RetryOptions = {}
): Promise<boolean> {
	return new Promise(resolve => {
		let remaining = tries;
		const tick = () => {
			if (signal?.aborted) return resolve(false);
			try {
				if (fn()) return resolve(true);
			} catch {
				// 继续重试
			}
			if (remaining-- <= 0) return resolve(false);
			setTimeout(tick, intervalMs);
		};
		tick();
	});
}
```

## 6. submitHiddenForm

```ts
import { submitHiddenForm } from '@/utils';

submitHiddenForm('/api/export', { token, id: 123 });
```

源码：

```ts
export function submitHiddenForm(action: string, fields: Record<string, string | number | null | undefined>) {
	const form = document.createElement('form');
	form.method = 'POST';
	form.action = action;
	form.style.display = 'none';
	Object.entries(fields).forEach(([name, value]) => {
		const input = document.createElement('input');
		input.type = 'hidden';
		input.name = name;
		input.value = value === null || value === undefined ? '' : String(value);
		form.appendChild(input);
	});
	document.body.appendChild(form);
	form.submit();
}
```

用途：需要浏览器原生 POST（文件下载、SSO、第三方表单）时绕过 XHR。

注意：

- 提交后页面会导航，`form` 不自动移除；在当前页停留的场景要手动清理。
- 不能设置自定义请求头；鉴权字段要放进 `fields`。
- `action` 与字段值需要调用方自行 URL 编码/校验，防止注入。
- `document` 不存在时不可用。

## 7. getQueryParam / getHashQueryParam

源码：

```ts
// 普通 query：只分割 ?
export function getQueryParam(name: string): string {
	const url = document.location.toString();
	const arrObj = url.split('?');
	if (arrObj.length > 1) {
		const arrPara = arrObj[1].split('&');
		for (const item of arrPara) {
			const arr = item.split('=');
			if (arr[0] == name) return arr[1];
		}
	}
	return '';
}

// hash query："#/setting/roleManage/?tab=x"
export function getHashQueryParam(key: string): string | null {
	const queryString = window.location.hash.split('?')[1];
	if (!queryString) return null;
	return new URLSearchParams(queryString).get(key);
}
```

注意：

- `getQueryParam` 不处理 hash 路由；hash 里的 query 要用 `getHashQueryParam`。
- `getQueryParam` 不 `decodeURIComponent`，`getHashQueryParam` 会。
- 同名参数只取第一个。
- 公共版本建议统一：

```ts
export function getQueryParam(name: string, search = window.location.search): string | null {
	if (!search) return null;
	return new URLSearchParams(search).get(name);
}
```

## 8. 设备判断

### 源码差异

`space` / `jy-player-ui` 版本：

```ts
import isMobileJS from 'ismobilejs';
const ua = navigator.userAgent.toLowerCase();
const outerWidth = window.outerWidth;
const outerHeight = window.outerHeight;

export const isMobile = isMobileJS(ua).any;
export const isPad = /iPad|Android|Tablet/i.test(ua) && outerWidth > 768 && outerHeight > 768;
```

`tqmp` 版本：

```ts
const isIPadOS = () => /macintosh/i.test(navigator.userAgent) && navigator.maxTouchPoints > 1;

export const isMobile = () => {
	const forceMobile = new URLSearchParams(window.location.search).get('mobile') === '1';
	if (forceMobile) return true;
	if (isIPadOS()) return true;
	const ua = navigator.userAgent.toLowerCase();
	return ['android', 'webos', 'iphone', 'ipad', 'ipod', 'blackberry', 'windows phone', 'mobile', 'ios'].some(key => ua.includes(key));
};
```

### 推荐公共版本

```ts
const ua = () => (typeof navigator === 'undefined' ? '' : navigator.userAgent.toLowerCase());

export const isIPadOS = () =>
	typeof navigator !== 'undefined' && /macintosh/i.test(navigator.userAgent) && navigator.maxTouchPoints > 1;

export const isMobile = () => {
	const value = ua();
	return isIPadOS() || ['android', 'webos', 'iphone', 'ipad', 'ipod', 'blackberry', 'windows phone', 'mobile', 'ios'].some(k => value.includes(k));
};

export const isTablet = () => isIPadOS() || /ipad|android(?!.*mobile)|tablet/i.test(ua());
export const isPhone = () => isMobile() && !isTablet();
export const isIOS = () => isIPadOS() || /iphone|ipad|ipod/i.test(ua());
export const isAndroid = () => /android/i.test(ua());
export const isWeChat = () => /micromessenger/i.test(ua());
export const getDeviceType = (): 'mobile' | 'tablet' | 'desktop' => (isPhone() ? 'mobile' : isTablet() ? 'tablet' : 'desktop');
```

注意：

- **不要在模块顶层读 `navigator`**；上面的 `ua()` 是函数。
- iPadOS 13+ 默认桌面 UA，必须用 `Macintosh + maxTouchPoints > 1`。
- `ismobilejs` 可选；不装也能用 UA 版本。
- `?mobile=1` 是测试强制开关，公共版本不要默认带。
- `isWeChat` 的正则是 `/micromessenger/i`。

## 9. getLocalCover

```ts
import { getLocalCover } from '@/utils';

const cover = await getLocalCover(file, '320', '180');
```

源码要点：

- 音频或 `.mp3` / `.m4a` 直接返回 `''`。
- 创建隐藏 `<video>`，设置 `crossOrigin='anonymous'`、`autoplay`、`muted`、`playsInline`。
- `loadeddata` 后 seek 到 `min(2, duration)`；`seeked` 时用 canvas `drawImage` 截 JPEG。
- 5 秒超时返回 `''`。
- 无论成功失败都会 `revokeObjectURL`、移除事件、清空 `src`。

注意：

- 跨域视频未配置 CORS 时 canvas 会被污染，`toDataURL` 抛错 → 返回 `''`。
- 某些浏览器 `seeked` 不触发；已有 5 秒超时兜底。
- 返回值是 data URL，体积可能较大；上传前可再压缩。
- 不适合长视频/批量封面；会创建 video 元素并解码首帧。

## 10. generateUUID

源码有两种：

```ts
// patrol/space：Math.random
export function generateUUID() {
	let uuid = '';
	for (let i = 0; i < 32; i++) {
		const random = (Math.random() * 16) | 0;
		if (i === 8 || i === 12 || i === 16 || i === 20) uuid += '-';
		uuid += (i === 12 ? 4 : i === 16 ? (random & 3) | 8 : random).toString(16);
	}
	return uuid;
}

// tqmp：crypto.getRandomValues
export function generateUUID(): string {
	return (([1e7] as any) + '-1e3-4e3-8e3-1e11').replace(/[018]/g, (c: string) =>
		(parseInt(c) ^ (crypto.getRandomValues(new Uint8Array(1))[0] & (15 >> (parseInt(c) / 4)))).toString(16)
	);
}
```

推荐：

```ts
export function generateUUID(): string {
	if (typeof crypto !== 'undefined' && 'randomUUID' in crypto) return crypto.randomUUID();
	// RFC4122 v4 fallback
	return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
		const r = (Math.random() * 16) | 0;
		const v = c === 'x' ? r : (r & 0x3) | 0x8;
		return v.toString(16);
	});
}
```

注意：`Math.random` 版本不保证密码学随机，不要用于 token、密钥、防重放。

## 11. md5Hash / generateFileMD5

```ts
import { md5Hash, generateFileMD5 } from '@/utils/hash';
import SparkMD5 from 'spark-md5';

md5Hash('abc'); // '900150983cd24fb0d6963f7d28e17f72'

await generateFileMD5(file, {
	onProgress: (current, total) => console.log(current, total)
});
```

源码差异：

- `md5Hash(str)` = `SparkMD5.hash(str)`。
- `generateFileMD5`：
  - patrol 版本接收原生 `File`，按 `chunkSize` 全量分片。
  - resourcemanage 版本接收 `file.file`（uploader 包装），**超过 5 个分片只采样 5 个**，速度快但非完整哈希。

推荐签名：

```ts
interface FileMD5Options {
	chunkSize?: number; // default 10MB
	sample?: boolean; // default false
	onProgress?: (current: number, total: number) => void;
}

export function generateFileMD5(file: File, options?: FileMD5Options): Promise<string>;
```

注意：

- 采样 MD5 不能用于完整性校验，只适合快速指纹/秒传。
- 大文件计算会读盘/占 CPU，放在 Web Worker 更稳。
- `SparkMD5.ArrayBuffer` 需要 `FileReader.readAsArrayBuffer`；`File.slice` 兼容旧浏览器。
- `spark-md5` 作为可选 peer，不装就不导出 hash 子路径。

## 12. base64ToFile

```ts
import { base64ToFile } from '@/utils';

const file = base64ToFile('data:image/png;base64,xxxx', 'avatar.png');
```

源码：

```ts
export function base64ToFile(base64: string, filename: string): File {
	const arr = base64.split(',');
	const mimeMatch = arr[0].match(/:(.*?);/);
	const mime = mimeMatch ? mimeMatch[1] : 'application/octet-stream';
	const bstr = atob(arr[1]);
	let n = bstr.length;
	const u8arr = new Uint8Array(n);
	while (n--) u8arr[n] = bstr.charCodeAt(n);
	return new File([u8arr], filename, { type: mime });
}
```

注意：

- 只支持 base64 data URL 或纯 base64；没有 `,` 时 `arr[1]` 为 `undefined`。
- `atob` 对非 Latin1 字符会抛错；二进制数据通常没问题。
- 大文件转 `Uint8Array` 占内存；可用 `fetch(dataUrl).then(r => r.blob())`。
- 生成的是内存 `File`，不会落盘。

## 13. 统一检查

- [ ] SSR 下所有 `window` / `document` / `navigator` 都有守卫。
- [ ] 加载脚本失败 reject `Error`，不是 `false`。
- [ ] 剪贴板返回 `Promise<boolean>`，不依赖 UI 库。
- [ ] `changeDomPosition` 的调用点确认不破坏框架 diff。
- [ ] 轮询有取消和超时。
- [ ] 设备判断在函数内读取 UA。
- [ ] UUID 优先 `crypto.randomUUID()`。
- [ ] MD5 明确「全量」还是「采样」。
