# codegen sql

根据已保存的 SQL 查询生成 TypeScript 调用代码。

## 命令

```bash
rabetbase codegen sql --sqlcode 2305f915-dd48cd4c --format json
rabetbase codegen sql --sqlcode 2305f915-dd48cd4c --target bff --format json
rabetbase codegen sql --sqlcode 2305f915-dd48cd4c > ./src/api/getUserList.ts
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--sqlcode <code>` | string | 是 | — | SQL code 标识符（格式：`xxxxxxxx-xxxxxxxx`） |
| `--target <target>` | string | 否 | `sdk` | 生成目标：`sdk`（SDK 调用） / `bff`（Backend Function 脚本内调用） |
| `--no-imports` | boolean | 否 | — | 不包含 import 语句 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

生成的 TypeScript 代码直接打印到 stdout。`--target sdk` 生成前端 SDK 调用代码，`--target bff` 生成 Backend Function 脚本内调用代码。

## 提示

- `--target bff` 生成的代码使用 `context.client.sql.byName(sqlName).execute({ params })`；生成时仍以 `--sqlcode` 定位 SQL 元数据，但产物不硬编码 `sqlCode`。Backend Function 中 SQL 返回直接是数组
- `--target sdk` 生成的代码使用 `client.sql.execute()`，返回 `{ execSuccess, execResult }`
- `sqlName` 在当前应用内必须唯一；未找到或重名时，运行时返回 `SQL_NAME_NOT_FOUND` 或 `SQL_NAME_AMBIGUOUS`，不会任选一个 SQL
- `--target bff` 产物默认使用语义 SQL 名，`sql.execute({ sqlCode, params })` 是 Backend Function 的兼容调用方式；前端 SDK 产物使用 `sqlCode`

## 参考

- [SKILL.md](../SKILL.md)
