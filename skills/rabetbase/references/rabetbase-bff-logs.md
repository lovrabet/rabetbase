# bff logs

查询当前 App 的 Backend Function 运行日志，用于开发调试。命令只读，不执行或修改脚本。

## 命令

```bash
rabetbase bff logs --format compress
rabetbase bff logs --since 10 --level ERROR --keyword timeout --format compress
rabetbase bff logs --start-time 1788433200000 --end-time 1788435000000 --limit 500 --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--app <name>` | string | 否 | — | 多应用模式下指定应用名称 |
| `--appcode <code>` | string | 否 | — | 直接指定 App Code |
| `--since <minutes>` | number | 否 | `30` | 从结束时间向前查询的分钟数；不能与 `--start-time` 同时使用 |
| `--start-time <ms>` | number | 否 | — | 查询开始时间，Unix 毫秒时间戳 |
| `--end-time <ms>` | number | 否 | 当前时间 | 查询结束时间，Unix 毫秒时间戳 |
| `--level <level>` | string | 否 | — | `DEBUG` / `INFO` / `WARN` / `ERROR` |
| `--keyword <text>` | string | 否 | — | 对服务端格式化后的整行日志做关键字过滤 |
| `--limit <count>` | number | 否 | `200` | 最大返回条数，范围 `1-1000` |

## 时间范围

- 未传时间参数时查询最近 30 分钟。
- `--end-time` 可与 `--since` 配合，查询以指定结束时间为基准的时间窗。
- 需要精确范围时传 `--start-time`，可同时传 `--end-time`；未传结束时间时使用当前时间。
- `--since` 与 `--start-time` 互斥，时间戳必须为正整数，开始时间不得晚于结束时间。

## 输出

结构化输出的 `data` 包含：

- `query`：最终发送的 `appCode`、`startTime`、`endTime`、可选过滤条件和 `limit`
- `count`：本次返回的日志行数
- `logs`：服务端已格式化的原始日志字符串数组

无结果时先确认 App Code、运行环境和时间范围，再逐步移除 `level` 或 `keyword`。不要把查询不到日志解释为脚本一定未执行。

## 参考

- [backend-function.md](../guides/backend-function.md)
- [bff detail](rabetbase-bff-detail.md)
- [bff status](rabetbase-bff-status.md)
