---
name: seacraft-vue-skills
description: Company Vue frontend development baseline based on the installed Vue best-practice, debugging, router, Pinia, testing, JSX, Options API, and adaptable composable skills.
license: MIT
metadata:
  author: seacraft
  version: "0.1.0"
---

# Seacraft Vue 前端开发基础规范

这是公司的 Vue 前端基础 skill，适用于 Vue 3、`.vue` 文件、Vue Router、Pinia、Vite + Vue、TypeScript、Vue Test Utils、Vitest 和 Playwright 相关任务。

## 使用原则

- 默认使用 Vue 3 + Composition API + `<script setup lang="ts">`。
- 项目已有明确的 Options API 或 JSX 约定时，遵循项目现状，并读取对应参考文档。
- 优先保持数据流显式：props down、events up。
- 保持状态来源单一，使用 `computed` 派生状态，使用 `watch` 处理副作用。
- 先保证行为正确，再处理性能优化。
- 组件、composable、store 和 route guard 的边界应清晰、可测试。
- 先阅读当前项目的现有模式，再引入新的抽象或依赖。

## 按场景读取参考文档

- Vue 组件结构、响应式、数据流和性能：[`vue-best-practices.md`](references/vue-best-practices.md)
- Vue 运行时错误、warning、异步问题和 SSR/hydration：[`vue-debug-guides.md`](references/vue-debug-guides.md)
- Vue JSX / render function：[`vue-jsx-best-practices.md`](references/vue-jsx-best-practices.md)
- Options API：[`vue-options-api-best-practices.md`](references/vue-options-api-best-practices.md)
- Pinia：[`vue-pinia-best-practices.md`](references/vue-pinia-best-practices.md)
- Vue Router：[`vue-router-best-practices.md`](references/vue-router-best-practices.md)
- Vue 测试：[`vue-testing-best-practices.md`](references/vue-testing-best-practices.md)
- 可接受 plain value、ref 或 getter 的 composable：[`create-adaptable-composable.md`](references/create-adaptable-composable.md)

## 完成前检查

- 需求对应的参考文档已读取。
- 没有把可派生状态重复存储为独立 source of truth。
- 组件 props、emits、`v-model` 和 `provide/inject` contracts 清晰且有类型。
- 异步副作用、watcher、event listener 和资源已正确清理。
- 测试覆盖了用户可观察行为，而不是只锁定内部实现。
- 没有为了预期中的需求增加无用 abstraction。
## 组件实现后格式化检查

- 组件实现完成后，检查项目中是否已有 `Prettier`、`ESLint` 和 `Stylelint` 配置。
- 如果存在，按项目现有规则格式化本次修改的文件，并修复格式化、lint 和 stylelint 相关错误。
