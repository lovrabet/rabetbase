# rabetbase app list

列出**合并后**多应用配置中的全部应用：与 CLI 运行时一致，合并 **全局** `~/.rabetbase.json`（或同目录下首个匹配的旧名）与**项目** `process.cwd()` 下的配置文件。`app add` 默认写项目文件（`--global` 写全局）。`app use` / `app remove` 按应用名所在层与 `--global` 决定写入项目或全局文件（见各子命令 reference）。

## 命令

```bash
rabetbase app list
```

> **说明：** `App Management` 未设置默认子命令，单独执行 `rabetbase app`（无子命令）会打印该分组的帮助，**不会**自动执行 `list`。必须显式写 `app list`。

## 参数

无额外参数。

## 结构化输出（`--format json` / `compress`）

多应用模式下，`data` 为对象（**不是**顶层数组），便于携带元信息：

| 字段 | 说明 |
|------|------|
| `items` | 应用条目数组 |
| `meta` | `globalPath`、`projectPath`（解析到的配置文件路径，无则为 `null`）、`defaultApp`、`defaultAppSource`（`project` \| `global` \| `null`） |

每个 `items[]` 元素除 `name`、`appcode`、`env` 等外，另有：

| 字段 | 说明 |
|------|------|
| `definedIn` | `global` — 仅出现在全局配置；`project` — 仅项目；`both` — 两边都有同名 key（合并后以项目 profile 为准） |
| `isDefault` | 是否为当前解析出的默认应用 |
| `isCurrent` | 是否为当前激活应用 |

**jq 示例**（只取应用名）：`--format json --jq '.data.items[].name'`

单应用模式（无 `apps`、仅有顶层 `appcode`）：`data` 仍为单应用说明对象（`mode: "single-app"` 等）。  
无任何应用时：`data` 可能为空数组 `[]`（见命令 `message`）。

## 人类可读（`pretty` / `table`）

先打印全局/项目配置文件路径、当前 **Default app** 及默认来自哪一侧文件，再列出各应用。

## 提示

- 若列表与预期不符，先检查项目或全局 JSON **是否为合法 JSON**（例如尾随逗号会导致该侧配置被忽略）；`rabetbase doctor` 的 **Config JSON** 段可检测语法。
- 无 `apps` 时显示单应用信息，并提示可用 `app add` 迁移到多应用模式。

## 参考

- [SKILL.md](../SKILL.md) — 总索引
- [`.rabetbase.json` 配置参考](rabetbase-config.md) — 合并策略与路径
- [rabetbase doctor](rabetbase-doctor.md) — 配置与 JSON 诊断
- [rabetbase app add](rabetbase-app-add.md) — 添加应用
- [rabetbase app use](rabetbase-app-use.md) — 切换默认应用
