---
name: vue-options-api-best-practices
description: Chinese company baseline for Vue 3 Options API, TypeScript inference, methods, lifecycle hooks, and common gotchas.
license: MIT
metadata:
  based-on: vue-options-api-best-practices
---

# Vue Options API Best Practices

仅当项目明确使用 Options API 时读取此文档。不要在一个 feature 中无理由混合 Options API 与 Composition API。

## TypeScript

- 使用 `defineComponent` 保持 component properties 的 type inference。
- 在 strict TypeScript 下为 `this`、event handler、object prop 和 array prop 提供明确类型。
- 复杂 computed property 写清楚 return type。
- `provide/inject` 的注入值需要补齐类型，避免在消费侧退化为 `any`。
- prop validator 和默认值使用与当前 TypeScript/Vue 版本兼容的写法。

## Methods 与 Lifecycle

- `methods` 和 lifecycle hooks 使用普通方法函数，以便 Vue 正确绑定 component instance 的 `this`。
- 不要使用 arrow function 代替依赖 `this` 的 methods 或 lifecycle hooks。
- 具有 debounce、throttle 或其他 mutable state 的方法不要在 `data()` 外共享同一个实例。
- 每个组件实例应拥有自己的 stateful method state。

## 维护边界

- data、computed、methods 和 watch 的职责要清晰。
- 不要在 computed 中产生副作用。
- 组件较大时优先迁移或拆分为 composable，而不是继续堆积 Options API 配置。
