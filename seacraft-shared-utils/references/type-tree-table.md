# 类型判断、树与表格显示

本文覆盖 `is.ts` / `getType` / `getFlatArr` / `findItemNested` / `handleProp` / `handleRowAccordingToProp` / `filterEnum` / `formatValue`。这些函数在 `template`、`patrol`、`supervision`、`tqmp`、`space` 等后台项目中大量重复，是最稳定的公共 utils。

## 1. 类型守卫

### 引入

```ts
import {
	is,
	isFunction,
	isDef,
	isUnDef,
	isObject,
	isDate,
	isNumber,
	isAsyncFunction,
	isPromise,
	isString,
	isBoolean,
	isArray,
	isClient,
	isWindow,
	isElement,
	isNull,
	isNullOrUnDef,
	isHexColor
} from '@/utils/is';
```

### 用法

```ts
is('abc', 'String'); // true
is([], 'Array'); // true
is([], 'array'); // false（区分大小写）
isObject({}); // true
isObject([]); // false
isObject(new Date()); // false
isElement(document.body); // true
isHexColor('#fff'); // true
isHexColor('#ffffff'); // true
```

### 实现

```ts
export function is(val: unknown, type: string) {
	return Object.prototype.toString.call(val) === `[object ${type}]`;
}

export function isFunction<T = Function>(val: unknown): val is T {
	return is(val, 'Function');
}

export const isDef = <T = unknown>(val?: T): val is T => typeof val !== 'undefined';
export const isUnDef = <T = unknown>(val?: T): val is T => !isDef(val);
export const isObject = (val: any): val is Record<any, any> => val !== null && is(val, 'Object');
export function isDate(val: unknown): val is Date { return is(val, 'Date'); }
export function isNumber(val: unknown): val is number { return is(val, 'Number'); }
export function isString(val: unknown): val is string { return is(val, 'String'); }
export function isBoolean(val: unknown): val is boolean { return is(val, 'Boolean'); }
export function isArray(val: any): val is Array<any> { return val && Array.isArray(val); }
export const isClient = () => typeof window !== 'undefined';
export const isWindow = (val: any): val is Window => typeof window !== 'undefined' && is(val, 'Window');
export const isElement = (val: unknown): val is Element => isObject(val) && !!val.tagName;
export function isNull(val: unknown): val is null { return val === null; }
export function isNullOrUnDef(val: unknown): val is null | undefined { return isUnDef(val) || isNull(val); }
export const isHexColor = (str: string) => /^#?([0-9A-Fa-f]{3}|[0-9A-Fa-f]{6})$/.test(str);
```

### 注意

- `isObject` 只认普通对象；数组、`Date`、`Map` 都是 `false`。
- `isArray` 返回 `val && Array.isArray(val)`，空值会返回 `null` / `undefined` 而不是严格布尔；公共版本改为 `Array.isArray(val)`。
- `isUnDef` 的类型谓词 `val is T` 不准确，建议返回 `boolean`。
- `isAsyncFunction` 返回 `val is Promise<T>`，但判断的是 `AsyncFunction`；不要用它缩窄 Promise。
- `isWindow` 在 SSR 下会因为 `typeof window` 短路，安全。
- 模块顶层不要读 `navigator` / `window`；`is/index.ts` 本身没有副作用，`isMobile` 版本要注意。

## 2. getType

```ts
import { getType } from '@/utils';

getType(null); // 'null'
getType([]); // 'array'
getType(new Date()); // 'date'
getType(/a/); // 'regexp'
getType(new Map()); // 'map'
getType(1); // 'number'
getType(undefined); // 'undefined'
```

实现：

```ts
export function getType(val: any) {
	if (val === null) return 'null';
	if (typeof val !== 'object') return typeof val;
	return Object.prototype.toString.call(val).slice(8, -1).toLowerCase();
}
```

与 `is` 的分工：

| 需求 | 用 |
| --- | --- |
| 判断具体类型 | `is(val, 'Array')` |
| 获取类型字符串（日志、分支） | `getType(val)` |
| 判断普通对象 | `isObject(val)` |

## 3. 树扁平化

### getFlatArr

```ts
import { getFlatArr } from '@/utils';

const flat = getFlatArr(tree);
```

源码：

```ts
export function getFlatArr(menuList: Menu.MenuOptions[]) {
	const newMenuList = JSON.parse(JSON.stringify(menuList));
	return newMenuList.reduce((pre, current) => {
		let flatArr = [...pre, current];
		if (current.children) flatArr = [...flatArr, ...getFlatArr(current.children)];
		return flatArr;
	}, []);
}
```

问题：

- `JSON.parse(JSON.stringify())` 丢失 `undefined`、函数、`Date`、`Map/Set`，循环引用直接抛错。
- 每次都创建新数组，节点被克隆，后续修改不会回写原树。
- 如果只需要读，公共版本不要克隆：

```ts
export function flattenTree<T extends { children?: T[] }>(tree: T[]): T[] {
	const result: T[] = [];
	const walk = (nodes: T[]) => {
		for (const node of nodes) {
			result.push(node);
			if (node.children?.length) walk(node.children);
		}
	};
	walk(tree);
	return result;
}
```

需要保留父链时：

```ts
export interface FlatNode<T> {
	node: T;
	parent?: T;
	level: number;
}

export function flattenTreeWithParent<T extends { children?: T[] }>(
	tree: T[],
	parent?: T,
	level = 0,
	result: FlatNode<T>[] = []
): FlatNode<T>[] {
	for (const node of tree) {
		result.push({ node, parent, level });
		if (node.children?.length) flattenTreeWithParent(node.children, node, level + 1, result);
	}
	return result;
}
```

### findItemNested

```ts
import { findItemNested } from '@/utils';

const target = findItemNested(tree, 3, 'value', 'children');
```

源码：

```ts
export function findItemNested(enumData: any[], callValue: any, value: string, children: string) {
	return enumData.reduce((accumulator: any, current: any) => {
		if (accumulator) return accumulator;
		if (current[value] === callValue) return current;
		if (current[children]) return findItemNested(current[children], callValue, value, children);
	}, null);
}
```

注意：

- 返回 `null` / `undefined`（`reduce` 初值 `null`），调用方要判空。
- 严格相等；`'1'` 与 `1` 不匹配。
- 无循环保护；树有环会栈溢出。
- 只返回第一个匹配节点，不保证唯一。

## 4. 表格显示辅助

### handleProp

```ts
handleProp('user.name'); // 'name'
handleProp('name'); // 'name'
```

用于 Element Plus 表格列 `prop` 是多级路径时，取最后一段作为列 key。

### handleRowAccordingToProp

```ts
handleRowAccordingToProp({ user: { name: '张三' } }, 'user.name'); // '张三'
handleRowAccordingToProp({ user: {} }, 'user.name'); // '--'
handleRowAccordingToProp({ count: 0 }, 'count'); // 0
```

实现：

```ts
export function handleRowAccordingToProp(row: Record<string, any>, prop: string) {
	if (!prop.includes('.')) return row[prop] ?? '--';
	prop.split('.').forEach(item => (row = row[item] ?? '--'));
	return row;
}
```

注意：

- 只按 `.` 分割，不支持数组下标表达式以外的语法（`list.0.name` 可以，因为数组也是对象）。
- 返回 `'--'` 会与真实字符串 `'--'` 混淆；需要区分空值时不要用。
- 局部变量 `row` 被重赋值，不会修改原对象。

### filterEnum

```ts
filterEnum(2, [
	{ value: 1, label: '启用' },
	{ value: 2, label: '停用', tagType: 'danger' }
]); // '停用'

filterEnum(
	2,
	[{ id: 2, name: '停用', type: 'danger' }],
	{ value: 'id', label: 'name' },
	'tag'
); // 'danger'
```

注意：

- `enumData` 不是数组时返回 `'--'`。
- 默认字段 `value` / `label` / `children`；字典字段不同必须传 `fieldNames`。
- `type='tag'` 返回 `tagType`，不是 label。
- 未命中普通模式返回 `'--'`，tag 模式返回 `''`。

### formatValue

```ts
formatValue([1, 2]); // '1 / 2'
formatValue([]); // '--'
formatValue(null); // '--'
formatValue(undefined); // '--'
formatValue(0); // 0
formatValue(false); // false
```

实现依赖 `isArray`：

```ts
export function formatValue(callValue: any) {
	if (isArray(callValue)) return callValue.length ? callValue.join(' / ') : '--';
	return callValue ?? '--';
}
```

注意：返回值不总是字符串，`0` / `false` 会原样返回；模板会隐式转字符串，逻辑判断时要注意。

## 5. 组合示例

```ts
import { getFlatArr, findItemNested, filterEnum, formatValue } from '@/utils';

const flat = getFlatArr(tree);
const target = findItemNested(tree, row.type, 'value', 'children');
const typeLabel = filterEnum(row.type, typeOptions);
const tags = formatValue(row.tags);
```

## 6. 常见坑

1. **`is` 的 type 大小写**：`'Array'` 不是 `'array'`。
2. **`isObject` 不含数组**：需要对象或数组时组合判断。
3. **`isArray` 返回非布尔**：源码 `val && Array.isArray(val)`，空值返回本身。
4. **`getFlatArr` JSON 克隆**：丢 `Date`/函数/`undefined`，循环引用抛错。
5. **`findItemNested` 严格相等**：接口返回字符串、字典是数字时匹配不到。
6. **`handleRowAccordingToProp` 的 `'--'`**：与真实值冲突。
7. **`filterEnum` 的 `tagType`**：tag 模式返回的是颜色类型，不是文案。
8. **`formatValue` 返回非字符串**：不要直接 `.toLowerCase()`。
