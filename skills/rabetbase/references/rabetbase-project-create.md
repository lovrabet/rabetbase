# project create

创建新项目。

> **风险等级：write** — 创建项目文件和配置。

## 命令

```bash
# 交互模式
rabetbase project create

# 指定项目名
rabetbase project create my-project

# 通过 flag 指定
rabetbase project create --name my-project

# 非交互模式（必须传 name）
rabetbase project create my-project --appcode <code> --env production
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `<project-name>` | positional | 否 | — | 项目名称（也可用 `--name`） |
| `--name <name>` | string | 否 | — | 项目名称（优先于位置参数） |
| `--env <env>` | string | 否 | — | 目标环境 |
| `--appcode <code>` | string | 否 | — | 绑定的应用 code（跳过交互选择） |

## 两种模式

1. **交互模式** — 通过提示选择项目配置
2. **非交互模式**（`--non-interactive` 或非 TTY）— 必须提供项目名称，支持 `--appcode` 跳过应用选择

## 提示

- `--name` 和位置参数二选一，`--name` 优先
- 非交互模式下缺少项目名会报错
- 项目名只接受字母、数字、`-`、`_`，不能传绝对路径、`../`、子目录路径或 Windows 保留名称
- 交互与非交互模式使用同一创建流程，都会安装依赖、格式化代码并写入项目配置
- 项目模板从 CDN 下载并校验 SHA-256；CDN 不可用、模板不兼容或校验失败时命令停止，修复后重试
- 新项目中的 `.rabetbase.json` **只继承**全局中的少量偏好及 `region` / Domain 路由配置，**不会**把全局的 `apps` / `defaultApp` 复制进新项目
- `src/api/sdk-config.ts` 只生成浏览器可公开的非默认 SDK 路由，不会写入 `cookie`、`accessKey` 等认证配置；后续执行 `rabetbase api pull` 会按当前有效配置刷新该文件。`api.ts` / `client.ts` 仅在缺失时由 CLI 写脚手架，已有文件按 [`sdk-client-generation.md`](../guides/sdk-client-generation.md) 更新

| 当前有效路由 | `src/api/sdk-config.ts` |
|--------------|-------------------------|
| 默认 CN，且没有非默认 `runtimeDomain` | `{}`，由 SDK 使用默认 CN 映射 |
| ID，且没有非默认 `runtimeDomain` | `{ region: "id" }`，由 SDK 结合项目 env 选择 ID Runtime |
| 非默认企业 `runtimeDomain` | `{ runtimeDomain: "https://…" }`，不再冗余写 region |

如果显式 `runtimeDomain` 恰好等于当前 region/env 的 SDK 内置地址，CLI 会把它视为默认值：CN 不写路由，ID 仍只写 `region: "id"`。这样生成项目不会固定当前的具体公共 URL，后续仍可跟随 SDK 的内置映射升级。

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase config init](rabetbase-init.md)
