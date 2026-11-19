# Seacraft Frontend Skills

Seacraft 前端开发规范 skill，当前包含 Vue、React、文件落位规范、公共组件、公共 hooks 和公共 utils 六部分。文档采用“英文 YAML 元数据 + 中文规范说明 + 英文技术标识”的形式。

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

### `seacraft-shared-components`

公司公共前端组件的接入规范，当前覆盖 `jy-player-ui` 播放器套件及其在两个业务仓中的封装：

- `JyKmediaPlayer` + `JyPlayerControl`（PC / kmedia 引擎）
- `JyKmediaMobilePlayer` + `JyMobilePlayerControl`（H5 / kmedia-mobile 引擎）
- `JyAliPlayer`（阿里播放器，HLS / FLV / MP4）
- `JyControlProgress`、`JySlider`（进度条与滑杆）
- 多屏同步、双屏对比、画中画、stage 分屏
- `MultiWindowPlayer`（`jy-digital-patrol-ui` 业务封装，1–4 画面、分屏布局、视角切换、拖拽换位、反向语音）
- `PlayerMask`（加载 / 错误 / 暂停 / 结束 / 空态遮罩）
- `SvgIcon`（SVG 雪碧图图标，`vite-plugin-svg-icons`）
- `GlobalUploader`（`jy-application-resourcemanage-ui` 业务封装，分片上传、MD5、秒传/断点续传、全局面板）
- 工程级接入：依赖版本、Vite/Webpack 配置、入口初始化、引擎脚本、运行时配置、打包与产物验证

包含安装与私有源、引擎脚本、注册方式、样式定制、数据源格式、组件 API、常用配方、故障排查与已知陷阱。后续公共组件按同一结构在 `references/` 中追加文档。

### `seacraft-shared-hooks`

公司公共 Vue 3 hooks 的使用与抽象规范，只保留不依赖业务 store / API / 路由 / socket / 业务 id 约定的**真正通用** hook：

- 弹窗 / 确认 / 下载：`useDialog`、`useConfirm`、`useDownload`
- 表格：`useTable`、`useSelection`、`useSelectionOrigin`
- 样式与图表：`useInlineCssVars`、`useEcharts`

业务专用的 `useDarkMode`、`useTheme`、`useWatermark`、`useStompSocket`、`useRecordPlayerState`、`useGlobalVar` 不进入公共 hook，需在原业务仓使用或按提取指南重写。

包含每个 hook 的签名、返回、依赖、用法、已知坑，以及把业务 hook 抽象成公共 hook 的依赖注入、类型、测试与打包指南。

### `seacraft-shared-utils`

公司公共前端 utils 的使用与提取规范，来源 `D:\LSWork` 下多个 Vue 后台项目：

- 类型与值：`is`、`isFunction`、`isObject`、`isArray`、`getType` 等
- 树与表格：`getFlatArr`、`findItemNested`、`handleProp`、`handleRowAccordingToProp`、`filterEnum`、`formatValue`
- 格式化：`getTimeState`、`convertToChinaNum`、`formatNumberUnit`、`formatFileSize`、时间/日期换算
- 浏览器与文件：脚本加载、剪贴板、DOM 交换、设备判断、query、UUID、MD5、base64、视频封面
- 颜色：`hexToRgb`、`rgbToHex`、`getLightColor`、`getDarkColor`

只收录跨项目复用、无业务/UI 依赖、模块加载无副作用的函数；菜单/权限/主题/播放器/socket 等业务 utils 明确排除。包含每个函数的签名、来源、行为、已知坑，以及去重、依赖治理、测试与 Vite 打包指南。

## 目录结构

```text
hz-skills/
├── README.md
├── AGENTS.md.example
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
├── seacraft-shared-components/
│   ├── SKILL.md
│   └── references/
│       ├── player-overview.md
│       ├── player-kmedia.md
│       ├── player-kmedia-mobile.md
│       ├── player-ali.md
│       ├── player-multi-screen.md
│       ├── multi-window-player.md
│       ├── player-mask.md
│       ├── svg-icon.md
│       ├── global-uploader.md
│       └── project-bootstrap.md
├── seacraft-shared-hooks/
│   ├── SKILL.md
│   └── references/
│       ├── hook-catalog.md
│       ├── ui-hooks.md
│       ├── table-hooks.md
│       ├── inline-css-vars.md
│       ├── echarts-hook.md
│       └── extraction-guide.md
├── seacraft-shared-utils/
│   ├── SKILL.md
│   └── references/
│       ├── catalog.md
│       ├── type-tree-table.md
│       ├── format.md
│       ├── browser-file.md
│       ├── color.md
│       └── extraction-guide.md
└── seacraft-react-skills/
    ├── SKILL.md
    └── references/
        ├── react-composition-patterns.md
        ├── react-native-best-practices.md
        ├── react-performance-best-practices.md
        └── react-view-transitions.md
```

每个 skill 目录都必须包含一个 `SKILL.md`。`SKILL.md` 是入口文件，`references/` 中的文档按任务需要读取。

根目录的 `AGENTS.md.example` 是项目级路由表模板，用于让 AI 在具体项目中优先加载对应 skill。

## 安装到 Codex

当前目录是 skill 源目录。要让 Codex 自动发现这些 skill，需要将各 skill 目录复制到 Codex 的 skill 搜索路径。

Windows PowerShell：

```powershell
$skillRoot = Join-Path $HOME '.codex\skills'
New-Item -ItemType Directory -Force $skillRoot | Out-Null
Copy-Item -Recurse -Force .\seacraft-vue-skills $skillRoot
Copy-Item -Recurse -Force .\seacraft-react-skills $skillRoot
Copy-Item -Recurse -Force .\seacraft-frontend-spa-conventions $skillRoot
Copy-Item -Recurse -Force .\seacraft-shared-components $skillRoot
Copy-Item -Recurse -Force .\seacraft-shared-hooks $skillRoot
Copy-Item -Recurse -Force .\seacraft-shared-utils $skillRoot
```

macOS / Linux：

```bash
skill_root="${CODEX_HOME:-$HOME/.codex}/skills"
mkdir -p "$skill_root"
cp -R ./seacraft-vue-skills "$skill_root/"
cp -R ./seacraft-react-skills "$skill_root/"
cp -R ./seacraft-frontend-spa-conventions "$skill_root/"
cp -R ./seacraft-shared-components "$skill_root/"
cp -R ./seacraft-shared-hooks "$skill_root/"
cp -R ./seacraft-shared-utils "$skill_root/"
```

如果使用了自定义 `CODEX_HOME`，将 skill 搜索路径替换为：

```text
$CODEX_HOME/skills
```

安装后重新打开或刷新 Codex，使 skill 列表重新加载。

### 让 AI 一键安装

把下面的提示词发给有本地文件读写权限的 AI（Codex CLI、Claude Code、Cursor 等）。Web 聊天窗口无法直接写本地目录。

```text
请把 D:\LSWork\hz-skills 仓库中的 6 个 skill 安装到 Codex：

1. 源仓库：D:\LSWork\hz-skills（如果路径不同，先问我确认，不要猜）。
2. 目标目录：$HOME/.codex/skills；如果设置了 CODEX_HOME，则用 $CODEX_HOME/skills。
3. 复制以下目录，保持目录名不变：
   seacraft-vue-skills
   seacraft-react-skills
   seacraft-frontend-spa-conventions
   seacraft-shared-components
   seacraft-shared-hooks
   seacraft-shared-utils
4. 不要修改 SKILL.md 或 references 内容。目标已存在同名目录时先备份并告诉我，不要静默覆盖。
5. 每个目录必须存在 SKILL.md。如果本机有
   ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py，
   逐个执行校验并汇总结果。
6. 最后列出已安装的 skill、实际目标路径，并提醒我重启或刷新 Codex。
```

### 让 AI 安装并写入 AGENTS.md 路由表

```text
请完成两件事：

A. 安装 skill
把 D:\LSWork\hz-skills 下的 6 个 skill 复制到 $HOME/.codex/skills
（如果设置了 CODEX_HOME，则用 $CODEX_HOME/skills），保持目录名不变；
已存在同名目录时先备份并询问，不要静默覆盖。

B. 在当前项目根目录创建或更新 AGENTS.md
1. 以 D:\LSWork\hz-skills\AGENTS.md.example 为模板。
2. 如果 AGENTS.md 已存在，保留原有内容，只合并「前端公共资产路由」章节；
   不要覆盖项目已有的构建、测试、代码风格约定。
3. 根据当前项目的 package.json / 技术栈删减无关行：
   - 纯 Vue 项目删除 React 行；
   - 不使用播放器/上传的项目删除对应行；
   - 没有 Pinia/Router 的项目可精简 Vue 行描述。
4. 如果已存在同名章节，先展示差异，再合并。
5. 安装完成后逐个校验 SKILL.md，并展示 AGENTS.md 的 git diff。
6. 不要自动 git commit / push，等我确认。
```

### 只给已有项目加 AGENTS.md 路由表

```text
请在当前项目根目录创建或更新 AGENTS.md：

1. 以 D:\LSWork\hz-skills\AGENTS.md.example 为模板。
2. 保留现有 AGENTS.md 的所有内容，只新增/合并「前端公共资产路由」章节。
3. 按当前项目技术栈删减无关行，不要照抄全部。
4. 技能名与实际安装目录保持一致；默认路径是 ~/.codex/skills/<name>/SKILL.md。
5. 完成后展示 diff，不要自动 commit。
```

模板文件：[`AGENTS.md.example`](AGENTS.md.example)。

## 使用方式

### 自动调用

当任务涉及 Vue 或 React 前端开发时，Codex 会根据 `SKILL.md` 中的 `description` 判断是否使用对应 skill。

典型匹配范围：

- Vue 页面、组件、composable、store、router、测试和调试任务使用 `seacraft-vue-skills`。
- React、Next.js、React Native、Expo、组件设计、性能和 View Transition 任务使用 `seacraft-react-skills`。
- 新增、移动、重命名前端文件，或需要判定目录归属、分层与跨层引用时使用 `seacraft-frontend-spa-conventions`。目录结构与框架写法冲突时，以该 skill 为准。
- 接入公司公共组件（`jy-player-ui` 播放器、`MultiWindowPlayer` 多画面、`PlayerMask` 遮罩、`SvgIcon` 图标、`GlobalUploader` 分片上传）、配置播放引擎、处理多屏同步或排查播放器异常时使用 `seacraft-shared-components`。
- 使用或抽象通用公共 hooks（弹窗、确认、下载、表格、多选、CSS 变量、ECharts）时使用 `seacraft-shared-hooks`。
- 新增或提取通用 utils（类型判断、树、时间/数字格式化、浏览器/DOM、设备、文件、颜色）时使用 `seacraft-shared-utils`。

安装后仍可能因为描述匹配、宿主实现或任务表述而漏选。需要稳定命中时，在项目根目录加入 `AGENTS.md` 路由表，或显式写 `$seacraft-shared-utils` 这类 skill 名。

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

```text
$seacraft-shared-components
在巡课页接入 jy-player-ui 的 PC 播放器，带自定义控制栏和暂停蒙层。
```

```text
$seacraft-shared-hooks
把这个页面的表格查询和跨页多选改用公共 hooks，并说明需要替换哪些依赖。
```

```text
$seacraft-shared-utils
从这几个项目里提取时间格式化和设备判断的公共 utils，并说明需要修掉的边界问题。
```

显式调用前，需要确保对应目录已安装到 Codex 的 skill 搜索路径。

## 规范执行顺序

1. 确认项目框架、版本和现有代码模式。
2. 需要新增文件时，先按 `seacraft-frontend-spa-conventions` 的决策树确定落位，再开始写代码。
3. 读取与当前任务相关的 `references/` 文档。新 Vite 工程或首次接入先读 `project-bootstrap.md`；播放器任务再读 `player-overview.md`；多画面/遮罩/图标任务读 `multi-window-player.md` / `player-mask.md` / `svg-icon.md`；分片上传任务读 `global-uploader.md`；公共 hooks 任务先读 `hook-catalog.md`，再按分组读 `ui-hooks.md` / `table-hooks.md` / `inline-css-vars.md` / `echarts-hook.md`；公共 utils 任务先读 `catalog.md`，再按域读 `type-tree-table.md` / `format.md` / `browser-file.md` / `color.md`；需要抽象时读对应的 `extraction-guide.md`。
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
- 公共组件 API 以发布版本源码为准；`README` 与源码冲突时在 skill 中标注差异，不要把错误默认值写进规范。
- 公共 hooks 的行为以源码为准；把业务耦合写成「适配点」，不要伪装成开箱即用。
- 公共 utils 只收录跨项目、无业务/UI 依赖的纯函数；旧实现缺陷要在 skill 中标注并给出修复建议。
- 新增参考文档后，在入口 `SKILL.md` 中补充链接。
- 修改入口文件后执行 skill 校验：

```powershell
$env:PYTHONUTF8 = "1"
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-vue-skills
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-react-skills
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-frontend-spa-conventions
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-shared-components
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-shared-hooks
py -3 C:\Users\用户\.codex\skills\.system\skill-creator\scripts\quick_validate.py .\seacraft-shared-utils
```

macOS / Linux：

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-vue-skills
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-react-skills
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-frontend-spa-conventions
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-shared-components
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-shared-hooks
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-creator/scripts/quick_validate.py" ./seacraft-shared-utils
```
