---
name: create-adaptable-composable
description: Chinese company baseline for Vue composables that accept plain values, refs, shallow refs, computed refs, or getters.
license: MIT
metadata:
  based-on: create-adaptable-composable
compatibility: Requires Vue 3 or above, or Nuxt 3 or above.
---

# Create Adaptable Composable

当 composable 需要同时接受 plain value、ref、computed ref 或 getter 时，使用 adaptable composable 设计。

## 设计步骤

1. 明确 composable 的职责、输入、输出和副作用。
2. 标记哪些输入需要保持 reactive。
3. 在 reactive effect 内使用 `toRef()` 或 `toValue()` 归一化输入。
4. 使用 Vue reactivity API 实现核心逻辑，并保持 API 小而稳定。

## 类型选择

```ts
import type {
  ComputedRef,
  Ref,
  ShallowRef,
  WritableComputedRef,
} from 'vue'

export type MaybeRef<T = any> =
  | T
  | Ref<T>
  | ShallowRef<T>
  | WritableComputedRef<T>

export type MaybeRefOrGetter<T = any> =
  | MaybeRef<T>
  | ComputedRef<T>
  | (() => T)
```

- 只读或 computed-friendly 输入使用 `MaybeRefOrGetter`。
- 需要可写或双向更新的输入使用 `MaybeRef`。
- 如果参数本身可能是 callback、predicate 或 comparator，不要使用 `MaybeRefOrGetter`，否则可能误把函数执行为 getter。
- DOM 或 element target 需要支持派生值时，可使用 `MaybeRefOrGetter`。

## 归一化规则

- watcher source 使用 `toRef()`。
- 只需要读取当前值时使用 `toValue()`。
- 不要在 composable 外部强制调用方先把输入转换为 ref。

```ts
import { toRef, watch } from 'vue'
import type { MaybeRefOrGetter } from 'vue'

export function useDocumentTitle(title: MaybeRefOrGetter<string>) {
  watch(toRef(title), (value) => {
    document.title = value
  }, { immediate: true })
}
```
