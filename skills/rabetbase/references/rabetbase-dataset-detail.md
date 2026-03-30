# dataset detail

获取指定 Dataset 的完整详情，包含字段定义和操作列表。

## 命令

```bash
rabetbase dataset detail --code 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase dataset detail --alias order --format json
rabetbase dataset detail --code xxx --verbose --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--code <code>` | string | 与 alias 二选一 | — | Dataset code（32 位 hex UUID） |
| `--alias <name>` | string | 与 code 二选一 | — | `api.ts` 中定义的别名 |
| `--verbose` | boolean | 否 | — | 返回完整原始对象 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

返回 dataset 元数据、字段列表（含类型/PK/FK/枚举值）、操作列表。

## 提示

- 写 SQL / BFF 前必须先调用此命令确认真实字段名和类型
- `--alias` 需要先执行 `rabetbase api pull` 生成 `api.ts`
- 使用 `--verbose` 获取关联关系、枚举定义等完整信息

## 参考

- [SKILL.md](../SKILL.md)
