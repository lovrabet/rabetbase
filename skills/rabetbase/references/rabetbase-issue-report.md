# issue report

向平台 Issue 采集链路上报一个问题。该命令只负责提交问题标题与 Markdown 描述；是否产生后续平台处理结果，以平台侧实际处理链路为准。

## 命令

```bash
rabetbase issue report --title "page push 保存失败" --description "## 实际现象\n- 点击保存后报错" --format json

cat <<'EOF' >/tmp/platform-issue.md
## 问题概述
page push 保存失败

## 实际现象
- 点击保存后报错
- 命令返回 validation error

## 期望结果
页面 schema 能成功推送
EOF

rabetbase issue report --title "page push 保存失败" --description-file /tmp/platform-issue.md --format json
rabetbase issue report --title "TEST issue smoke" --description-file /tmp/platform-issue.md --dry-run --format json
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
- 单次 issue 的业务字段只写：
  - `issue.title`
  - `issue.description`
- `--dry-run` 中若出现 `descriptionFile`，它只用于本地确认读取了哪个文件，不属于正式上报字段
- 当前仅支持 `report`；不支持 `issue update`、`issue close`

## 提示

- 推荐先由 Skill 把上下文整理成 Markdown，再通过 `--description-file` 调用命令
- 需要确认当前 CLI 环境、登录态和配置时，先执行 [`rabetbase doctor`](rabetbase-doctor.md)
- 需要审计本次命令将写入什么内容时，先执行 `--dry-run`

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-doctor.md](rabetbase-doctor.md)
