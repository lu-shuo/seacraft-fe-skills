---
name: vue-testing-best-practices
description: Chinese company baseline for Vue testing with Vitest, Vue Test Utils, component tests, composables, Pinia, Suspense, Teleport, and Playwright.
license: MIT
metadata:
  based-on: vue-testing-best-practices
---

# Vue Testing Best Practices

## 测试工具

- Vue 组件单元测试默认使用 Vitest + Vue Test Utils。
- 需要验证真实 DOM、样式、浏览器事件或跨页面行为时使用 browser runner 或 Playwright。
- 测试用户可观察行为和公共 contract，不要把 snapshot 当作唯一测试方式。

## 组件测试

- 通过 props、emits、DOM、slots 和可见状态验证组件，而不是依赖内部 ref 或私有 method。
- 异步组件、`async setup` 和 Suspense 测试要等待正确的异步边界。
- Teleport 内容可能挂载在 wrapper 外部，查询时明确 target 和 cleanup。
- `defineAsyncComponent` 测试需要处理加载、成功和失败状态。

## 异步与响应式

- 使用 `await`、`nextTick` 和 `flushPromises` 等待实际更新，不要使用任意 sleep。
- 覆盖 race condition、请求失败和 stale result。
- 测试 composable 时提供必要的 wrapper、生命周期和 inject context。

## Pinia

- 测试中显式创建并安装 testing Pinia 或 active Pinia。
- 验证 store 的 state、getters 和 actions 对组件行为的影响，不要依赖全局测试顺序。

## E2E

- 关键用户流程使用 Playwright 覆盖。
- E2E 不替代单元测试；单元测试关注逻辑和 contract，E2E 关注真实浏览器中的集成行为。

## 测试文件目录结构

- 组件测试应位于项目根目录test/unit文件夹，如果没有test/unit目录，需要先创建。
- 组件测试文件命名格式为`[component name].spec.ts`。
