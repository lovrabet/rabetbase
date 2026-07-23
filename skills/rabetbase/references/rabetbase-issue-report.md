# issue report

向平台 Issue 采集链路上报一个问题。description 的核心是完整组织客观事实，供平台侧复现和排查；该入口不负责代替平台侧判断根因或设计解决方案。

## 核心边界

- 只写已经观察、执行或读取到的事实，包括环境、版本、时间、命令、输入、原始输出、状态变化、错误信息和定位标识
- 实际结果与期望结果分开记录；“期望结果”只描述用户可验证的目标，不延伸成技术实现建议
- 不确定的信息放入“尚未确认”，明确说明缺少什么证据；不得把推测写成事实
- 不得推测根因，不得提供修复方案，不得给出主观优先级或未经验证的影响判断
- 不得为了简洁删除唯一事实；可以去除完全重复的输出，但必须保留首次出现位置、重复次数和可定位标识
- Cookie、Authorization、AccessKey、密码、私钥和 JWT 等敏感值仍需由命令脱敏，不得以“保持原始事实”为由绕过安全处理

## 事实模板

```markdown
## 问题概述

用一句话描述已经发生的异常，不写原因判断。

## 环境与标识

- CLI 与版本：
- AppCode / AppEnv：
- 发生时间与时区：
- TraceId / SpanId / JobId：

## 复现步骤

1. 按实际顺序记录操作。
2. 命令、参数和必要输入保持可复现；敏感值使用安全占位符。

## 实际结果

- 原样记录状态、错误码、错误信息和关键返回字段。
- 长输出注明完整证据的位置、字节数和摘要值。

## 期望结果

- 描述用户能够验证的正确行为。

## 客观证据

- 日志、截图、文件、URL、时间线和对应标识。

## 尚未确认

- 只列缺少证据、尚未执行或平台侧需要继续核对的事实问题。
```

不要在模板中增加“根因分析”“建议修复方案”“解决方案”或“建议排查方向”等章节。

## 命令

```bash
rabetbase issue report --title "page push 保存失败" --description "## 实际现象\n- 点击保存后报错" --dry-run --format json

cat <<'EOF' >/tmp/platform-issue.md
## 问题概述
page push 保存失败

## 环境与标识
- rabetbase: 2.3.6
- AppCode: app-example

## 复现步骤
1. 执行 `rabetbase page push ...`

## 实际结果
- 命令返回 `validation_error`

## 期望结果
页面 schema 能成功推送

## 客观证据
- TraceId: `<trace-id>`

## 尚未确认
- 当前返回内容未包含具体字段信息
EOF

wc -c /tmp/platform-issue.md
rabetbase issue report --title "TEST issue smoke" --description-file /tmp/platform-issue.md --dry-run --format json
# 确认 issue.description_original_bytes 与上方字节数一致，且 issue.description_truncated=false
rabetbase issue report --title "page push 保存失败" --description-file /tmp/platform-issue.md --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--title <text>` | string | 是 | — | 一句话问题摘要 |
| `--description <markdown>` | string | 与 `--description-file` 二选一 | — | 内联 Markdown 问题描述 |
| `--description-file <path>` | string | 与 `--description` 二选一 | — | Markdown 文件路径，适合长上下文；最终会上报文件内容文本，而不是路径 |
| `--dry-run` | boolean | 否 | `false` | 仅预览将要写入的 trace/span/logData |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 行为说明

- 命令会自动补充 CLI 宿主公共上下文，不需要手动重复传入环境或用户字段
- `--description-file` 只是本地输入来源；命令会先读取文件内容，最终上报的是 `issue.description` 的 Markdown 文本，不是本机文件路径
- 命令不会自动附带 Agent 对话、工具历史或工作目录文件；只有显式写入 `--description` / `--description-file` 的事实会进入上报正文
- 单次 issue 的正文业务字段只写：
  - `issue.title`
  - `issue.description`
- 命令还会上报 `issue.description_original_bytes`、`issue.description_truncated`、`issue.description_redactions` 和 `issue.description_redaction_kinds`，用于核对完整性与脱敏结果
- `--dry-run` 中若出现 `descriptionFile`，它只用于本地确认读取了哪个文件，不属于正式上报字段
- 当前仅支持 `report`；不支持 `issue update`、`issue close`

## 上报前完整性检查

1. 先用 `wc -c /path/to/platform-issue.md` 记录源文件字节数，并确认文件包含所有唯一事实。
2. 执行 `rabetbase issue report ... --dry-run --format compress`。
3. 检查 `issue.description_original_bytes` 与源文件字节数一致，并确认 `issue.description_truncated=false`。
4. 检查 `issue.description` 的结尾仍是源文件的最后一条事实，且 `issue.description_redactions` 只命中预期的敏感信息。
5. 只有上述检查通过后才正式上报；正式提交成功前保留源 Markdown，便于逐字核对。

如果 `issue.description_truncated=true`，不得把该结果当作完整上报。去除完全重复内容，或将大体积原始日志/文件通过文件服务保存，在正文中记录其 `filePath`、访问链接、字节数和摘要值；不得静默丢弃超出部分。

## 文件与图片证据

- 如需加入图片证据，先执行 `rabetbase file upload --file <图片路径>`，再用 `rabetbase file query-url --filepath <filePath> --long-term` 获取 Markdown 图片 URL
- OSS 签名 URL 查询参数中的 `OSSAccessKeyId` 会保持原样，避免签名被脱敏改写；URL 外的 AccessKey 文本仍会脱敏
- 需要确认当前 CLI 环境、登录态和配置时，先执行 [`rabetbase doctor`](rabetbase-doctor.md)

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-file.md](rabetbase-file.md)
- [rabetbase-doctor.md](rabetbase-doctor.md)
