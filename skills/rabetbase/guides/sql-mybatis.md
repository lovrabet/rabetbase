# SQL 与 MyBatis 语法指南

> 目标：指导平台 Custom SQL 的语义处理与 MyBatis 动态 SQL 编写。
>
> 前置阅读：[SQL 工作流](sql-creation-workflow.md)、[数据接口约束](data-api-guidelines.md)

## 何时使用

当任务满足任一条件时，必须阅读并遵守本指南：

* 编写或修改平台 Custom SQL
* 编写需要动态条件的复杂 SQL（如可选参数、范围过滤）
* SQL 验证报错，需要排查语法或字段问题

## Instant API 与 Custom SQL 的逻辑删除边界

- **Instant API**：`create`、`update`、`filter` 接口由平台按逻辑删除配置自动处理 `is_deleted` 字段；`filter` 默认排除已删除记录。该行为仅适用于相应标准接口，不能推导为所有平台 SQL 都会自动过滤。
- **Custom SQL**：平台不会自动添加 `is_deleted` 条件，也不会替开发者补齐逻辑删除字段处理。即使表结构已标记“逻辑删除”，查询正常记录仍须在 SQL 中显式编写过滤条件，例如 `is_deleted = 0`；字段名和正常值以实际表结构及配置为准。自定义 INSERT / UPDATE 所需的逻辑删除字段赋值或筛选也由 SQL 明确表达。
- 编写或修改 SQL 前，通过 `dataset detail` 核对主表、关联表、子查询涉及表的逻辑删除字段和执行连接，逐处判断需要排除还是包含已删除记录。不得以“平台自动补充”为由移除已有条件；列表、计数、聚合和写入条件应保持业务口径一致。
- `LEFT JOIN` 需要保留没有有效右表记录的主记录时，右表的逻辑删除条件写在 `ON` 中；不要移到外层 `WHERE` 导致主记录被过滤。主表和子查询各自显式处理删除条件，并保留业务状态、租户或门店范围及权限条件。
- 已授权的历史查询可在 Custom SQL 中显式使用删除态条件（例如 `is_deleted = 1`），或按业务要求包含两种状态。`includeDeleted` 等自定义参数只有被 SQL 实际消费才有效；查询范围仍须受服务端授权和业务边界约束，不把读取历史记录等同于允许恢复或重复创建。
- Java Mapper、数据库直连及其他执行通道按各自契约处理；不能套用 Instant API 的自动行为。CLI 的校验和同步不会替业务 SQL 补齐或删除逻辑删除条件。
- `FOR UPDATE`、`FOR SHARE` 和 `LOCK IN SHARE MODE` 属于加锁查询，与逻辑删除过滤是两个独立要求。保留原锁语义与事务边界，不为通过校验删除锁或互换锁类型；`valid=true` 只表示静态校验通过，不证明锁在实际事务中生效。
- 运行验证应覆盖主表、关联表与子查询的删除过滤，特别是 `LEFT JOIN` 右表已删除或不存在时的主记录保留语义，以及列表、计数和聚合的一致性。真实业务结果与锁效果交接运行态验证；不能仅凭 SQL 同步成功宣称业务验收通过。

资源复用、创建、同步、检查纠错与运行验收统一遵循 [SQL 工作流](sql-creation-workflow.md)。单条命令参数读取对应 reference；表结构与字段按 [dataset detail](../references/rabetbase-dataset-detail.md) 的实际输出获取。

## 📚 MyBatis 动态 SQL 语法参考（核心）

平台支持 MyBatis 语法。在自定义 SQL 中处理动态参数时，必须遵守以下规范。`<foreach>`、`<include>` 等动态内容由平台处理，CLI 上传时保留原文；本地提示不证明所有参数分支可执行。

### 简单参数 vs 动态参数

| 类型 | 语法 | `jdbcType` | 适用场景 |
|---|---|---|---|
| **简单参数** | `#{param}` | ❌ 不加 | 必定传入的固定条件 |
| **动态参数** | `#{param, jdbcType=TYPE}` | ✅ 必须加 | `<if>` 标签内的可选条件 |

### 动态 SQL 示例（推荐）

以下示例假设 `company.is_deleted = 0` 表示正常记录；固定删除过滤不依赖可选参数。

```sql
SELECT id, name, status_code 
FROM company
<where>
    is_deleted = 0
    <if test="statusCode != null and statusCode != ''">
        AND status_code = #{statusCode, jdbcType=VARCHAR}
    </if>
    <if test="name != null and name != ''">
        AND name LIKE CONCAT('%', #{name, jdbcType=VARCHAR}, '%')
    </if>
    <if test="startDate != null">
        AND created_at &gt;= #{startDate, jdbcType=DATE}
    </if>
</where>
ORDER BY id DESC
```

### 常用标签说明

* `<where>`：自动处理内部的 `AND` / `OR` 前缀，当内部没有条件成立时，整个 `WHERE` 关键字也不会出现。
* `<if test="条件">`：用于判空。注意：字符串应同时判断 `!= null` 和 `!= ''`，数字/布尔只需判断 `!= null`。
* `<foreach>`：常用于 `IN` 语句的数组遍历。

### `jdbcType` 对照表

在 `<if>` 标签内使用参数时，必须带上对应的 `jdbcType`：

| 数据类型 | `jdbcType` |
|---|---|
| 字符串 | `VARCHAR` |
| 整数 | `INTEGER` |
| 小数 | `DECIMAL` |
| 日期 | `DATE` |
| 时间戳 | `TIMESTAMP` |

### XML 转义规则

由于 SQL 内容会被解析为 XML，符号需要正确转义：

* **SQL 文本中**：必须转义！
  * 大于号 `>` 写作 `&gt;`
  * 小于号 `<` 写作 `&lt;`
  * 示例：`AND created_at &gt;= #{startDate, jdbcType=DATE}`
* **`<if test="...">` 属性内**：不需要转义！
  * 示例：`<if test="amount > 100">`

## 禁止事项

* ❌ 禁止在未执行 `rabetbase sql validate` 通过前，直接执行 `rabetbase sql push`
* ❌ 禁止在 `<if>` 标签的参数绑定中漏写 `jdbcType`
* ❌ 禁止在简单固定参数（不在标签内）里加 `jdbcType`
* ❌ 禁止把 `<` 或 `>` 直接写在 SQL 正文中（必须用 `&lt;` / `&gt;`）
