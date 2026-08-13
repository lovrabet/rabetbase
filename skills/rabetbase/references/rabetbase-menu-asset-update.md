# menu asset-update

更新指定线上微前端菜单的 CDN 资源 URL。

> **风险等级：write** — 修改线上菜单运行资源；建议先 dry-run，正式执行复用相同参数并移除 `--dry-run`。

## 命令

```bash
# 交互模式（TTY）
rabetbase menu asset-update

# 使用 menu list 返回的真实 path 精确预览并更新
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://...css"}' --dry-run --format compress
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://...css"}' --format compress

# 按菜单 ID 精确更新，并显式切换加载方式
rabetbase menu asset-update --menu-ids "<menu-id>" --load-mode fetch --params '{"jsUrl":"https://...js"}' --dry-run --format compress

# 明确需要全量发布时显式选择全部
rabetbase menu asset-update --all --params '{"jsUrl":"https://...js","cssUrl":"https://...css"}' --dry-run --format compress
```

## 高频 SOP：修改菜单资源 URL

修改菜单中的 JS / CSS 资源 URL 是 `menu asset-update` 的主路径，不需要 `menu detail` 或其他配置更新命令。

1. **先确认资源现状**，只看已配置资源的菜单：

```bash
rabetbase menu list --format json --jq '.data.menus[] | select(.resources | length > 0)'
```

2. **精确选择目标**：

- 优先从 `menu list` 取得稳定 `id` 或 `path`，使用 `--menu-ids` / `--paths`。
- 只有明确要更新所有已有 resources 的菜单时才使用 `--all`。
- `--all` 不能与 ID/path 组合；未知 ID/path 会失败，不会静默扩大更新范围。

3. **选择更新模式**：

- 默认 `--mode patch`，只替换传入类型并保留其他资源。
- 被更新类型已有多个资源时，单个 `jsUrl` / `cssUrl` 无法确定替换目标，patch 会拒绝；`--force` 不绕过该歧义。
- 同时换 JS + CSS：仍可用 `--mode patch`；只有确实要把菜单 resources 改成传入 URL 的完整集合时，才用 `replace`。
- 不要用裸 `replace` 做“只换 CSS”，否则可能删除已有 JS；出现 JS 删除 warning 时，除非用户明确要求，否则停止。

4. **必须先 dry-run**，检查 `diffs[]` 的 `before.resources`、`after.resources` 和 `warnings`；显式传 `--load-mode` 时还要核对 before/after `loadScriptMode`：

```bash
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://...css"}' --dry-run --format compress
```

5. **正式执行必须复用 dry-run 的同一组 selector、`--mode` 和 `--params`**：

```bash
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://...css"}' --format compress
```

6. **执行后回查资源现状**：

```bash
rabetbase menu list --format json --jq '.data.menus[] | select(.resources | length > 0)'
```

### 常用模板

```bash
# 只替换 CSS，保留已有 JS
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://cdn.example.com/app.css"}' --dry-run --format compress
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://cdn.example.com/app.css"}' --format compress

# 只替换 JS，保留已有 CSS
rabetbase menu asset-update --menu-ids "<menu-id>" --params '{"jsUrl":"https://cdn.example.com/app.js"}' --dry-run --format compress
rabetbase menu asset-update --menu-ids "<menu-id>" --params '{"jsUrl":"https://cdn.example.com/app.js"}' --format compress

# 显式更新全部菜单的 JS + CSS
rabetbase menu asset-update --all --params '{"jsUrl":"https://cdn.example.com/app.js","cssUrl":"https://cdn.example.com/app.css"}' --dry-run --format compress
rabetbase menu asset-update --all --params '{"jsUrl":"https://cdn.example.com/app.js","cssUrl":"https://cdn.example.com/app.css"}' --format compress
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--params <json>` | string | 非交互必填 | — | 预填 JS/CSS URL。JSON 格式：`{"jsUrl":"...","cssUrl":"..."}`；至少包含 `jsUrl` 或 `cssUrl` |
| `--menu-ids <csv>` | string | 至少一个 selector | — | 精确菜单 ID；只接受正安全整数；可与 paths 组合，不可与 all 组合 |
| `--paths <csv>` | string | 至少一个 selector | — | 精确菜单 path；可与 menu-ids 组合，不可与 all 组合 |
| `--all` | boolean | 至少一个 selector | `false` | 显式选择所有已有 resources 的菜单；不可与 ID/path 组合 |
| `--mode <mode>` | string | 否 | `patch` | `patch` 只在目标唯一时替换传入的同类型资源并保留其他资源；`replace` 将传入 URL 作为完整 `resources` |
| `--load-mode <mode>` | string | 否 | 保留线上值 | 显式修改为 `import`、`script` 或 `fetch` |
| `--force` | boolean | 否 | `false` | 允许 `replace` 删除已有 JS 资源；仅用于明确的资源降级/迁移 |
| `--dry-run` | boolean | 否 | `false` | 输出每个菜单的 before/after resources；显式修改加载方式时同时输出 loadScriptMode diff，不写入 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 执行模式

1. **selector + `--params` + `--dry-run`** — 非交互预览精确目标；全量必须显式 `--all`
2. **selector + `--params`** — 正式更新精确目标；必须复用 dry-run 参数并移除 `--dry-run`
3. **TTY 交互式** — 未给 selector 时展示全部有资源菜单并二次确认；给 selector 时只展示目标

## 输出

- 成功：`✓ Menu asset update completed: N menu(s) updated`
- 无目标：`! No menus with existing resources found`
- 部分失败：`! N menu(s) failed`
- `--dry-run`：返回 `diffs[]`，包含 `id / label / path / before.resources / after.resources / warnings`；显式传 `--load-mode` 时还包含 before/after `loadScriptMode`

## 提示

- 仅更新已配置了资源 URL 且被 selector 命中的菜单
- 非交互和 dry-run 缺 selector 会被拒绝；空 `--params` 也会被拒绝
- 写入采用 read-modify-write：默认只更新 `extend.resources`，保留 `loadScriptMode` 和其他扩展字段
- 只有显式传 `--load-mode` 才修改加载方式
- 默认 `patch`；使用 `replace` 时仍会阻止无意删除已有 JS
- patch 被更新类型已有多个资源时会拒绝，不通过 `--force` 猜测替换目标
- 典型场景：新版本构建后批量更新 CDN 地址
- 交互模式会展示受影响菜单的摘要表

## 常见错误

- 跳过 `--dry-run` 直接写入。
- 缺 selector 或传空 `--params`。
- 需要单菜单更新却使用 `--all`。
- 只想换 CSS 却显式使用 `replace`，导致已有 JS 被删除风险。
- dry-run 已出现删除 JS warning，仍未取得用户明确确认就继续执行。
- patch 报告同类型资源不唯一时尝试用 `--force` 绕过；应先确认完整资源集合，只有明确重写时才改用 `replace`。
- 把资源 URL 修改误判为需要 `menu detail` / `config-update`；资源 URL 变更统一使用 `menu asset-update`。

## 参考

- [SKILL.md](../SKILL.md)
- [menu list](rabetbase-menu-list.md)
- [menu external-link-update](rabetbase-menu-external-link-update.md)
- [menu sync](rabetbase-menu-sync.md)
