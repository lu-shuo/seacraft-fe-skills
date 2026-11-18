# Seacraft Frontend Skills

Seacraft 前端开发规范 skill，当前包含 Vue、React 和文件落位规范三部分。文档采用“英文 YAML 元数据 + 中文规范说明 + 英文技术标识”的形式。

## 包含的 Skill

### `seacraft-vue-skills`

适用于：

- Vue 3、`.vue` 文件和 Vite + Vue
- Composition API、`<script setup>` 和 TypeScript
- Vue Router
- Pinia
- Vue JSX / render function
- Options API
- Vue Test Utils、Vitest 和 Playwright
- 可接受 plain value、ref 或 getter 的 composable
- Vue 响应式、组件数据流、调试和 SSR/hydration

### `seacraft-react-skills`

适用于：

- React 和 Next.js
- React Native 和 Expo
- React 组件组合模式
- React / Next.js 性能优化
- React View Transition

### `seacraft-frontend-spa-conventions`

技术栈无关的文件落位与目录分层规范，适用于：

- 新增、移动或重命名任何前端文件（页面、组件、composable/hook、api、store、类型、工具、样式、测试）
- 判定一个文件应落在哪一层（页面私有 / 跨页共享 / 基础层 / 资源层）
- 单应用与一仓多应用仓库（PC + 移动端、admin + portal）
- 代码评审时核查目录结构是否违规

该 skill 只约束目录与分层，不涉及组件写法；写法规范见 `seacraft-vue-skills` 和 `seacraft-react-skills`。

## 目录结构

```text
hz-skills/
├── README.md
├── seacraft-frontend-spa-conventions/
│   ├── SKILL.md
│   └── references/
│       ├── directory-baseline.md
│       └── placement-examples.md
├── seacraft-vue-skills/
│   ├── SKILL.md
│   └── references/
│       ├── create-adaptable-composable.md
│       ├── vue-best-practices.md
│       ├── vue-debug-guides.md
│       ├── vue-jsx-best-practices.md
│       ├── vue-options-api-best-practices.md
│       ├── vue-pinia-best-practices.md
│       ├── vue-router-best-practices.md
│       └── vue-testing-best-practices.md
└── seacraft-react-skills/
    ├── SKILL.md
    └── references/
        ├── react-composition-patterns.md
        ├── react-native-best-practices.md
        ├── react-performance-best-practices.md
        └── react-view-transitions.md
```

每个 skill 目录都必须包含一个 `SKILL.md`。`SKILL.md` 是入口文件，`references/` 中的文档按任务需要读取。

## 安装到 Codex

当前目录是 skill 源目录。要让 Codex 自动发现这些 skill，需要将各 skill 目录复制到 Codex 的 skill 搜索路径。

Windows PowerShell：

```powershell
$skillRoot = "C:\Users\用户\.codex\skills"
Copy-Item -Recurse -Force .\seacraft-vue-skills $skillRoot
Copy-Item -Recurse -Force .\seacraft-react-skills $skillRoot
Copy-Item -Recurse -Force .\seacraft-frontend-spa-conventions $skillRoot
```

macOS / Linux：

```bash
skill_root="${CODEX_HOME:-$HOME/.codex}/skills"
mkdir -p "$skill_root"
cp -R ./seacraft-vue-skills "$skill_root/"
cp -R ./seacraft-react-skills "$skill_root/"
cp -R ./seacraft-frontend-spa-conventions "$skill_root/"
```

如果使用了自定义 `CODEX_HOME`，将 skill 搜索路径替换为：

```text
$CODEX_HOME/skills
```

安装后重新打开或刷新 Codex，使 skill 列表重新加载。

## 使用方式

### 自动调用

当任务涉及 Vue 或 React 前端开发时，Codex 会根据 `SKILL.md` 中的 `description` 判断是否使用对应 skill。

典型匹配范围：

- Vue 页面、组件、composable、store、router、测试和调试任务使用 `seacraft-vue-skills`。
- React、Next.js、React Native、Expo、组件设计、性能和 View Transition 任务使用 `seacraft-react-skills`。
- 新增、移动、重命名前端文件，或需要判定目录归属、分层与跨层引用时使用 `seacraft-frontend-spa-conventions`。目录结构与框架写法冲突时，以该 skill 为准。

### 显式调用

需要明确指定时，在请求中使用 skill 名称：

```text
$seacraft-vue-skills
实现一个带 Pinia 状态管理的 Vue 3 页面。
```

```text
$seacraft-react-skills
重构这个 React 组件，减少 boolean props 并保持组件可组合。
```

```text
$seacraft-frontend-spa-conventions
给这个仓库新增一个巡课计划页面，按规范决定文件放在哪里。
```

显式调用前，需要确保对应目录已安装到 Codex 的 skill 搜索路径。

## 规范执行顺序

1. 确认项目框架、版本和现有代码模式。
2. 需要新增文件时，先按 `seacraft-frontend-spa-conventions` 的决策树确定落位，再开始写代码。
3. 读取与当前任务相关的 `references/` 文档。
4. 按 skill 约定实现组件或功能。
5. 组件实现完成后，检查项目中是否已有 `Prettier`、`ESLint` 和 `Stylelint` 配置。
6. 如果存在，按项目已有规则格式化本次修改的文件，并修复相关格式化、lint 和 stylelint 错误。
7. 根据任务风险执行必要的类型检查、单元测试或端到端测试。

格式化和 lint 只使用项目已有配置，不在没有明确需求时新增工具、配置或依赖。

## 更新 Skill

修改规范时：

- 优先更新对应 skill 的 `SKILL.md` 或相关 `references/` 文档。
- 保持 Vue 和 React 规则边界清晰，不把 Vercel 部署规范加入 React skill。
- 目录分层规则只写在 `seacraft-frontend-spa-conventions`，Vue / React skill 不重复定义目录结构。
- 新增参考文档后，在入口 `SKILL.md` 中补充链接。
- 修改入口文件后执行 skill 校验：

```powershell
$env:PYTHONUTF8 = "1"
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-vue-skills
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-react-skills
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-frontend-spa-conventions
```

macOS / Linux：

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-vue-skills
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-react-skills
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-frontend-spa-conventions
```
