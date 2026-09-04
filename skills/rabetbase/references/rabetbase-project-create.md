# project create

创建新项目。

> **风险等级：write** — 创建项目文件和配置。

## 命令

```bash
# 交互模式
rabetbase project create

# 指定项目名
rabetbase project create my-project

# 通过 flag 指定
rabetbase project create --name my-project

# 在当前空目录创建指定 AppCode 的工程
rabetbase project create --current-dir --appcode <code>

# 非交互模式
rabetbase project create my-project --appcode <code>
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `<project-name>` | positional | 否 | — | 项目名称（也可用 `--name`） |
| `--name <name>` | string | 否 | — | 项目名称（优先于位置参数） |
| `--current-dir` | boolean | 否 | `false` | 初始化当前空目录；项目名取目录名，不创建子目录 |
| `--appcode <code>` | string | 否 | — | 绑定的应用 code（跳过交互选择） |

## 两种模式

1. **交互模式** — 通过提示选择项目配置
2. **非交互模式**（`--non-interactive` 或非 TTY）— 必须提供项目名称，或使用 `--current-dir`；支持 `--appcode` 跳过应用选择

## 当前目录创建工程

当用户表达“在当前文件夹创建 `app-xxxxxx` 应用工程”等意图时，将其视为授权在当前空目录创建并验证本地工程。使用 `rabetbase project create --current-dir --appcode <code>`。

**完成口径：**

- 工程绑定目标 AppCode，项目配置、公开路由和 SDK 客户端保持一致。
- 已获取 Dataset 事实，并确认本地模型配置与返回事实一致。
- 项目提供的类型检查和构建验证通过。
- 向用户报告创建结果、Dataset 数量、验证结果及仍需处理的问题。

**关键边界：**

- 当前目录非空时不得覆盖、移动或删除已有内容，应报告冲突。
- `needsAgentMerge` 只表示已有 TypeScript 被保留，是检查提示，不代表必须改写文件；仅在模型事实存在差异时合并。
- `--force --yes` 只用于首次脚手架拉取明确失败，或用户明确要求放弃本地定制的场景。
- 此意图只授权本地工程创建与平台事实读取，不授权修改线上业务数据或平台资源。

`api pull` 的单应用/项目应用清单输出结构见 [`rabetbase-api-pull.md`](rabetbase-api-pull.md)，TypeScript 合并规则见 [`sdk-client-generation.md`](../guides/sdk-client-generation.md)。Agent 可根据实际输出选择检查、修复和验证方式。


项目创建完成后，如需在子应用页面中使用公共组件，先阅读 [`components.md`](../knowledge/components.md) 确认最新组件用法，并可参考项目内 `src/pages/components-demo` 的交互和实现。`components-demo` 必须在项目配置了有效 AppCode 后使用，优先在创建工程时传入 `--appcode <code>`；如创建后补充 AppCode，需执行 `rabetbase config set appcode <code>` 和 `rabetbase api pull` 完成初始化，否则用户选择、附件上传等依赖应用信息的组件无法使用。示例中的 mock 数据和富文本图片 mock 上传仅用于展示效果，正式页面需按实际业务替换。

## 提示

- `--name` 和位置参数二选一，`--name` 优先
- `--current-dir` 不可与 `--name` 或位置项目名同时使用
- `--current-dir` 只接受空目录；创建失败会保留该目录并清理本次暂存内容
- 非交互模式下缺少项目名且未使用 `--current-dir` 会报错
- 项目名只接受字母、数字、`-`、`_`，不能传绝对路径、`../`、子目录路径或 Windows 保留名称
- 交互与非交互模式使用同一创建流程，都会安装依赖、格式化代码并写入项目配置
- 项目模板从 CDN 下载并校验 SHA-256；CDN 不可用、模板不兼容或校验失败时命令停止，修复后重试
- 新项目中的 `.rabetbase.json` **只继承**全局中的少量偏好及 `region` / Domain 路由配置，**不会**把全局的 `apps` / `defaultApp` 复制进新项目
- `src/api/sdk-config.ts` 与 `rabetbase.domain-routing.json` 只生成浏览器可公开的最终路由，不会写入 `cookie`、`accessKey` 等认证配置；`rabetbase run start|dev|build|preview` 会在执行脚本前刷新公开 Domain 快照，也可用 [`project domain-routing-sync`](rabetbase-project-domain-routing-sync.md) 立即显式刷新。`api.ts` / `client.ts` 仅在缺失时由 CLI 写脚手架，已有文件按 [`sdk-client-generation.md`](../guides/sdk-client-generation.md) 更新；若项目创建时首次拉取失败，按 CLI 提示执行 `rabetbase api pull --force --yes` 完成占位脚手架初始化

| 当前有效路由 | `src/api/sdk-config.ts` |
|--------------|-------------------------|
| 默认中国大陆节点 | `{}`，由 SDK 使用默认映射 |
| ID | `{ runtimeDomain: "https://runtime.lovrabet.id" }` |
| Global | `{ runtimeDomain: "https://runtime.lovrabet.ai" }`，兼容 SDK 1.5.3 |
| 独立部署 | `{ runtimeDomain: "https://…" }` |

模板依赖 SDK 1.5.3+。CLI 从统一 Routing Profile 生成 `rabetbase.domain-routing.json`，其中只包含最终 API/User/App/Runtime/Skill/KB Domain、`cdn.libraries`、`cdn.lovrabet`、本地地址和证书 URL，不复制节点注册表。每个官方节点直接配置两个 CDN 地址；中国大陆当前使用 AliCDN，印尼当前使用 Cloudflare cdnjs，任一新节点都可以使用独立 CDN。企业独立部署使用清单中的显式值。

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase config init](rabetbase-init.md)
