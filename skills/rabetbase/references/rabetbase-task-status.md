# task status

按 `taskId + appCode` 查询平台通用异步任务是否到达终态。该命令不限定业务任务类型。

> **风险等级：read** — 只查询任务，不取消、重启或重提业务请求。

## 命令

```bash
rabetbase task status --task-id <taskId> --format compress
```

## Flags

| Flag | 类型 | 必填 | 说明 |
|---|---|---|---|
| `--task-id <id>` | string | 是 | 异步接口返回的任务 ID |
| `--appcode <code>` | string | 否 | 任务归属应用；未传时使用当前应用 |

## 输出与决策

| status | `isTerminal` | 动作 |
|---|---:|---|
| `PENDING` | `false` | 等待后再查询同一 taskId |
| `PROCESSING` | `false` | 等待后再查询同一 taskId |
| `SUCCESS` | `true` | 任务成功；按业务命令的回读步骤验证最终事实 |
| `FAILED` | `true` | 任务失败；读取 `errorMessage`，不自动重提原业务请求 |

未知 status 返回 `knownStatus=false`，必须人工复核，不推断为成功、失败或终态。

## Agent 编排规则

- `PENDING / PROCESSING` 时执行响应中的 `data.query.command`，始终查询同一个 taskId。
- 只有 `SUCCESS` 才能按来源业务命令的回读步骤确认最终完成。
- `FAILED` 时报告 `errorMessage`，不得自动重提来源业务请求。
- 未知状态进入人工复核，不循环提交来源业务请求。

## 相关命令

- [menu regroup-start](rabetbase-menu-regroup-start.md)
