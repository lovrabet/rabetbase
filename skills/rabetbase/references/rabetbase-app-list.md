# rabetbase app list

列出应用。

默认列出**本地配置视图**，与 CLI 运行时一致：合并全局 `~/.rabetbase.json`（或同目录下首个匹配的旧名）与**项目**当前目录下的配置文件。
加 `--remote` 时改为查询**平台目录视图**：当前登录账号在平台上可访问的应用，不读取或修改本地 app 配置。

- **不加 flag**：合并视图（global + project）。
- **`--global`**：仅列出**全局**文件中的应用。
- **`--project`**：仅列出**项目**文件中的应用（与 `--global` 互斥）。
- **`--remote`**：列出当前登录账号在平台上可访问的应用（与 `--global` / `--project` 互斥）。

`app add` 默认写项目（`--global` 写全局）。当前工作目录默认应用优先用 `workspace use` 修改；`app use` 仅保留兼容。`app remove` 的落盘规则见对应 reference。

## 命令

```bash
rabetbase app list
rabetbase app list --global
rabetbase app list --project
rabetbase app list --remote
```

> **说明：** `rabetbase app` 本身只显示 `app` 服务帮助；查看本地配置中的应用请显式执行 `rabetbase app list`。旧命令 `rabetbase app remote` 仍可兼容使用，但会提示迁移到 `rabetbase app list --remote`。

## 参数

| Flag | 说明 |
|------|------|
| `--global` | 仅全局配置中的应用 |
| `--project` | 仅项目配置中的应用 |
| `--remote` | 查询平台目录中的可访问应用；需要登录；不能与 `--global` / `--project` 同用 |

## 结构化输出（`--format json` / `compress`）

本地配置视图中，`data` 为对象（**不是**顶层数组），包含 `items` 与 `meta`：

| 字段 | 说明 |
|------|------|
| `items` | 应用条目数组 |
| `meta` | **合并视图**：`globalPath`、`projectPath`、`defaultApp`（无命名默认时为**有效 appcode**）、`defaultAppSource`（`project` \| `global` \| `null`）。**`--global` / `--project`**：`scope`、`configPath` |

每个 `items[]` 元素除 `name`、`appcode`、`env` 等外，另有：

| 字段 | 说明 |
|------|------|
| `named` | `true` — 来自 `apps` 中的命名应用；`false` — 仅顶层 `appcode`/`app` 的**旧版单应用**行（`name` 可能为 `null`） |
| `definedIn` | `global` / `project` / `both`（合并视图下；单边视图下为对应侧） |
| `isDefault` | 是否为当前解析出的默认应用 |
| `isCurrent` | 是否为当前激活应用（合并视图） |

**jq 示例**（只取命名应用名）：`--format json --jq '.data.items[] | select(.named) | .name'`

无任何应用时：`data` 可能为空数组 `[]`（见命令 `message`）。

`--remote` 输出同样是 `{ items, meta }`，但语义不同：

| 字段 | 说明 |
|------|------|
| `items` | 平台返回的可访问应用数组，常见字段为 `id`、`appCode`、`appName`、`appDesc` |
| `meta.source` | 固定为 `platform` |

## 人类可读（`pretty` / `table`）

合并视图：先打印全局/项目配置文件路径、当前 **Default app**（及来源），再列出各应用。`--global` / `--project` 时元信息对应该单层。`--remote` 展示平台目录返回的应用。

## 提示

- 若列表与预期不符，先检查项目或全局 JSON **是否为合法 JSON**（例如尾随逗号会导致该侧配置被忽略）；`rabetbase doctor` 的 **Config JSON** 段可检测语法。
- 无 `apps` 时显示单应用信息，并提示可用 `app add` 迁移到多应用模式。
- 如果你想知道“本地已经登记了哪些应用”，执行 `rabetbase app list`。
- 如果你想知道“当前登录账号在平台上还能访问哪些应用”，执行 `rabetbase app list --remote`。
- `rabetbase app remote` 是兼容入口，不再作为 Agent 主路径。

## 参考

- [SKILL.md](../SKILL.md) — 总索引
- [`.rabetbase.json` 配置参考](rabetbase-config.md) — 合并策略与路径
- [rabetbase doctor](rabetbase-doctor.md) — 配置与 JSON 诊断
- [rabetbase workspace](rabetbase-workspace.md) — 推荐的工作目录应用绑定入口
- [rabetbase app add](rabetbase-app-add.md) — 添加应用
- [rabetbase app use](rabetbase-app-use.md) — 兼容入口
