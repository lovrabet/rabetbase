# db detail

按 **dblink id** 拉取单条连接的完整元数据（含分析相关字段）。密码在输出中脱敏。

## 何时用

- 已用 `db list` 拿到 `id`，需要比列表更全的字段
- 读取 `tableCount`，按 200 张表阈值决定是否先刷新差异结果
- 需要当前 **`latestAnalysisTraceId`** 查询最新分析状态；取消已知任务时仍使用正在跟踪的原 planId

## 命令

```bash
rabetbase db detail --id 10157 --format compress
```

## 参数

| Flag | 类型 | 必填 | 说明 |
|------|------|------|------|
| `--id` | number | **是** | `db list` / 工作台连接列表上的 dblink id |
| `--appcode` | string | 否 | 覆盖当前 app |
| `--format` | string | 否 | 默认 **compress** |

## 参考

输出中的 `tableCount` 是当前连接表数量：不大于 200 时通常直接读取 `db diff`；大于 200 时建议先执行 `db diff-refresh-start/status`。字段缺失时不猜测数量，也不隐式刷新。

- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
