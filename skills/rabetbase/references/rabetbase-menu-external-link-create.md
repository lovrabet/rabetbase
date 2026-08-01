# menu external-link-create

创建一个外部网站链接菜单。使用产品打开方式选择应用内嵌入或新窗口打开，不要求使用者理解服务端菜单类型。

> **风险等级：high-risk-write** — 创建后会直接影响当前 App 的运行时菜单；先 dry-run，确认后显式传入 `--yes`。

## 命令

```bash
rabetbase menu external-link-create \
  --label "帮助中心" \
  --url "https://help.example.com" \
  --open-mode new-window \
  --dry-run \
  --format compress

rabetbase menu external-link-create \
  --label "帮助中心" \
  --url "https://help.example.com" \
  --open-mode new-window \
  --yes \
  --format compress
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--label <label>` | string | 是 | — | 菜单名称 |
| `--open-mode <embedded\|new-window>` | string | 是 | — | `embedded` 应用内嵌入；`new-window` 新窗口打开 |
| `--url <https-url>` | string | 是 | — | 无内嵌用户名密码的绝对 HTTPS URL |
| `--parent-id <id\|root>` | string | 否 | `root` | 父节点必须是当前 App 中已有的 folder |
| `--sort <n>` | number | 否 | 同级末尾 | `0..2147483647` 的整数 |
| `--visible` | boolean | 否 | `true` | 是否显示菜单；隐藏菜单传 `--visible=false` |
| `--appcode <code>` | string | 否 | 配置解析 | 显式指定 App Code |
| `--dry-run` | boolean | 否 | `false` | 输出创建计划，不写入 |
| `--yes` | boolean | 正式执行必填 | — | 明确确认高风险写入 |

## 安全与输出

- `embedded` 映射为服务端 `type=iframe`；`new-window` 映射为服务端 `type=link`。HTML `_self` 表示在当前浏览上下文导航，不等同于应用内嵌入，因此不是本命令参数。
- URL 必须使用 HTTPS，且不能包含 URL userinfo。
- URL 查询参数属于菜单配置，不按参数名拦截或脱敏。
- URL 是可审计的菜单事实，dry-run、成功、结构化失败结果和 `menu list` 均按输入或线上值原样输出。
- 依赖原始异常等不可信错误文本仍会清洗，错误码只使用稳定白名单。
- 创建请求不发送 `path`，平台负责生成非空 path。
- 正式写入前重新读取菜单快照；任一菜单事实漂移都会阻断本次写入。

## 响应丢失

创建请求不自动重试。响应丢失后，只有同时满足以下条件才返回恢复成功：

1. 回查到唯一匹配的菜单；
2. 该 ID 在写入前快照中不存在；
3. ID 是正安全整数；
4. parent、type、label、URL、visible、sort 和非空 path 全部一致。

无法唯一确认时，执行错误中返回的 `verification.lookupCommand`，核对线上事实后再决定下一步，不要直接重提创建。

## 参考

- [menu list](rabetbase-menu-list.md)
- [menu group-create](rabetbase-menu-group-create.md)
- [SKILL.md](../SKILL.md)
