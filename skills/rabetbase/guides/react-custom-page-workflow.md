# React 自定义页面工作流

本指南说明如何使用 `rabetbase` 创建、修改和发布 React 自定义页面。页面内容由完整 JSX、样式和多语言文件组成。

## 适用范围

适用于以下目标：

- 新建或修改 React 自定义页面、看板、门户、落地页或复杂交互页面

## 前置上下文

- 创建页面前确认 `appCode`；修改页面前确认 `pageId`
- 确认页面目标，以及影响实现的布局、交互、数据和样式约束
- 无法确定新建页面或目标页面时，先向用户确认

## 编排规则

### 新建页面

1. 先阅读 [`rabetbase-page-react-create.md`](../references/rabetbase-page-react-create.md)
2. 可选传入完整 `--page-content`；省略时 CLI 生成默认文件。先执行 `page react-create --dry-run`，确认页面名称、父菜单和最终页面文件
3. 经用户确认后创建页面，记录 `data.after.pageId`。新页面默认尚未发布
4. 执行 `page react-detail --id <pageId>`，以 `data.codeContent` 中的完整页面文件为修改基线
5. 需要定制初始文件时，再按“更新页面”流程保存完整页面文件

创建页面会同时创建菜单入口。不得额外调用菜单创建命令重复创建入口。

### 查找或修改已有页面

1. 未知页面 ID 时，先阅读 [`rabetbase-page-react-list.md`](../references/rabetbase-page-react-list.md)，执行 `page react-list --appcode <appCode>`
2. 根据 `data.pages[].pageId` 与 `label` 与用户确认目标页面；`data.pages[].pageUrl` 用于查看最新保存内容，`data.pages[].editPageUrl` 用于打开页面编辑器
3. 阅读 [`rabetbase-page-react-detail.md`](../references/rabetbase-page-react-detail.md)，执行 `page react-detail --id <pageId>`
4. 只在返回的最新 `data.codeContent` 基础上编辑，不得根据旧缓存或猜测覆盖文件

### 更新页面

1. 阅读 [`rabetbase-page-react-update.md`](../references/rabetbase-page-react-update.md)
2. 将新增、修改和保留的所有文件整理为完整 `page-content` JSON
3. 先执行 `page react-update --dry-run` 审阅完整页面文件
4. 经用户确认后执行更新
5. 再执行 `page react-detail --id <pageId>`，确认 `codeContent` 已包含预期文件和内容
6. 已发布页面更新后，最新保存内容需要再次发布，用户访问的已发布页面才能看到本次修改

`page react-update` 会完整保存页面文件：未提交的文件会被删除。不得仅提交本次修改的单个文件，也不得为删除文件再调用不存在的独立删除命令。

当前 CLI 没有逐文件新增、编辑或删除命令。将本地文件操作整理为完整 `codeContent` 后再调用 `react-update`；每次更新都会创建一个新的保存版本，`react-detail` 默认读取最新页面内容。

### 发布页面

1. 先执行 `page react-detail --id <pageId>`，确认当前 `data.status`
2. 当前状态为 `FORMAL` 时，表示当前保存内容已发布，不执行 `page react-publish`，直接查看已发布页面或继续后续流程
3. 当前状态不是 `FORMAL` 时，读取 [`rabetbase-page-react-publish.md`](../references/rabetbase-page-react-publish.md)，再执行 `page react-publish --dry-run`
4. 发布会影响用户实际访问的页面，必须取得用户明确确认后再执行
5. 从 `data.after.status` 确认 `FORMAL`，表示内容已发布

`data.pageUrl` 用于查看最新保存内容，包含尚未发布的修改；`data.editPageUrl` 用于打开页面编辑器；`page react-publish` 成功返回的 `data.runtimePageUrl` 用于查看已发布内容。不要将三者混用。

## 页面生命周期

React 页面有“未发布”和“已发布”两种状态。未发布的修改可通过 `pageUrl` 或编辑器查看，已发布页面只展示最近一次发布的内容。

```text
page react-create
  → 未发布
  → 生成或更新完整页面文件
  → 未发布
  → page react-publish
  → 已发布（FORMAL）→ 可通过 runtimePageUrl 查看已发布内容
  → page react-update
  → 未发布（已发布页面仍显示上一次发布内容）
  → page react-publish
  → 已发布（FORMAL）→ 已发布页面展示最新发布内容
```

| 阶段 | `pageUrl` 或编辑器可见内容 | 已发布页面可见内容 | 下一步 |
| --- | --- | --- | --- |
| 创建后 | 初始页面文件，尚未发布 | 无新发布内容 | 读取详情并生成或更新内容 |
| 更新后、未发布 | 最新保存内容，尚未发布 | 最近一次已发布内容 | 完成自检并发布 |
| 发布后 | 当前已发布版本，状态为 `FORMAL` | 本次已发布内容 | 通过 `runtimePageUrl` 查看或继续修改 |
| 已发布后再次更新 | 最新保存内容，尚未发布 | 更新前的已发布内容 | 再次发布后才切换已发布页面内容 |

`page react-detail` 用于确认最新保存内容，`page react-publish` 用于发布当前保存内容。不要仅因 `react-update` 成功就认为已发布页面已经更新。

`react-detail` 返回 `FORMAL` 时，表示当前保存内容已经发布。没有新的修改时，跳过重复发布，继续查看已发布页面或后续流程。

### 地址语义

| 操作结果 | 输出字段 | 用途 |
| --- | --- | --- |
| 创建/保存成功 | `pageUrl`、`editPageUrl` | `pageUrl` 查看最新保存内容，`editPageUrl` 打开页面编辑器 |
| 发布成功 | `runtimePageUrl` | 查看最近一次已发布内容 |
| 页面查询 | `pageUrl`、`editPageUrl` | 快速打开最新保存内容或页面编辑器 |

## 数据与服务契约

### 选型

根据页面的实际业务需求选择数据访问方式，不要为了简单查询引入 SQL 或 BFF：

| 方式 | 适用场景 | 边界 |
| --- | --- | --- |
| 单一数据集请求 | 只涉及一个数据集的查询、详情、分页、筛选、创建、更新或删除 | 严格按照当前数据集 API-doc 返回的调用说明实现，不得自行推测或自由发挥调用标识、方法、字段、参数、返回值或异常处理 |
| 自定义 SQL | 需要组合多个数据集，或需要数据库完成关联、聚合、分组、排序、计算和复杂筛选 | 先确认 SQL 资源、参数和执行结果，再按 `@lovrabet/sdk` 规范调用；不要在页面中拼接 SQL，也不要把 CLI 的 `data.rows` 当作运行时返回结构 |
| BFF | 需要注入业务校验、数据转换、加密、条件分支、多步业务编排或外部服务调用 | 将业务逻辑放入已确认的 BFF，由页面通过 SDK 调用；简单数据集请求或单纯数据关联不应额外包装成 BFF |

判断顺序：先确认单一数据集请求能否满足需求；数据组合和数据库计算是主要问题时选择自定义 SQL；业务规则、数据处理流程或外部服务协同是主要问题时选择 BFF。三种方式可以根据已确认的 SDK 契约配合使用，但不得自行推测方法、参数或返回结构。

页面需要读取或写入数据集时，先按以下顺序确认事实：

1. `rabetbase dataset list --appcode <appCode>` 查找可用数据集
2. `rabetbase dataset detail --code <datasetCode>` 确认字段、类型和必填约束
3. `rabetbase dataset operations --code <datasetCode>` 确认可用操作及其输入
4. 需要生成调用示例时，阅读 [`rabetbase-codegen-sdk.md`](../references/rabetbase-codegen-sdk.md)，使用 `rabetbase codegen sdk`

页面需要调用自定义 SQL 或 BFF 时，先通过当前 CLI 获取目标资源事实：

1. SQL：执行 `rabetbase sql list`，再执行 `rabetbase sql detail --sqlcode <sqlCode>` 确认目标 SQL
2. BFF：执行 `rabetbase bff list`，再执行 `rabetbase bff detail --id <id>` 确认目标脚本
3. 按 `@lovrabet/sdk` 的使用规范实现 `client.sql` 或 `client.bff` 调用，不得根据 CLI 返回结构猜测页面调用参数或返回值
4. 页面使用自定义 SQL 前，使用代表性参数执行 `rabetbase sql exec --sqlcode <sqlCode> --params <json> --format json`，确认执行没有报错并检查实际数据结构
5. `sql exec` 仅用于确认 SQL 可执行及返回数据结构。页面代码按 `@lovrabet/sdk` 的使用规范处理 `client.sql` 的返回值，不使用 CLI 输出字段 `data.rows`

当前 CLI 没有按单个数据集返回页面数据访问说明的专用命令。在页面中使用 `client` 前，必须以数据集、SQL 或 BFF 的事实，以及对应的 `@lovrabet/sdk` 使用规范为依据；无法确认的字段、接口路径和响应结构不得猜测。

## 页面开发参考

编写页面内容前，先阅读 [`generation-standards.md`](../knowledge/react-page/generation-standards.md)。

- 选择、组合或新增 UI 组件时，按需阅读 [`components.md`](../knowledge/react-page/components.md)
- 调用页面上下文、国际化、路由或数据客户端时，按需阅读 `generation-standards.md` 中的“页面上下文内置能力”
- 未登记的组件、方法、参数和返回结构不得猜测，先补充确认后的规范再复用

## 页面文件与代码约束

默认文件结构：

```text
src/app/index.jsx
src/app/index.css
src/locales/index.js
src/components/*.jsx
```

代码遵循以下约束：

- 仅导入 [`generation-standards.md`](../knowledge/react-page/generation-standards.md)“依赖白名单”中的包及其子路径
- 使用页面上下文时，从 `@/context/app-context` 导入 `useSdkClient`、`useI18n`、`useNavigate`、`useLocation`
- 数据读取或写入前，先确认数据集及 SDK 用法；接口或字段无法从现有事实确认时，不得猜测请求协议
- 所有用户可见文案通过 `$i18n.t("key")` 获取，并在 `src/locales/index.js` 提供有效翻译
- 路由跳转使用 `navigate`，读取地址信息使用 `location`；关联页面优先使用 `navigate("/pagecode", { onePage: true })`
- CSS 使用 Ant Design CSS 变量，不写死颜色或其他样式常量
- 不新增白名单外依赖，不使用 `window.location`

未传 `--page-content` 时，`react-create` 生成基础 JSX、样式和词包。首次自定义时，将初始文案和样式一并调整为上述约束后再发布，不把初始化模板视为最终页面实现。

## 页面内容自检

在首次提交或每次更新完整 `codeContent` 前，按以下顺序自检：

1. 先用 `page react-detail --id <pageId>` 读取最新页面文件，避免覆盖其他尚未发布的修改
2. 确认提交内容包含全部需要保留的文件，入口、样式、词包和组件之间的导入路径一致
3. 对照 [`generation-standards.md`](../knowledge/react-page/generation-standards.md) 检查依赖白名单、国际化、路由、样式和数据契约
4. 使用组件或内置方法时，分别对照 `components.md` 和 `generation-standards.md` 中的“页面上下文内置能力”，确认不存在未登记的属性、方法或返回结构假设
5. 先完成“页面生成规范”中的静态自检，再执行 `page react-update --dry-run`。CLI 会阻止缺少 React 源码或存在基础 JS/JSX/TS/TSX 语法错误的内容，但不推断变量引用、模块导出、数据契约或运行时行为
6. 更新后再次执行 `page react-detail --id <pageId>`，确认最新保存的 `codeContent` 与提交内容一致
7. 需要让用户查看已发布页面时，先检查最新 `data.status`。状态为 `FORMAL` 时跳过发布；否则执行 `page react-publish --dry-run`、取得确认后发布，并确认 `data.after.status` 为 `FORMAL` 和 `data.runtimePageUrl` 已返回

当前 CLI 不提供已发布页面截图或自动浏览器预览。发布成功后，使用返回的 `data.runtimePageUrl` 打开已发布页面进行人工预览。

## 最小验证闭环

```text
新建：react-create --dry-run → react-create → react-detail → [按需] react-update --dry-run → react-update → react-detail → [尚未发布（status 非 FORMAL）且经确认] react-publish --dry-run → react-publish

修改：react-list（按需）→ react-detail → react-update --dry-run → react-update → react-detail → [尚未发布（status 非 FORMAL）且经确认] react-publish --dry-run → react-publish
```

CLI 当前不提供本地预览或独立删除命令。

## 参考

- [`rabetbase-page-react-create.md`](../references/rabetbase-page-react-create.md)
- [`rabetbase-page-react-list.md`](../references/rabetbase-page-react-list.md)
- [`rabetbase-page-react-detail.md`](../references/rabetbase-page-react-detail.md)
- [`rabetbase-page-react-update.md`](../references/rabetbase-page-react-update.md)
- [`rabetbase-page-react-publish.md`](../references/rabetbase-page-react-publish.md)
- [`generation-standards.md`](../knowledge/react-page/generation-standards.md)
- [`components.md`](../knowledge/react-page/components.md)
- [`rabetbase-codegen-sdk.md`](../references/rabetbase-codegen-sdk.md)
