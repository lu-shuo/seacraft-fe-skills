---
name: react-view-transitions
description: Chinese company baseline for React View Transition API, shared elements, route changes, Suspense reveals, and reduced-motion handling.
license: MIT
metadata:
  based-on: react-view-transitions
---

# React View Transitions

使用 browser native `document.startViewTransition` 在 React UI state 之间建立连续性。用 `<ViewTransition>` 声明 what，用 `startTransition`、`useDeferredValue` 或 `Suspense` 触发 when，用 CSS classes 控制 how。

## 何时使用

每个 `<ViewTransition>` 都应表达空间关系或连续性。不能说明它表达什么时，不要添加动画。

优先考虑：

1. shared element：同一个对象进入更深层页面。
2. Suspense reveal：数据加载完成。
3. list identity：同一批 item 重新排序。
4. state change：内容出现或消失。
5. route change：前往新页面。

层级导航使用 type-keyed `nav-forward` / `nav-back`；tab-to-tab 等横向导航使用 fade 或 `default="none"`。有顺序的 carousel、上一张/下一张和分页结果可以使用方向动画。

## 运行条件

- Next.js App Router 不要额外安装 `react@canary`，由 App Router 提供内部版本。
- 非 Next.js 项目根据当前 React 版本安装对应的 canary 依赖。
- 支持情况必须允许 graceful degradation；不支持的 browser 应跳过动画。

## 实现规则

- React 自动管理 `view-transition-name`，不要自己调用 `startViewTransition`。
- 普通 `setState` 不会触发 View Transition，必须使用支持的 transition trigger。
- `<ViewTransition>` 要出现在目标 DOM 节点之前，否则 enter/exit 可能不生效。
- 默认使用 `default="none"`，只显式打开需要的 `enter`、`exit`、`share` 或 `update`。
- shared element 的 `name` 同一时间只能有一个匹配实例，必要时使用 `photo-${id}` 等唯一 name。
- shared morph 页面不要使用普通 fade-out exit，避免与 morph 冲突。
- list identity 和 shared element 是两种独立关系，需要时使用嵌套的两个 `<ViewTransition>`。
- `router.back()` 和 browser back/forward 不应假设会触发 transition；需要时用显式 URL 和 `router.push()`。
- nested VT 中，父 VT exit 时子 VT 不会各自触发 enter/exit。

## Accessibility

全局 stylesheet 必须提供 reduced-motion CSS。动画应增强信息层级，不能成为完成任务的必要条件。
