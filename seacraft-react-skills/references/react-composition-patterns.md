---
name: react-composition-patterns
description: Chinese company baseline for scalable React composition, compound components, context providers, explicit variants, and React 19 APIs.
license: MIT
metadata:
  based-on: react-composition-patterns
---

# React Composition Patterns

## 适用场景

- component 有大量 boolean props。
- 需要构建可复用 component library。
- 需要设计灵活的 component API。
- 使用 compound components、context provider 或 render-related pattern。

## 规则

- 不要通过不断增加 boolean props 表达多个模式，优先使用 composition。
- 复杂组件可使用 compound components，并通过共享 context 协调内部 parts。
- provider 是唯一知道 state 如何管理的地方，消费者依赖稳定的 context interface。
- 需要让兄弟组件访问 state 时，把 state 提升到 provider。
- 用显式 variant components 替代互斥的 boolean mode。
- 优先使用 `children` 进行 composition，不要为了替代 composition 创建大量 `renderX` props。
- 状态 interface 可拆为 state、actions 和 meta，便于 dependency injection 和测试。
- React 19 项目遵循 React 19 API 约定；不要把 React 19-only 写法带入 React 18 项目。

组件 API 的目标是让使用方组合能力，而不是让一个组件承担所有分支。
