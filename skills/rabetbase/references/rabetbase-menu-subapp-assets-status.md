# menu subapp-assets-status

只读核查指定菜单宿主中一个前端子应用的共享资源、关联菜单及回退资源。

查询需要目标宿主应用的应用开发者或应用管理员权限；更新共享资源需要该应用的应用管理员权限。系统管理员、所属租户管理员沿用平台现有权限规则。“宿主”只表示目标应用，不是独立权限角色；其他业务应用的权限不能替代目标宿主应用的权限。

```bash
rabetbase menu subapp-assets-status --app-name store-app --format json
```

`--app-name` 必填，去除首尾空白后精确匹配宿主 `extend.assets[].appName` 和菜单 `extend.appName`。CLI 不从包名、目录名或应用 profile 自动推导该值；标识来源与冲突处理见[宿主前端集成指南](../guides/host-frontend-integration.md)。菜单宿主沿用 CLI 的应用选择：可用 `--appcode` / `--app` 指定，否则使用工作区默认应用或唯一应用；多应用未选定目标时提示选择。`defaultApp` 是默认应用上下文，在本命令中作为菜单宿主。前端子应用访问哪些业务 appCode 是独立关系，不根据业务应用列表遍历或批量操作。默认应用与目标宿主不同时，用参数覆盖。

输出 `data.appCode` 是本次查询的菜单宿主，`data.appName` 是输入标识去除首尾空白后的回显，即使未匹配到配置也会返回。`sharedConfigured` 表示同名共享配置条目是否存在，`boundMenus` 表示关联菜单数量；条目存在不代表资源非空。`sharedResources` 返回该子应用共享资源；`menus[].menuFallbackResources` 返回每个关联菜单自己的资源。菜单回退资源格式异常时，该值及 `fallbackDiffers` 为 `null`，`menuFallbackError` 返回原因，不阻断有效共享资源的核查；无共享资源可用时，该菜单的 `preferredResourcesIfSupported` 也为 `null`，应报告无法判断，不能当成版本一致或空资源。

检测资源版本时，先读取子应用共享资源；共享资源非空时优先使用，否则逐菜单使用回退资源。按 `menus[].preferredSourceIfSupported` 和 `preferredResourcesIfSupported` 与目标资源 URL 比较。`staleMenuFallbacks` 只表示菜单回退配置与共享资源有差异，不代表当前优先资源版本不一致。没有关联菜单时仍可核查 `sharedResources`，但不能据此声称页面已挂载。

比较本地发布版本时，先按[资源版本核验](../guides/host-frontend-integration.md#子应用资源版本核验)取得可信发布记录或构建上传结果中的完整资源 URL 列表；不能仅凭 `package.json.version` 判断发布一致。缺少资源对应关系时报告无法判断，并展示已查到的配置。URL 列表相同只表示配置一致。

`hostCapabilityVerified=false` 表示未验证目标宿主的加载能力；优先级字段描述支持共享资源的宿主。不同宿主版本可能只支持菜单资源，发布前需核实目标宿主的加载行为。本命令不校验 CDN 文件内容，不更新配置；URL 一致不能代替发布验证。
