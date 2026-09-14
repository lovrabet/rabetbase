# rabetbase instant-api-policy

管理应用级 Instant API 数据集访问策略。策略可以按 `datasetCode + api` 放行、禁止，或路由到同一应用的普通 Backend Function ENDPOINT。

## 固定本地文件

每个应用只有一个 CLI 策略源文件：

```text
.rabetbase/instant-api-policy/<appCode>/policy.json
```

`init`、`pull`、`validate` 和 `publish` 只使用该路径，不提供 `--file`。目标应用来自当前工作区、`--app <name>` 或 `--appcode <code>`。策略不存入 `.rabetbase.json` 或运行态 app-config。

`version` 是 JSON 结构版本：`"1"` 保持精确匹配兼容，`"2"` 支持通配选择和排除项；新版 CLI 的 `init` 默认创建版本 2。`revision` 是服务端发布历史版本，首次发布为 1，后续发布和回滚递增；两者含义不同。

## 配置结构

### 先取得真实 Dataset 选择器

策略匹配的是平台 Dataset，不是前端 SDK 的变量名。编辑前按下面顺序取得当前应用的事实：

```bash
# 更新当前应用的本地模型事实；多应用项目显式选择目标
rabetbase api pull --appcode <appCode> --yes --format compress

# 对将被规则覆盖的 Dataset 确认真实 code、字段和支持的操作
rabetbase dataset detail --appcode <appCode> --code <datasetCode> --format compress
```

`api pull` 生成的模型条目包含 `datasetCode`、可选 `tableName` 和 SDK `alias`。规则中优先只写这个 `datasetCode`；**不能**写 `alias`。例如模型中的 `alias: "brand"` 只是 SDK 调用名，不是策略选择器。只有无法取得 code 的兼容场景才使用 `tableName`；若两者同时填写，服务端要求它们解析为同一个 Dataset。

认证不可用、`api pull` 失败或模型文件明显过期时，停止配置并先恢复认证；不要根据业务名称、SDK alias 或旧截图猜测 `datasetCode`。

### Dataset 选择器规则

| 场景 | 合法写法 | 约束 |
| --- | --- | --- |
| v1/v2 的精确 Dataset | `{ "datasetCode": "<real-dataset-code>" }` | 首选；code 来自当前应用的 `api pull` 或 `dataset detail` |
| v1/v2 的精确物理表 | `{ "tableName": "<real-table-name>" }` | 可用作兼容选择器；不能填 SDK alias |
| v1/v2 双字段精确选择 | `{ "datasetCode": "…", "tableName": "…" }` | 两值必须解析为同一个 Dataset；已知 code 时不要冗余填写 tableName |
| 仅 v2 的全部 Dataset | `{ "datasetCode": "*" }` | `datasets` 的唯一元素；不能附带 `tableName`，也不能与精确选择器混用 |

`{ "tableName": "*" }` 在任何版本都不支持。版本 1 只接受上面的精确选择器，不能使用 Dataset/API 通配或 `excludes`。

### 顶层字段

| 字段 | 必填 | 取值 / 约束 | 作用 |
| --- | --- | --- | --- |
| `version` | 是 | `"1"` 或 `"2"`；新文件使用 `"2"` | JSON 结构版本，不是服务端发布 revision |
| `appCode` | 是 | 必须等于当前命令解析出的应用 | 防止跨应用误发布 |
| `defaults.instantApi` | 是 | `"allow"` 或 `"deny"` | 没有任何规则命中时的最终动作；不能是 `"route"` |
| `rules` | 否 | 规则数组；缺省或空数组表示全部使用默认动作 | 对特定 Dataset + API 组合覆盖默认动作 |

### 规则字段

| 字段 | 必填 | 取值 / 约束 |
| --- | --- | --- |
| `name` | 是 | 稳定、可读的规则标识，方便审阅和版本回溯 |
| `description` | 否 | 说明规则的业务目的和影响范围 |
| `datasets` | 是 | 非空数组；每项优先仅 `{ "datasetCode": "<real-dataset-code>" }` |
| `apis` | 是 | 非空数组，每项是独立 API 名；不能把多个 API 拼为一个逗号字符串 |
| `action` | 是 | `"allow"`、`"deny"` 或 `"route"` |
| `backendFunction` | 条件必填 | 仅 `action: "route"` 时填写；必须是同一应用的普通 `ENDPOINT`，不能是 Personal Backend Function |
| `excludes` | 仅 v2 可用 | 从本规则的 Dataset × API 匹配集合中扣除的数组；每项都要有 `datasets` 和 `apis`，且不得超出父规则范围 |

支持的 API 名称为：`getSelectOptions`、`getList`、`excelExport`、`filter`、`aggregate`、`getOne`、`getOneOrigin`、`create`、`batchCreate`、`update`、`delete`。

规则数组**没有优先级**。服务端会先按每条规则的 Dataset × API 组合（扣除 `excludes` 后）计算覆盖范围；仍有重叠就校验失败，而不是“后一条覆盖前一条”。未被任何规则覆盖的组合回退到 `defaults.instantApi`。

### 精确匹配示例

下面的 `<real-dataset-code>` 必须替换为刚由 `api pull` 得到的 code，不能替换成 alias 或展示名称：

```json
{
  "version": "2",
  "appCode": "app-example",
  "defaults": { "instantApi": "allow" },
  "rules": [
    {
      "name": "deny-sensitive-export",
      "description": "禁止敏感数据集通过 Instant API 导出",
      "datasets": [{ "datasetCode": "<real-dataset-code>" }],
      "apis": ["excelExport"],
      "action": "deny"
    }
  ]
}
```

这是默认放行下的最小收紧方式：仅该 Dataset 的 `excelExport` 被禁止，其他组合继续由默认动作处理。

### 版本 2 通配与排除示例

```json
{
  "version": "2",
  "appCode": "app-example",
  "defaults": {
    "instantApi": "allow"
  },
  "rules": [
    {
      "name": "route-all-except-student-create",
      "description": "除学生新增外，全部数据集操作交给统一 Backend Function",
      "datasets": [{"datasetCode": "*"}],
      "apis": ["*"],
      "excludes": [
        {
          "datasets": [{"datasetCode": "student23"}],
          "apis": ["create"]
        }
      ],
      "action": "route",
      "backendFunction": "unifiedGateway"
    },
    {
      "name": "allow-student-create",
      "datasets": [{"datasetCode": "student23"}],
      "apis": ["create"],
      "action": "allow"
    }
  ]
}
```

- `defaults.instantApi` 只允许 `allow` 或 `deny`，不能是 `route`。
- `rules[].action` 允许 `allow`、`deny`、`route`。
- `route` 必须填写 `backendFunction`；其他 action 禁止填写。
- `datasets[]` 至少填写 `datasetCode` 或 `tableName`；已知 code 时只填写稳定的 `datasetCode`。两者同时填写时必须指向同一 Dataset。
- 版本 1 只支持精确匹配，不允许 `*` 或 `excludes`。
- 版本 2 的全部数据集写为 `datasets: [{"datasetCode":"*"}]`。通配项必须是数组唯一元素且不能包含 `tableName`；不支持 `tableName: "*"`。
- 版本 2 的全部 API 写为 `apis: ["*"]`，`"*"` 必须是数组唯一元素。
- `excludes[]` 每项包含 `datasets` 和 `apis`，表示从父规则中扣除两者笛卡尔积；排除范围不能超出父规则。
- 被排除的组合可以由后续其他规则处理；没有其他规则时执行 `defaults.instantApi`。
- 排除后仍重复的 `datasetCode + api` 会校验失败，规则数组顺序不表示优先级。
- `apis` 的每个元素是一个独立值。`["filter,create"]` 或包含中文逗号的 `["filter，create"]` 都不是两个 API。
- 支持的 API：`getSelectOptions`、`getList`、`excelExport`、`filter`、`aggregate`、`getOne`、`getOneOrigin`、`create`、`batchCreate`、`update`、`delete`。

路由目标必须属于同一应用、类型为 `ENDPOINT`、不是 Personal Backend Function，并且函数配置可被 Runtime 解析。命中 route 后直接执行 Backend Function，原 Dataset 操作、Dynamic Hook 和通知链路不执行。策略判定优先于 Dynamic Hook。

## 本地配置工作流

```bash
# 没有远端策略时创建默认放行草稿
rabetbase instant-api-policy init --format compress

# 已有远端策略时拉到固定文件；本地不同内容不会被静默覆盖
rabetbase instant-api-policy pull --format compress
rabetbase instant-api-policy pull --force --yes --format compress

# 编辑固定 policy.json 后进行服务端校验
rabetbase instant-api-policy validate --format compress

# 预览发布；0 表示预期当前尚未配置
rabetbase instant-api-policy publish --expected-revision 0 --change-summary "initial policy" --dry-run --format compress

# 人工确认高风险影响后发布
rabetbase instant-api-policy publish --expected-revision 0 --change-summary "initial policy" --yes --format compress
```

`init --default-action deny` 可以创建版本 2 的默认禁止草稿。`init` 和 `pull` 遇到不同的既有文件时停止；只有 `--force --yes` 可以替换。已有版本 1 文件仍可继续校验和发布。

`validate` 调用服务端完整校验，返回 `configHash` 和 `normalizedConfig`，但不发布、不生成 revision。Dataset、API、规则重叠以及 Backend Function 合法性以该结果为准。

`publish` 和 `rollback` 是 `high-risk-write`。它们受 CLI `riskLevel` 门禁约束；Agent 和自动化脚本不得修改 `riskLevel` 提权。正式执行需要 `--yes`，建议使用 `--expected-revision` 做写前漂移检查。该检查是非原子的，因为服务端 mutation 请求没有 revision 条件。

## 查询与回滚

```bash
rabetbase instant-api-policy current --format compress
rabetbase instant-api-policy revisions --format compress
rabetbase instant-api-policy revision --revision 2 --format compress

rabetbase instant-api-policy rollback --revision 2 --expected-revision 5 --dry-run --format compress
rabetbase instant-api-policy rollback --revision 2 --expected-revision 5 --change-summary "restore revision 2" --yes --format compress
```

回滚不会把 revision 倒退，也不会覆盖历史。例如当前 revision 为 5，恢复 revision 2 后会生成新的 revision 6。

发布和回滚最多提交一次。CLI 随后读取当前策略，要求 configHash 匹配且 revision 推进。响应丢失但回读可以证明结果时返回 verified；证据不足时停止并提示执行 `current`，不得自动重提。Runtime 最迟约 30 秒感知新策略。

数据集通配项按应用当前有效数据集展开；应用新增数据集后，会在 Runtime 策略缓存刷新时自动进入通配规则范围。
