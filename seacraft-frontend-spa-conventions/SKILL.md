---
name: seacraft-frontend-spa-conventions
description: Company-wide frontend SPA file-placement and directory-layering convention, framework-agnostic (Vue, React, Svelte, any bundler), covering single-app and multi-app repositories (PC + mobile, admin + portal). Use before creating, moving, or renaming any frontend file - pages, components, composables, API clients, stores, types, utilities, styles, tests - to decide which layer it belongs to, how to name it, and whether cross-layer imports are allowed. Also use when reviewing code for directory-structure violations.
license: MIT
metadata:
  author: seacraft
  version: "0.1.0"
---

# Seacraft 前端 SPA 文件落位规范

## 适用范围

适用：Seacraft 所有浏览器端 SPA / C 端 web 应用，单应用或一仓多应用仓库；技术栈不限 Vue、React、Svelte 或自研框架。

不适用：只改样式值、文案、接口参数等不新增文件的任务；SSR 框架的服务端专用目录（server actions、loader 等）需按框架约定在本规范之上叠加。

分工：本 skill 只约束**文件落位与目录分层**。组件写法、响应式、状态管理与性能等实现规范分别见 `seacraft-vue-skills` 和 `seacraft-react-skills`；两者冲突时，目录结构以本 skill 为准，代码写法以框架 skill 为准。

## 三条底层原则

先记这三条，再查表和目录。绝大多数落位争议都由它们裁决。

1. **依赖单向**：页面 → 跨页共享层 → 基础层。基础层（utils / api / store / config）永远不得反向 import 页面、布局或业务组件。
2. **就近优先**：默认放在使用点旁边（页面目录内）。只有出现**第二个真实使用方**时才提升到共享层。"以后可能复用"不算理由。
3. **一个职责一个家**：同一职责不并存两个目录（`component` 与 `components`、`utils` 与 `helpers`）。拿不准时跟随仓库既有惯例，不要引入第二种风格。

## 通用分层模型

与技术栈无关，只定义"层"，不绑定具体目录名。

| 层 | 职责 | 典型落点 | 允许被谁引用 |
| --- | --- | --- | --- |
| L0 入口与构建 | 应用入口、路由表、构建配置、环境变量 | 入口文件、`router/`、`vite.config.*`、`build/`、`.env*` | 全部 |
| L1 页面 | 路由可达页面，一个业务模块一个目录 | `views/<模块>/` 或 `pages/<模块>/` | 路由 |
| L2 页面私有 | 只服务单个页面的组件/逻辑/类型/样式 | `<页面>/{components,composables,utils,interface,*}.{vue,tsx,scss}` | 该页面 |
| L3 跨页共享 | ≥2 处复用的业务组件与业务纯逻辑 | `components/<业务组件>/`、`shared/<域>.ts`、`features/<feature>/` | 页面、布局 |
| L4 基础层 | 请求、状态、通用工具、全局配置、枚举、指令、国际化 | `api/`、`store/`、`utils/`、`hooks/`、`config/`、`enums/`、`directives/`、`locales/` | 上层任意，**不反向** |
| L5 资源与样式 | 参与打包的资源、全局样式体系 | `assets/`、`styles/` | 全部 |
| L6 测试与工具 | 测试、脚本、本地调试沙箱 | `tests/`、`scripts/`、`dev/` | — |
| L7 不打包资源 | 部署期可替换的文件与配置 | `public/`、`static/` | 运行时 |

## 落位决策树

新增文件前按顺序问，命中即停。

1. **是构建/工具链配置？** → 仓库根或 `build/`，不要塞进 `src`。
2. **是入口或路由表？** → 入口文件与 `router/` 目录，不要在页面文件里定义路由。
3. **是路由页面？** → L1 页面目录，一模块一目录，同时补路由配置。
4. **只被一个页面使用？** → L2 页面内：UI 放 `components/`，纯函数放 `utils/`，依赖响应式/生命周期的逻辑放 `composables/`（React 放页面内 `hooks/`），类型放 `interface/`。
5. **多页复用且含业务语义？** UI → L3 `components/<组件名>/`；纯逻辑 → L3 `shared/<域>.ts`。
6. **多页复用但不含业务语义？** 普通函数 → L4 `utils/`；依赖响应式/生命周期 → L4 `hooks/`。
7. **是数据形状？** 接口出入参 → `api/models`；跨页枚举 → `enums`；全局声明 → `types/`；单页专用 → 页面 `interface/`。
8. **是服务端数据还是客户端状态？** 服务端 → `api/`（或 `services/`）；客户端状态 → `store/`。两者不互相内联。
9. **是静态资源？** 参与打包 → `assets/`；部署后可替换 → `public/` 或 `static/`。
10. **是测试？** → 仓库约定的测试根目录（多数为根级 `tests/`），不要新开第二处测试位置。

命中不了任何一条，说明职责没想清：先拆，或就地放在使用点，**不要**新建 `misc.ts`、`common2.ts`、`temp/` 之类的垃圾桶。

## 目录命名映射

同一层在不同生态里的常见叫法。以仓库路由与别名配置为准，不强行改名。

| 层 | Vue 生态 | React 生态 | 备注 |
| --- | --- | --- | --- |
| 页面 | `views/` 或 `pages/` | `pages/` 或 `app/` | 以路由指向为准 |
| 页面私有逻辑 | `composables/` | 页面内 `hooks/` | 全局才进根级 `hooks/` |
| 跨页组件 | `components/<Name>/index.vue` | `components/<Name>/index.tsx` | 组件目录 PascalCase + index 入口 |
| 请求 | `api/modules` + `api/models` | `services/` 或 `api/` | 请求函数与出入参类型分离 |
| 状态 | `stores/modules/<域>.ts` | `store/<域>.ts` | 一域一文件 |
| 类型 | `types/` + `*.d.ts` | 同 | 与框架无关 |
| 样式 | `styles/*.scss` + 组件 scoped | `styles/*.css` + CSS Modules / 原子类 | 全局样式只从入口引 |

## 硬性规则

1. **禁止反向依赖**：`utils` / `api` / `store` / `config` 不得 import 页面、布局、业务组件。
2. **禁止跨页面引用内部文件**：`views/a/...` 不能被 `views/b` 直接 import；提升到 L3。
3. **禁止手改生成物**：自动导入声明（`auto-import.d.ts`、`components.d.ts`）、OpenAPI/GraphQL 生成的 client 与类型、路由自动生成文件。
4. **禁止垃圾桶文件**：`misc`、`common`、`helpers`、`utils2`、`temp/`、`new/`。
5. **禁止同义目录并存**：`component` 与 `components`、`interface` 与 `interfaces`、`hook` 与 `hooks` 只留一个。
6. **服务端数据不经 store 中转**：请求收敛在 api 层；store 只放客户端状态（UI 状态、会话、持久化偏好）。
7. **请求不写在页面/组件里**：不裸调 fetch/axios；鉴权、错误提示、loading 由请求层统一处理。
8. **单页专用组件不进全局组件目录**：全局目录会被自动注册或打进公共包，造成体积与命名污染。
9. **全局样式只在入口引入**：页面与组件样式必须限定作用域（scoped / CSS Modules / 原子类）。
10. **资源引用走别名**：不写 `../../../assets/...`；别名以 `tsconfig paths` 与构建配置为准。
11. **新增复杂文件前先对齐既有同类文件**：命名、导出方式、类型组织保持一致，不引入第二种风格。
12. **多应用仓库各应用独立**：别名、构建、依赖互不越界；共享代码抽成 `packages/`，或各应用各存一份，禁止用相对路径穿透到另一个应用。

## 命名基线

仓库已有约定优先；新仓库无约定时按此执行，且全仓统一。

- 页面目录：kebab-case 或 camelCase，二选一全仓统一。
- 组件目录与组件文件：PascalCase；目录入口统一 `index.*`；引用写到目录层级。
- 组合式函数 / hook：`useXxx`；请求函数：`xxxApi` 或 `getXxx` / `fetchXxx`，二选一。
- 常量：`UPPER_SNAKE_CASE`；类型与接口：PascalCase，按文件或 namespace 聚合。
- 组件事件：kebab-case；CSS 类：BEM 或原子类，二选一。
- 测试文件：`<被测模块>.test.ts`，位置以仓库测试根为准。

## 接管陌生仓库的适配流程

首次在某个仓库使用本规范时先做这一步，再动手写文件。

1. 读 `package.json`：判断单应用/多应用、框架、包管理器、是否 workspaces。
2. 读别名：`tsconfig` 的 `paths`、`vite/webpack` 的 `alias`，记下可用别名。
3. `ls` 根目录：识别 `apps/` `packages/` 还是平铺结构。
4. 抽样 2~3 个既有页面：归纳页面内部分层（有无 `components` / `composables` / `utils` / `interface`）。
5. 抽样 1 个请求模块 + 1 个 store：归纳请求封装与状态管理约定。
6. 把结论整理成该仓库的「落位映射表」，建议写入仓库 `AGENTS.md` 或 `CONTRIBUTING.md`，后续复用。
7. 冲突时：**仓库既有惯例优先**，除非命中下面的"必须纠正"清单。

必须纠正（即使仓库现状如此，也要在新代码中纠正、并建议单独 PR 修复）：

- 反向依赖、跨页面 import 内部文件；
- 手改生成物；
- 单页组件污染全局组件目录；
- 测试散落在业务目录（除非仓库统一 colocate 且有测试配置）；
- 同一职责并存两套目录。

注意：存量违规目录不要顺手大规模重构。新代码按基线走，迁移单独开 PR，避免混在功能提交里。

## 交付前自检

- [ ] 文件落在正确层级，未新建同义目录；
- [ ] 依赖方向正确，无反向 import、无跨页面内部引用；
- [ ] 复用性有真实第二使用方，或已明确标注待提升；
- [ ] 请求 / 状态 / 类型各归其位，页面内无裸请求；
- [ ] 未修改生成物；lint 与 typecheck 通过；
- [ ] 命名与仓库既有实践一致；
- [ ] 测试位置符合仓库约定。

## 参考文件

- `references/directory-baseline.md`：完整目录基线（单应用 / 多应用 / 小项目裁剪版三套），确定层级边界与目录职责时读。
- `references/placement-examples.md`：9 个场景的落位示例与反例集（Vue / React 混合），拿不准时对照。
