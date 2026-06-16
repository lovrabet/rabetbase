# menu update

批量更新线上菜单的 CDN 资源 URL。

> **风险等级：high-risk-write** — 修改线上菜单运行资源；非交互正式执行必须显式 `--yes`。

## 命令

```bash
# 交互模式（TTY）
rabetbase menu update

# 静默更新（指定 URL）
rabetbase menu update --yes --params '{"jsUrl":"https://...js","cssUrl":"https://...css"}'

# 仅替换 CSS，保留已有 JS
rabetbase menu update --mode patch --yes --params '{"cssUrl":"https://...css"}'

# 预览资源变更，不写入
rabetbase menu update --mode patch --params '{"cssUrl":"https://...css"}' --dry-run --format json
```

## 高频 SOP：修改菜单资源 URL

修改菜单中的 JS / CSS 资源 URL 是 `menu update` 的主路径，不需要 `menu detail` 或其他配置更新命令。

1. **先确认资源现状**，只看已配置资源的菜单：

```bash
rabetbase menu list --format json --jq '.data.menus[] | select(.resources | length > 0)'
```

2. **选择更新模式**：

- 只换 JS 或只换 CSS：默认用 `--mode patch`，保留未传入类型的资源。
- 同时换 JS + CSS：仍可用 `--mode patch`；只有确实要把菜单 resources 改成传入 URL 的完整集合时，才用 `replace`。
- 不要用裸 `replace` 做“只换 CSS”，否则可能删除已有 JS；出现 JS 删除 warning 时，除非用户明确要求，否则停止。

3. **必须先 dry-run**，检查 `diffs[]` 的 `before.resources`、`after.resources` 和 `warnings`：

```bash
rabetbase menu update --mode patch --params '{"cssUrl":"https://...css"}' --dry-run --format json
```

4. **正式执行必须复用 dry-run 的同一组 `--mode` 和 `--params`**：

```bash
rabetbase menu update --mode patch --yes --params '{"cssUrl":"https://...css"}'
```

5. **执行后回查资源现状**：

```bash
rabetbase menu list --format json --jq '.data.menus[] | select(.resources | length > 0)'
```

### 常用模板

```bash
# 只替换 CSS，保留已有 JS
rabetbase menu update --mode patch --params '{"cssUrl":"https://cdn.example.com/app.css"}' --dry-run --format json
rabetbase menu update --mode patch --yes --params '{"cssUrl":"https://cdn.example.com/app.css"}'

# 只替换 JS，保留已有 CSS
rabetbase menu update --mode patch --params '{"jsUrl":"https://cdn.example.com/app.js"}' --dry-run --format json
rabetbase menu update --mode patch --yes --params '{"jsUrl":"https://cdn.example.com/app.js"}'

# 同时替换 JS + CSS
rabetbase menu update --mode patch --params '{"jsUrl":"https://cdn.example.com/app.js","cssUrl":"https://cdn.example.com/app.css"}' --dry-run --format json
rabetbase menu update --mode patch --yes --params '{"jsUrl":"https://cdn.example.com/app.js","cssUrl":"https://cdn.example.com/app.css"}'
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--params <json>` | string | 非交互必填 | — | 预填 JS/CSS URL。JSON 格式：`{"jsUrl":"...","cssUrl":"..."}`；至少包含 `jsUrl` 或 `cssUrl` |
| `--yes` | boolean | 否 | — | 非交互：跳过确认并更新所有已有资源的菜单，必须配合有效 `--params` |
| `--mode <mode>` | string | 否 | `replace` | `replace` 将传入 URL 作为完整 `resources`；`patch` 只替换传入的同类型资源并保留其他资源 |
| `--force` | boolean | 否 | `false` | 允许 `replace` 删除已有 JS 资源；仅用于明确的资源降级/迁移 |
| `--dry-run` | boolean | 否 | `false` | 只输出每个菜单的 before/after resources diff，不写入 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 三种执行模式

1. **`--yes` + `--params`** — 静默批量更新所有已有资源的菜单
2. **`--params` + 非 TTY** — 静默更新，使用传入的 URL
3. **TTY 交互式** — 展示有资源的菜单列表 → 输入 JS URL → 输入 CSS URL → 确认 → 执行更新

## 输出

- 成功：`✓ Menu update completed: N menu(s) updated`
- 无目标：`! No menus with existing resources found`
- 部分失败：`! N menu(s) failed`
- `--dry-run`：返回 `diffs[]`，包含 `id / label / path / before.resources / after.resources / warnings`

## 提示

- 仅更新已配置了资源 URL 的菜单（`filterMenusWithResources` 过滤）
- `--yes` 不能裸跑；空 `--params` 会被拒绝，避免把已有资源批量更新为空数组
- 写入采用 read-modify-write：先读取线上菜单，再只更新 `extend.resources` / `loadScriptMode`，保留 `extend` 里的其他字段
- 默认 `replace` 模式会阻止删除已有 JS 资源；如只想替换 CSS，用 `--mode patch`
- 典型场景：新版本构建后批量更新 CDN 地址
- 交互模式会展示受影响菜单的摘要表

## 常见错误

- 跳过 `--dry-run` 直接写入。
- `--yes` 裸跑或传空 `--params`。
- 只想换 CSS 却使用默认 `replace`，导致已有 JS 被删除风险。
- dry-run 已出现删除 JS warning，仍未取得用户明确确认就继续执行。
- 把资源 URL 修改误判为需要 `menu detail` / `config-update`；资源 URL 变更统一使用 `menu update`。

## 参考

- [SKILL.md](../SKILL.md)
- [menu list](rabetbase-menu-list.md)
- [menu sync](rabetbase-menu-sync.md)
