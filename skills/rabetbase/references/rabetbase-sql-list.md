# sql list

列出当前 App 下的自定义 SQL 查询。

## 命令

```bash
rabetbase sql list --format json
rabetbase sql list --name getUserList --format json
rabetbase sql list --sqlcode 2305f915-dd48cd4c --format json
rabetbase sql list --page 2 --pagesize 20 --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--sqlcode <code>` | string | 否 | — | 按 SQL code 过滤（格式：`xxxxxxxx-xxxxxxxx`） |
| `--name <name>` | string | 否 | — | 按名称过滤 |
| `--page <n>` | number | 否 | `1` | 页码 |
| `--pagesize <n>` | number | 否 | `50` | 每页数量 |
| `--verbose` | boolean | 否 | — | 返回完整 SQL 对象（含 SQL 内容和参数） |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 多应用过滤

`sql list` 总是查询**当前已经决议到的单个应用**：

- 默认使用当前项目 `defaultApp`
- 传 `--app <name>` 时，切到指定本地应用名
- 传 `--appcode <code>` 时，直接使用指定 appcode

```bash
# 仅列出 order 应用的 SQL
rabetbase sql list --app order

# 指定 appcode
rabetbase sql list --appcode app-xxxxxxxx
```

## 输出

默认返回精简列表（sqlCode, sqlName, description, db）。`--verbose` 返回完整对象含 SQL 内容。

## 提示

- `--name` 是服务端过滤，`--sqlcode` 也是服务端过滤
- 修改已有 SQL 前先用此命令找到 sqlCode
- SQL Code 格式为 `{8位hex}-{8位hex}`，如 `2305f915-dd48cd4c`

## 参考

- [SKILL.md](../SKILL.md)
- [sql-creation-workflow.md](../guides/sql-creation-workflow.md)
