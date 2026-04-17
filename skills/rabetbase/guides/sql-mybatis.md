# SQL CLI 命令与 MyBatis 语法指南

> 目标：指导 AI 如何正确使用 CLI 命令来完成 SQL 的查询、本地同步、验证、推送与测试，并提供平台支持的 MyBatis 动态 SQL 语法参考。
>
> 前置阅读：`sql-creation-workflow.md`（整体流程）、`data-api-guidelines.md`（字段约束）

## 何时使用

当任务满足任一条件时，必须阅读并遵守本指南：

* 需要使用 CLI 命令处理自定义 SQL
* 编写需要动态条件的复杂 SQL（如可选参数、范围过滤）
* SQL 验证报错，需要排查语法或字段问题

## CLI 命令调用规范

在执行 `sql-creation-workflow.md` 规定的流程时，AI 必须严格按以下方式使用 CLI 命令。

### 1. `rabetbase sql list --format json`
* **用途**：创建或修改前，搜索同名或相似语义的 SQL。
* **动作**：若发现相似项，必须与开发者确认是否复用或另起新名，避免冗余创建。

### 2. `rabetbase dataset detail --code xxx --format json`
* **用途**：写 SQL 前获取表结构，**绝对禁止凭空编造表名或字段名**。
* **提取项**：
  * 真实表名：`basic.tableName`
  * 真实字段列表：`fields` 下的 `code`、`type`、`required`
  * 数据库 ID：`basic.database.dbId`（`sql create` 时需要）

### 3. `rabetbase sql pull --sqlcode xxx` / `rabetbase sql create --name ... --db-id ... --mode ...`
* **用途**：把 SQL 先落到本地同步目录，再编辑。
* **动作**：
  * 修改已有 SQL → 先 `sql pull --sqlcode <sqlCode>`
  * 新建 SQL → 先 `sql create --name <sqlName> --db-id <dbId> --mode sql|mybatisXml`
* **目录**：长期维护的文件统一放在 `.rabetbase/sql/<appCode>/<dbName|db-<id>>/<sqlCode>_<sqlName>.sql|xml`

### 4. `rabetbase sql validate --file xxx --format json`
* **用途**：推送前的建议卡点。
* **动作**：如果 `valid: false`，必须根据 `errors` 提示修改 SQL 内容，然后重新执行此命令，直到通过。

### 5. `rabetbase sql status --format json`
* **用途**：确认当前本地文件是否被识别为 `modified` / `added` / `missing` / `unchanged`。
* **动作**：若状态与预期不符，先解释原因再继续，不要跳过状态检查直接推送。

### 6. `rabetbase sql push --sqlcode xxx --dry-run|--yes --format json`
* **用途**：将同步目录中的本地文件上传到平台。
* **动作**：
  * 先 `--dry-run` 看预览
  * 再 `--yes` 正式执行
* **补充**：
  * 文件名变化会驱动远端 `sqlName` 更新
  * 文件移动到新的数据库目录时，会尝试按目录重绑 `dbId`
  * 如果提示 `missing remote version`，先执行 `sql pull`

### 7. `rabetbase sql exec --sqlcode xxx --format json`
* **用途**：推送成功后的功能测试。
* **动作**：测试失败（如语法错、数据不符合预期）时，必须在本地文件中修改 SQL -> 重新 validate -> 重新 status -> 重新 push -> 重新 execute，形成闭环。

### 8. `rabetbase sql save`（已废弃）
* **用途**：不要再作为主路径使用。
* **替代**：
  * 新建 SQL → `sql create`
  * 修改已有 SQL → 编辑同步目录文件后 `sql push`

---

## 📚 MyBatis 动态 SQL 语法参考（核心）

平台支持 MyBatis 语法。在自定义 SQL 中处理动态参数时，必须遵守以下规范。

### 简单参数 vs 动态参数

| 类型 | 语法 | `jdbcType` | 适用场景 |
|---|---|---|---|
| **简单参数** | `#{param}` | ❌ 不加 | 必定传入的固定条件 |
| **动态参数** | `#{param, jdbcType=TYPE}` | ✅ 必须加 | `<if>` 标签内的可选条件 |

### 动态 SQL 示例（推荐）

```sql
SELECT id, name, status_code 
FROM company
<where>
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
