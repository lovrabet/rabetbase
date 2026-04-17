# 冲突检测与未完成写入处理

**文档版本**: v4.0
**适用于**: SQL / BFF 资源的远端写入、同步与历史兼容返回

> **与 SKILL.md 的关系**：[SKILL.md](../SKILL.md) 只保留硬性规则；本文件负责解释什么叫“远端未写入成功”，以及 AI 应该如何沟通。

---

## 当前主工作流

当前团队主路径不是 `save`：

- **SQL**
  - 新建：`rabetbase sql create`
  - 修改：编辑 `.rabetbase/sql/...` 下的同步文件后执行 `rabetbase sql push`
  - 删除：`rabetbase sql delete`
- **BFF**
  - 新建本地脚手架：`rabetbase bff create`
  - 推送远端：`rabetbase bff push`
  - 删除远端：`rabetbase bff delete`

`sql save` 只保留为**迁移提示**入口；`bff save` 不再是主工作流命令。AI 不应再围绕它们设计新步骤。

---

## 什么时候算“没有写入成功”

只要 CLI 返回结果表明**远端没有完成创建 / 更新 / 删除 / 同步**，就必须按失败处理。常见信号包括：

- `blocked: true`
- `action: "blocked"`
- `success: false`
- `ok: false`
- 返回 message 明确表示冲突、被平台拦截、校验失败、未同步
- 非零退出且错误原因发生在远端写入前后

**核心原则**：

- 本地文件已生成，不等于远端已保存
- `--dry-run` 预览成功，不等于远端已执行
- 命令执行失败时，不能用“已处理”“已保存”这类表述蒙混过去

---

## AI 的沟通义务

只要远端未写入成功，回复里必须同时做到：

1. **明确状态**
   - 直接说“未保存到平台”“未完成远端同步”“只完成了本地生成/预览”。
2. **说明原因**
   - 引用返回里的 `message`、`error.message`、`blocked`、或关键上下文。
3. **给出下一步**
   - 去平台处理
   - 修改本地内容后重试
   - 先 `--dry-run`
   - 回到 `create + 本地编辑 + push` 主路径

---

## 典型分支

| 场景 | 是否算远端成功 | AI 应怎么说 |
|------|----------------|-------------|
| `sql create --dry-run` 成功 | 否 | 说明只是预览，尚未创建远端 SQL |
| `bff create` 成功 | 否 | 说明本地脚手架已生成，尚未推送远端 |
| `sql push --yes` 成功 | 是 | 告知已同步远端，可附带 sqlCode / 路径 |
| `bff push --yes` 成功 | 是 | 告知已推送远端，可附带 lockKey / 名称 |
| `blocked: true` / `action: blocked` | 否 | 明确说平台未保存，提示手动处理或联系相关人 |
| 校验失败 / validation error | 否 | 说明未写入远端，先修内容或参数 |
| `sql save` 返回 deprecated | 否 | 说明旧命令未执行保存，引导迁移到 `sql create` / `sql push` |

---

## blocked 场景

当返回 `blocked: true` 或语义等价的“平台冲突 / 非本人资源 / 平台限制”时：

- 必须明确说：**这次没有写入到 Lovrabet 平台**
- 可以建议：去平台手动处理、联系上次提交人、或先保留本地草稿
- 禁止：重试绕过、换参数偷偷再试、把失败包装成成功

建议话术：

```text
这次没有同步到 Lovrabet 平台。平台返回了冲突/阻断提示：<message>。
如果需要继续修改，请先到平台处理冲突，或联系上次提交人；本地内容可以先保留。
```

---

## 历史兼容返回

如果用户或旧脚本仍触发历史 `save` 链路：

- `sql save`
  - 当前应视为**迁移提示**
  - AI 要直接引导到 `sql create` / `sql push`
- `bff save`
  - 不再作为当前主命令说明
  - 若文档或用户提到它，统一迁移到 `bff create` / `bff push`

---

## 响应示例

### 仅预览成功

```json
{
  "ok": true,
  "dryRun": true,
  "data": {
    "method": "POST"
  }
}
```

解释：**预览成功，不等于远端已执行**。

### 迁移提示

```json
{
  "ok": false,
  "error": {
    "message": "`rabetbase sql save` has been deprecated."
  }
}
```

解释：**旧命令未执行保存，应该迁移流程**。

### 冲突/阻断

```json
{
  "ok": false,
  "blocked": true,
  "message": "Resource was blocked by platform"
}
```

解释：**远端未写入成功，必须明确告知用户**。

---

## 相关文档

- **SQL 工作流**: `sql-creation-workflow.md`
- **BFF 工作流**: `bff-creation-workflow.md`
- **最佳实践**: `best-practices.md`
