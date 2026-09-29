# SQL 工作流规则

前置知识：`data-api-guidelines.md`

## 核心原则

平台是唯一 source of truth。团队长期维护 SQL 时，优先使用 **本地同步工作流**：`sql create / pull / status / push / delete` + `.rabetbase/sql.lock.json`。

SQL 内容编写、参数绑定与 MyBatis 语法以 [`sql-mybatis.md`](sql-mybatis.md) 为准。页面执行已发布的 Custom SQL 时使用 `sqlCode` + `params`；Backend Function 默认使用 `context.client.sql.byName(sqlName).execute({ params })`，`sql.execute({ sqlCode, params })` 仅作兼容路径。

## 工作流

```
确认需求 → 查现有 SQL → 校验字段 → 拉/落本地（pull/create）→ 编辑本地文件 → 检查、修复与复验 → status → push/delete → detail/exec 验证
```

### 1. 确认需求
写 SQL 前必须明确：查询目标、字段、筛选条件、排序分页、是否 JOIN、新建还是修改。先读取已有需求、项目资料与相关元数据；仍缺少影响实现的业务信息时再向用户确认。

### 2. 查现有 SQL
目标 `sqlCode` 已明确时直接回读 `sql detail`；否则执行 `rabetbase sql list --format json` 定位目标或可复用资源。

发现相似资源后，主动比较业务语义、参数、返回结构、目标连接及调用方。满足需求且兼容时直接复用；任务范围内的明确错误直接修复并验证。确需新建且已获授权时自行完成。只有存在无法从资料确定的业务取舍或影响既有调用方的范围变更时，才准备推荐方案并请求决策；同名不等于同语义。

### 3. 校验字段

通过现有连接元数据确认目标数据库类型；版本等信息无法取得时明确标注未知，不默认采用 MySQL 语法。
执行 `rabetbase dataset detail --code <数据集编码> --format json`，按 [dataset detail 输出契约](../references/rabetbase-dataset-detail.md)确认表名、字段名、字段类型及数据库 ID。`sql create --db-id` 使用已核实的目标连接 ID，不得猜测或复用其他应用的 ID。

同时核对各表的逻辑删除字段及正常值。Instant API 的 `create`、`update`、`filter` 自动处理逻辑删除字段；Custom SQL 不会自动添加 `is_deleted` 条件，即使表结构已标记逻辑删除，也须在主表、关联表和子查询中按业务需要显式编写过滤条件。历史查询由 SQL 明确表达范围，并遵守授权约束。具体规则见 [Instant API 与 Custom SQL 的逻辑删除边界](sql-mybatis.md#instant-api-与-custom-sql-的逻辑删除边界)。

同时核对各表所属连接与该 SQL 的执行连接；同名表须按真实连接消歧。不同连接的逻辑关联由 [跨库 BFF 查询与拼接](cross-database-bff.md)编排分库读取，不能用跨库 JOIN、子查询或视图绕过边界。`sql validate` 的静态检查不证明跨连接可执行、权限完整或全局结果正确。

### 4. 先把 SQL 拉/落到同步目录

#### 修改已有 SQL

先检查本地是否有待保留的修改，再执行；出现分歧按下方“冲突处理”完成比较：

```bash
rabetbase sql pull --sqlcode <sqlCode> --format json
```

把平台最新内容同步到本地。

#### 新建 SQL

先使用相同参数执行 `--dry-run`，由 Agent 核对名称、连接、模式与创建范围；符合已有授权后执行：

```bash
rabetbase sql create --name <sqlName> --db-id <dbId> --mode sql --format json
```

或 MyBatis XML：

```bash
rabetbase sql create --name <sqlName> --db-id <dbId> --mode mybatisXml --format json
```

`sql create` 会先在远端创建 SQL，再生成本地文件并写入 `.rabetbase/sql.lock.json`。

### 5. 编辑本地 SQL（规范路径）

长期维护的文件路径统一为：

```text
.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml
```

不要默认把长期源文件放在 `queries/`、`src/` 等目录；长期维护统一使用同步目录。

CLI 会自动维护 `@lovrabet` 头注释，例如：

```sql
-- @lovrabet.sqlCode: 2305f915-dd48cd4c
-- @lovrabet.sqlName: getUserList
-- @lovrabet.dbId: 10001
-- @lovrabet.dbName: sample_db
-- @lovrabet.mode: sql
-- @lovrabet.syncedAt: 2026-04-11T06:47:24.325Z

select * from users
```

可以保留并编辑这段头注释；`sql push` 上传时会自动剥离，不会把本地元信息写回平台正文。

### 6. 检查、修复与复验

Agent 应主动核对 SQL 与业务意图、目标连接、字段、参数和必要条件是否一致。对证据明确且属于任务授权范围的错误，直接修复并复验，不因用户提供了原始实现或本地检查能力有限而忽略问题。

修改 SQL 后，默认执行 `rabetbase sql validate --file <sql文件路径> --format json` 获取辅助诊断。已知超出其覆盖范围时，使用现有可用的元数据、相关测试或获授权的运行验证，并说明未覆盖部分；不为取得 `valid=true` 反复调用已知不适用的检查。

| 诊断情况 | 后续动作 |
| --- | --- |
| 已确认的实现错误 | 修正错误，验证原问题消失，并检查受影响的业务结果 |
| 事实不足 | 优先读取相关元数据、调用契约和已有测试；补充事实后继续判断，仅对仍无法取得的关键业务信息询问用户 |
| 有依据的检查器误报 | 保留正确实现，说明依据与限制，使用适用的验证方式继续推进任务 |

`valid=true` 后仍须检查业务条件与结果是否满足需求。保留的是业务意图及必要的锁、权限和范围条件；已证实写错或漏写的实现应修正，不得仅为消除告警删除必要条件。本地检查不参与 `sql push` 保存决策，实际提交仍须符合用户授权及平台规则。

同一问题没有新证据却重复出现时，停止无依据的改写或重试，继续完成不受阻碍的工作。交付时列明已修复内容、实际验证、未验证项和具体阻碍；不以“检查器不支持”直接结束可继续处理的任务。

### 7. 查看同步状态

执行：

```bash
rabetbase sql status --format json
```

必要时补充：

```bash
rabetbase sql status --remote --format json
```

状态含义：

* `added`：本地文件存在，但 lock 未跟踪
* `modified`：本地文件 hash、路径或文件名（`sqlName`）变化
* `missing`：lock 有记录，但本地文件缺失
* `unchanged`：本地与 lock 一致
* `remoteOnly`：只有 `--remote` 时检查远端孤儿项

### 8. 推送到平台

先预览：

```bash
rabetbase sql push --sqlcode <sqlCode> --dry-run --format json
```

Agent 核对预览与已有授权一致、命令前置要求满足后正式执行；只有新增关键取舍才请用户决策：

```bash
rabetbase sql push --sqlcode <sqlCode> --format json
```

补充规则：

* 仅文件名变化时，`sql push` 会把新的文件名视作新的 `sqlName` 并回写远端
* 文件移动到新的数据库目录时，`sql push` 会尝试按目录名重新绑定 `dbId`
* 若提示 `missing remote version`，先保留本地修改、回读远端，再按“冲突处理”恢复同步；由 `sql pull` 刷新版本，不手改 lock 伪造版本

### 9. 测试

推送成功后先回读内容：

```bash
rabetbase sql detail --sqlcode <sqlCode> --format json
```

任务授权包含真实执行时，再使用 `rabetbase sql exec --sqlcode <sqlCode> --params '<JSON参数>' --format json` 验证相关业务结果；单纯保存成功不代表功能或锁效果已验收。

根据失败原因处理：实现错误按第 6 步修复并复验；连接、权限或版本冲突先处理对应原因，不凭这些错误改写 SQL。运行验证受阻时，完成可进行的本地检查并报告剩余项。

### 10. 删除工作流

先预览：

```bash
rabetbase sql delete --sqlcode <sqlCode> --dry-run --format json
```

Agent 核对预览中的资源与影响，满足删除授权和命令确认要求后执行；`--yes` 仅用于已获授权的确认，不能绕过用户取消：

```bash
rabetbase sql delete --sqlcode <sqlCode> --yes --format json
```

成功后会删除远端记录，并把本地文件移动到 `.rabetbase/sql-trash/`。

## 非 SELECT 语句

SQL 资源的保存与语句的真实执行分开判断。`sql exec` 的 `read` 标记不代表 SQL 没有写入或锁定影响；本地 `valid=true`、风险配置或 `--yes` 都不能替代对实际执行范围的授权。

**编写与准备由 Agent 完成**：确认目标连接、影响范围、实际入口能力与授权，写出具体 SQL，核对影响行数或对象范围，准备适用的验证方式与恢复方案；不因语句类型就把编写、排查和验证直接交给用户。

**按具体方案或批次授权**：目标环境、连接、影响范围及后果已明确授权，且实际入口支持时，Agent 连续执行并验证，无需逐条重复确认。涉及尚未授权的数据删除、破坏性结构变更或不可逆后果时，先完成可审阅的 SQL、影响评估与恢复方案，再就整个具体方案请求一次确认；不把概括性任务目标视为这些后果的授权。目标环境或连接改变、影响范围扩大或出现新的重大后果时，重新提交关键决策。

用户确认不扩展平台能力或权限。不为验证候选 SQL 而隐式执行它，也不把 `sql exec` 当作任意 SQL 执行器；入口不支持或用户取消时停止该操作，不通过换接口、包装 SQL 或反复尝试绕过，继续其他已授权且不受阻碍的工作。

能力、权限或关键取舍仍阻塞时，完成可独立推进的准备，交付具体候选、影响、已验证结果和最小待办。未准备同步的候选文件放在同步目录外；`.draft.sql` 后缀不代表 CLI 自动忽略，不能把草稿写入同步目录后整批推送。

## 冲突处理

* `sql pull` 提示 `local differs from remote` → 保留本地修改，使用 `sql detail` 回读远端并比较；有可靠基线且语义明确、互不冲突的修改由 Agent 合并并验证。基线不足先补证；涉及覆盖他人修改或业务冲突时，准备差异、推荐方案与影响再请求决策，不把 `--force` 当默认恢复方式。
* `sql push` 失败或结果未知 → 按[写入结果与恢复动作](conflict-detection.md#写入结果与恢复动作)处理；逐项核对返回的 `pushed`、`skipped`、`failed` 与 `sqlCode`，不重复提交已成功项。

## SQL 调用差异

| 场景 | 前端 SDK | Backend Function (context.client) |
|------|---------|---------------------|
| 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 调用 | `client.sql.execute({ sqlCode, params })` | `context.client.sql.execute({ sqlCode, params })` |
