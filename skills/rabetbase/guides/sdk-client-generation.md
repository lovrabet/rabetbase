# SDK 客户端代码生成（稳定入口与模型注册表）

> 目标：根据 `rabetbase api pull` 返回的数据集事实，维护浏览器子应用的稳定 SDK 入口，并让同一套业务 API 的不同部署使用各自正确的 `datasetCode`。
>
> 单应用或显式配置同一 `apiGroup` 的 profile 使用注册表模式：`project create` / `project upgrade` 维护稳定源码结构，`api pull` 只刷新 `models.generated.ts` 中的模型事实。`apiGroup` 只负责归并，不进入文件名或运行时对象。

## 重要运行边界：一次运行只选择一个 profile

注册表模式解决的是：**同一份业务代码分别部署到不同节点，并由当前 Runtime Domain 选择唯一 profile**。它不提供跨节点数据访问，也不把多个 profile 合并成一个可同时查询的浏览器 client。

- 业务代码始终通过稳定 alias 调用 `lovrabetClient.models.<alias>`；不要读取注册表后自行挑选其他 profile 的 `datasetCode`，也不要把该代码硬编码进业务模块。
- 当前 profile 只物化自己的模型。仅其他 profile 存在的 alias 不会注册到当前 client；误用时应保留 SDK 的 `MODEL_NOT_FOUND`，不得回退默认 profile 或借用其他节点的代码。
- 节点独有功能可先检查 `LOVRABET_PROFILE_NAME` 再决定是否调用；共享功能不需要分支，同一 alias 会自动绑定当前 profile 的代码。
- 浏览器 Cookie 与 Runtime Domain 都属于当前节点。不要为了“一次查询两端”在浏览器中创建两个 profile client、跨 Domain 转发认证信息或绕过 Domain 选择器。
- 真正需要跨节点读取、比对或聚合时，使用具备明确认证和授权边界的可信服务端或 Backend Function 编排；CLI 侧排查则分别用 `--app <profile>` 查询，不把两个 profile 当成一个运行实例。

## 何时执行

出现以下任一情况时，先读本指南再改 `src/api/`：

- 刚执行 `rabetbase api pull` 或 `rabetbase project create`（带 appCode）
- 平台增删了 Dataset，本地 `models` 过期
- `api.ts` / `client.ts` 缺失、注册表为空、或生成 profile 的 AppCode 仍是 `NOT-SET`
- 用户要求修复 SDK 客户端初始化

不要凭空编造 `datasetCode`。不要把本指南里的浏览器 Cookie 客户端写成服务端 `accessKey` 客户端。

## 工作流

```
rabetbase project upgrade --dry-run
  → 审阅 apiArchitecture.groups[].files[]
rabetbase project upgrade --yes
  → 标准旧脚手架自动备份迁移；业务定制生成候选入口
rabetbase api pull --format compress
  → 单应用读 data；项目应用清单逐项读 data.apps[]
  → 读取每项的 models / modelProfile / files / needsProjectUpgrade
  → 注册表模式检查 models.generated.ts / model-runtime.ts / api.ts / client.ts
  → 验证最终 Runtime Domain 能选中目标 profile
```

1. 已有项目先执行 `rabetbase project upgrade --dry-run`。标准旧脚手架可直接正式升级；`manual-merge` 表示原入口含业务定制，正式升级只会写候选文件，不会覆盖原文件。
2. 执行 `rabetbase project upgrade --yes` 后审阅 `apiArchitecture.manualMergeRequired`。候选文件在 `.rabetbase/project-upgrade/api-model-registry-v2/candidates/`；完成合并前不要删除原入口。
3. 执行 `rabetbase api pull --format compress`（需要人类可读缩进时用 `--format json`），刷新远端 Dataset 事实。不要自动添加 `--force`；它在注册表模式只用于明确接受空结果。
4. 直接单应用结果使用 `data.models`；项目应用清单结果逐项使用 `data.apps[].models`。不要手写或猜测 `datasetCode`。
5. `data.modelProfile` 存在时检查 `data.files.modelRegistry`：同一平台 `alias` 只有一条模型，`tableName` 必须在各 profile 间一致。所有当前 profile 都存在且代码相同时使用 `datasetCode`；代码不同或仅部分 profile 存在时使用 `datasetCodes[modelProfile]`，缺失 profile 不写空字符串。
6. `model-runtime.ts` 由项目创建/升级维护，`models.generated.ts` 由 pull 刷新；都不要写入凭据或应用业务函数。若 `data.needsProjectUpgrade === true`，回到第 1 步，不要手工覆盖入口。
7. 一个 AppCode 对应多个 profile 时，所有拉取与别名写操作都用 `--app <profile>`；不要使用有歧义的 `--appcode`。AppCode 相同不会自动归并，只有显式相同的 `apiGroup` 才表示共享业务 API。

## 权威结构

浏览器子应用的权威形态与 `templates/projects/sub-app-react-demo/src/api/` 一致。脚手架模板是 `templates/generate-api/*.tpl`，生成结果应对齐 demo，而不是另造一套。

### models.generated.ts（CLI 生成事实）

```typescript
/* rabetbase-model-registry:v2 */
export const LOVRABET_MODEL_REGISTRY = {
  defaultProfile: "oa-cn",
  profiles: {
    "oa-cn": {
      appCode: "app-xxxxxxxx",
      runtimeDomain: "https://runtime.lovrabet.com",
    },
    "oa-id": {
      appCode: "app-xxxxxxxx",
      runtimeDomain: "https://runtime.lovrabet.id",
    },
  },
  models: [
    {
      alias: "paymentApplication",
      tableName: "payment_application",
      name: "Payment Application",
      datasetCode: "shared-dataset-code",
    },
    {
      alias: "orders",
      tableName: "orders",
      name: "Orders",
      datasetCodes: {
        "oa-cn": "cn-dataset-code",
        "oa-id": "id-dataset-code",
      },
    },
  ],
} as const;
```

- 以平台稳定 `alias` 作为跨 profile 的模型归并键，`tableName` 作为一致性校验；同 alias 指向不同表或同表出现不同 alias 时停止写入。
- 默认 profile 提供公共 `alias` / `name`；其他 profile 的同表展示名不同不会制造第二份模型。
- 所有当前 profile 都存在且 Dataset 代码相同时只写单数 `datasetCode`。
- 某 profile 独有的表只包含自己的 `datasetCodes` key；运行时不会把它提供给其他 profile。
- 文件版本由 CLI 专用注释标记识别，不进入应用运行时对象。
- 此文件由 CLI 原子刷新，不手工编辑。

### model-runtime.ts（通用选择器）

该文件根据平台注入的 Runtime Domain 选择唯一 profile；本地构建读取项目根目录的 `rabetbase.domain-routing.json`。零匹配或多匹配都直接报错，不回退 `defaultProfile`。它只将所选 profile 物化为 SDK 的 `{ appCode, models }`，不会创建多节点 client，不放业务函数，也不硬编码具体 Dataset。

### api.ts

```typescript
import { registerModels, CONFIG_NAMES } from "@lovrabet/sdk";
import projectRouting from "../../rabetbase.domain-routing.json";
import { LOVRABET_MODEL_REGISTRY } from "./models.generated";
import { resolveActiveModelConfig } from "./model-runtime";

const activeModelConfig = resolveActiveModelConfig(
  LOVRABET_MODEL_REGISTRY,
  projectRouting.domains.runtime,
);

export const LOVRABET_PROFILE_NAME = activeModelConfig.profileName;
export const LOVRABET_APP_CODE = activeModelConfig.appCode;
export const LOVRABET_RUNTIME_DOMAIN = activeModelConfig.runtimeDomain;
export const LOVRABET_MODELS_CONFIG = activeModelConfig.modelsConfig;
registerModels(LOVRABET_MODELS_CONFIG, CONFIG_NAMES.DEFAULT);
```

`api.ts` 只负责连接注册表、路由事实和 SDK 注册。业务层继续从原路径 import，无需感知 profile 数量。

`registerModels` 是 `@lovrabet/sdk` 的独立命名导出，**不是** `client.registerModels()`。

历史未分组项目仍兼容读取具名前缀文件及项目内相对转导出。例如 `api.ts` 可写 `export { APP_MODELS_CONFIG as LOVRABET_MODELS_CONFIG } from "./app-api"`。CLI 会静态读取本地 `export … from` 目标，不执行 TypeScript；HOOK 目录和 `--code` alias 会使用汇总后的模型别名。每个 Dataset 必须只有一个 alias，且一个 alias 不能对应多个 Dataset；冲突时 CLI 会停止而不是猜测绑定目标。旧式前缀路径的默认应用使用 `CONFIG_NAMES.DEFAULT`，具名应用使用 `data.configName`；数字开头的文件前缀继续使用 `APP_` 变量前缀。

### client.ts（Cookie-first）

```typescript
import { createClient, CONFIG_NAMES } from "@lovrabet/sdk";
import { LOVRABET_RUNTIME_DOMAIN } from "./api";
import { LOVRABET_SDK_CONFIG } from "./sdk-config";

export const lovrabetClient = createClient({
  apiConfigName: CONFIG_NAMES.DEFAULT,
  ...LOVRABET_SDK_CONFIG,
  runtimeDomain: LOVRABET_RUNTIME_DOMAIN,
});
```

- 浏览器子应用默认登录态 Cookie：省略 `authMode`，**不要**在此文件写入 `accessKey` / `token`。
- `runtimeDomain` 放在 spread 之后，确保运行时选中的 profile 是最终路由事实。

### sdk-config.ts（路由，无凭证）

该文件仍由 CLI 刷新并保持浏览器安全。注册表模式的最终 Runtime Domain 来自 `api.ts`，项目的完整公开 Domain 与资源策略由 `rabetbase project domain-routing-sync` 写入 `rabetbase.domain-routing.json`。

## apiGroup、目录隔离与旧式兼容

CLI 仍能读取已有 `[name-]api.ts` 中的 `models`，页面与 BFF 的 `--alias` 不会立即失效。迁移时：

- 先执行 `project upgrade --dry-run`，审阅标准生成文件与业务定制文件的分类。
- 有本地定制时，将业务 helper 移到独立业务模块；不要放进 `models.generated.ts` 或 `model-runtime.ts`。
- 正式执行 `project upgrade --yes`；标准旧入口先备份再替换，定制入口只生成候选文件。
- 执行 `api pull --format compress` 刷新远端事实；业务 import 继续指向 `src/api/api.ts` / `client.ts`。
- 旧的 `[name-]api.ts` / `[name-]client.ts` 可在确认无引用后再删除；CLI 不自动删除用户文件。

`apiGroup` 是显式 join key：同组 profile 的 AppCode 可以相同或不同，但必须使用同一规范化 `apiDir`；同一 `apiDir` 也只能承载一个显式组。`./src/api`、`src/api`、`src/api/` 会视为同一目录。第二套业务使用独立目录，例如 `src/api/finance`，目录内仍是无前缀的 `api.ts` / `client.ts`。同一目录下多个未配置 `apiGroup` 的历史 profile 继续使用 `[name-]api.ts` / `[name-]client.ts`，不会被静默归并。

## 认证（不要写进浏览器 client.ts）

浏览器子应用走 Cookie，本指南的默认 `client.ts` 不声明 `authMode`。

仅当用户明确要求**服务端**文件时才使用密钥，且必须显式 `authMode`：

- 仅 `accessKey` → `authMode: "client-ak"`，且只放服务端
- `accessKey`（+ 可选 `secretKey`）签名，或预计算 `token` + 配对 `timestamp` → `authMode: "openapi"`
- 预计算 token 必须同时提供 `timestamp`，否则首次请求报 `timestamp is required`
- OpenAPI 凭据通过 `X-Token` / `X-Time-Stamp` 传递，不是 `Authorization: Bearer`

服务端写法详见 [`typescript-sdk.md`](typescript-sdk.md)。

## 禁止

- `client.registerModels(...)`
- 把 `filter()` 返回值当数组：必须 `result.tableData`
- `result.total`：总数是 `result.paging.totalCount`
- `client.models.dataset_[code]`：动态访问用 `client.models[\`dataset_${code}\`]`
- 传了 `accessKey` / `token` 却省略 `authMode`
- 把 Cookie / AccessKey 写进 `sdk-config.ts` 或浏览器 `client.ts`
- 在已有正确 `createClient` 形态上改成另一种参数风格

## pull 输出字段

`rabetbase api pull --format compress` 的 `data`（单应用）：

| 字段 | 含义 |
|------|------|
| `appCode` | 当前应用 |
| `modelProfile` | 本次刷新的 profile；旧式输出可省略 |
| `models[]` | `datasetCode` / `tableName` / `name` / `alias` |
| `sdkRouting` | `{ runtimeDomain?: string }` |
| `files.api` / `files.client` / `files.sdkConfig` | 稳定入口与 SDK 路由的路径及动作 |
| `files.modelRegistry` / `files.modelRuntime` | 注册表模式的模型事实与通用选择器路径及动作 |
| `needsAgentMerge` | 兼容字段；旧式或自定义入口尚未接入注册表时为 `true` |
| `needsProjectUpgrade` | 注册表事实已刷新，但稳定源码入口缺失或仍为旧结构时为 `true` |
| `apiFilePath` / `clientFilePath` / `sdkConfigPath` | 稳定入口与 SDK 路由路径 |
| `modelRegistryPath` / `modelRuntimePath` | 注册表模式下的生成文件路径 |
| `modelCount` / `datasetCount` | 模型数量 |

多应用时 `data.apps[]` 为上述对象的数组。
