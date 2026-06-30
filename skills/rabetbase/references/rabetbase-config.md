# `.rabetbase.json` 配置参考

项目级配置文件，放在项目根目录。CLI 启动时自动读取，全局配置（`~/.rabetbase.json`）作为 fallback。

这份文件描述的是**本地配置模型**：默认应用、应用名到 `appcode`/`env`/`apiDir` 的映射，以及输出格式、风险等级、认证信息等本地偏好。

它**不是**平台应用目录。若要查看当前登录账号在平台上能访问哪些应用，应使用 `rabetbase app list --remote`，而不是直接查看 `.rabetbase.json`。

兼容旧名：`.lovrabet.json`、`.lovrabetrc`（按优先级 `.rabetbase.json` > `.lovrabet.json` > `.lovrabetrc`，首个存在的生效）。

## 初始化

```bash
rabetbase project init
```

交互式创建 `.rabetbase.json`，新写入优先使用 canonical 的 `apps + defaultApp` 结构；旧的顶层 `appcode` 仍兼容读取。

## canonical 主模型

当前推荐、也是所有新写入默认采用的结构，是 **`apps + defaultApp`**：

```json
{
  "defaultApp": "main",
  "apps": {
    "main": {
      "appcode": "app-xxxxxxxx",
      "env": "daily"
    }
  }
}
```

CLI 对旧版顶层 `appcode` 仍兼容读取，但它已经不是推荐主模型。

## 兼容读取的单应用模式

兼容读取的最简配置只需 `appcode` 和 `env`：

```json
{
  "appcode": "app-xxxxxxxx",
  "env": "daily"
}
```

### 完整字段

```json
{
  "appcode": "app-xxx",
  "env": "production",
  "locale": "en-US",
  "cookie": "session-cookie-value",
  "accessKey": "ak-xxx",
  "format": "json",
  "pageSize": 20,
  "riskLevel": "high-risk-write",
  "apiDir": "./src/api",
  "template_base_url": "https://custom-cdn.example.com/dist",
  "apiDomain": "https://custom-api.example.com",
  "userDomain": "https://custom-user.example.com",
  "runtimeDomain": "https://custom-runtime.example.com"
}
```

## 多应用模式

一个项目对接多个 Lovrabet 应用时，使用 `apps` + `defaultApp`：

```json
{
  "defaultApp": "order",
  "apps": {
    "order": {
      "appcode": "app-yyyyyyyy",
      "env": "daily",
      "riskLevel": "write"
    },
    "product": {
      "appcode": "app-zzzzzzzz",
      "env": "production",
      "apiDir": "./src/api/product"
    }
  }
}
```

每个 app profile 可单独覆盖顶层字段。CLI 运行时根据 `--app <name>` 或 `defaultApp` 选择激活的 profile。

管理命令（声明式 flags，非位置参数）：

```bash
rabetbase app add order --appcode app-order-001 --env daily
rabetbase app add product --appcode app-product-002
rabetbase app use order
rabetbase app list
rabetbase app remove product --yes
```

## 字段说明

### 顶层字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `appcode` | string | — | 顶层单应用兼容字段。新写入优先使用 `apps + defaultApp`。兼容旧名 `app` |
| `env` | string | `"production"` | 环境。可选值：`production`、`daily`（配置文件若仍为旧值 `online`，加载时会规范为 `production`） |
| `locale` | string | `"en-US"` | 语言设置 |
| `cookie` | string | — | 内联 session cookie。设置后优先于 `~/.lovrabet/cookie` 文件 |
| `accessKey` | string | — | Access Key 认证（预留，未来替代 cookie） |
| `format` | string | — | 默认输出格式。可选值：`json`、`pretty`、`compress`。不设则命令默认 `compress` |
| `pageSize` | number | — | 默认分页大小，用于 `sql list` 等分页命令 |
| `riskLevel` | string | `"high-risk-write"` | 允许执行的最高风险等级。可选值：`read`、`write`、`high-risk-write`。兼容旧名 `maxRisk` |
| `inherit` | boolean | 省略 | 配置继承模式。**省略**（默认）：项目主导，仅从全局白名单继承 cookie/accessKey/locale/format/riskLevel/pageSize，不继承 apps/defaultApp/appcode。**`true`**：全量合并全局+项目（旧行为，需显式开启）。**`false`**：完全隔离，不继承任何全局字段 |
| `apiDir` | string | `"./src/api"` | `api pull` 生成代码的输出目录 |
| `template_base_url` | string | 平台默认 CDN | 模板 CDN 基础 URL，一般无需修改 |
| `defaultApp` | string | — | 多应用模式下的默认应用名称 |
| `apps` | object | — | 多应用配置。key 为应用名，value 为 AppProfile（见下方） |
| `apiDomain` | string | 平台默认 | 自定义 API 域名（独立部署场景）。覆盖后所有 API 请求使用此域名 |
| `userDomain` | string | 平台默认 | 自定义用户域名 |
| `runtimeDomain` | string | 平台默认 | 自定义运行时域名 |

### AppProfile 字段（`apps.*` 内的每个应用）

| 字段 | 类型 | 说明 |
|------|------|------|
| `appcode` | string | **必填**。该应用的 appcode |
| `env` | string | 覆盖顶层 `env` |
| `apiDir` | string | 覆盖顶层 `apiDir` |
| `cookie` | string | 覆盖顶层 `cookie` |
| `accessKey` | string | 覆盖顶层 `accessKey` |
| `format` | string | 覆盖顶层 `format` |
| `pageSize` | number | 覆盖顶层 `pageSize` |
| `riskLevel` | string | 覆盖顶层 `riskLevel` |
| `locale` | string | 覆盖顶层 `locale` |

## 优先级

每个配置项的解析优先级从高到低（`appcode` 不从环境变量自动解析，必须来自 `--appcode` 或配置文件）：

```
CLI flag (--appcode, --env, --format, --app ...)
  ↓
当前项目 app profile / 顶层字段
  ↓
环境变量 (RABETBASE_ENV, RABETBASE_FORMAT ...)
  ↓
全局级 ~/.rabetbase.json app profile / 顶层字段
  ↓
内置默认值
```

业务落点类字段以当前项目为主。`appcode` 不允许从 shell 残留环境变量自动推断；CI 或一次性脚本如需使用环境变量，必须显式传 `--appcode "$RABETBASE_APPCODE"`。对需要 App Code 的业务命令，如果检测到 `RABETBASE_APPCODE` / `LOVRABET_APPCODE` 已设置但未显式传 `--appcode`，且当前解析结果为空或与环境变量不同，CLI 会直接拒绝执行，避免误操作其他应用。`RABETBASE_APP`、`RABETBASE_ENV` 仍只作为无项目配置时的显式选择 / 环境 fallback。

## 环境变量

所有环境变量以 `RABETBASE_` 为前缀，兼容旧前缀 `LOVRABET_`。

| 环境变量 | 对应配置项 | 说明 |
|----------|-----------|------|
| `RABETBASE_APPCODE` | — | 仅作为 shell 变量供脚本显式传给 `--appcode`；业务命令检测到冲突时会拒绝执行 |
| `RABETBASE_ENV` | `env` | 无项目 env 时的环境 fallback |
| `RABETBASE_COOKIE` | `cookie` | Session cookie |
| `RABETBASE_ACCESS_KEY` | `accessKey` | Access Key |
| `RABETBASE_FORMAT` | `format` | 输出格式 |
| `RABETBASE_PAGE_SIZE` | `pageSize` | 分页大小 |
| `RABETBASE_RISK_LEVEL` | `riskLevel` | 最高风险等级 |
| `RABETBASE_VERBOSE` | — | 全局 verbose 开关（`1` 或 `true` 启用），仅环境变量，不支持配置文件 |
| `RABETBASE_APP` | — | 无项目显式选择时的临时应用名 fallback（显式切换优先用 `--app`） |

## 配置文件查找规则

| 作用域 | 查找目录 | 文件名优先级 |
|--------|---------|------------|
| 项目级 | `process.cwd()` | `.rabetbase.json` > `.lovrabet.json` > `.lovrabetrc` |
| 全局级 | `~`（用户 HOME） | 同上 |

合并策略由项目级 `inherit` 字段控制（三态）：

- **省略 `inherit`（默认，项目主导）**：项目级配置自给自足；仅从全局白名单继承 `cookie`、`accessKey`、`locale`、`format`、`riskLevel`、`pageSize`，**不继承** `apps` / `defaultApp` / `appcode` / `env` 等业务字段。项目级显式声明的字段覆盖全局白名单。
- **`inherit: true`（显式全量合并）**：标量字段项目级覆盖全局级；`apps` 深度合并（项目同名覆盖全局同名，其余保留）；`defaultApp` 仅当项目级显式声明时才覆盖全局级。等价于历史默认行为，需要跨项目共享全局 `apps` 时显式开启。
- **`inherit: false`（完全隔离）**：忽略全局配置，仅使用项目级字段；适用于强隔离场景（如多客户）。

> **为什么默认改成项目主导**：避免 `~/.lovrabet.json` 中的 `defaultApp` / `apps` 在用户进入空白目录或新项目时被无声继承（曾导致 menu update 误改其他应用资源）。`api list` / `api pull` / `app list` 仍可通过 `--global` 显式查看合并视图。

## 示例

### 最小化（CI 环境用环境变量）

```json
{}
```

```bash
export RABETBASE_APPCODE=app-xxx
export RABETBASE_ENV=daily
rabetbase dataset list --appcode "$RABETBASE_APPCODE"
```

### 开发环境单应用

```json
{
  "appcode": "app-xxxxxxxx",
  "env": "daily",
  "riskLevel": "high-risk-write"
}
```

### 多应用 + 限制风险

```json
{
  "defaultApp": "order",
  "apps": {
    "order": {
      "appcode": "app-order-001",
      "env": "daily",
      "riskLevel": "write"
    },
    "product": {
      "appcode": "app-product-002",
      "env": "production",
      "riskLevel": "read"
    }
  }
}
```

```bash
rabetbase sql list --app order
rabetbase dataset list --app product
```

## config 命令

### config set

写入配置项：

```bash
# 写入项目级配置（默认；须在能解析到项目 .rabetbase.json 的目录下执行）
rabetbase config set apiDomain https://custom-api.example.com
rabetbase config set env daily

# 写入全局配置（任意目录）
rabetbase config set apiDomain https://custom-api.example.com --global
```

`config set` **默认写入项目**配置文件（当前工作目录能解析到 `.rabetbase.json` 等）；加 **`--global`** 则写入 **`~/.rabetbase.json`**。

若当前目录**没有**项目配置文件且**未**指定 `--global`，CLI **拒绝执行**并提示使用 `--global` 或先 `rabetbase init`，**不会**静默改全局。

### config get

读取配置项（读取合并后的值）：

```bash
rabetbase config get apiDomain
# 输出: https://custom-api.example.com
```

### config list

以 JSON 格式输出当前合并后的完整配置：

```bash
rabetbase config list
```

## 诊断命令

遇到配置问题时，用 `rabetbase doctor` 快速定位问题：

```bash
rabetbase doctor
```

输出包括：CLI 版本、配置文件路径、**各侧配置文件 JSON 语法是否合法**（非法则该侧内容在合并时会被忽略）、合并后的所有配置项（appCode / env / cookie / apiDomain 等）、API 域名、认证状态。

多应用列表与来源说明见 [`rabetbase app list`](rabetbase-app-list.md)（`items` / `meta` / `definedIn`）。

## 独立部署场景

如果 Lovrabet 部署在自定义域名下，通过 `apiDomain` / `userDomain` / `runtimeDomain` 覆盖默认域名：

```bash
# 写入全局配置（所有项目生效）
rabetbase config set apiDomain https://your-api.example.com --global
rabetbase config set userDomain https://your-user.example.com --global
rabetbase config set runtimeDomain https://your-runtime.example.com --global
```

优先级：配置文件中显式设置 > 环境变量 > 默认平台域名。项目级配置可覆盖全局配置。
