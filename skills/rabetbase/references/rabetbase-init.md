# rabetbase config init

首次使用、切换官方节点或切换企业独立部署地址时，重建 Lovrabet 连接配置。默认写入当前项目；显式传 `--global` 时写入全局配置。该命令只处理 region/Domain，不负责认证或项目应用绑定。

```bash
# 交互选择 cn / id / global
rabetbase config init

# 自动化选择官方节点
rabetbase config init --region id

# 企业独立部署
rabetbase config init --domain-config ./lovrabet-domains.json

# 企业独立部署也可直接传 Domain flags
rabetbase config init \
  --user-domain https://user.customer.example.com \
  --api-domain https://api.customer.example.com \
  --runtime-domain https://runtime.customer.example.com \
  --skill-domain https://skills.customer.example.com \
  --kb-domain https://kb.customer.example.com \
  --app-domain https://app.customer.example.com \
  --local-domain https://local.customer.example.com \
  --certificate-domain https://cert.customer.example.com
```

## 交互与默认行为

- 交互执行时选择 `Mainland China (cn)`、`Indonesia (id)` 或 `Global (global)`，默认选中 `cn`
- 非交互执行时应显式传 `--region cn|id|global`；未传 region/Domain 时回退 `cn`
- 当前开放 `cn`、`id`、`global`；例如 `rabetbase config init --region global`
- 已保存的 `region` 无法识别时，依赖连接配置的命令会在请求前停止；先用 `rabetbase doctor` 定位作用域，再执行对应作用域的 `config init` 或 `config delete region` 修复

## 写入行为

- 默认写入当前项目 `.rabetbase.json`；`--global` 写入 `~/.rabetbase.json`
- 默认中国节点 `cn` 不写冗余 `region` 字段
- 官方节点模式会清除所有显式 Domain，以及遗留的 `agentDomain` / `platformDomain` / `skillHubDomain`
- 独立部署模式会清除旧 `region`、显式 Domain 和遗留 Domain，再写入本次提供的 Domain
- 保留 Cookie、AccessKey、format、locale、apps 等无关配置
- 不负责登录，也不绑定当前项目应用

## 独立部署 Domain

`--domain-config` 推荐使用两个 CLI 共用的版本化企业路由清单：

```json
{
  "protocol": "lovrabet-routing/v1",
  "kind": "enterprise",
  "cdn": {
    "libraries": "https://cdnjs.cloudflare.com/ajax/libs",
    "lovrabet": "https://g.lovrabet.com"
  },
  "domains": {
    "userDomain": "https://user.customer.example.com",
    "apiDomain": "https://api.customer.example.com",
    "runtimeDomain": "https://runtime.customer.example.com",
    "skillDomain": "https://skills.customer.example.com",
    "kbDomain": {
      "rabetbase-cli": "https://kb-admin.customer.example.com",
      "lovrabet-cli": "https://kb.customer.example.com"
    },
    "kbServiceDomain": "https://kb-service.customer.example.com",
    "appDomain": "https://app.customer.example.com"
  }
}
```

新版清单要求完整服务拓扑和 `cdn.libraries` / `cdn.lovrabet`，不会把缺失服务回退到官方节点；不能再叠加单独的 `--*-domain` flag。`cdn.libraries` 是第三方库的完整基地址，可包含 `/code/lib` 或 `/ajax/libs`；`cdn.lovrabet` 是 Lovrabet 自有资源 origin。Domain 可写成所有消费者共用的 HTTPS 字符串；确有差异时写成带 `default` 或消费者键的对象。对象优先使用当前消费者键，其次使用 `default`，两者都没有时命令报错。旧扁平 JSON 继续兼容：

```json
{
  "userDomain": "https://user.customer.example.com",
  "apiDomain": "https://api.customer.example.com",
  "runtimeDomain": "https://runtime.customer.example.com",
  "skillDomain": "https://skills.customer.example.com",
  "kbDomain": "https://kb.customer.example.com",
  "kbServiceDomain": "https://kb-service.customer.example.com",
  "appDomain": "https://app.customer.example.com",
  "localDomain": "https://local.customer.example.com",
  "certificateDomain": "https://cert.customer.example.com"
}
```

- 每个值必须是 HTTPS origin，不能包含账号、密码、path、query 或 fragment
- 旧扁平文件至少提供一个受支持 Domain，并维持既有回退语义
- 每个官方节点直接配置自己的 `cdn.libraries` 与 `cdn.lovrabet`；中国大陆当前使用 AliCDN，印尼当前使用 Cloudflare cdnjs，企业独立部署使用清单中的显式值
- 未提供 `localDomain` / `certificateDomain` 时，本地回调使用 `http://localhost`；自定义 HTTPS 本地回调时两者必须同时提供
- 同名 `--*-domain` flag 仅覆盖旧扁平文件中的值
- `--region` 与任意独立部署 Domain 互斥，不能混用
- 旧扁平配置的其他未提供 Domain 保持既有默认节点回退；知识搜索在独立部署缺少 `kbServiceDomain` 时失败，不回退官方节点

## 结果语义

- 官方节点返回 `data.mode="official"` 与选中的 `data.region`
- 独立部署返回 `data.mode="independent"` 与实际写入的 `data.domains`
- 重复执行用于切换连接配置，不会清除认证、输出偏好或应用绑定

首次使用推荐顺序：

```bash
rabetbase config init
rabetbase auth login
rabetbase workspace init --appcode <code>
```

旧的项目初始化能力已统一到 `rabetbase workspace init`。
