---
name: vue-router-best-practices
description: Chinese company baseline for Vue Router 4 routes, navigation guards, route params, and route-component lifecycle.
license: MIT
metadata:
  based-on: vue-router-best-practices
---

# Vue Router Best Practices

## Navigation Guards

- `beforeEnter` 不会因为同一路由的 params 变化而重新触发；需要响应 params 变化时使用组件内 watcher、`onBeforeRouteUpdate` 或重新设计 route。
- `beforeRouteEnter` 中不能直接访问 `this`，需要通过 callback 在组件实例创建后处理。
- guard 中的异步请求必须 `await`，并明确返回 allow、redirect 或 error。
- redirect guard 必须有清晰的终止条件，避免无限循环。
- 新代码优先使用 return-based guard，不要继续使用已废弃的 `next()` 流程，除非项目兼容性明确要求。

## Route Lifecycle

- 同一 route 不同 params 之间切换时，组件可能复用，不能假设完整 lifecycle 会重新执行。
- 监听 params 变化并刷新对应数据，处理请求取消和 stale result。
- route 级 event listener、subscription 和 timer 必须在离开或组件卸载时清理。

## Setup

- 生产 SPA 使用 Vue Router 管理 route，不要通过手写 pathname 分支替代。
- route record、lazy-loaded component、meta 和 guard contract 保持可读、可测试。
- 只在确实需要时使用 nested route、keep-alive 和 transition，避免增加无法解释的生命周期复杂度。
