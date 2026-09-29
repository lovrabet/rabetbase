# menu subapp-assets-update

更新指定菜单宿主中一个已存在的前端子应用共享资源 URL。只替换该条目的 `resources`，保留读取到的加载方式、基础路径、其他条目以及 `extend` 中的其他配置（包括直连数据库设置）。通过 `--resources` 提供完整资源 URL 列表。

正式写入需要目标宿主应用的应用管理员权限。系统管理员、所属租户管理员沿用平台现有权限规则。“宿主”只表示目标应用，不是独立权限角色；其他业务应用的权限不能替代目标宿主应用的权限。仅具有该应用的应用开发者权限时可以查询和预演，但不能更新；预演成功不代表具有写权限。遇到权限不足时说明所需权限，不通过逐菜单更新或切换业务 appCode 绕过。

```bash
rabetbase menu subapp-assets-update --app-name store --resources '["https://cdn.example.com/store/2/app.js","https://cdn.example.com/store/2/app.css"]' --dry-run --format json
rabetbase menu subapp-assets-update --app-name store --resources '["https://cdn.example.com/store/2/app.js","https://cdn.example.com/store/2/app.css"]' --expected-hash <data.body.preview.expectedHash> --format json
rabetbase menu subapp-assets-status --app-name store --format json
```

`--app-name` 必填，去除首尾空白后精确匹配宿主中已有的 `extend.assets[].appName`。标识沿用平台已有绑定，不随包名或版本变化；CLI 不自动从工程推导，来源与冲突处理见[宿主前端集成指南](../guides/host-frontend-integration.md)。菜单宿主沿用 CLI 的应用选择：可用 `--appcode` / `--app` 指定，否则使用工作区默认应用或唯一应用；多应用未选定目标时提示选择。`defaultApp` 是默认应用上下文，在本命令中作为菜单宿主。前端子应用访问哪些业务 appCode 是独立关系，不根据业务应用列表批量更新。默认应用与目标宿主不同时，用参数覆盖。

本命令不创建子应用条目、不改名、不修改菜单绑定。已有菜单绑定但缺少共享条目时，保留原绑定和菜单资源，使用平台已有配置入口建立同名共享条目，再查询核对；不能因为共享配置缺失就自动改成逐菜单更新。

用户要求更新“子应用版本／子应用资源”时使用本命令；只有明确要求修改菜单自身资源时才使用 `menu asset-update`。同名子应用在不同宿主下分别配置，本次只修改选中的宿主。预演与正式写入必须保持同一宿主、子应用和目标 URL 列表；修改其中任何一项后应重新预演。

用户只给版本号或要求“从 A 升到 B”时，先按[资源版本核验](../guides/host-frontend-integration.md#子应用资源版本核验)确认完整资源 URL 和当前起点；不猜测目标 URL。起始版本不符或无法确认时先澄清，当前配置已匹配目标 URL 时不重复写入。预演指纹用于检测配置漂移，不证明当前配置对应用户指定的起始版本。

`--resources` 必填，值为资源 URL 的 JSON 字符串数组。JSON/compress 预演结果位于 `data.body.preview`，返回 `preview.appCode`、`preview.appName`、`preview.before`、`preview.after`、`preview.expectedHash`，不写入。正式结果返回 `data.appCode` 与 `data.appName` 标明实际目标。指纹同时绑定宿主、子应用、目标 URL 列表和当前应用配置；正式执行要求 `--expected-hash` 与本次输入及写前回读配置匹配，写后再次读取管理端应用并核对资源和扩展字段；URL 未变化时不重复写入。命令要求目标子应用条目唯一、应用发布状态可读、资源 URL 为绝对 HTTPS 地址且至少包含一个 JS 入口。

现有应用更新接口整列替换 `extend`，不提供原子并发比较；预演摘要只能阻止已发生的变化，不能消除写前检查与实际写入之间的竞争窗口。更新期间应避免并发修改同一宿主配置。

CLI 的保留和回读校验范围是应用读取接口返回的配置，无法证明数据库中未被接口返回的字段得到保留。直接访问数据库不是日常更新的通用前置条件；若发现未知扩展字段或有具体证据表明接口遗漏字段，应单独协调有权限的维护人员进行只读核查，核清后再决定是否更新，不得据管理端回读声称所有数据库字段均已保留。

服务端保存应用时可能触发独立部署自动同步，但命令仅确认管理端回读，`independentDeploymentVerified=false` 不表示目标已生效。本命令不上传文件，也不校验 CDN 文件字节；写入前需确认 CDN 资源已就绪。若写入结果或回读验证不确定，先用 `menu subapp-assets-status` 按相同宿主和子应用核查，不要直接重试。
