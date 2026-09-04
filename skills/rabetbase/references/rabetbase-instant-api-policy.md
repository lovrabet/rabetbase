# rabetbase instant-api-policy

管理应用级 Instant API 数据集访问策略。策略可以按 `datasetCode + api` 放行、禁止，或路由到同一应用的普通 Backend Function ENDPOINT。

## 固定本地文件

每个应用只有一个 CLI 策略源文件：

```text
.rabetbase/instant-api-policy/<appCode>/policy.json
```

`init`、`pull`、`validate` 和 `publish` 只使用该路径，不提供 `--file`。目标应用来自当前工作区、`--app <name>` 或 `--appcode <code>`。策略不存入 `.rabetbase.json` 或运行态 app-config。

`version` 固定为字符串 `"1"`，表示 JSON 结构版本。`revision` 是服务端发布历史版本，首次发布为 1，后续发布和回滚递增；两者含义不同。

## 配置结构

```json
{
  "version": "1",
  "appCode": "app-example",
  "defaults": {
    "instantApi": "allow"
  },
  "rules": [
    {
      "name": "route-student-filter",
      "description": "学生筛选交给业务 Backend Function",
      "datasets": [
        {
          "datasetCode": "student23"
        }
      ],
      "apis": ["filter"],
      "action": "route",
      "backendFunction": "filterStudents"
    }
  ]
}
```

- `defaults.instantApi` 只允许 `allow` 或 `deny`，不能是 `route`。
- `rules[].action` 允许 `allow`、`deny`、`route`。
- `route` 必须填写 `backendFunction`；其他 action 禁止填写。
- `datasets[]` 至少填写 `datasetCode` 或 `tableName`；建议优先使用稳定的 `datasetCode`。两者同时填写时必须指向同一 Dataset。
- 同一个 `datasetCode + api` 只能命中一条规则；重复覆盖会校验失败。
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

`init --default-action deny` 可以创建默认禁止草稿。`init` 和 `pull` 遇到不同的既有文件时停止；只有 `--force --yes` 可以替换。

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
