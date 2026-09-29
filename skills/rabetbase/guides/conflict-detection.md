# 冲突检测与未完成写入处理

适用于 SQL / Backend Function 的远端写入与同步。通用查证、纠错、授权与交付原则遵循 [AI First 执行原则](../SKILL.md#ai-first-执行原则)；本指南说明如何根据实际返回结果恢复任务。

## 写入结果与恢复动作

不能只根据 `ok=false` 或非零退出断言所有资源都未保存。结合逐项结果、错误发生阶段及远端回读判断：

| 结果 | 判断与动作 |
| --- | --- |
| `--dry-run` 成功或 `bff create` 生成本地脚手架 | 只完成预览或本地准备，尚未写入远端；按对应工作流继续 |
| 资源明确写入成功 | 报告成功项及其真实标识，继续任务需要的回读与运行验证 |
| 批量操作部分成功 | 分别报告成功、跳过和失败项；保留成功项，只处理失败项，不盲目重提整批 |
| 明确拒绝写入，如 `blocked: true` / `action: "blocked"` | 说明该资源未保存；核对事实并处理授权内可恢复的原因，不换参数或入口绕过权限、平台限制 |
| 写入前的输入校验失败 | 查证并修复确认的内容或参数错误，再按命令工作流复验 |
| 超时、响应丢失等结果未知 | 先回读远端，确认是否已生效；确认前不能声称成功或未保存，也不能直接重新提交 |
| 用户取消 | 停止该操作，保留准备结果并继续其他已授权工作；不得用 `--yes` 或修改风险配置绕过取消 |

## 响应样例

### 仅预览成功

```json
{ "ok": true, "dryRun": true, "data": { "method": "POST" } }
```

`dryRun: true` 只说明预览通过，远端未执行；不能表述为已保存或已同步。

### 明确拒绝写入

```json
{ "ok": false, "blocked": true, "message": "Resource was blocked by platform" }
```

该资源未保存。引用 `message` 说明原因，处理授权内可恢复的部分；不得换参数或入口重试绕过，也不得粉饰为已保存。

### 批量部分成功

```json
{
  "ok": false,
  "data": {
    "pushed": [{ "sqlCode": "2305f915-dd48cd4c", "remoteId": 10241 }],
    "skipped": [{ "sqlCode": "7b1e0c42-90aa31de", "reason": "unchanged" }],
    "failed": [{ "sqlCode": "c48d21a7-51fe07bb", "error": "missing remote version" }]
  }
}
```

此处 `ok: false` 只因存在 `failed` 项，`pushed` 里的资源确实已写入远端。逐项按 `sqlCode` 报告，只处理 `failed` 项，不重提已成功项。

### 沟通参考

`blocked` 或平台限制时可参考：

```text
这次没有同步到 Lovrabet 平台。平台返回：<message>。
已完成 <已做的检查/准备>；需要 <具体待办> 才能继续，本地修改已保留。
```

## 同步分歧

- SQL 本地与远端漂移、缺少版本时，按 [SQL 冲突处理](sql-creation-workflow.md#冲突处理)保留修改、回读比较并恢复同步。
- `bff pull` / `bff push` 的 `data.conflicts` 表示可恢复分歧，不等同于 `failed`。逐项报告 `lockKey`、`code` 和 `nextAction`：`BFF_LOCAL_UNSYNCED` 经审阅后可精确 push；`BFF_REMOTE_VERSION_CHANGED` / `BFF_REMOTE_VERSION_MISSING` 先按返回的 `bff detail` 命令回读源码并合并。不得自动 `--force` 或在分歧未解决时声称已同步。

## 交付结果

按资源列明实际状态、返回原因、已完成的处理与验证。仍需关键决策或人工操作时，提供具体差异、推荐方案、影响和最小待办；只完成本地准备或保存配置时，不宣称业务运行已验收。

## 相关工作流

- [SQL 工作流](sql-creation-workflow.md)
- [Backend Function 工作流](bff-creation-workflow.md)
