---
name: vue-debug-guides
description: Chinese company baseline for diagnosing Vue 3 runtime errors, warnings, async failures, reactivity issues, and SSR/hydration problems.
license: MIT
metadata:
  based-on: vue-debug-guides
---

# Vue Debug Guides

诊断 Vue 问题时先复现并定位实际数据流，再修改代码。不要用临时 `nextTick`、额外 watcher 或强制 remount 掩盖根因。

## Reactivity

- 检查 `ref` 在 script 中是否正确访问 `.value`。
- reactive object 解构时使用 `toRefs` 或保持对象引用，避免丢失响应式。
- 数组、`Map`、`Set` 中的 ref 不会像 template 顶层 ref 一样自动 unwrap。
- 不要依赖 reactive proxy 与原始对象的直接 identity comparison。
- 第三方 class instance、DOM 节点等非响应式对象可使用 `markRaw`。
- 同一 tick 内多个变更会被 batching，检查 watcher 的 flush 时机。
- `computed` getter 不应修改 state、发请求或产生其他副作用。
- 不要修改 readonly computed 的返回值，也不要在 computed 中原地 `sort` / `reverse` 原始数组。

## Watcher 与生命周期

- watch source 必须是正确的 ref、reactive object 或 getter。
- async watcher 需要处理 stale result 和 cleanup。
- 不要在 async callback 中创建未自动绑定到组件生命周期的 watcher。
- `watchEffect` 只会追踪第一个 `await` 之前访问的依赖。
- 需要读取最新 DOM 时使用合适的 `flush` 或 `nextTick`。
- event listener、timer、subscription 和请求取消逻辑必须在卸载时清理。
- 生命周期 hook 应同步注册；DOM 访问应发生在 mount 之后。
- SSR 场景不要在共享 module state 中保存请求级用户数据。

## Components、Props 与 Emits

- component ref 需要子组件显式 `defineExpose` 才能暴露 script setup 中的内容。
- `defineProps`、`defineEmits`、`defineModel` 等宏应遵循 Vue 的 top-level 和 type/runtime 限制。
- 所有自定义事件都应声明，避免 native listener 与 component emit 产生重复触发。
- multi-root component 要明确 class、attrs 和 event 的 fallthrough 行为。
- 处理 component naming collision 和 recursive component 的 name 配置。
- `provide/inject` 必须在同步 setup 中注册；注入的 reactive value 不会自动响应，除非 provider 提供 ref/reactive。

## Template 与表单

- template expression 只能写表达式，复杂分支和副作用放到 script。
- `v-else` 必须紧跟 `v-if` / `v-else-if`。
- 避免 `v-if` 与 `v-for` 同时使用。
- `v-for` 使用稳定且唯一的 `key`；range 从 1 开始。
- template ref 可能为 `null`，尤其是在 `v-if`、异步渲染和卸载之后。
- `v-model` 是组件或表单的状态 contract，不会简单等价于所有原生 HTML 属性。
- textarea 使用 `v-model`，不要依赖插值设置初始内容。
- 中文、日文输入需要注意 IME composition；`.number` 可能把空输入保留为 empty string。
- passive event listener 不能同时阻止默认行为。

## Async、Transition、Teleport 与 Suspense

- async component 需要明确 loading、error、timeout 和 retry 行为。
- KeepAlive、Transition 和 Router 嵌套时检查重复 mount、缓存和 cleanup。
- `TransitionGroup` 中的元素应满足 FLIP 动画的布局要求。
- JavaScript transition hook 必须在异步动画完成时调用 `done`。
- Teleport target 必须存在，并单独检查 SSR hydration 和 scoped style 限制。
- Suspense 不自动提供所有 error handling；异步错误需要由应用边界处理。

## SSR 与 TypeScript

- 服务器和客户端首次 render 必须保持一致，避免 browser-only API 直接进入 SSR 路径。
- 模板 ref、boolean prop、imported `defineProps` type 和 dynamic component ref 都要显式处理 TypeScript 的 nullable 与 inference。
- 发现 warning 时优先修复数据流和类型 contract，不要通过 `as any` 消除信号。
