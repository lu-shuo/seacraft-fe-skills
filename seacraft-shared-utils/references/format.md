# 时间、数字、文件大小与中文数字

本文覆盖 `getTimeState`、`convertToChinaNum`、`formatNumberUnit`、`formatFileSize`、`parseTimeToSeconds`、`parseSecondsToTimeString`、`formatDate`、`formatDuration`、`timestampToDateString`、`getBrowserLang`。

这些函数在后台项目中大量复制，差异主要在边界条件。公共版本应先统一口径，再复用。

## 1. getTimeState

```ts
import { getTimeState } from '@/utils';

const greeting = getTimeState(); // '早上好 ⛅'
```

源码：

```ts
export function getTimeState() {
	const hours = new Date().getHours();
	if (hours >= 6 && hours <= 10) return '早上好 ⛅';
	if (hours >= 10 && hours <= 14) return '中午好 🌞';
	if (hours >= 14 && hours <= 18) return '下午好 🌞';
	if (hours >= 18 && hours <= 24) return '晚上好 🌛';
	if (hours >= 0 && hours <= 6) return '凌晨好 🌛';
}
```

问题：

- 10、14、18 同时命中两个区间，按顺序取第一个。
- 返回类型推断为 `string | undefined`，虽然 0–24 已覆盖。
- 不同项目边界不同（supervision 是 6–11 / 11–14）。

公共版本建议：

```ts
export function getTimeState(now = new Date()): string {
	const hours = now.getHours();
	if (hours < 6) return '凌晨好 🌛';
	if (hours < 11) return '早上好 ⛅';
	if (hours < 14) return '中午好 🌞';
	if (hours < 18) return '下午好 🌞';
	return '晚上好 🌛';
}
```

显式传 `now` 便于测试。

## 2. getBrowserLang

```ts
import { getBrowserLang } from '@/utils';

getBrowserLang(); // 'zh' | 'en'
```

源码：

```ts
export function getBrowserLang() {
	const browserLang = navigator.language ? navigator.language : navigator.browserLanguage;
	if (browserLang.toLowerCase() === 'cn' || browserLang.toLowerCase() === 'zh' || browserLang.toLowerCase() === 'zh-cn') {
		return 'zh';
	}
	return 'en';
}
```

注意：

- `navigator.browserLanguage` 是 IE 遗留字段，现代浏览器用 `navigator.language`。
- 只区分中/英；`zh-TW`、`zh-HK` 会落到 `en`。
- 需要完整语言标签用 `navigator.languages` 或 i18n 库。
- SSR 下 `navigator` 不存在，必须加 `typeof navigator !== 'undefined'`。

公共版本：

```ts
export function getBrowserLang(): 'zh' | 'en' {
	if (typeof navigator === 'undefined') return 'en';
	const lang = (navigator.languages?.[0] || navigator.language || 'en').toLowerCase();
	return lang.startsWith('zh') || lang === 'cn' ? 'zh' : 'en';
}
```

## 3. convertToChinaNum

```ts
import { convertToChinaNum } from '@/utils';

convertToChinaNum(0); // '零'
convertToChinaNum(10); // '十'
convertToChinaNum(101); // '一百零一'
convertToChinaNum(10001); // '一万零一'
```

源码：

```ts
export function convertToChinaNum(num: number) {
	const arr1 = ['零', '一', '二', '三', '四', '五', '六', '七', '八', '九'];
	const arr2 = ['', '十', '百', '千', '万', '十', '百', '千', '亿', '十', '百', '千', '万', '十', '百', '千', '亿'];
	if (!num || isNaN(num)) return '零';
	const english = num.toString().split('');
	let result = '';
	for (let i = 0; i < english.length; i++) {
		const desIndex = english.length - 1 - i;
		result = arr2[i] + result;
		result = arr1[parseInt(english[desIndex])] + result;
	}
	result = result.replace(/零(千|百|十)/g, '零').replace(/十零/g, '十');
	result = result.replace(/零+/g, '零');
	result = result.replace(/零亿/g, '亿').replace(/零万/g, '万');
	result = result.replace(/亿万/g, '亿');
	result = result.replace(/零+$/, '');
	result = result.replace(/^一十/g, '十');
	return result;
}
```

边界：

- 只支持非负整数；负数会得到 `'零'`（`!num` 为 false，但 `toString().split` 会带 `-`，`parseInt('-')` 为 NaN）。
- 超过 16 位（万亿级）会错位；`arr2` 只到亿。
- `0`、`NaN`、`null`、`''` 都返回 `'零'`。
- 不做小数/千分位处理。

公共版本建议：

```ts
export function convertToChinaNum(num: number): string {
	if (!Number.isInteger(num) || num < 0 || num > 9999999999999999) {
		throw new RangeError('convertToChinaNum supports 0 ~ 9999999999999999');
	}
	// 原算法...
}
```

## 4. formatNumberUnit

```ts
import { formatNumberUnit } from '@/utils';

formatNumberUnit(9999); // '9,999'（locale 相关）
formatNumberUnit(10000); // '1.0万'
formatNumberUnit(123456789); // '1.2亿'
```

实现：

```ts
export function formatNumberUnit(value: number) {
	const n = Number(value || 0);
	if (n < 10_000) return n.toLocaleString();
	if (n < 100_000_000) return `${(n / 10_000).toFixed(1)}万`;
	return `${(n / 100_000_000).toFixed(1)}亿`;
}
```

注意：

- `< 1 万` 的千分位依赖 locale；测试时不要写死 `'9,999'`。
- 固定 1 位小数，不做「去掉 `.0`」。
- 负数返回 `toLocaleString()`（负数永远 `< 10000`）。
- `NaN` / `null` 经 `Number(value || 0)` 变成 `0`。

## 5. formatFileSize

```ts
import { formatFileSize } from '@/utils';

formatFileSize(0); // '0B'
formatFileSize(1024); // '1.0KB'
formatFileSize(1024 * 1024); // '1.0M'
formatFileSize(1024 ** 3); // '1.0G'
```

实现：

```ts
export function formatFileSize(size: number) {
	if (size < 1024) return size + 'B';
	if (size < 1024 * 1024) return (size / 1024).toFixed(1) + 'KB';
	if (size < 1024 * 1024 * 1024) return (size / (1024 * 1024)).toFixed(1) + 'M';
	return (size / (1024 * 1024 * 1024)).toFixed(1) + 'G';
}
```

注意：

- 单位是 `M` / `G`，不是 `MB` / `GB`。
- `size` 为负数/`NaN` 时返回 `NaN B` 或错误结果；公共版本先 `Number.isFinite` + `Math.max(0, size)`。
- 1024 边界：`1024` 进入 KB 分支输出 `1.0KB`，符合预期。
- 需要 B/KB/MB/GB 标准单位时改文案。

## 6. parseTimeToSeconds

```ts
import { parseTimeToSeconds } from '@/utils';

parseTimeToSeconds('01:02:03'); // 3723
parseTimeToSeconds('02:03'); // 123
parseTimeToSeconds('59'); // 59
parseTimeToSeconds(''); // 0
```

实现：

```ts
export function parseTimeToSeconds(timeStr: string) {
	const parts = timeStr.split(':').map(Number);
	let totalSeconds = 0;
	if (parts.length === 3) totalSeconds = parts[0] * 3600 + parts[1] * 60 + parts[2];
	else if (parts.length === 2) totalSeconds = parts[0] * 60 + parts[1];
	else if (parts.length === 1) totalSeconds = parts[0];
	return totalSeconds;
}
```

注意：

- `''` → `[''].map(Number)` → `[0]`，返回 `0`，不是 `NaN`。
- 非法段（`'aa:bb'`）→ `NaN`。
- 不支持毫秒、超过 24 小时会正常累加。
- 公共版本应校验 `parts.length` 和 `Number.isFinite`。

## 7. parseSecondsToTimeString

```ts
import { parseSecondsToTimeString } from '@/utils';

parseSecondsToTimeString(59); // '00:59'
parseSecondsToTimeString(61); // '01:01'
parseSecondsToTimeString(3661); // '01:01:01'
```

实现：

```ts
export function parseSecondsToTimeString(seconds: number) {
	const hours = Math.floor(seconds / 3600);
	const minutes = Math.floor((seconds % 3600) / 60);
	const remainingSeconds = Math.floor(seconds % 60);
	const formattedMinutes = String(minutes).padStart(2, '0');
	const formattedSeconds = String(remainingSeconds).padStart(2, '0');
	if (hours === 0) return `${formattedMinutes}:${formattedSeconds}`;
	const formattedHours = String(hours).padStart(2, '0');
	return `${formattedHours}:${formattedMinutes}:${formattedSeconds}`;
}
```

注意：

- 负数会输出 `-1:-1` 之类；先 `Math.max(0, seconds)`。
- 超过 99 小时会输出 3 位小时。
- 不处理毫秒。

## 8. formatDate

```ts
import { formatDate } from '@/utils';

formatDate('yyyy-MM-dd HH:mm:ss', new Date()); // '2026-11-19 14:30:00'
formatDate('yyyy/MM/dd', '2026-11-19'); // '2026/11/19'
formatDate('yyyy-MM-dd', ''); // ''
```

实现（`app-core/comm.js`）：

```js
function formatDate(fmt, date) {
	if (!date) return '';
	if (typeof date !== 'object') {
		if (typeof date === 'string') date = date.replaceAll('-', '/');
		date = new Date(date);
	}
	const o = {
		'M+': date.getMonth() + 1,
		'd+': date.getDate(),
		'h+': date.getHours(),
		'm+': date.getMinutes(),
		's+': date.getSeconds(),
		'q+': Math.floor((date.getMonth() + 3) / 3),
		S: date.getMilliseconds()
	};
	if (/(y+)/.test(fmt)) fmt = fmt.replace(RegExp.$1, (date.getFullYear() + '').substr(4 - RegExp.$1.length));
	for (const k in o)
		if (new RegExp('(' + k + ')').test(fmt))
			fmt = fmt.replace(RegExp.$1, RegExp.$1.length == 1 ? o[k] : ('00' + o[k]).substr(('' + o[k]).length));
	return fmt;
}
```

注意：

- `MM` / `dd` 通过长度补零；`M` / `d` 不补零。
- `replaceAll` 在旧环境不支持；用 `replace(/-/g, '/')`。
- 使用 `RegExp.$1`，在严格模式/并发调用下可读性差。
- 时区取本地时间；跨时区需求用 dayjs + timezone 插件。
- 建议公共版本直接使用 `dayjs(date).format(fmt)`，并把 dayjs 作为可选 peer。

## 9. formatDuration

```ts
import { formatDuration } from '@/utils';

formatDuration(5); // '00:05'
formatDuration(65); // '01:05'
formatDuration(3665); // '01:01:05'
```

实现（`comm.formatTime`）：

```js
function formatTime(time) {
	time = parseInt(time);
	if (time < 60) return `00:${('00' + time).slice(-2)}`;
	if (time < 3600) {
		const min = ('00' + Math.floor(time / 60)).slice(-2);
		const ss = ('00' + (time - Number(min) * 60)).slice(-2);
		return `${min}:${ss}`;
	}
	const hh = ('00' + Math.floor(time / 3600)).slice(-2);
	const min = ('00' + Math.floor((time - Number(hh) * 3600) / 60)).slice(-2);
	const ss = ('00' + (time - Number(hh) * 3600 - Number(min) * 60)).slice(-2);
	return `${hh}:${min}:${ss}`;
}
```

注意：

- `Number(min)` 把补零后的字符串转回数字，逻辑上等价于直接取余。
- 负数/`NaN` 未处理；`parseInt(null)` 为 `NaN`。
- 与 `parseSecondsToTimeString` 的区别：后者 `<1h` 输出 `mm:ss`，`formatDuration` `<60s` 也输出 `00:ss`，本质相同。

## 10. timestampToDateString

```ts
import { timestampToDateString } from '@/utils';

timestampToDateString(1731990000000); // '2024-11-19'
```

实现：

```ts
export function timestampToDateString(timestamp: number) {
	const date = new Date(timestamp);
	const year = date.getFullYear();
	const month = String(date.getMonth() + 1).padStart(2, '0');
	const day = String(date.getDate()).padStart(2, '0');
	return `${year}-${month}-${day}`;
}
```

注意：无效时间戳会得到 `NaN-NaN-NaN`；先 `Number.isFinite`。

## 11. 统一建议

| 需求 | 推荐 |
| --- | --- |
| 简单时分秒 | `parseSecondsToTimeString` / `formatDuration` |
| 日期格式化 | dayjs（时区、本地化、ISO） |
| 相对时间 | dayjs relativeTime 插件 |
| 大数字单位 | `formatNumberUnit`，产品确认「万/亿」与小数位 |
| 文件大小 | `formatFileSize`，产品确认 `M` 还是 `MB` |
| 中文数字 | `convertToChinaNum`，限制 0–亿级整数 |
| 浏览器语言 | `getBrowserLang`，只用于中英二选一 |
