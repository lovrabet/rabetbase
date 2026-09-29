# menu sync

将前端子应用工程中平台缺失的页面路由注册为 `procode` 菜单。优先读取 `src/features/routing/routes.json` 中声明的页面路由，包括 `visible: false`；没有该清单时沿用 `src/pages/**/*.tsx` 扫描。命令只注册菜单与资源引用，不上传页面源码或构建资源。

数据列表页和自定义页面走各自的页面创建与发布流程，不使用本命令接入；操作入口见[宿主前端集成指南](../guides/host-frontend-integration.md)。

> **风险等级：write** — 修改平台菜单配置。

## 默认引导：Agent 按菜单树注册

前端子应用优先由 Agent 按业务菜单树规划，再调用 `menu sync` 注册页面并引用子应用共享资源。`menu sync` 是新旧流程共用的注册命令；作为兼容兜底保留的是手动扫描、逐菜单填写资源及原有批量操作方式。交互向导、`--yes` 全量同步、`--paths` 精确选择及默认根级创建均保持兼容，不增加 CLI 参数门禁。

1. 读取项目路由和 `menu list --verbose` 的 `id / parentId / type / path / extend.appName`，核实菜单宿主，按[宿主前端集成指南](../guides/host-frontend-integration.md) 确定本工程的绑定，优先复用现有分组。源码目录和 URL 层次不等于业务菜单层次。
2. 展示拟创建的菜单树，注明新增分组、页面显隐和子应用绑定。只有业务归属不明确时才询问用户；已明确的根级页面可正常创建。
3. 缺失分组用 `menu group-create` 按父子顺序创建并回读，取得真实 ID。分组重名时按父链定位，不猜测名称对应关系。
4. 用 `menu subapp-assets-status` 核查共享资源。子应用页面优先传 `appName`，通常不传菜单 JS/CSS。查询失败或共享资源缺失不等于用户选择了旧路径，不自动改写成逐菜单资源。
5. Agent 默认按明确 `--paths`、`parentId` 或路由清单里的 `group` 预演和创建。不同父节点分批处理，或复用清单中能唯一匹配的分组；不要给跨组页面统一覆盖一个父节点。用户明确采用原有全量或交互流程时，沿用其选择。
6. 用 `menu list --verbose` 回读父节点、排序、显隐及 `extend.appName` 绑定，不只按创建数量报告成功。请求结果未知时先回读，不盲目重试。已有菜单默认保留层次，移动或重组是独立操作。

子应用版本升级使用 `menu subapp-assets-update`，不批量更新菜单 URL。共享资源与菜单兜底 URL 不同，应分别报告，不能直接判断子应用版本不一致。

```bash
# 示例中的 42 应替换为已核实的目录 ID
rabetbase menu list --appcode <host-appcode> --verbose --format compress
rabetbase menu subapp-assets-status --appcode <host-appcode> --app-name store-app --format compress
rabetbase menu sync --appcode <host-appcode> --paths "/record-detail" --params '{"appName":"store-app","parentId":42,"visible":false}' --dry-run --format compress
rabetbase menu sync --appcode <host-appcode> --paths "/record-detail" --params '{"appName":"store-app","parentId":42,"visible":false}' --yes --format compress
```

## 兜底：原有终端与脚本路径

这些用法保持兼容。没有分组信息的新页面按原行为创建到根级，不会把已有菜单打平或移动。涉及业务分组时先核对预演结果，避免误把新页面全部放到根级。

```bash
# 交互向导仍可使用
rabetbase menu sync

# 查看全部待同步页面，不写入
rabetbase menu sync --dry-run --format json

# 原有全量同步仍可使用
rabetbase menu sync --yes

# 原有指定路径、菜单资源用法仍可使用
rabetbase menu sync --paths "<local-page-path>" --params '{"jsUrl":"https://cdn.example.com/app.js"}' --dry-run --format compress
rabetbase menu sync --paths "<local-page-path>" --params '{"jsUrl":"https://cdn.example.com/app.js"}' --yes --format compress
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--params <json>` | string | 否 | — | 可填 `jsUrl`、`cssUrl`、`appName`、`visible`、`parentId`。`visible` 为布尔值，`parentId` 为非负整数 |
| `--paths <csv>` | string | 否 | — | 按本地页面 path 精确选择；值来自 dry-run 的 `targetPages[].path` |
| `--yes` | boolean | 否 | — | 确认非交互执行；未传 `--paths` 时创建全部本地未上线页面 |
| `--dry-run` | boolean | 否 | `false` | 复用真实扫描、线上 diff 和 URL 校验，返回计划但不创建菜单 |
| `--format <fmt>` | string | 否 | `compress` | 输出格式 |

## 三种执行模式

1. **`--yes`** — 非交互同步；未传 `--paths` 时创建全部本地未在平台上的页面
2. **`--paths` + `--yes`** — 按本地 path 精确同步指定页面
3. **TTY 交互式** — 展示名称、path 与线上状态 → checkbox 选择未注册页面 → 填写 JS/CSS URL 或使用已配置子应用 → 确认 → 执行创建

## 输出

- 成功：`✓ Menu sync completed: N menu(s) created`
- 无需同步：`✓ All local pages are already on platform` 或 `! No local pages found in src/pages`
- `--dry-run`：返回 `targetPages` 和 `plannedMenus`；`created=0`，且不调用菜单创建接口

## 提示

- 有 `src/features/routing/routes.json` 时只同步清单中的有效页面；每项必须有存在的 `pageFile`、一致的 `path`、非空 `label` 和布尔 `visible`。隐藏页面也必须注册菜单，`visible: false` 只控制导航显隐
- 清单可选 `group` 与 `order`。显式 `--params.parentId` 优先于清单 `group`：非零 ID 指向的节点不存在或不是当前宿主中的 folder 时拒绝，`0` 表示根节点。未指定 `parentId` 时，若提供了 `group`，则必须匹配宿主中唯一的 folder；两者均未提供时，保留原有根级创建行为
- 无清单时扫描 `src/pages`，保持原有项目兼容
- 同步前会获取线上菜单列表做 diff
- path 是稳定选择器，名称只用于展示；多个 path 必须全部匹配
- path 比较统一忽略首尾斜杠，重复选择同一路径只创建一次；已在平台存在的 path 不会重复创建
- 交互模式下会展示 compare table（名称、path、本地与线上状态）
- CLI 只注册菜单和资源引用，不构建、不上传源码或静态资源
- `--params.appName` 将页面绑定到菜单宿主的同名前端子应用共享资源。菜单宿主使用 CLI 当前选中的应用，支持工作区默认应用或唯一应用，也可用 `--appcode` / `--app` 覆盖；子应用访问的业务 appCode 是独立关系。子应用资源在支持该机制的宿主中优先生效；不填 JS/CSS 时 CLI 会先确认同名共享资源非空，菜单资源留空。仍须核对目标宿主支持共享资源
- 当前 React/Vite 模板使用 `loadScriptMode=import`；资源 URL 指向已部署的 CDN 构建产物

## 参考

- [SKILL.md](../SKILL.md)
