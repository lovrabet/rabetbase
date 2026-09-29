# 宿主前端集成

按用户目标和已有对象选择流程。已有 pageId 且页面产品类型明确时，直接进入对应页面工作流；类型未知时先读取页面事实。处理前端子应用接入或资源版本时，再确定宿主与 appName；CLI 通过 `--app-name` 显式接收标识，不自动从当前工程推导。

## 产品对象与操作入口

`appCode` 唯一标识一个应用；该应用承载前端集成时称为“宿主应用”。确定目标 `appCode` 后即确定宿主，无需在该 appCode 下再选择另一个宿主实体。应用权限也直接归属于这个 appCode；同一应用可以集成多个以 `appName` 区分的前端子应用。

一个宿主应用可以集成多个前端子应用，以及多个平台页面。平台页面包括数据列表页（DATA_LIST）和自定义页面（CUSTOM）；前端子应用是拥有自身工程、构建资源和页面路由的集成单位。菜单承载导航、路由及资源关联，不要求每个菜单都属于某个子应用。

| 产品对象 | 定位与操作入口 |
| --- | --- |
| 前端子应用 | 按宿主 appCode + appName 定位共享资源；需要注册本地页面路由时使用 `menu sync`，保留菜单分组与绑定 |
| 数据列表页（DATA_LIST） | `page generate-start` 发起异步生成，生成流程自动创建菜单入口；通过 `page generate-status` 跟进任务，以 `page data-list-status` 核对页面组及对应菜单，无需额外绑定 appName 或用 `menu sync` 注册菜单 |
| 自定义页面（CUSTOM） | `page create` 同时创建页面与菜单入口；后续按 pageId 使用 `page custom-update` 保存、`page custom-publish` 发布，无需额外绑定 appName 或用 `menu sync` 注册菜单 |

平台页面的完整工作流分别见[数据列表页开发](page-development-workflow.md)和[自定义页面开发](custom-page-workflow.md)。自定义页面的创建来源、完整文件更新与发布核验分别见 [`page create`](../references/rabetbase-page-create.md)、[`page custom-update`](../references/rabetbase-page-custom-update.md)、[`page custom-publish`](../references/rabetbase-page-custom-publish.md)。仅在明确需要调整菜单资源配置时，按 [`menu asset-update`](../references/rabetbase-menu-asset-update.md) 精确维护对应菜单；这不属于每次页面发布的必做步骤。

数据列表页的生成与核验见 [`page generate-start`](../references/rabetbase-page-generate-start.md)、[`page generate-status`](../references/rabetbase-page-generate-status.md)、[`page data-list-status`](../references/rabetbase-data-list-status.md)。任务提交成功不代表页面和菜单已就绪；生成后核对完整页面组及对应菜单事实，不把菜单可见性等同于菜单是否存在，也不要求每个辅助页面都有可见菜单。预期菜单入口缺失时报告并核查，不自动用 `menu sync` 补建。

根据已有页面产品类型、菜单绑定、工程部署方式和用户目标选择流程。PageSchema 是数据列表页的页面描述技术，JSX 是自定义页面的实现技术，不作为额外产品类别；文件类型、页面数量、目录或包名也不能单独决定是否属于子应用。未绑定 appName 可能是尚未接入的子应用，应结合已有事实判断。归属明确时继续对应流程，不明确时只澄清本次目标的集成方式。平台页面不为使用共享资源而自动改造成子应用，也不强制补 appName。

子应用共享资源属于宿主的前端集成配置，菜单通过 appName 关联。同一宿主中的多个子应用分别使用各自的 appName；更新一个子应用时保留其他子应用与平台页面的配置。资源优先级判断仅适用于关联该子应用的菜单：共享资源优先，菜单资源回退；不扩展到宿主中的所有页面。`menu sync` 负责子应用本地路由注册，不替代平台页面的创建、内容更新与发布。

## 子应用标识与来源

`appName` 是同一宿主内稳定的前端子应用标识：宿主 `extend.assets[].appName` 与关联菜单 `extend.appName` 精确匹配。版本升级沿用该标识，只更新资源 URL。子应用访问的业务 appCode、CLI 应用 profile 和 `.rabetbase.json` 的 `defaultApp` 都不是该标识。

1. 沿用 CLI 的应用选择确定宿主：支持默认应用、唯一应用以及 `--appcode` / `--app` 覆盖。默认应用与目标宿主不同才覆盖，不要求每次显式传宿主。
2. 用户已提供准确标识时，用该值查询共享资源并核对关联菜单；用户只给了业务名称时，先定位真实标识。
3. 未提供标识时，用 `menu list --verbose` 读取 `extend.appName`，结合当前工程的页面路由、已知部署资源及用户提供的对应关系，定位属于本工程的菜单绑定。已知宿主共享配置中的标识也可作为候选。仅宿主内候选唯一、名称相似或单个路径相同，不足以证明属于当前工程。
4. 有明确对应关系且标识唯一时，沿用平台已有值。多个候选、来源冲突或无法建立工程对应关系时，展示候选和来源，请用户选择；不要自行改名或覆盖绑定。一个工程包含多个子应用时按本次目标分别定位。
5. 首次接入且没有已有绑定时，可参考工程现有命名提出候选，由用户确定标识；用户已明确指定时无需重复确认。`package.json.name`、目录名及构建目录中的名称只是线索，不作为平台标识的自动默认值；包名改变也不自动修改已有绑定。

```bash
# 读取标识必须带 --verbose；将占位符替换为已确定的宿主与标识
rabetbase menu list --appcode <host-appcode> --verbose --format json
rabetbase menu subapp-assets-status --appcode <host-appcode> --app-name <app-name> --format json
```

执行前说明已定位的宿主、标识及其来源，例如：“已定位宿主 A 下的子应用 store-app，标识来自当前工程对应菜单的绑定；本次更新它的共享资源 URL。”已有授权和明确对应关系时继续执行，无需重复确认。

## 子应用资源版本核验

1. 用户只提供版本号时，先从用户提供的完整 URL、项目已有发布记录或构建上传结果中确认该版本对应的完整资源 URL 列表；版本号本身不是资源地址。不要仅凭 `package.json.version`、目录名称或替换旧 URL 中的版本字符串构造目标 URL。查不到可信对应关系时，说明缺少哪一版本的资源地址并请用户补充，暂不写入。
2. 用户指定“从 A 升到 B”时，先按共享资源优先、关联菜单资源回退的规则读取当前配置，并核对它与 A 的资源 URL 对应。当前配置已经等于 B 的完整 URL 列表时，报告配置已匹配，无需重复写入；当前配置与 A 不符、当前采用菜单回退且各关联菜单版本不一致，或无法确认 A 时，展示实际配置和差异，先澄清起点，不把预演指纹当成 A 版本已核验的证明。
3. 检测环境与本地发布版本是否一致时，比较实际发布资源的完整 URL 列表，并注明来源。本地包版本或源码版本不足以证明发布产物一致；没有可信发布资源地址时，报告“缺少发布资源依据，无法判断”，仍可列出已读取的共享配置和菜单回退配置。
4. 结论区分“资源 URL 配置一致”“CDN 文件内容已验证”“目标宿主加载已验证”。仅查询或更新管理端配置时只确认第一项，不能据此声称发布内容或实际加载版本已验证。

## 查询结果与后续操作

- `data.appName` 是输入 `--app-name` 去除首尾空白后的回显，不表示已匹配到配置。`sharedConfigured` 表示存在对应共享配置条目，`boundMenus` 表示关联菜单数量；共享资源是否可用还要检查 `sharedResources` 是否非空。
- 有菜单绑定但没有共享配置时，沿用菜单中的标识，引导在宿主中配置共享资源，并保留原有菜单资源。`menu subapp-assets-update` 只更新已有条目；首次建立条目需使用平台已有配置入口，配置后再查询核对，不把更新命令当作创建命令。
- 没有共享条目且没有关联菜单时，报告“该宿主下未找到此标识的共享配置或菜单绑定”，核对目标或按首次接入处理；不能凭回显报告已找到子应用。
- 检测子应用版本先比较非空共享资源，其次比较关联菜单资源；共享配置缺失时可以如实报告菜单当前配置，不自动写入或切换到逐菜单更新。数据列表页和自定义页面按各自页面流程核查，需要菜单资源时单独核查该菜单。
- 注册子应用页面时，将已确定的同一标识传给 `menu sync --params` 中的 `appName`，按现有菜单树保留分组；回读绑定时使用 `menu list --verbose`。共享资源更新和菜单注册分别按下列 reference 预演、执行和核验。

## 命令契约

- [查询共享资源](../references/rabetbase-menu-subapp-assets-status.md)
- [更新已有共享资源](../references/rabetbase-menu-subapp-assets-update.md)
- [注册菜单及保留原有操作路径](../references/rabetbase-menu-sync.md)
