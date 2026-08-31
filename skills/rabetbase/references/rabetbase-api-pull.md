# api pull

拉取当前 App 下所有数据集（Dataset）的元信息，刷新 `src/api/sdk-config.ts`，并在 `api.ts` / `client.ts` **缺失时**写入 Cookie-first 脚手架。项目公开 Domain 路由由 [`project domain-routing-sync`](rabetbase-project-domain-routing-sync.md) 独立维护。

已有 TypeScript 默认**不会被覆盖**。Agent 必须用返回的 `data.models` 按 [`sdk-client-generation.md`](../guides/sdk-client-generation.md) 更新 `api.ts` / `client.ts`。人类明确要放弃本地定制并重新生成完整脚手架时，可使用 `--force --yes`。

## 命令

```bash
# 拉取事实并刷新 sdk-config.ts（默认 ./src/api/）
rabetbase api pull --format compress

# 指定输出目录
rabetbase api pull --output ./src/generated/api/ --format compress

# 放弃本地定制，按最新 Dataset 事实整体重建 TypeScript
rabetbase api pull --force --yes --format compress

# 多应用：默认只解析「项目级」apps；与全局合并配置一起拉取时加 --global
rabetbase api pull --global --format compress

# 多应用模式：指定某个应用
rabetbase api pull --app order --format compress
rabetbase api pull --appcode app-order-001 --format compress
```

## 参数

| 参数 | 说明 |
|------|------|
| `--output <dir>` | 输出目录，默认 `./src/api/` |
| `--global` | 多应用时显式从「全局+项目」双层解析 `apps`；默认仅项目级 `apps` |
| `--force` | 整体替换已有 `api.ts` / `client.ts`；必须同时传 `--yes`，本地定制会丢失 |
| `--app <name>` | 多应用模式下，指定应用名称 |
| `--appcode <code>` | 直接指定 appcode，跳过配置查找 |
| `--format compress\|json` | 结构化信封。Agent 应读 `data.models`，不要刮 stderr |

## 多应用过滤

多应用模式下：
- **不加 `--app` / `--appcode`**：遍历已配置应用（默认仅 **项目** `.rabetbase.json` 中的 `apps`；若需包含全局里合并进来的应用，加 **`--global`**）
- **加 `--app <name>`**：仅拉取指定应用
- **加 `--appcode <code>`**：反查到对应 app profile，使用其 cookie/env/apiDir

## 配置作用域

- 默认只从项目文件解析 `apps`，同时可从全局白名单继承 cookie/accessKey 等标量配置。
- `--global` 显式从全局和项目双层读取 apps；项目同名项覆盖全局同名项。
- `inherit` 不是受支持的配置项。

## CLI 实际写入

| 文件 | 行为 |
|------|------|
| `sdk-config.ts` | **每次刷新**。只含必要的最终 `runtimeDomain`，不含凭证 |
| `<prefix>-api.ts` | 文件缺失时写入脚手架（含当前 `models`）；已存在时默认跳过，`--force` 时整体替换 |
| `<prefix>-client.ts` | 文件缺失时写入 Cookie-first `createClient`；已存在时默认跳过，`--force` 时整体替换 |

默认 `prefix` 为空（单应用），多应用非 default 应用使用应用名作为 prefix。

`project create` 在拷贝空 demo 脚手架后会覆盖写入一次 TypeScript，以便带上真实 models。首次拉取失败时，恢复提示会要求执行 `rabetbase api pull --force --yes`，避免占位脚手架被继续保留。日常 `api pull` 遇到已有 `api.ts` / `client.ts` 时跳过，只刷新 `sdk-config.ts` 并返回 Dataset 事实。

## 输出（单应用 `data`）

| 字段 | 说明 |
|------|------|
| `appCode` | 当前应用 |
| `configName` | `"default"` 或具名应用名（不含 TS 引号） |
| `isDefaultConfig` | 默认应用为 `true` |
| `models[]` | `datasetCode` / `tableName` / `name` / `alias` |
| `sdkRouting` | `{ runtimeDomain?: string }` |
| `files.api` / `files.client` / `files.sdkConfig` | `{ path, action }`：`created` / `overwritten` / `preserved` / `refreshed` |
| `needsAgentMerge` | 任一 TypeScript 被 `preserved` 时为 `true`；这是本地检查提示，不表示文件一定存在差异 |
| `apiFilePath` / `clientFilePath` / `sdkConfigPath` | 与 `files.*.path` 相同 |
| `modelCount` / `datasetCount` | 模型数量 |

项目使用 `apps` 清单解析时，`data` 为 `{ apps, succeeded, failed }`，每个 `data.apps[]` 元素都使用上表结构。因此 Dataset 事实从直接单应用结果的 `data.models` 读取，或从项目应用清单结果的 `data.apps[].models` 读取；以实际信封为准。

## 生成标识符

脚手架路径遵循 [`best-practices.md`](../guides/best-practices.md) 的动态标识符原则。数字开头的 App Namespace 使用 `APP` 领域前缀，运行时 AppCode 保持不变。

## 前置条件

- 已完成 `rabetbase auth` 登录
- 已配置 appcode（单应用或多应用）

## 示例

```bash
# 单应用：刷新 sdk-config.ts；缺失则写 api.ts/client.ts
rabetbase api pull --format compress

# 多应用 order
rabetbase api pull --app order --format compress

# 独立输出目录
rabetbase api pull --output ./src/generated/api/ --format compress

# 仅在明确放弃本地 TypeScript 定制时整体重建
rabetbase api pull --force --yes --format compress
```

更新已有 TypeScript：见 [`sdk-client-generation.md`](../guides/sdk-client-generation.md)。
