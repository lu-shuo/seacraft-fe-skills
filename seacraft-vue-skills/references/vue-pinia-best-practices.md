---
name: vue-pinia-best-practices
description: Chinese company baseline for Pinia store setup, state ownership, reactivity, URL filters, and application-scale state management.
license: MIT
metadata:
  based-on: vue-pinia-best-practices
---

# Pinia Best Practices

## Store 初始化

- 在 app mount 前创建 Pinia，并通过 `app.use(pinia)` 安装。
- 测试、SSR 和独立 composable 中要明确提供 active Pinia 或测试 Pinia。
- setup store 必须 return 所有需要被 DevTools、SSR 或其他消费者识别的 state。

## Reactivity

- 不要直接 destructure store 导致响应式丢失；需要拆分时使用 `storeToRefs`。
- actions 可以直接通过 store 调用；不要把 method 当作普通 callback 传递而丢失上下文。
- store state、getters 和 actions 的职责保持清晰。

## State 设计

- 只有跨组件、跨 route 或跨 feature 的共享 state 才进入 Pinia。
- 页面临时 filter、排序和分页参数如果需要刷新保留、分享或可回退，应优先考虑 URL。
- 不要把纯 UI 局部 state 全部提升到 store。
- 大型生产应用统一 store 命名、目录和测试约定，并保持 DevTools 可追踪。
