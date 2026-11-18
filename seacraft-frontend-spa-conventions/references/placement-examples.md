# 落位示例与反例

占位符：`<module>` 业务模块、`<Page>` 页面组件名、`<Name>` 组件名。Vue 用 `.vue`，React 用 `.tsx`，结论一致。

## 场景 1：新增一个"列表页"（筛选 + 表格 + 分页）

```text
src/views/<module>/
├─ index.vue                 # 只做编排：组合筛选、表格、分页
├─ index.scss                # 本页样式
├─ components/
│  ├─ FilterBar/index.vue    # 仅本页的筛选区
│  └─ RecordTable/index.vue  # 仅本页的表格
├─ composables/useListQuery.ts   # 分页、查询参数、竞态处理（有状态逻辑）
├─ utils/formatRow.ts        # 行数据格式化，纯函数
└─ interface/index.ts        # 本页视图模型类型
```

配套改动：`src/api/modules/<module>.ts` 加 `getListApi`，`src/api/models/<module>.ts` 加 `ListReq / ListRsp`（复用通用分页类型）；`router/` 补路由，component 指向 `views/<module>/index.vue`。

**提升时机**：`RecordTable` 只在这里用 → 留在页面内。等到第二个页面也要这张表，再移到 `src/components/RecordTable/`，并把数据获取留在页面、组件只吃 props。

## 场景 2：页面 A 的卡片要在页面 B 复用

```text
src/components/<Name>Card/
├─ index.vue          # 通过 props 接收数据，去掉对页面 A store 的直接依赖
└─ interface/index.ts # props/emit 复杂时才建
```

- 先做"去业务耦合"这一步：组件内不能读 A 的页面状态、不能自己发请求。
- 取值/格式化逻辑不进组件，进 `src/shared/<域>.ts`。
- 只有一处复用时**不要**抽；抽象成本通常高于重复成本。

## 场景 3：新增一组后端接口

```text
src/api/modules/<module>.ts   # export function getXxxApi(params) { return http.post<...>(...) }
src/api/models/<module>.ts    # namespace 内追加 Req / Rsp
```

- 页面只 `import { getXxxApi } from '@/api/modules/<module>'`。
- 分页统一用基础分页类型（`BasicPageParams` / `BasicPageResult<T>`），不要每处自定义。
- 错误提示、鉴权、loading 交给请求层拦截器；页面不做 `catch` 弹提示（除非需要特殊文案）。
- 同一接口在多个页面用：请求函数仍只写一份，页面各自调用，不要复制。

## 场景 4：新增一个跨页面的客户端状态

例如"侧栏折叠状态"要在多个页面共享并持久化。

```text
src/stores/modules/<domain>.ts   # defineStore：state / getters / actions
src/stores/interface/index.ts    # 追加该 store 的类型（若仓库这样组织）
```

- 只在单页用 → 页面内 `ref` / `useState`，不进 store。
- 服务端数据不进 store：那是请求层的职责，否则会出现缓存与真实数据不一致。
- 持久化沿用仓库已有的持久化插件配置，不要自带一套 localStorage 封装。

## 场景 5：抽出跨页业务规则（两种栈都要用）

规则：把接口字典值转成下拉选项，并按角色过滤。

```text
src/shared/<domain>Rules.ts   # getOptions(dict) / shouldShowFilter(role, values)，纯函数
```

- 不 import 任何框架 API、不发请求、不读 localStorage。
- 单测直接覆盖：`tests/<domain>Rules.test.ts`。
- 判断标准：如果文件顶部出现 `import { ref } from 'vue'` 或 `useState`，说明位置错了 → 该进 `hooks/`。

## 场景 6：PC 端与移动端需要同一份逻辑

```text
packages/shared-utils/src/<domain>.ts    # 方案 A：抽包，两应用以包名引用（有 workspace 机制时）
src/shared/<domain>.ts                   # 方案 B：各存一份
mobile/src/utils/<domain>.ts
```

- 有 monorepo 机制 → 方案 A；两应用物理隔离且无 workspace → 方案 B。
- 方案 B 必须保持同名同签名，改动时两边同步。
- **禁止** `import ... from '../../pc/src/shared/...'`：跨应用别名不存在，构建必挂，即使本地能跑也会在 CI 失败。

## 场景 7：给纯逻辑补测试

```text
tests/<domain>.test.ts
```

- 位置由仓库约定决定：多数项目集中在根 `tests/`；若仓库已全量 colocate（`__tests__/` 与源码同级）且有测试配置，则跟随。
- 一个新仓库不要同时存在两种测试位置。
- 只测纯逻辑时不要挂载组件；组件测试另建目录（如 `tests/components/`），避免和纯函数测试混成一锅。

## 场景 8：需要一个独立预览页验证效果

```text
dev/<feature>.html     # 独立入口
dev/<feature>.ts       # 创建轻量应用实例 + 假数据
```

- 用环境判断守卫（`if (!import.meta.env.DEV) throw ...`），确保不进生产入口。
- 不修改 `src/views` 里正在开发的页面来"临时验证"。
- 长期需要的演示页应升级成 Storybook 之类的组件文档体系，而不是堆在 `dev/`。

## 场景 9：新增全局指令 / 全局 hook

```text
src/directives/modules/<name>.ts   # 实现
src/directives/index.ts            # 统一注册（唯一注册入口）
src/hooks/use<Name>.ts             # hook 直接放 hooks/，无需注册
```

- 指令必须有第二个使用方才全局化；单页用就在页面内定义局部指令。
- 全局 hook 的类型放 `hooks/interface/`（若仓库这样组织），不要散落在 hook 文件里导出公共类型。

## 反例集

| 错误写法 | 后果 | 修正 |
| --- | --- | --- |
| 单页组件放进 `src/components/` | 被自动注册或打进公共包，体积与命名污染，删除时不敢动 | 回页面 `components/`，第二处复用时再提升 |
| `utils/api` 里 import 页面或 store | 反向依赖，模块循环、单测无法独立运行 | 把依赖改成入参，或上移到调用方 |
| 页面 B `import '@/views/a/components/x'` | 页面边界消失，A 改结构就炸 B | 提升到 `components/` 或 `shared/` |
| 页面里直接 `axios.get('/xxx')` | 鉴权/错误提示/取消重复请求全部失效 | 收敛到 `api/modules` |
| 把接口响应缓存进 store | 双份真相，刷新后不一致 | 请求层 + 组件内请求/查询库缓存 |
| 新建 `utils/common.ts` 什么都往里塞 | 变成垃圾桶，无人敢删 | 按职责拆成具名文件 |
| `component` 与 `components` 并存 | 同一层两种写法，AI 与人都要猜 | 统一复数，存量单独 PR 迁移 |
| 手改 `auto-import.d.ts` / 生成的 api client | 下次构建被覆盖，类型与实际不符 | 改配置源文件并重新生成 |
| 测试散落在各业务目录 | 两套约定并行，CI 覆盖不全 | 统一到根 `tests/` 或全量 colocate |
| 移动端 `import '@/shared/x'` 复用 PC 逻辑 | 别名不存在，构建失败或产物串味 | 抽包或各存一份 |

## 判定口诀

> 只服务一个页面 → 待在页面里；
> 两个页面要它，带业务 → 上层共享；
> 两个页面要它，不带业务 → 基础工具；
> 数据形状 → 类型层；请求 → 请求层；状态 → 状态层；
> 说不上职责 → 先别建文件。
