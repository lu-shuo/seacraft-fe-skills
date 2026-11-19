# useInlineCssVars（主题对象 → CSS 变量）

`useInlineCssVars` 把一个「亮色 / 暗色」主题对象展平成 CSS 变量，绑定到容器 `:style`，实现按区块换肤。它不依赖具体业务 store，只需要一个暗色来源；是本 skill 保留的通用样式 hook。

## 1. 用法

```ts
import { useInlineCssVars } from '@/hooks/useInlineCssVars';

const { cssVars } = useInlineCssVars({
	scope: 'room-detail',
	light: {
		player: {
			bgColor: '#ffffff',
			borderColor: '#dcdfe6'
		}
	},
	dark: {
		player: {
			bgColor: '#141414',
			borderColor: '#3a3a3a'
		}
	}
});
```

```vue
<template>
	<div class="room-detail" :style="cssVars">
		<PlayerPanel />
	</div>
</template>

<style scoped>
.room-detail :deep(.player) {
	background: var(--room-detail-player-bgColor);
	border-color: var(--room-detail-player-borderColor);
}
</style>
```

## 2. API

```ts
useInlineCssVars({
  light: ThemeObject;
  dark: ThemeObject;
  scope?: string;
}): { cssVars: ComputedRef<Record<string, string>> }

type ThemeObject = { [key: string]: string | ThemeObject };
```

| 参数 | 说明 |
| --- | --- |
| `light` | 亮色主题对象 |
| `dark` | 暗色主题对象 |
| `scope` | CSS 变量前缀；不传则直接输出 `--key` |

| 返回 | 说明 |
| --- | --- |
| `cssVars` | `ComputedRef`，亮/暗切换时自动重算 |

## 3. 展平规则

| 输入 | `scope: 'room-detail'` 输出 |
| --- | --- |
| `{ player: { bgColor: '#fff' } }` | `--room-detail-player-bgColor` |
| `{ titleColor: '#000' }` | `--room-detail-titleColor` |
| `{ top: { left: { color: '#fff' } } }` | `--room-detail-top-left-color` |

- 连接符是 `-`，但**键名本身保留 camelCase**；CSS 里必须写 `var(--room-detail-player-bgColor)`，不能写 `bg-color`。
- `scope` 为空时直接输出 `--player-bgColor`。
- 数组不会展开，会作为值写入（CSS 通常无效）；只传字符串。
- 数字在类型上不允许；需要数字时先 `String(value)`。

## 4. 暗色来源

源码依赖项目的 `useDarkMode()`：

```ts
const isDark = useDarkMode(); // route '/dashboard' 强制暗色 + appStore.isDark
const cssVars = computed(() => {
	const vars = isDark.value ? darkVars : lightVars;
	// flatten...
});
```

这个依赖是**业务耦合**，公共版本必须改成注入：

```ts
import { computed, toValue, type MaybeRefOrGetter } from 'vue';

export function useInlineCssVars(options: {
	light: ThemeObject;
	dark: ThemeObject;
	scope?: string;
	isDark: MaybeRefOrGetter<boolean>;
}) {
	const cssVars = computed(() => {
		const vars = toValue(options.isDark) ? options.dark : options.light;
		return flattenToCssVars(vars, options.scope);
	});
	return { cssVars };
}
```

`flattenToCssVars` 注意：

- 只递归普通对象；数组、日期、`Map`、`Set` 作为叶子值处理。
- `null` / `undefined` 值当前源码会原样写入结果对象；Vue 绑定到 `:style` 时通常按空值处理，不会生成有效 CSS 变量。公共版本应显式跳过或转成 `''`。
- 键名不转 kebab-case，保持与源码一致，避免既有 CSS 失效。

## 5. 与组件主题系统的区别

- `useInlineCssVars` 是**局部换肤**：变量挂在某个容器上，只影响该子树。
- `useTheme`（已移出本 skill）是**全局换肤**：改 `html` class、Element Plus 主色、侧边栏/头部变量，属于业务主题系统。
- 两者可以组合：全局主题决定亮暗，局部变量决定区块配色。

## 6. 常见坑

1. **变量名大小写**：`bgColor` 不会变成 `bg-color`；CSS 里写错就取不到值。
2. **数组**：`{ colors: ['red', 'blue'] }` 会生成 `--scope-colors: red,blue`，不能作为颜色列表使用。
3. **数字**：类型只接受字符串；传数字运行时会被写成字符串，但类型检查会报错。
4. **暗色来源写死**：直接复制源码会依赖项目的 `useDarkMode` / app store；公共版本必须注入 `isDark`。
5. **`scope` 冲突**：同一页面多个区块用同一个 scope 会互相覆盖；scope 应按组件/模块唯一。
6. **响应性**：`cssVars` 是 computed，绑定 `:style` 即可；不要解构成普通对象。

## 7. 提取检查

- [ ] `isDark` 由参数注入，未 import 业务 store/route。
- [ ] `flattenToCssVars` 不修改输入对象。
- [ ] 只递归普通对象，数组/日期/null 有明确策略。
- [ ] 键名保持 camelCase，文档写清 CSS 变量名。
- [ ] 有单测覆盖亮/暗切换与嵌套展平。
