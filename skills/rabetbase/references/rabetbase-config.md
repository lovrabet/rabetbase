# `.rabetbase.json` 配置参考

项目级配置文件，放在项目根目录。CLI 启动时自动读取，全局配置（`~/.rabetbase.json`）作为 fallback。

这份文件描述的是**本地配置模型**：默认应用、应用名到 `appcode`/`env`/`apiDir` 的映射，以及输出格式、风险等级、认证信息等本地偏好。

它**不是**平台应用目录。若要查看当前登录账号在平台上能访问哪些应用，应使用 `rabetbase app list --remote`，而不是直接查看 `.rabetbase.json`。

只自动读取 `.rabetbase.json`，不会把 `.lovrabet.json` 或 `.lovrabetrc` 当作 Rabetbase 配置。旧文件需要迁移时，显式执行 `rabetbase project upgrade`。

## 初始化

```bash
rabetbase workspace init --appcode <code>
```

绑定当前目录到某个应用并写入 `.rabetbase.json`，采用 canonical 的 `apps` 结构；单应用自动选中且省略 `defaultApp`，旧的顶层 `appcode` 仍兼容读取。首次安装后的全局引导用 `rabetbase config init`。

## canonical 主模型

当前推荐、也是单应用新写入默认采用的结构，是只有一个 profile 的 **`apps`**：

```json
{
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

下面分别展示字段形态；官方 `region` 模式与企业显式 Domain 模式不要混写。官方印尼节点只需：

```json
{
  "region": "id"
}
```

企业独立部署才显式提供 Domain：

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
  "runtimeDomain": "https://custom-runtime.example.com",
  "skillDomain": "https://custom-skills.example.com",
  "kbDomain": "https://custom-kb.example.com",
  "appDomain": "https://custom-app.example.com",
  "localDomain": "https://custom-local.example.com",
  "certificateDomain": "https://custom-cert.example.com"
}
```

## 多应用模式

一个项目对接多个 Lovrabet 应用时，才使用 `apps` + `defaultApp`：

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

每个 app profile 可单独覆盖普通顶层字段。唯一 app 自动激活；两个及以上 app 必须通过 `--app <name>` 临时选择，或用 `defaultApp` 持久选择，CLI 不按 JSON key 顺序猜测。`riskLevel` 是安全例外：默认值为 `write`，全局、项目与 profile 决定基础上限并取最严格值。权限不足时停止执行并请求授权人员处理；Agent 和自动化脚本不得自行修改 `riskLevel`、配置文件、环境变量或尝试提权。

管理命令（声明式 flags，非位置参数）：

```bash
rabetbase workspace add order --appcode app-order-001 --env daily
rabetbase workspace add product --appcode app-product-002
rabetbase workspace use --app order
rabetbase app list
rabetbase workspace remove product --yes
```

## 字段说明

### 顶层字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `appcode` | string | — | 顶层单应用兼容字段。新写入优先使用 `apps`。兼容旧名 `app` |
| `env` | string | `"production"` | 环境。可选值：`production`、`daily`（配置文件若仍为旧值 `online`，加载时会规范为 `production`） |
| `locale` | string | `"en-US"` | 应用本地化设置（不是 CLI 语言；目前命令尚未消费） |
| `cookie` | string | — | 内联 session cookie。设置后优先于 `~/.lovrabet/cookie` 文件 |
| `accessKey` | string | — | Access Key 认证（预留，未来替代 cookie） |
| `format` | string | — | 默认输出格式。可选值：`json`、`pretty`、`compress`。不设则命令默认 `compress` |
| `pageSize` | number | — | 默认分页大小，用于 `sql list` 等分页命令 |
| `riskLevel` | string | `"write"` | 允许执行的最高风险等级。可选值：`read`、`write`、`high-risk-write` |
| `apiDir` | string | `"./src/api"` | `api pull` 生成代码的输出目录 |
| `region` | string | `"cn"`（省略） | 官方节点快捷配置：当前开放 `cn`、`id`；历史 `global` 配置仅保留读取兼容。显式未知值会阻止业务命令回落默认节点，先用 `rabetbase doctor` 定位后修复 |
| `template_base_url` | string | 平台默认 CDN | 模板 CDN 基础 URL，一般无需修改 |
| `defaultApp` | string | — | 多应用模式下的默认应用名称；单应用省略 |
| `apps` | object | — | 多应用配置。key 为应用名，value 为 AppProfile（见下方） |
| `apiDomain` | string | 节点默认 | 普通研发平台 API 域名 |
| `userDomain` | string | 平台默认 | 自定义用户域名 |
| `runtimeDomain` | string | 平台默认 | 自定义运行时域名 |
| `skillDomain` | string | 节点默认 | SkillHub HTTPS origin 覆盖 |
| `kbDomain` | string | 跟随 `apiDomain` / 节点 API 默认值 | 企业知识库管理与 `kb search` 的 SmartCode Java HTTPS origin；KB Service 下游地址由 Java 配置 |
| `appDomain` | string | 节点默认 | 工作台、页面编辑器和发布页面域名 |
| `localDomain` | string | 独立部署默认 `http://localhost` | 自定义 HTTPS 本地回调 origin；与 `certificateDomain` 同时配置 |
| `certificateDomain` | string | 节点默认；独立部署无 | 本地 HTTPS 证书服务 origin；与 `localDomain` 同时配置 |

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
| `riskLevel` | string | 只能继续收紧顶层 `riskLevel`，不能提权 |
| `locale` | string | 覆盖顶层 `locale`（应用本地化；目前命令尚未消费） |

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
| `RABETBASE_VERBOSE` | — | 全局 verbose 开关（`1` 或 `true` 启用），仅环境变量，不支持配置文件 |
| `RABETBASE_APP` | — | 无项目显式选择时的临时应用名 fallback（显式切换优先用 `--app`） |

## 配置文件查找规则

| 作用域 | 查找目录 | 文件名优先级 |
|--------|---------|------------|
| 项目级 | 从 `process.cwd()` 向父目录查找 | 仅 `.rabetbase.json` |
| 全局级 | `~`（用户 HOME） | 仅 `.rabetbase.json` |

配置合并使用固定模型，`inherit` 不是受支持的配置项：

- 项目显式值覆盖全局显式值。
- 项目存在时，仅从全局白名单继承 `cookie`、`accessKey`、`locale`、`format`、`riskLevel`、`pageSize`、`region` 和显式 Domain（服务、本地回调、证书服务）；不继承 `apps` / `defaultApp` / `appcode` 等项目状态。
- `apps` / `defaultApp` 始终项目隔离，避免研发写操作落到全局应用。
- `region` 是 Domain fallback profile；显式 Domain 按普通 key 继承和覆盖，不会被 `region` 清除或屏蔽。
- `riskLevel` 始终取全局与项目顶层的更严格值，项目不能借配置覆盖提权。
- 旧配置中的 `inherit` 会被忽略，可用 `rabetbase config delete inherit` 清理。

> **为什么固定为项目主导**：避免全局 `.rabetbase.json` 中的 `defaultApp` / `apps` 在用户进入项目后被无声继承（曾导致写操作误选应用）。`api list` / `api pull` 仍可通过 `--global` 显式读取双层 apps。

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
  "riskLevel": "write",
  "defaultApp": "order",
  "apps": {
    "order": {
      "appcode": "app-order-001",
      "env": "daily"
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
rabetbase config set --key apiDomain --value https://custom-api.example.com
rabetbase config set --key env --value daily

# 写入全局配置（任意目录）
rabetbase config set --key apiDomain --value https://custom-api.example.com --global
```

`config set` **默认写入项目**配置文件（当前工作目录能解析到 `.rabetbase.json` 等）；加 **`--global`** 则写入 **`~/.rabetbase.json`**。

若当前目录**没有**项目配置文件且**未**指定 `--global`，CLI **拒绝执行**并提示使用 `--global` 或先 `rabetbase workspace init --appcode <code>`，**不会**静默改全局。

### config get

读取配置项（读取合并后的值）：

```bash
rabetbase config get --key apiDomain
# 输出: https://custom-api.example.com
```

### config delete

删除当前作用域显式配置的 key；命令幂等，key 已不存在时仍成功返回。默认删除项目层，`--global` 删除全局层。`riskLevel` 受保护，不能通过 CLI 删除。

```bash
rabetbase config delete apiDomain
rabetbase config delete apiDomain --global
rabetbase config delete inherit  # 清理已移除的旧字段
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

官方节点使用内置完整映射，默认 `cn` 可省略：

```bash
rabetbase config set --key region --value id --global
```

企业独立部署推荐通过连接配置入口一次性初始化：

```bash
rabetbase config init --domain-config ./lovrabet-domains.json
```

推荐使用 `lovrabet-routing/v1`，完整声明 `userDomain`、`apiDomain`、`runtimeDomain`、`skillDomain`、`kbDomain`、`appDomain`、`cdn.libraries` 与 `cdn.lovrabet`，并可选声明成对出现的 `localDomain`、`certificateDomain`。Domain 默认写成共用 HTTPS 字符串，确有差异时可按消费者覆盖。旧扁平 Domain 文件仍兼容。完整行为与命令行 flags 见 [`rabetbase config init`](rabetbase-init.md)。

也可以逐项覆盖：

```bash
# 写入全局配置（所有项目生效）
rabetbase config set --key apiDomain --value https://your-api.example.com --global
rabetbase config set --key userDomain --value https://your-user.example.com --global
rabetbase config set --key runtimeDomain --value https://your-runtime.example.com --global
rabetbase config set --key skillDomain --value https://your-skills.example.com --global
rabetbase config set --key kbDomain --value https://your-kb.example.com --global
rabetbase config set --key appDomain --value https://your-app.example.com --global
```

官方模式：`region` 内置 Routing Profile，默认 `cn`，配置文件可省略；当前新配置只开放 `cn`、`id`，`id` 需保存对应 region。历史文件中的 `global` 仍可读取，但不能通过 `config init` 或 `config set` 新写入。每个官方节点直接配置最终 `cdn.libraries` 与 `cdn.lovrabet`，因此可按节点使用独立 CDN；中国大陆当前使用 AliCDN，印尼当前使用 Cloudflare cdnjs。企业独立部署模式保存显式 Domain 与 CDN。项目级配置可覆盖全局配置；默认项目合并会继承全局节点配置。
