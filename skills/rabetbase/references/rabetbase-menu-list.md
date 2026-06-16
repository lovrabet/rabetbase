# menu list

查询当前 App 的线上菜单事实。

> **风险等级：read** — 只读查询，不修改平台菜单配置。

## 命令

```bash
# AI / 脚本优先
rabetbase menu list --format compress

# 人类阅读
rabetbase menu list --format json

# 按 path 过滤
rabetbase menu list --format json --jq '.data.menus[] | select(.path | contains("orders"))'

# 按 label 过滤
rabetbase menu list --format json --jq '.data.menus[] | select(.label | contains("用户"))'

# 仅查看配置了资源 URL 的菜单
rabetbase menu list --format json --jq '.data.menus[] | select(.resources | length > 0)'

# 排查 extend.resources 原始形态
rabetbase menu list --verbose --format json
```

如需找出重复 path、根级重名等异常菜单并生成平台手动删除清单，先按 [`menu-anomaly-manual-cleanup`](../guides/menu-anomaly-manual-cleanup.md) 执行。清单中的 `<appBaseUrl>` 由当前 `rabetbase` 环境解析：production 为 `https://app.lovrabet.com/app/<appCode>`，daily 为 `https://daily.lovrabet.com/web-app/app/<appCode>`。

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--appcode <code>` | string | 否 | 配置解析 | 显式指定 App Code，覆盖配置文件 |
| `--verbose` | boolean | 否 | `false` | 每个菜单项额外返回 `extend` 排障字段 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |
| `--jq <expr>` | string | 否 | — | 对最终 JSON 信封执行 jq；仅配合 `json` / `compress` |

`menu list` 不提供 `--path` / `--label` / `--resources-only` 等领域筛选 flag。需要过滤时使用 `--jq`。

## 输出

`json` / `compress` 信封的 `data`：

```json
{
  "total": 1,
  "menus": [
    {
      "id": 21,
      "parentId": null,
      "depth": 0,
      "label": "Orders",
      "path": "/orders",
      "type": "procode",
      "visible": true,
      "sort": 0,
      "resources": [
        "https://cdn.example.com/app.js"
      ]
    }
  ]
}
```

默认菜单项只返回稳定字段：

- `id`
- `parentId`
- `depth`
- `label`
- `path`
- `type`
- `visible`
- `sort`
- `resources`

`--verbose` 只额外返回 `extend`，不返回服务端原始 `raw`。

## 字段约定

- `menus[]` 顺序是 DFS 前序：父节点在前，子节点紧随其后，兄弟节点按 `sort` 升序。
- 字符串字段缺失时返回空字符串。
- 布尔 / 数值字段缺失时返回 `null`。
- `extend.resources` 会被归一化为 `resources: string[]`。
- `extend.resources` 为 JSON 字符串、数组、异常字符串或缺失值时，命令都不会因为资源字段解析失败而失败。

## 适用场景

- 查询当前 App 已有哪些线上菜单。
- 确认菜单路径、层级和可见状态。
- 查看菜单是否已配置 CDN 资源 URL。
- 审计重复 path、根级 label 重复、孤儿父节点等菜单异常，并生成平台人工处理清单。
- 修改菜单资源 URL 前确认当前资源现状；写入入口使用 `menu update`。
- 提供稳定的菜单事实入口，避免使用底层数据源查询菜单配置。

## 不负责

- 不做本地 `src/pages` 与线上菜单 diff；这类对比使用 `menu sync`。
- 不做智能列表页菜单审计；dataset-scoped 页面事实使用 `page standard-page-status`。
- 不修改菜单资源；更新 JS / CSS 资源 URL 使用 `menu update`，并先执行 dry-run。
- 不直接删除菜单；当前高风险清理优先生成平台手动删除清单，由用户在随当前 `rabetbase` 环境解析的 `<appBaseUrl>/pages` 或精细页面入口确认后处理。

## 参考

- [menu sync](rabetbase-menu-sync.md)
- [menu update](rabetbase-menu-update.md)
- [SKILL.md](../SKILL.md)
