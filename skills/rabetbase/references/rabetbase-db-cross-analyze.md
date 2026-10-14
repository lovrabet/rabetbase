# db cross-analyze-start / cross-analyze-status / cross-analyze-latest

分析同一应用下多个数据库连接之间的 DB_TABLE 逻辑关系。范围是所选连接已有的有效 V2 数据集及分析元数据，先完成各库的单库分析。按连接选择范围，不接受 `--tables`，不扫描业务行数据；缺少可用元数据的连接可能被跳过。

沿用当前平台、应用和连接 ID 编排分析。任务另涉及独立部署的同步、发布或运行验证时，按需读取[独立部署指南](../guides/independent-deployment.md)；该指南不是普通 DB 分析的执行前置。

## 命令

| 命令 | 输入 | 输出 |
| --- | --- | --- |
| `db cross-analyze-start` | `--ids <id1,id2,...>` 必填，至少两个不同的正安全整数；需要应用上下文；支持 `--dry-run` | 预览范围或提交后的 `data.planId` |
| `db cross-analyze-status` | `--plan <planId>` 必填，不要求应用上下文 | 同一任务的状态、进度和结果 |
| `db cross-analyze-latest` | 需要应用上下文 | `data.appCode` 和 `data.planId`，无任务时为 null |

所有命令需要通过 `rabetbase auth` 登录的 Cookie。应用上下文来自配置、`--app <name>` 或 `--appcode <code>`。具体 flags 与风险等级以 `rabetbase schema` 为准。

```sh
# 从当前应用的列表取得真实连接 ID；确认各库单库分析结果
rabetbase db list --appcode <appCode> --format compress

# 只读校验连接身份、归属和影响，不创建任务
rabetbase db cross-analyze-start --ids 10,20 --appcode <appCode> --dry-run --format compress

# 审阅后提交一次，保存 data.planId
rabetbase db cross-analyze-start --ids 10,20 --appcode <appCode> --format compress

# 每次只查询一次，持续使用原 planId
rabetbase db cross-analyze-status --plan <originalPlanId> --format compress

# 标识丢失时取得应用最新候选，再按返回的 planId 查询
rabetbase db cross-analyze-latest --appcode <appCode> --format compress
```

`--ids` 去除首尾空白、去重、升序排序。空项、非十进制整数、非正数、超出 JavaScript 安全整数范围或去重后少于两个 ID 在请求前失败。连接不存在、响应身份不可靠或归属另一应用时阻止提交。预览仅输出连接 ID、库名和类型，不输出账号、主机、密码或连接内部环境标记。

库名和类型来自连接详情，用于识别与展示。执行范围按 `selector.dbLinkIds` 对应的平台已有数据集与元数据确定，不凭展示库名推断分析覆盖范围。

## 预览和提交输出

预览输出 `operation`、`selector: {appCode, dbLinkIds}`、`connections`、`before: null`、`after: {planId: null}`、`dryRun`、`backend` 和 `warnings`。`before` 不表示当前关系为空；此操作创建新的分析任务，预览不枚举或预测关系变更。

正式提交额外返回顶层 `data.planId`，`data.after.planId` 为同一标识。标识只证明任务已提交，不证明分析完成。

分析消耗积分并写入跨库关系事实，同一应用的跨库任务串行。预览不验证元数据就绪、积分余额、配额或并发任务，不提供积分或耗时估算。提交接口检查数据集数量、余额、配额与并发任务；可用分析元数据在执行阶段检查。至少两个库需要可用数据集与分析元数据；缺少可用元数据的库可能被跳过。任务不自动配置数据集字段关联或页面绑定。

## 状态与结果

| 字段 | 含义 |
| --- | --- |
| `data.planId` | 本次查询的任务标识 |
| `data.status` | 原始状态，保留未知值 |
| `data.currentStep` / `progressRate` / `errorMsg` | 步骤、进度和失败原因，按平台事实展示 |
| `data.isTerminal` | `SUCCESS / PARTIAL_SUCCESS / FAILED / CANCELLED` 为 true，其它状态为 false |
| `data.result` | 平台结果对象，缺失时为 null，不补造统计值 |
| `data.result.analyzedTableCount` / `relationCount` / `newlyAddedRelationCount` | 已分析表数、关系数、新增关系数 |
| `data.result.participatingDbLinkIds` / `skippedDbLinkIds` | 实际参与库和跳过库，保留平台 ID 类型 |
| `data.result.elapsedMs` | 平台记录的分析耗时 |
| `data.gmtCreate` / `gmtModified` | 创建与更新时间，不推算为分析耗时 |

已知非终态 `PENDING / RUNNING / RETRYING / CANCELING` 继续查询同一 planId；未知状态停止自动推进，保留标识并报告原始状态。result 存在、进度为 100 或错误文案不能替代 status 判断生命周期。终态失败也不能保证未写入部分关系，需按实际意图回读。

## 恢复和关系回读

- 已有 planId 时查询同一任务，不用 latest 替换它。查询失败不等于任务失败，不触发新的 start。
- 提交超时、连接中断、成功响应无法解析或未返回有效标识时，结构化错误包含 `data.outcome: "unknown"` 和原 selector。停止提交，用 latest 辅助查找，再查候选状态。latest 是应用最新任务，不是原请求的幂等证明；旧任务或 null 都不能证明原请求没有提交，不自动重提。
- latest 的 `planId: null` 是正常无任务结果；损坏响应报错，不能当作无任务。
- 成功后检查 participatingDbLinkIds 与 skippedDbLinkIds，对每个实际参与库读取出向关系，并按两端库 ID 筛选目标范围：

目标库被跳过或参与信息缺失时，报告覆盖不足或无法确认覆盖，不把任务成功等同于所有所选库分析完成，不自动重新提交。

```sh
rabetbase dataset cross-relation-list --from-dblink-id 10 --appcode <appCode> --format compress
rabetbase dataset cross-relation-list --from-dblink-id 20 --appcode <appCode> --format compress
```

回读使用 [跨库关系命令](rabetbase-dataset-cross-relations.md)，不以 dataset relations 或单库 db diff 代替。任务成功和关系存在不代表跨库 JOIN、字段绑定、页面绑定或跨库事务已验证。

单库 schema 分析见 [db analyze](rabetbase-db-analyze.md)；跨库业务查询见 [跨库 BFF 指南](../guides/cross-database-bff.md)。
