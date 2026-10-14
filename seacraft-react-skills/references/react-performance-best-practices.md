---
name: react-performance-best-practices
description: Chinese company baseline for React and Next.js performance optimization.
license: MIT
metadata:
  based-on: react-performance-best-practices
---

# React Performance Best Practices

适用于编写、review 或重构 React / Next.js 代码，尤其是 data fetching、bundle、render 和 performance 任务。

## 优先级

1. 消除 waterfalls：`async-`
2. Bundle size：`bundle-`
3. Server-side performance：`server-`
4. Client-side data fetching：`client-`
5. Re-render optimization：`rerender-`
6. Rendering performance：`rendering-`
7. JavaScript performance：`js-`
8. Advanced patterns：`advanced-`

## 关键规则

- 独立异步任务使用 `Promise.all()` 并行执行；把 `await` 延迟到真正使用结果的分支。
- API route 尽早创建 promise、尽晚 await；使用 Suspense 处理可流式展示的边界。
- 避免 barrel import，优先使用可静态分析的直接路径。
- 大型或低频组件使用 dynamic import；第三方 analytics/logging 延迟到 hydration 之后。
- 服务端 action 与 API route 一样需要 authentication。
- 使用 `React.cache()` 做 request 内 dedupe；跨 request cache 需要明确失效策略。
- 不要在 RSC/SSR module scope 保存可变的 request state。
- 不要订阅 callback 不会读取的 state；使用 primitive dependencies。
- 派生 state 在 render 中计算，不要用 effect 再写回 state。
- 复杂初始化使用 `useState(() => initialValue)`；频繁 transient value 优先使用 `useRef`。
- 不要在 component 内声明 component，避免每次 render 产生新 identity。
- 静态 JSX 提升到 component 外；长列表评估 `content-visibility`。
- 条件渲染不要依赖会意外渲染 `0` 的 falsy `&&`，需要明确的 ternary 或条件。
- JavaScript 热路径中使用 `Map` / `Set`、early exit、缓存 property access 和 function result。
- 不要为了简单 primitive 表达式滥用 `memo`。

## 工作方式

先确认实际 bottleneck，再选择对应 prefix 的规则。性能建议必须能对应到数据、用户路径、代码位置或构建产物，不能只凭规则名改代码。
