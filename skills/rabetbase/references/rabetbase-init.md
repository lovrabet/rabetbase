# rabetbase config init

首次使用、切换官方节点或切换企业独立部署地址时，重建全局 Lovrabet 连接配置。该命令只处理 region/Domain，不负责认证或项目应用绑定。

```bash
# 交互选择 cn / id
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
  --app-domain https://app.customer.example.com
```

## 交互与默认行为

- 交互执行时选择 `Mainland China (cn)` 或 `Indonesia (id)`，默认选中 `cn`
- 非交互执行时应显式传 `--region cn|id`；未传 region/Domain 时回退 `cn`
- 当前只开放 `cn`、`id`；`global` 不可选择

## 写入行为

- 固定写入全局 `~/.rabetbase.json`
- 不使用也不需要 `--global`，不能写入项目配置
- 默认中国节点 `cn` 不写冗余 `region` 字段
- 官方节点模式会清除六个显式 Domain，以及遗留的 `agentDomain` / `platformDomain` / `skillHubDomain`
- 独立部署模式会清除旧 `region`、显式 Domain 和遗留 Domain，再写入本次提供的 Domain
- 保留 Cookie、AccessKey、format、locale、apps 等无关配置
- 不负责登录，也不绑定当前项目应用

## 独立部署 Domain

`--domain-config` 接受只包含以下字段的 JSON 对象：

```json
{
  "userDomain": "https://user.customer.example.com",
  "apiDomain": "https://api.customer.example.com",
  "runtimeDomain": "https://runtime.customer.example.com",
  "skillDomain": "https://skills.customer.example.com",
  "kbDomain": "https://kb.customer.example.com",
  "appDomain": "https://app.customer.example.com"
}
```

- 每个值必须是 HTTPS origin，不能包含账号、密码、path、query 或 fragment
- 文件至少提供一个受支持 Domain；企业完整独立部署建议显式提供全部六个
- 同名 `--*-domain` flag 覆盖文件中的值
- `--region` 与任意独立部署 Domain 互斥，不能混用
- 未提供的 Domain 会按默认 `cn` 映射回退，因此完整独立部署不要遗漏实际由客户部署的服务入口

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
