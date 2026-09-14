# api pull

拉取当前 App 下所有数据集（Dataset）的元信息并刷新 SDK 模型事实。单应用与显式配置同一 `apiGroup` 的多部署项目使用稳定的 `api.ts` / `client.ts`，CLI 将模型写入 `models.generated.ts`，由 `model-runtime.ts` 按最终 Runtime Domain 选择 profile。项目公开 Domain 路由由 [`project domain-routing-sync`](rabetbase-project-domain-routing-sync.md) 独立维护。

`apiGroup` 只作为归并键，不进入文件名。第二套业务使用独立 `apiDir`，目录内仍是无前缀入口。同一目录中多个未配置 `apiGroup` 的历史 profile 继续使用 `[name-]api.ts` / `[name-]client.ts`。`api pull` 只刷新模型事实；已有应用的稳定入口迁移由 [`project upgrade`](rabetbase-project-upgrade.md) 负责。Agent 按 [`sdk-client-generation.md`](../guides/sdk-client-generation.md) 检查输出。

## 命令

```bash
# 拉取事实并刷新 sdk-config.ts（默认 ./src/api/）
rabetbase api pull --format compress

# 指定输出目录
rabetbase api pull --output ./src/generated/api/ --format compress

# 确认远端清单确实为空，允许清空当前 profile
rabetbase api pull --app oa-id --force --yes --format compress

# 多应用：默认只解析「项目级」apps；与全局合并配置一起拉取时加 --global
rabetbase api pull --global --format compress

# 多应用模式：指定某个应用
rabetbase api pull --app order --format compress
rabetbase api pull --appcode app-order-001 --format compress

# 一个 AppCode 对应多个 profile 时必须按名称选择
rabetbase api pull --app oa-id --format compress
```

## 参数

| 参数 | 说明 |
|------|------|
| `--output <dir>` | 输出目录，默认 `./src/api/` |
| `--global` | 多应用时显式从「全局+项目」双层解析 `apps`；默认仅项目级 `apps` |
| `--force` | 注册表模式允许空结果替换当前 profile；历史前缀模式允许替换对应生成文件；必须同时传 `--yes` |
| `--app <name>` | 多应用模式下，指定应用名称 |
| `--appcode <code>` | 指定唯一 AppCode；匹配多个 profile 时必须改用 `--app` |
| `--format compress\|json` | 结构化信封。Agent 应读 `data.models`，不要刮 stderr |

## 多应用过滤

多应用模式下：
- **不加 `--app` / `--appcode`**：遍历已配置应用（默认仅 **项目** `.rabetbase.json` 中的 `apps`；若需包含全局里合并进来的应用，加 **`--global`**）
- **加 `--app <name>`**：仅拉取指定应用
- **加 `--appcode <code>`**：唯一匹配时使用该 profile 的 `region`、cookie 和 apiDir；同一 AppCode 匹配多个 profile 时拒绝猜测，改用 `--app <name>`
- 每个应用请求前都会切换到其有效 `region`，不会沿用上一个应用的官方服务地址

## 配置作用域

- 默认只从项目文件解析 `apps`，同时可从全局白名单继承 cookie/accessKey 等标量配置。
- `--global` 显式从全局和项目双层读取 apps；项目同名项覆盖全局同名项。
- `inherit` 不是受支持的配置项。

## CLI 实际写入

| 文件 | 行为 |
|------|------|
| `models.generated.ts` | 注册表模式下**每次刷新当前 profile**并保留其他 profile；公共代码写 `datasetCode`，差异或部分可用代码写 `datasetCodes[profile]`，不含凭证 |
| `model-runtime.ts` | 由 `project create` / `project upgrade` 维护；`api pull` 不改写 |
| `api.ts` / `client.ts` | 由 `project create` / `project upgrade` 维护稳定结构；注册表模式的 `api pull` 不创建、不覆盖 |
| `sdk-config.ts` / `<prefix>-sdk-config.ts` | 每次刷新，只含必要的最终 Runtime Domain，不含凭证；旧式多应用继续使用前缀文件 |
| `<prefix>-api.ts` / `<prefix>-client.ts` | 同目录未显式分组的历史多应用结构；默认保留，仍可被命令侧别名解析读取 |

注册表模式不使用国家/地区文件名前缀。业务代码始终 import `api.ts` / `client.ts`；页面加载时优先读取平台注入的 Runtime Domain，本地构建使用 `rabetbase.domain-routing.json`，据此物化该 profile 的 SDK `models` 数组。

同一 `apiGroup` + 同一规范化 `apiDir` 才进入同一注册表，组内 AppCode 可以相同或不同。相同 `apiGroup` 分散到不同目录、同一目录出现不同组、或显式分组与未分组混用都会在请求前报错。多个未分组 profile 共用目录时继续使用旧式具名入口；要共享模型必须显式配置同一个 `apiGroup`。

`project create` 直接生成注册表结构。旧项目先执行 `project upgrade --dry-run`，确认后执行 `project upgrade --yes`；标准旧脚手架会备份后迁移，业务定制入口会保留并生成候选文件。随后运行普通 pull 刷新事实。只拉取一个 profile 时会保留同组其他 profile 的模型代码。注册表已有非空 profile 而远端返回空清单时，普通 pull 会拒绝清空；仅在确认远端确实为空时使用 `--force --yes`。

## 输出（单应用 `data`）

| 字段 | 说明 |
|------|------|
| `appCode` | 当前应用 |
| `configName` | `"default"` 或具名应用名（不含 TS 引号） |
| `isDefaultConfig` | 默认应用为 `true` |
| `models[]` | `datasetCode` / `tableName` / `name` / `alias` |
| `sdkRouting` | `{ runtimeDomain?: string }` |
| `files.api` / `files.client` / `files.sdkConfig` | `{ path, action }`：`created` / `overwritten` / `preserved` / `refreshed` / `missing` |
| `files.modelRegistry` / `files.modelRuntime` | 注册表模式下的生成文件及动作 |
| `needsAgentMerge` | 旧式 TypeScript 被保留且尚未使用注册表时为 `true` |
| `needsProjectUpgrade` | 注册表事实已刷新，但稳定源码入口缺失或仍为旧结构时为 `true` |
| `modelProfile` | 已刷新的 profile 名称；旧式输出可省略 |
| `apiFilePath` / `clientFilePath` / `sdkConfigPath` | 稳定入口与 SDK 路由路径 |
| `modelRegistryPath` / `modelRuntimePath` | 注册表模式下的模型事实与选择器路径 |
| `modelCount` / `datasetCount` | 模型数量 |

项目使用 `apps` 清单解析时，`data` 为 `{ apps, succeeded, failed }`，每个 `data.apps[]` 元素都使用上表结构。因此 Dataset 事实从直接单应用结果的 `data.models` 读取，或从项目应用清单结果的 `data.apps[].models` 读取；以实际信封为准。

## 生成标识符

脚手架路径遵循 [`best-practices.md`](../guides/best-practices.md) 的动态标识符原则。数字开头的 App Namespace 使用 `APP` 领域前缀，运行时 AppCode 保持不变。

## 前置条件

- 已完成 `rabetbase auth login`；不同官方节点的登录态不通用时，先对目标 profile 执行 `rabetbase auth login --app <name>`
- 已配置 appcode（单应用或多应用）

## 示例

```bash
# 单应用：刷新模型事实
rabetbase api pull --format compress

# 多应用 order
rabetbase api pull --app order --format compress

# 独立输出目录
rabetbase api pull --output ./src/generated/api/ --format compress

# 仅在确认目标 profile 的远端 Dataset 清单确实为空时
rabetbase api pull --app oa-id --force --yes --format compress
```

更新已有 TypeScript：见 [`sdk-client-generation.md`](../guides/sdk-client-generation.md)。
