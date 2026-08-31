# project domain-routing-sync

根据当前项目最终生效的国家/地区与显式 Domain 配置，生成项目根目录的公开 Domain 路由快照。

```bash
rabetbase project domain-routing-sync --format compress
```

## 行为

- 从当前目录向上定位包含 `.rabetbase.json` 的项目根目录。
- 按项目优先、全局白名单补充的规则解析最终 Domain。
- 原子覆盖项目根目录的 `rabetbase.domain-routing.json`。
- 文件中的 `cdn.libraries` 是 React、Day.js、Ant Design 等第三方库的完整基地址，`cdn.lovrabet` 是 Lovrabet 自有资源 origin。
- 每个官方节点直接配置自己的 `cdn.libraries` 与 `cdn.lovrabet`；中国大陆当前使用 AliCDN，印尼当前使用 Cloudflare cdnjs。任一新节点都可以配置独立 CDN，企业独立部署使用 `lovrabet-routing/v1` 清单中的显式值。
- 不要求登录或 AppCode，不访问平台，不生成全局副本。
- 该文件供项目模板与 Vite 消费，不是 React Router、页面 path 或菜单路由配置。

生成内容只包含公开信息：User/App/Runtime Domain、资源策略、本地开发 Domain 与可选证书 URL；不会写入 Cookie、AccessKey 等认证配置。

## 使用时机

- `project create` 会自动完成首次生成，无需紧接着重复执行。
- 修改项目级 Domain 配置后执行。
- 全局 Domain 配置变化且当前项目需要采用新结果时执行。
- 文件缺失或模板提示重新生成时执行。

命令必须在 Rabetbase 项目目录或其子目录中运行；找不到项目 `.rabetbase.json` 时会停止，不会在任意目录创建快照。
