# 目录基线（三套形态）

按项目规模与形态选一套，不要混着用。所有目录都是"层"的载体，改名可以，层不能少。

## A. 单应用标准版（推荐起点）

```text
<repo>/
├─ public/                  # 不打包资源：favicon、可替换配置、第三方脚本
├─ src/
│  ├─ main.{ts,tsx}         # 应用入口：创建实例、注册插件、挂载
│  ├─ App.{vue,tsx}         # 根组件
│  ├─ router/               # 路由表 + 守卫（单文件或 modules/ 拆分）
│  ├─ views|pages/          # L1 页面，一个业务模块一个目录
│  │  └─ <module>/
│  │     ├─ index.{vue,tsx} # 页面入口，只做编排
│  │     ├─ components/     # L2 仅本页使用的组件
│  │     ├─ composables|hooks/  # L2 仅本页使用的组合式逻辑
│  │     ├─ utils/          # L2 仅本页使用的纯函数
│  │     ├─ interface/      # L2 仅本页使用的类型
│  │     └─ index.scss      # L2 仅本页使用的样式
│  ├─ components/           # L3 跨页业务组件，一组件一目录，index 入口
│  ├─ shared/               # L3 跨页业务纯逻辑（无框架、无 DOM）
│  ├─ api|services/         # L4 请求层
│  │  ├─ client.{ts}        # 实例与拦截器（鉴权、错误、loading、竞态）
│  │  ├─ modules/           # 请求函数，按业务一文件
│  │  ├─ models/            # 出入参类型，与 modules 一一对应
│  │  └─ types|interface/   # 通用响应体、状态码枚举
│  ├─ store|stores/         # L4 客户端状态，一域一文件
│  ├─ hooks/                # L4 全局组合式逻辑
│  ├─ utils/                # L4 与业务无关的工具函数
│  ├─ config/               # L4 全局配置：请求参数、主题、路由常量、第三方初始化
│  ├─ enums/                # L4 跨页枚举
│  ├─ directives/           # L4 全局指令（Vue）
│  ├─ locales|languages/    # L4 国际化词条与初始化
│  ├─ layouts/              # 布局壳，与页面解耦
│  ├─ assets/               # L5 参与打包的资源：icons / images / fonts
│  └─ styles/               # L5 全局样式：变量、重置、主题、组件库覆盖
├─ tests/                   # L6 单元/集成测试
├─ scripts/                 # L6 构建与运维脚本
├─ dev/                     # L6 本地调试沙箱（仅 dev 环境生效）
├─ build/                   # L0 构建配置拆分（插件、代理、环境变量处理）
├─ types/                   # 全局 TS 声明（declare global、window 扩展）
├─ *.config.{js,ts,cjs}     # L0 构建/校验/提交规范配置
└─ AGENTS.md|CONTRIBUTING.md # 落位约定的落地文本
```

## B. 一仓多应用（PC + 移动端 / admin + portal）

```text
<repo>/
├─ apps/ 或平铺的 pc/ + mobile/
│  ├─ pc/
│  │  ├─ src/               # 层级同 A，别名 @ → pc/src
│  │  └─ vite.config.ts
│  └─ mobile/
│     ├─ src/               # 层级同 A，别名 @mobile → mobile/src
│     └─ vite.config.ts
├─ packages/                # 仅当确有双端共享诉求时才建
│  ├─ shared-utils/         # 纯逻辑，无框架、无 DOM
│  └─ ui/                   # 若两应用同框架才可共享组件
├─ tests/                   # 测试统一在根
└─ package.json             # workspaces 编排各应用
```

关键约束：

- 每个应用独立别名、独立构建、独立依赖。移动端 Vite root 指向自己的目录时，另一个应用的别名根本不存在，跨应用 import 会直接编译失败。
- **禁止**用相对路径穿透到另一个应用（`../../pc/src/...`）。
- 双端要同一份业务规则时，二选一：抽到 `packages/shared-utils` 并以包名引用，或各应用各存一份并保持同名同签名。不要为了"少写一遍"做跨端路径引用。
- 共享包一旦引入，就必须有发布/引用机制（workspace 依赖或构建别名），不能靠 `../../../`。

## C. 小项目裁剪版

页面少于 10 个、单人维护时，可删掉 L2 的多层子目录和 `shared/`，但下列结构必须保留：

```text
src/
├─ main.ts
├─ router.ts
├─ views/            # 页面，页面内直接用同目录文件拆分即可
├─ components/       # 跨页组件
├─ api/              # 请求函数 + 类型（同一文件可合并）
├─ store.ts          # 或 stores/
├─ utils/
├─ styles/
└─ assets/
```

裁剪的是脚手架，不是规则：层间依赖方向、请求不在页面里裸调、生成物不可手改仍然生效。

## 目录职责速查

| 目录 | 装什么 | 不该装什么 |
| --- | --- | --- |
| `views|pages` | 路由可达页面，一模块一目录 | 全局可复用组件、请求实例 |
| `<页面>/components` | 仅本页使用的展示/容器组件 | 跨页复用的组件 |
| `<页面>/composables|hooks` | 仅本页使用的有状态逻辑 | 纯函数（放 `utils/`） |
| `<页面>/utils` | 仅本页使用的纯函数 | 有副作用、依赖响应式的逻辑 |
| `components/` | 跨页业务组件，含自身子组件与样式 | 单页专用、无业务语义的基础控件（进组件库） |
| `shared/` | 跨页业务纯逻辑：字段映射、格式化、规则判定 | Vue/React API、DOM 操作、请求 |
| `api/modules` | 请求函数，一业务一文件 | 类型定义、业务计算 |
| `api/models` | 出入参类型、DTO、分页结构 | 请求函数、默认值常量 |
| `store` | 客户端状态与派生值 | 服务端缓存副本、请求 URL |
| `hooks/` | 全局有状态逻辑（表格、弹窗、主题、权限） | 只服务单页的逻辑 |
| `utils/` | 与业务无关的通用函数 | 业务规则（应进 `shared/`）、全局单例 |
| `config/` | 全局常量与第三方初始化 | 组件、状态 |
| `enums/` | 跨页枚举与字典 | 单页枚举、接口 DTO |
| `types/` | 全局声明、环境变量类型、`declare global` | 业务 DTO |
| `assets/` | 参与打包的图标、图片、字体、雪碧图源 | 部署后要替换的配置 |
| `public|static` | 原样拷贝、运行时可替换的文件 | 需要被构建处理的模块 |
| `tests/` | 单测、集成测试 | 测试 fixture 之外的可执行代码 |
| `dev/` | 本地起独立预览页的沙箱 | 生产入口逻辑 |

## 三个容易放错的判断

1. **"工具函数"还是"业务规则"？** 换个业务就失效 → `shared/`；任何业务都能用 → `utils/`。
2. **"组件"还是"基础控件"？** 含本业务字段/流程 → `components/`；纯交互原子件（按钮、空态、骨架） → 组件库或 `components/base/`。
3. **"类型"还是"常量"？** 描述数据形状 → `models` / `interface`；描述可选项 → `enums` / `config`。
