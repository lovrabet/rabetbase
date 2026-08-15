# dataset business-groups

按数据库连接读取已有业务场景分组及有效 Dataset 数量。该命令为只读发现入口，统计范围固定为 `DB_TABLE`，不包含 METADATA。

## 命令

```bash
rabetbase dataset business-groups --dbid 10282 --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--dbid` | **是** | dblink id（先从 `db list` 获取） |
| `--appcode` | 否 | 覆盖当前 app |

## 输出

- `scope` 固定为 `DB_TABLE`。
- `totalDatasetCount` 是该连接下参与统计的有效 DB_TABLE Dataset 数。
- `groups[]` 只包含 `businessGroup` 与 `datasetCount`；空字符串原样表示未分组。
- 本命令不修改分组。写入仍使用 `dataset business-group-update --code ...`，建议先复用这里发现的已有业务场景名。

## 参考

- [dataset business-group-update](rabetbase-dataset-business-group-update.md)
- [dataset list](rabetbase-dataset-list.md)
