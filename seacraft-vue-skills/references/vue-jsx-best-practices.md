---
name: vue-jsx-best-practices
description: Chinese company baseline for Vue JSX and render-function syntax, including differences from React JSX.
license: MIT
metadata:
  based-on: vue-jsx-best-practices
---

# Vue JSX Best Practices

- 只有项目已经选择 JSX 或 template 无法表达需求时才使用 Vue JSX / render function。
- 不要直接把 React JSX 语法复制到 Vue JSX。
- Vue JSX 使用 `class`，不要默认替换为 `className`。
- Vue JSX 的事件、`v-model`、slots、directives 和 component props 应遵循 Vue JSX plugin 的配置与项目现有写法。
- JSX 中的响应式值、条件渲染和列表渲染仍需遵循 Vue 的响应式与 key 规则。
- 需要迁移 React JSX 或排查 attribute type error 时，先对照 Vue JSX 与 React JSX 的差异，再修改调用方式。
