# SQL 工作流规则

前置知识：`data-api-guidelines.md`

## 核心原则

平台是唯一 source of truth。团队长期维护 SQL 时，优先使用 **本地同步工作流**：`sql create / pull / status / push / delete` + `.rabetbase/sql.lock.json`。

## 工作流

```
确认需求 → [按需]查现有 SQL → 校验字段 → 拉/落本地（pull/new）→ 编辑本地文件 → [建议]validate → status → push/delete → detail/exec 验证
```

### 1. 确认需求
写 SQL 前必须明确：查询目标、字段、筛选条件、排序分页、是否 JOIN、新建还是修改。缺失则先问用户。

### 2. 查现有 SQL（按需）
* 新建 → 可跳过
* 修改已有 → 执行 `rabetbase sql list --format json` 找到目标，确认 `sqlCode`
* 不确定 → 查一下

命中同名或同语义 SQL 时，停下问用户：沿用还是另建。

### 3. 校验字段
执行 `rabetbase dataset detail --code <数据集编码> --format json` 确认表名、字段名、字段类型。禁止凭经验猜。

### 4. 先把 SQL 拉/落到同步目录

#### 修改已有 SQL

先执行：

```bash
rabetbase sql pull --sqlcode <sqlCode> --format json
```

把平台最新内容同步到本地。

#### 新建 SQL

先执行：

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

不要默认把长期源文件放在 `queries/`、`src/` 等目录；`sql save` 已废弃，不再作为长期工作流入口。

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

### 6. 验证（建议）
执行 `rabetbase sql validate --file <sql文件路径> --format json` 传入 SQL 文件。未通过则修正后重新验证。

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

确认无误后正式执行：

```bash
rabetbase sql push --sqlcode <sqlCode> --yes --format json
```

补充规则：

* 仅文件名变化时，`sql push` 会把新的文件名视作新的 `sqlName` 并回写远端
* 文件移动到新的数据库目录时，`sql push` 会尝试按目录名重新绑定 `dbId`
* 若提示 `missing remote version`，先执行 `sql pull` 刷新 lock 中的 `version`

### 9. 测试

推送成功后执行：

```bash
rabetbase sql detail --sqlcode <sqlCode> --format json
rabetbase sql exec --sqlcode <sqlCode> --format json
```

失败则修正 → validate → status → push → detail/exec。

### 10. 删除工作流

先预览：

```bash
rabetbase sql delete --sqlcode <sqlCode> --dry-run --format json
```

确认无误后：

```bash
rabetbase sql delete --sqlcode <sqlCode> --yes --format json
```

成功后会删除远端记录，并把本地文件移动到 `.rabetbase/sql-trash/`。

## 非 SELECT 语句

DELETE / DDL（DROP / ALTER / CREATE / TRUNCATE）属高风险，不建议进入同步主路径。
将 SQL 写入本地草稿文件，告知用户手动在平台操作。
草稿路径可放在同步目录旁的 `.draft.sql`，例如：

```text
.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.draft.sql
```

## 冲突处理

* `sql pull` 提示 `local differs from remote` → 说明本地与远端已漂移；先和用户确认是否 `--force`
* `sql push` 失败 → 明确告诉用户是哪一条 `sqlCode` 失败、为什么失败，不要粉饰为已同步

## SQL 调用差异

| 场景 | 前端 SDK | BFF (context.client) |
|------|---------|---------------------|
| 返回值 | `{ execSuccess, execResult }` | 直接返回数组 |
| 调用 | `client.sql.execute({ sqlCode, params })` | `context.client.sql.execute({ sqlCode, params })` |

## 已废弃命令

`rabetbase sql save` 已废弃。若用户提到旧流程，直接迁移为：

```bash
# 新建
rabetbase sql create --name <sqlName> --db-id <dbId> --mode sql --format json

# 修改
rabetbase sql push --sqlcode <sqlCode> --yes --format json
```
