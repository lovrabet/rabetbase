# doctor

诊断 CLI 配置问题，打印当前生效的配置、API 域名和认证状态。

遇到配置错误、`auth_required`、`api_error` 等问题时，先运行此命令定位根因。

## 命令

```bash
rabetbase doctor
```

无需参数，直接执行。

## 输出内容

| 段落 | 说明 |
|------|------|
| CLI Version | 当前 CLI 版本（含 build number） |
| Config Files | 全局和项目级配置文件路径 |
| Merged Config | 合并后的所有配置项（appCode / env / cookie / apiDir 等） |
| Apps | 多应用模式下的应用列表及各自配置 |
| API Endpoints | 最终生效的三个域名（apiDomain / userDomain / runtimeDomain） |
| Auth | 登录状态及 cookie 有效性 |

## 典型场景

```bash
# 独立部署后，验证域名是否正确
rabetbase doctor
# 查看 API Endpoints 是否为自定义域名

# 认证失效时，查看 Auth 状态
rabetbase doctor
# 如果显示 Expired → 重新 run rabetbase auth

# 多应用配置混乱时，查看 Merged Config
rabetbase doctor
# 确认 --app 指定的配置是否正确生效
```

## 独立部署场景

如果 Lovrabet 部署在自定义域名下，先通过 `rabetbase config set` 配置域名，再用 `doctor` 验证：

```bash
rabetbase config set apiDomain https://custom-api.example.com --global
rabetbase doctor
# API Endpoints 应显示自定义域名
```
