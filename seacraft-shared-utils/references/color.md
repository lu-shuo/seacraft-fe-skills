# 颜色转换与调色

来源：`jy-digital-patrol-ui/src/utils/theme/tool.ts`、`jy-application-resourcemanage-ui/app-core/color.ts`。两份实现几乎一致，区别是 resourcemanage 显式 import 了 `ElMessage`，patrol 依赖自动导入。

## 1. API

```ts
hexToRgb(str: string): number[] | void
rgbToHex(r: number, g: number, b: number): string | void
getLightColor(color: string, level: number): string | void
getDarkColor(color: string, level: number): string | void
```

| 函数 | 说明 |
| --- | --- |
| `hexToRgb` | `#RRGGBB` / `RRGGBB` → `[r, g, b]` |
| `rgbToHex` | `r/g/b` → `#rrggbb` |
| `getLightColor` | 与白色混合，`level` 越大越浅 |
| `getDarkColor` | 与 `20.5` 混合，`level` 越大越深 |

## 2. 用法

```ts
import { hexToRgb, rgbToHex, getLightColor, getDarkColor } from '@/utils/theme/tool';

hexToRgb('#027BF5'); // [2, 123, 245]
rgbToHex(2, 123, 245); // '#027bf5'
getLightColor('#027BF5', 0.2); // 更浅
getDarkColor('#027BF5', 0.2); // 更深
```

典型用途：

- Element Plus 主色派生 `--el-color-primary-light-1..9`。
- 根据标签/状态色生成浅色背景。
- 根据主题色生成 hover/active 色。

## 3. 实现

```ts
export function hexToRgb(str: string) {
	const reg = /^\#?[0-9A-Fa-f]{6}$/;
	if (!reg.test(str)) return ElMessage.warning('输入错误的hex');
	const hexs = str.replace('#', '').match(/../g)!;
	for (let i = 0; i < 3; i++) hexs[i] = parseInt(hexs[i], 16) as any;
	return hexs;
}

export function rgbToHex(r: number, g: number, b: number) {
	const reg = /^\d{1,3}$/;
	if (!reg.test(String(r)) || !reg.test(String(g)) || !reg.test(String(b))) {
		return ElMessage.warning('输入错误的rgb颜色值');
	}
	const hexs = [r.toString(16), g.toString(16), b.toString(16)];
	for (let i = 0; i < 3; i++) if (hexs[i].length === 1) hexs[i] = `0${hexs[i]}`;
	return `#${hexs.join('')}`;
}

export function getDarkColor(color: string, level: number) {
	const rgb = hexToRgb(color);
	if (!rgb) return;
	for (let i = 0; i < 3; i++) rgb[i] = Math.round(20.5 * level + rgb[i] * (1 - level));
	return rgbToHex(rgb[0], rgb[1], rgb[2]);
}

export function getLightColor(color: string, level: number) {
	const rgb = hexToRgb(color);
	if (!rgb) return;
	for (let i = 0; i < 3; i++) rgb[i] = Math.round(255 * level + rgb[i] * (1 - level));
	return rgbToHex(rgb[0], rgb[1], rgb[2]);
}
```

## 4. 公共化改造

旧实现的三个问题：

1. **依赖 UI 库**：校验失败调用 `ElMessage.warning`，util 不应依赖 Element Plus。
2. **返回 `void`**：调用方拿到 `undefined` 后继续做数学运算会 `NaN`。
3. **输入范围不严**：`rgbToHex` 只校验 1–3 位数字，`r=999` 会生成非法 hex。

推荐版本：

```ts
export interface RGB {
	r: number;
	g: number;
	b: number;
}

export function hexToRgb(hex: string): RGB | null {
	const value = hex.replace('#', '');
	if (!/^[0-9A-Fa-f]{6}$/.test(value)) return null;
	return {
		r: parseInt(value.slice(0, 2), 16),
		g: parseInt(value.slice(2, 4), 16),
		b: parseInt(value.slice(4, 6), 16)
	};
}

export function rgbToHex({ r, g, b }: RGB): string {
	const clamp = (n: number) => Math.min(255, Math.max(0, Math.round(n)));
	return `#${[r, g, b].map(n => clamp(n).toString(16).padStart(2, '0')).join('')}`;
}

export function getDarkColor(color: string, level: number): string | null {
	const rgb = hexToRgb(color);
	if (!rgb) return null;
	const next = { r: 20.5 * level + rgb.r * (1 - level), g: 20.5 * level + rgb.g * (1 - level), b: 20.5 * level + rgb.b * (1 - level) };
	return rgbToHex(next);
}

export function getLightColor(color: string, level: number): string | null {
	const rgb = hexToRgb(color);
	if (!rgb) return null;
	const next = { r: 255 * level + rgb.r * (1 - level), g: 255 * level + rgb.g * (1 - level), b: 255 * level + rgb.b * (1 - level) };
	return rgbToHex(next);
}
```

调用方负责提示：

```ts
const light = getLightColor(primary, 0.3);
if (!light) ElMessage.warning('主题色格式错误');
```

## 5. 注意

1. **只支持 6 位 hex**：`#fff` 会被判非法；需要 3 位缩写先展开。
2. **不支持 alpha**：`#RRGGBBAA` / `rgba()` 不处理；需要透明度用 CSS `color-mix()` 或单独解析。
3. **`getDarkColor` 不是变黑**：与 `20.5` 混合，结果是深灰；要纯黑用 `#000` 或自定义混合基准。
4. **`level` 没有范围校验**：传 `0.5` 正常，传 `1.5` 会溢出；公共版本 clamp 到 `[0,1]`。
5. **`rgbToHex` 旧实现不 clamp**：`r=300` 会输出 `12c` 之类的错误；公共版本 clamp。
6. **颜色缓存/主题**：这两组函数是纯计算，不读写 CSS 变量；主题切换由 `useTheme` / `useInlineCssVars` 负责。
7. **`ElMessage` 依赖**：复制到新项目时删掉提示逻辑，否则要么装 Element Plus，要么运行时报 `ElMessage is not defined`。

## 6. 测试

```ts
import { describe, expect, it } from 'vitest';
import { hexToRgb, rgbToHex, getLightColor, getDarkColor } from '../src/color';

describe('color', () => {
	it('converts hex and rgb', () => {
		expect(hexToRgb('#027BF5')).toEqual({ r: 2, g: 123, b: 245 });
		expect(rgbToHex({ r: 2, g: 123, b: 245 })).toBe('#027bf5');
	});

	it('rejects invalid input', () => {
		expect(hexToRgb('#fff')).toBeNull();
		expect(hexToRgb('not-a-color')).toBeNull();
	});

	it('clamps rgb', () => {
		expect(rgbToHex({ r: 300, g: -1, b: 128 })).toBe('#ff0080');
	});

	it('lightens and darkens', () => {
		expect(getLightColor('#000000', 0.5)).toBe('#808080');
		expect(getDarkColor('#ffffff', 0.5)).toBe('#8a8a8a');
	});
});
```
