---
name: vue-best-practices
description: Chinese company baseline for Vue 3 architecture, Composition API, SFC structure, component data flow, composables, optional built-ins, and performance.
license: MIT
metadata:
  based-on: vue-best-practices
---

# Vue Best Practices

## 核心原则

- 保持 state 可预测：只保留必要的 source state，其余内容使用 `computed` 派生。
- 让数据流显式：大多数场景遵循 props down、events up。
- 组件保持单一职责，避免一个组件同时负责数据编排、复杂状态和多个独立 UI 区域。
- 使用 `watch` 处理副作用，不要把副作用放进 `computed`。
- 代码应清晰、自解释，避免为了抽象而抽象。

## 开始编码前

1. 确认项目使用 Vue 3、Composition API、Options API 还是 JSX。
2. 默认采用 `<script setup lang="ts">`。
3. 非简单功能先画出组件 map：每个组件写一句职责说明，并定义 props/emits contract。
4. route-level view 和 root component 默认作为组合层，只负责 app shell、provider wiring 和 feature composition。
5. 多组件功能优先使用 `components/<feature>/...` 与 `composables/use<Feature>.ts` 的目录结构。

## 响应式与模板

- 使用 `ref` / `reactive` 保存最小状态，使用 `computed` 派生数据。
- 不要在 template 中重复执行昂贵计算。
- 保持 template declarative，把分支、转换和复杂判断移到 script。
- SFC 默认保持 `<template>`、`<script>`、`<style>` 顺序。
- 只有在确实需要时才使用 `v-html`、自定义 directive、render function 或插件。
- 不要混用 `v-if` 与 `v-for`；先通过 `computed` 得到过滤后的列表。

## 组件边界

出现以下任一情况时，考虑拆分组件：

- 同时负责 orchestration/state 和多个大块 presentational markup。
- 包含 3 个或更多独立 UI 区域，例如 form、filters、list、footer/status。
- template 中出现可复用的重复块，例如 item row、card 或 list entry。

CRUD、table、catalog、inbox 等功能默认拆成 feature container、input/form、list/item 和 footer/actions 或 filter/status。

## 数据流

- 子组件通过 `defineProps` 接收数据，通过 `defineEmits` 通知父组件。
- 只有真正的双向组件 contract 才使用 `v-model`。
- 只有深层依赖或共享 context 才使用 `provide/inject`，并用 `InjectionKey` 保持类型安全。
- 不要通过隐式的全局状态或未声明事件绕过组件 contract。

## Composable

- 可复用、有状态或副作用较多的逻辑放入 composable。
- composable API 保持小、稳定、有类型。
- 将 feature logic 与 presentational component 分离。
- 需要接受 plain value、ref 或 getter 时，读取 [`create-adaptable-composable.md`](create-adaptable-composable.md)。

## 可选能力

- 插槽控制子组件内容或布局时，使用 slots。
- wrapper/base component 需要转发 attrs 或 events 时，处理 fallthrough attrs。
- 需要保留视图状态时使用 `<KeepAlive>`。
- overlay 或 portal 场景使用 `<Teleport>`。
- async subtree 需要 fallback boundary 时使用 `<Suspense>`，并确认 error handling。
- 进入/离开动画使用 `<Transition>`，列表变化动画使用 `<TransitionGroup>`。
- 大量低频 UI 使用 async component。
- 应用跨 feature 共享状态时再引入 Pinia 或其他 state management。

## 性能

性能优化放在功能正确之后：

- 大列表优先评估 virtualize。
- 静态子树可评估 `v-once` / `v-memo`。
- 热路径中避免不必要的 component abstraction。
- 避免不稳定 props 导致子组件重复更新。
- 使用 `updated` hook 前先确认是否能用更直接的响应式建模解决问题。

## 完成前检查

- 响应式模型最小且可预测。
- SFC 结构和 template 规则符合项目约定。
- 组件拆分理由清楚，route view 没有承载完整 feature 实现。
- props/emits/composable contracts 有类型。
- optional feature 只在需求存在时使用。
