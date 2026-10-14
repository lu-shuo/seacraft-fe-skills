---
name: seacraft-react-skills
description: Company React frontend development baseline covering React, Next.js, React Native, component composition, performance, and View Transition patterns.
license: MIT
metadata:
  author: seacraft
  version: "0.1.0"
---

# Seacraft React 前端开发基础规范

这是公司的 React 前端基础 skill，适用于 React、Next.js、React Native、Expo、View Transition、组件设计和性能优化相关任务。

## 使用原则

- 先确认项目是 React Web、Next.js 还是 React Native/Expo。
- React Web 默认遵循组件可组合、状态最小化、避免不必要 re-render 和 bundle 增长。
- 只有在真实需求存在时才增加 animation、View Transition、dynamic import 或其他优化。
- 性能优化应有明确的用户路径、代码位置或构建产物证据。

## 按场景读取参考文档

- React / Next.js 性能：[`react-performance-best-practices.md`](references/react-performance-best-practices.md)
- React 组件组合模式：[`react-composition-patterns.md`](references/react-composition-patterns.md)
- React Native / Expo：[`react-native-best-practices.md`](references/react-native-best-practices.md)
- React View Transition：[`react-view-transitions.md`](references/react-view-transitions.md)

## 完成前检查

- 已确认框架和运行时。
- 组件 API 没有通过大量 boolean props 表达互斥模式。
- 异步请求、bundle、render 和 state 优化有明确证据。
## 组件实现后格式化检查

- 组件实现完成后，检查项目中是否已有 `Prettier`、`ESLint` 和 `Stylelint` 配置。
- 如果存在，按项目现有规则格式化本次修改的文件，并修复格式化、lint 和 stylelint 相关错误。
