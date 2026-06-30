# rabetbase workspace

配置当前工作目录使用哪个 Lovrabet 应用。它是面向工作空间的推荐入口，会写入当前目录的 `.rabetbase.json`，不修改全局配置。

## 命令

```bash
rabetbase workspace init --appcode <code> [--env daily|production]
rabetbase workspace init --app <name> --appcode <code>
rabetbase workspace use --app <name>
rabetbase workspace use --app <name> --appcode <code>
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

## 行为

- `workspace init --appcode app-xxx`：用 appcode 作为应用名，写入 `apps.app-xxx` 和 `defaultApp`。
- `workspace init --app crm --appcode app-xxx`：用业务名 `crm` 绑定 appcode，写入 `apps.crm` 和 `defaultApp`。
- `workspace use --app crm`：切换当前目录默认应用；如果 `crm` 只存在于全局配置，会复制非敏感 profile 字段到当前目录。
- `workspace use --app crm --appcode app-xxx`：直接在当前目录新增或更新 `crm`，并设为默认应用。

`workspace` 不会从全局配置复制 `cookie` / `accessKey` 到项目文件。若当前项目文件里原本已有这些字段，更新应用时会保留它们。

## 输出

结构化输出会包含：

- `data.operation`：`workspace.init` 或 `workspace.use`
- `data.configPath`：实际写入的配置文件
- `data.app` / `data.appcode`
- `data.credentialsWritten`：固定表示本命令是否写入凭证

## 参考

- [rabetbase app list](rabetbase-app-list.md) — 查看本地或平台应用列表
- [rabetbase app add](rabetbase-app-add.md) — 维护项目/全局应用配置
- [`.rabetbase.json` 配置参考](rabetbase-config.md) — 配置文件结构
