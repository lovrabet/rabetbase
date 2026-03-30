# dataset list

列出当前 App 下所有 Dataset，支持模糊过滤。

## 命令

```bash
rabetbase dataset list --format json
rabetbase dataset list --name "用户" --format json
rabetbase dataset list --format table
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--name <name>` | string | 否 | — | 按名称/代码/表名/描述模糊过滤 |
| `--verbose` | boolean | 否 | — | 返回完整 dataset 对象 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式：`json` / `pretty` / `table` |

## 输出

默认返回精简列表（id, name, code, description, source, db, table, fields）。`--verbose` 返回原始完整对象。

## 提示

- `--name` 支持跨 name/code/tableName/description 四个字段的模糊匹配
- AI Agent 推荐始终使用 `--format json`
- 列表为客户端过滤，数据量大时可能稍慢

## 参考

- [SKILL.md](../SKILL.md)
