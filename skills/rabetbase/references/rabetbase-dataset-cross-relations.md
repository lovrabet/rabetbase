# 跨库数据集关系

使用 `dataset cross-relation-list/create/update/delete` 管理同一应用内、不同数据库连接的 DB_TABLE 逻辑关联。同库关系命令见 [数据集关系](rabetbase-dataset-relation-mutations.md)。跨库关系事实使用本页的 list 查询，不能用同库关系查询命令 `dataset relations` 或 `dataset detail` 的关系列表替代回读。

需要实现跨库业务查询时，读取 [跨库 BFF 查询与拼接](../guides/cross-database-bff.md)，按目标字段用途确定查询顺序，并核对各端授权及结果完整性。业务关系清单不代表平台已登记；平台已有关系也需要与业务合同核对。

## 定位和查询

四个命令均支持 `--appcode` 覆盖当前应用、`--format json|compress`。参数使用真实 Dataset Code 和字段名，先用 `dataset detail` 确认端点字段。

```sh
rabetbase dataset cross-relation-list --from-datasetcode <source-code> --format json
rabetbase dataset cross-relation-list --from-dblink-id <source-connection-id> --format json
rabetbase dataset cross-relation-list --from-datasetcode <source-code> --from-column customer_id --format json
```

列表必须提供 `--from-datasetcode` 或 `--from-dblink-id`，可同时提供，但必须属于同一连接。提供数据集时自动解析源连接；只提供连接时列出该连接的所有出向跨库关系。`--from-column` 可进一步过滤源字段。

列表输出 `data.appCode`、`data.selector`、`data.total`、`data.relations`。每条包含 `relationId`、`from/to.source`、`datasetCode`、`datasetName`、`dblinkId`、`table`、`field`；目标展示字段为 `to.labelField`，关系属性为 `relation.cardinality`、`relation.relationType`。缺失的 Dataset Code 或名称保留 null，不能根据同名表猜测。Relation ID 是返回事实，不是这些命令的输入参数。

## 创建

```sh
rabetbase dataset cross-relation-create \
  --from-datasetcode <source-code> --from-column customer_id \
  --to-datasetcode <target-code> --to-column id \
  --to-label-column name --cardinality MANY_TO_ONE --dry-run
```

两端数据集与字段必填。`--to-label-column` 可选，省略时默认目标关联字段；`--cardinality` 可选，省略时不设置。合法基数：`ONE_TO_ONE`、`ONE_TO_MANY`、`MANY_TO_ONE`、`MANY_TO_MANY`。

每端只接收一个字段；复合键不能拆成多条单字段关系，也不能用逗号拼接字段传入。依据完整复合键编写 BFF 不代表这些命令已经登记该关系。

完全相同的端点不可重复创建。同一源字段可以关联不同目标；需要在创建前理解下述更新、删除的唯一匹配要求。

## 更新

```sh
rabetbase dataset cross-relation-update \
  --from-datasetcode <source-code> --from-column customer_id \
  --to-datasetcode <new-target-code> --to-column id \
  --to-label-column name --dry-run
```

源数据集和源字段定位原关系，目标数据集和目标字段必填并表示更新后的目标。可选 `--from-dblink-id` 校验源连接归属。展示字段和基数省略时保留原值；更换目标数据集时应显式给出有效的 `--to-label-column`。无实际字段变化时返回错误，不当作更新成功。

## 删除

```sh
rabetbase dataset cross-relation-delete \
  --from-datasetcode <source-code> --from-column customer_id --dry-run
```

源数据集和源字段必填；可选 `--from-dblink-id`。删除属于高风险写入：先审阅预演，再按用户授权执行，非交互执行使用 `--yes`，且当前配置必须允许 high-risk-write；`--yes` 不会提高风险上限。

更新和删除都要求源字段恰好匹配一条跨库关系。匹配多条时失败，结构化错误的 `data.candidates` 给出候选事实；不要选择第一条、循环删改，也不要追加 `--relation-id` 重试。先向用户说明该源字段当前无法由这组命令逐条维护。`--relation-id`、`--biz-relation-type` 和 `main_sub` 不属于本组命令。

## 预演与确认

所有写命令支持 `--dry-run`，只读取当前关系；输出 `operation`、`selector`、`before`、预计 `after`、`dryRun`、`backend`、`warnings` 和请求 body。预演不是实际写入，也不是并发锁。审阅后在用户授权范围内移除 `--dry-run` 执行。

正式写入后通过跨库列表回读：创建核对完整端点和属性；更新核对原 Relation ID、更新后的端点和属性；删除核对原 ID 已消失。仅在 `verification.status=matched` 时报告写入已验证。

网络中断会返回结果未确认；写入被接受但回读失败时返回 `verification.status=unconfirmed`。此时先运行 `cross-relation-list` 查询，不要自动重新创建、更新或删除。跨库关系管理完成不代表跨库 JOIN、页面绑定或跨库事务已经验证。
