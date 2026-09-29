# sql validate

对普通 SQL 或 MyBatis XML 提供本地辅助检查（类型、参数和风险提示），不实际保存。文件格式默认使用 `.sql`；只有内容实际采用 MyBatis XML 语法时才使用 `.xml`。

## 本地路径约定

从文件校验时，优先使用同步目录下的文件：

```text
.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml
```

内联 `--sql` 不受此限。

## 命令

```bash
# 从文件校验
rabetbase sql validate --file .rabetbase/sql/app-xxxxxxxx/sample_db/2305f915-dd48cd4c_getUserList.sql --format json

# 仅在内容使用 MyBatis XML 语法时选择 .xml
rabetbase sql validate --file .rabetbase/sql/app-xxxxxxxx/sample_db/2305f915-dd48cd4c_getUserList.xml --format json

# 内联 SQL 校验
rabetbase sql validate --sql "SELECT * FROM users WHERE id = #{userId}" --format json

# 带 schema 交叉校验
rabetbase sql validate --file .rabetbase/sql/app-xxxxxxxx/sample_db/report.sql --schemas datasetCode1,datasetCode2 --format json

# 审计顶层 JOIN 的索引匹配
rabetbase sql validate --file .rabetbase/sql/app-xxxxxxxx/sample_db/report.sql --schemas datasetCode1,datasetCode2 --check-indexes --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--file <path>` | string | 与 sql 二选一 | — | 默认传 `.sql`；MyBatis XML 内容传 `.xml` |
| `--sql <content>` | string | 与 file 二选一 | — | 内联 SQL 或 MyBatis XML 内容 |
| `--schemas <codes>` | string | 否 | — | 逗号分隔的 Dataset code 列表，交叉校验表名 |
| `--check-indexes` | boolean | 否 | `false` | 使用 Dataset 索引元数据审计顶层 JOIN；必须同时传 `--schemas` |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 输出

| 字段 | 说明 |
|------|------|
| `valid` | 是否通过当前本地辅助检查；不表示允许保存、完整语法有效或执行安全 |
| `sqlType` | SQL 类型（SELECT / INSERT / UPDATE / DELETE / DDL / UNKNOWN） |
| `isSelectOnly` | 是否为不含加锁子句的 SELECT；不代表数据库权限或完整执行安全判定 |
| `isLockingRead` | 是否识别到 `FOR UPDATE`、`FOR SHARE` 或 `LOCK IN SHARE MODE` 加锁查询 |
| `isDangerous` | 本地检查是否提示需明确审阅的操作；不是完整安全判定 |
| `tables` | 引用的表名列表 |
| `parameters` | 提取的 `#{param}` / `#{param, jdbcType=...}` / `${param}` 参数名 |
| `schemaWarnings` | 表名/Dataset 不匹配的警告（仅 `--schemas` 时） |
| `joinAudit` | JOIN 索引审计结果（仅 `--check-indexes` 时）；不会修改 `valid` |

## 提示

- 可按需调用本命令获取提示；`sql push` 不依赖本地分析结果，不按方言白名单限制提交
- 普通 SQL 保持 `.sql`；不要因为校验器支持 MyBatis XML 而改成 `.xml`
- MyBatis XML 会去除组装标签后做保守静态分析，不执行 OGNL、动态分支或集合展开
- 受 `<if>` / `<choose>` / `<foreach>` 影响的 JOIN 和无法展开的 `<include>` 返回无法判断，不报告确定的索引缺失或覆盖
- `--schemas` 可交叉检查 SQL 中引用的表是否在指定 dataset 中存在
- `--check-indexes` 只审计可安全识别的顶层 JOIN 等值条件，不支持时返回无法判断，不猜测缺索引
- 空索引元数据返回 `INDEX_METADATA_UNAVAILABLE`；静态审计不能替代数据库执行计划
- `JOIN_INDEX_MISSING` 仅表示非空索引元数据中没有相关字段；`JOIN_INDEX_NOT_USABLE` 表示相关字段不构成完整最左前缀
- `JOIN_PREDICATE_UNSUPPORTED` 表示 JOIN 方向、目标或谓词不能安全分析；发现这类 JOIN 时 `checked` 仍为 `true`
- `sql create` 仅创建固定的 `SELECT 1` 模板；推送不会调用本地 SQL 分析器
- 加锁查询仍属于 SELECT，可识别；必须在适当事务内执行，不能因为 `valid=true` 将其当作普通只读查询
- `WITH [RECURSIVE] ... AS (...)` 按完整 CTE 定义列表后的主语句分类，支持多 CTE、列名列表和嵌套查询；合法的 `WITH ... UPDATE` 按 UPDATE 校验。定义不完整或主语句无法识别时，本地检查返回 `valid=false`，不影响推送决策；不单独判定 CTE 的修改效果，也不解释数据库特有可执行注释的引擎或版本语义
- 词法扫描区分加锁子句与修改语句，忽略普通注释和引用文本中的关键字；该检查不替代数据库语法验证、权限检查或真实并发测试

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)

本命令不执行 MyBatis/OGNL，也不检查真实数据库版本、会话和权限。动态模板、未知写法的诊断只表示本地能力边界；Agent 应保留业务语义并说明不确定性，不把 `valid=true` 当作提交授权。
