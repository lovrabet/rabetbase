# menu asset-update

更新指定线上菜单自身的 CDN 资源 URL。宿主可同时集成多个前端子应用和平台页面（数据列表页、自定义页面）。平台页面按其配置与发布流程管理，需要菜单资源时，该资源是正常加载配置，无需绑定 appName；本命令不修改页面内容或代替页面发布。绑定有效子应用共享资源时，支持该机制的宿主优先加载对应子应用的共享资源，菜单资源才作为兜底。

仅在明确需要调整菜单资源时，平台页面才按菜单 ID/path 使用本命令。例如，自定义页面通过 [`page create`](rabetbase-page-create.md) 同时创建页面与菜单，后续按 pageId 使用 [`page custom-update`](rabetbase-page-custom-update.md)、[`page custom-publish`](rabetbase-page-custom-publish.md) 保存和发布，无需额外绑定 appName 或用 `menu sync` 注册菜单。前端子应用先用 `rabetbase menu subapp-assets-status --appcode <host-appcode> --app-name <name>` 核查，升级使用 `menu subapp-assets-update`；只有明确维护菜单自身的兜底配置时才使用本命令。

> **风险等级：write** — 修改菜单自身的资源配置；建议先 dry-run，正式执行复用相同参数并移除 `--dry-run`。

## 命令

```bash
# 交互模式（TTY）
rabetbase menu asset-update

# 使用 menu list 返回的真实 path 精确预览并更新
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://...css"}' --dry-run --format compress
rabetbase menu asset-update --paths "<menu-path>" --params '{"cssUrl":"https://...css"}' --format compress

# 按菜单 ID 精确更新，并显式切换加载方式
rabetbase menu asset-update --menu-ids "<menu-id>" --load-mode fetch --params '{"jsUrl":"https://...js"}' --dry-run --format compress

# 仅当明确要更新当前宿主所有已有 resources 的菜单时使用，不用于子应用版本升级
rabetbase menu asset-update --all --params '{"jsUrl":"https://...js","cssUrl":"https://...css"}' --dry-run --format compress
```

## 高频 SOP：修改菜单资源 URL

修改菜单自己的 JS / CSS 资源 URL 使用 `menu asset-update`：包括平台页面的正常资源配置，以及明确维护的子应用菜单兜底配置。不需要 `menu detail` 或其他配置更新命令。

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

# 显式更新当前宿主全部已有 resources 的菜单，不限定为某个子应用
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
- 典型场景：更新选定平台页面的 CDN 地址，或明确维护菜单兜底 URL；前端子应用升级优先更新共享资源
- 交互模式会展示受影响菜单的摘要表

## 常见错误

- 跳过 `--dry-run` 直接写入。
- 缺 selector 或传空 `--params`。
- 需要单菜单更新却使用 `--all`。
- 只想换 CSS 却显式使用 `replace`，导致已有 JS 被删除风险。
- dry-run 已出现删除 JS warning，仍未取得用户明确确认就继续执行。
- patch 报告同类型资源不唯一时尝试用 `--force` 绕过；应先确认完整资源集合，只有明确重写时才改用 `replace`。
- 修改菜单自己的资源 URL 时误用 `menu detail` / `config-update`；此类变更使用 `menu asset-update`。同名子应用共享资源版本另行管理。

## 参考

- [SKILL.md](../SKILL.md)
- [menu list](rabetbase-menu-list.md)
- [menu external-link-update](rabetbase-menu-external-link-update.md)
- [menu sync](rabetbase-menu-sync.md)
