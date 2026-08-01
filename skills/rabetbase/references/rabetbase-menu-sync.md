# menu sync

扫描本地 `src/pages/**/*.tsx`，把线上缺失的路由注册为平台 `procode` 菜单。

> **风险等级：write** — 修改平台菜单配置。

## 命令

```bash
# 交互模式（TTY）
rabetbase menu sync

# 发现待同步页面的真实 path，不写入
rabetbase menu sync --dry-run --format json

# 按 targetPages[].path 精确预览
rabetbase menu sync --paths "<local-page-path>" --params '{"jsUrl":"https://cdn.example.com/app.js"}' --dry-run --format compress

# 静默全量同步
rabetbase menu sync --yes

# 静默部分同步（预填 URL 和精确 path）
rabetbase menu sync --paths "<local-page-path-1>,<local-page-path-2>" --params '{"jsUrl":"https://...js","cssUrl":"https://...css"}' --yes
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--params <json>` | string | 否 | — | 预填 JS/CSS URL。JSON 格式：`{"jsUrl":"...","cssUrl":"..."}` |
| `--paths <csv>` | string | 否 | — | 按本地页面 path 精确选择；值来自 dry-run 的 `targetPages[].path` |
| `--yes` | boolean | 否 | — | 确认非交互执行；未传 `--paths` 时创建全部本地未上线页面 |
| `--dry-run` | boolean | 否 | `false` | 复用真实扫描、线上 diff 和 URL 校验，返回计划但不创建菜单 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 三种执行模式

1. **`--yes`** — 非交互同步；未传 `--paths` 时创建全部本地未在平台上的页面
2. **`--paths` + `--yes`** — 按本地 path 精确同步指定页面
3. **TTY 交互式** — 展示名称、path 与线上状态 → checkbox 选择未注册页面 → 输入 JS/CSS URL → 确认 → 执行创建

## 输出

- 成功：`✓ Menu sync completed: N menu(s) created`
- 无需同步：`✓ All local pages are already on platform` 或 `! No local pages found in src/pages`
- `--dry-run`：返回 `targetPages` 和 `plannedMenus`；`created=0`，且不调用菜单创建接口

## 提示

- 本地页面扫描 `src/pages` 目录
- 同步前会获取线上菜单列表做 diff
- path 是稳定选择器，名称只用于展示；多个 path 必须全部匹配
- 已在平台存在的 path 不会重复创建
- 交互模式下会展示 compare table（名称、path、本地与线上状态）
- CLI 只注册菜单和资源引用，不构建、不上传源码或静态资源
- 当前 React/Vite 模板使用 `loadScriptMode=import`；资源 URL 指向已部署的 CDN 构建产物

## 参考

- [SKILL.md](../SKILL.md)
