# rabetbase workspace

配置当前工作环境使用哪些 Lovrabet 应用。它是面向工作空间的推荐入口，一切「改我的配置」的写操作都在这里。

> **边界**：`workspace` 负责所有「配置我的工作环境」的写操作——绑定目录默认应用（`init` / `use`），以及登记/移除应用清单（`add` / `remove`）。默认写当前项目 `.rabetbase.json`；`add` / `remove` 支持 `--global`（全局视作一个大工作空间）；`init` / `use` 只写当前目录。纯事实查询「有哪些 app」用 [`app list`](rabetbase-app-list.md)。

**选型**：首次安装全局引导用 `rabetbase init`；当前目录还没绑定默认 app → `workspace init`（始终写 `defaultApp`）；已有配置只再登记 profile → `workspace add`；切换默认 → `workspace use`。

## 命令

```bash
rabetbase workspace init --appcode <code> [--env daily|production]
rabetbase workspace init --app <name> --appcode <code>
rabetbase workspace use --app <name>
rabetbase workspace use --app <name> --appcode <code>
rabetbase workspace add <name> --appcode <code> [--global]
rabetbase workspace remove <name> [--global]
```

## 参数

| Flag | 说明 |
|------|------|
| `--app <name>` | 工作空间里的应用名，如 `crm`、`order` |
| `--appcode <code>` | 应用 App Code；不知道本地应用名时可直接使用 |
| `--env <env>` | 写入该 app profile 的环境：`daily` / `production` |
| `--apiDir <dir>` | 写入该 app profile 的 API 目录 |
| `--defaultFormat <format>` | 写入该 app profile 的默认输出格式 |
| `--pageSize <n>` | 写入该 app profile 的分页大小 |
| `--riskLevel <level>` | 写入该 app profile 的风险等级 |
| `--locale <locale>` | 写入该 app profile 的地区设置 |

`add` / `remove` 用位置参数 `<name>` 指定应用名；`add` 必带 `--appcode <code>`，两者都支持 `--global`（写/删全局清单，全局视作一个大工作空间），其余 profile flags 与上表一致。

## 行为

- `workspace init --appcode app-xxx`：用 appcode 作为应用名，写入 `apps.app-xxx` 和 `defaultApp`。
- `workspace init --app crm --appcode app-xxx`：用业务名 `crm` 绑定 appcode，写入 `apps.crm` 和 `defaultApp`。
- `workspace use --app crm`：切换当前目录默认应用；如果 `crm` 只存在于全局配置，会复制非敏感 profile 字段到当前目录。
- `workspace use --app crm --appcode app-xxx`：直接在当前目录新增或更新 `crm`，并设为默认应用。
- `workspace add crm --appcode app-xxx`：登记应用 profile（默认写当前项目，`--global` 写全局）；当前无 `defaultApp` 时首个自动设为默认。已存在则更新。
- `workspace remove crm`：移除应用 profile（high-risk-write）。默认按「项目优先、其次全局」定位要删的那一层，`--global` 强制删全局；若删的是当前 `defaultApp`，自动切到剩余的第一个；若该应用在项目与全局都有定义，只删单侧并在结果里提示另一侧仍存在。

`workspace init` / `workspace use` 不会从全局配置复制 `cookie` / `accessKey` 到项目文件。若当前项目文件里原本已有这些字段，更新应用时会保留它们。

## 输出

结构化输出会包含：

- `data.operation`：`workspace.init` 或 `workspace.use`
- `data.configPath`：实际写入的配置文件
- `data.app` / `data.appcode`
- `data.credentialsWritten`：固定表示本命令是否写入凭证

`add` / `remove` 返回单行 `message`（如 `Added app "crm" ...` / `Removed app "crm" ...`），移除时若涉及默认应用切换或另一侧仍有定义会在同一 `message` 里说明。

## 参考

- [rabetbase app list](rabetbase-app-list.md) — 查看本地或平台应用列表（只读事实）
- [`.rabetbase.json` 配置参考](rabetbase-config.md) — 配置文件结构
