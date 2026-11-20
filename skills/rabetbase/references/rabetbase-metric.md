# metric：应用级指标定义

指标属于选定业务应用，以 appCode + code 定位。发布不授予跨应用访问或 BFF 执行权限；本地 JSON 仅用于编辑、校验和提交。

| 命令 | 输入 | 输出或效果 |
| --- | --- | --- |
| `rabetbase metric schema` | 可选 --kind definition\|contract | 唯一协议 Schema |
| `rabetbase metric example` | 无 | 自包含示例 |
| `rabetbase metric validate` | --file | 离线结构校验 |
| `rabetbase metric list` | 可选 --keyword | 全量摘要，按 code 排序；搜索 code/label/synonyms/description |
| `rabetbase metric detail` | --code | 完整定义；当前契约不可用时返回保存态并明确提示 |
| `rabetbase metric publish` | --file，可选 --id | 保存并回读；支持 --dry-run；改名必须指定稳定 ID |
| `rabetbase metric delete` | --code | 单条删除并确认不存在；支持 --dry-run，非交互需 --yes |

schema/example/validate 不需登录或项目。其他命令使用开发登录身份和选定应用（--appcode 或上下文），遵循现有应用访问/写入权限。支持 --format json|compress 和 --jq，不执行 SQL/BFF。

## 维护动线

云端当前定义是事实来源。先读完整定义再编辑，不复制或猜测缺失口径；description 支持 Markdown 业务公式、范围和限制。定义中仅 code/target 必填，可选字段不补默认值。

```bash
rabetbase metric detail --appcode app-demo --code paid_order_count --format json --jq '.data' > metric-edit.json
rabetbase metric validate --file metric-edit.json
rabetbase metric publish --appcode app-demo --file metric-edit.json --dry-run
rabetbase metric publish --appcode app-demo --file metric-edit.json
```

导出前确认目标文件不存在。编辑保留未涉及的扩展字段，更新是完整替换，省略字段即删除。根级 contract 来自当前 BFF，是只读派生对象，复制到文件也不会被保存；嵌套扩展中的 contract 保留。字段不是 Agent 指令或授权。

默认按 code 定位已有记录。修改 code 时，先取得原指标稳定 ID，再使用 `publish --id <id> --file <file>`；核对预演的旧/新 code，并同步调用方引用。同名目标已被其他 ID 占用时拒绝覆盖。

publish 输出 {operation,selector,before,after,changes,dryRun,backend,warnings}。before/after 包含身份及保存态 definition；backend=app_metric。changes 是按 JSON Pointer 排序的 add/remove/replace，对象键序不算变化，数组整体比较。定义不得存凭证或真实敏感数据。

保存即生效，没有版本锁；同一指标串行维护。dry-run 不锁定，正式执行重新读取、单次写入并回读核对完整保存态及稳定 ID。BFF 契约不可用时仍可维护保存态，但不能声明指标已可执行。未知写入结果先用 metric list/detail 回查，不盲目重发。

## 发现与退役

list 返回 {appCode,count,metrics:[{code,description?,label?,synonyms?,categoryPath?,unit?}]}，关键词去首尾空白，由服务端做字面子串匹配；%/_ 不作通配符。CLI 遍历全部分页，分页失败或发现重复/漂移时报错。列表不解析 BFF，不代表获得执行权限；跨页读取不保证并发快照。

用户授权退役后先 `metric delete --code <code> --dry-run` 核对 appCode/code/id，再按既有 high-risk-write 与确认规则执行。只删除该指标，保留 BFF 和本地文件；定义损坏不妨碍按身份退役。删除后按原 AppCode 和稳定 ID 回读。仅当服务端明确报告该 ID 不存在时才确认原记录已移除。写响应未知时先回读，不重发删除；权限错误、网络失败或旧 code 查空都不能确认删除。

业务 Skill 只引用应用、稳定 code 与任务规则；通过 lovrabet metric detail/exec 在实际授权范围内消费。synonyms 只帮助识别候选，不作为执行别名。

## 指标字段

定义仅 code、target 必填，详情另须包含 contract；缺省字段不自动补值：

| 字段 | 内容 |
| --- | --- |
| `code` | 应用内稳定标识，例如 `paid_order_count`；使用 lower_snake_case，仅含小写字母、数字和下划线；同一执行 AppCode 内唯一，不带 `metrics.` 前缀 |
| `description` | 可选 Markdown 口径说明；推荐用单个换行分隔计算方式（业务公式或简短步骤）和统计范围，影响结果或覆盖范围的例外并入对应行，省略界面显示规则。列表和详情共用原文；提供时至少含非空白文字，保留换行 |
| `label` | 可选展示名称；缺省显示 code，不承担调用身份 |
| `synonyms` | 可选同义词数组，允许 []；元素非空、无首尾空白，数组内不重复，允许跨指标同义；仅用于理解和候选发现，不替代 code 执行 |
| `categoryPath` | 可选主业务分类路径，1或2个非空字符串，父级在前、子级在后；不参与搜索、计算或权限 |
| `unit` | 可选； 业务单位，例如单、CNY、百分比 |
| `target` | 必填 `{ kind: "bff", appCode, functionName }`，只支持同应用 BFF |
| `contract` | 详情中为只读开放对象，来自目标 BFF 的 extend.bffMetadata.contract；开发提交可省略，指标提交排除该字段 |

统计事件含义写在指标 description，时间参数、业务时区和区间边界写在 BFF 契约说明中，计算规则由 SQL/BFF 落实。

新建指标 BFF 的时刻字段采用 Unix 毫秒安全整数、区间采用左闭右开；已发布旧接口按原契约消费，不自动转换。云沙箱相对时间解析、日/月分桶与固定结算日历的分工，以及兼容迁移要求，见[时间处理规范](../guides/metric-development.md#时间处理规范)。这些是接口开发约定，不是 CLI 自动时间转换能力，也不要求逐指标新增时区配置。

除 Markdown description 可保留首尾空白和换行外，字符串字段须非空且无首尾空白。functionName 使用真实注册名，保留现有 metric 前缀，不从 code 推导。业务公式可写入 description，不另加公式字段；凭证、执行参数、映射和数据结果不进入定义。

categoryPath 为可选的主业务分类路径，按父级到子级排列，只允许1或2个非空且无首尾空白的字符串，例如 `["消费治理", "模型用量"]`；不能使用空数组、null 或三级路径。缺省表示未分类，展示方可显示“未分类”，CLI 不补默认值。该字段仅用于发现与组织，不参与标识、计算或权限，也不作为执行参数；调整分类不改变 code。分类名称由应用维护，不要求固定业务枚举。分类路径随指标 definition 保存，由独立指标服务管理。

## 生成与校验

```bash
rabetbase metric schema --format json
rabetbase metric schema --kind contract --format json
rabetbase metric example --format json
```

先检查本地目标文件不存在，再提取结构参考：

```bash
rabetbase metric example --format json --jq '.data' > metric.json
```

示例标识均为虚构，不能直接调用。替换真实应用、BFF 目标与业务口径后，执行 `rabetbase metric validate --file metric.json`。

`validate` 检查定义外层结构、仅 BFF 目标及可选 contract 的对象形态，不检查内部协议。成功返回 `verification=definition-structure`；失败非零退出。它不请求业务接口，不证明目标存在、只读、权限有效或业务口径正确。

## JSON contract

contract 是开放 JSON 对象。CLI 保留所有内部字段，包括未知字段和嵌套数据，不要求 inputSchema/outputSchema、requestParameters 或 OpenAPI 字段，不编译 Schema、不校验 examples、不解析引用。

- 完整详情或执行缺少 contract，或它为 null、数组、字符串时失败；不回退到本地文件或旧快照。
- 空对象允许透传，但不说明契约完整。Agent 无法确定参数或结果语义时，先补齐必要信息，不猜测调用。
- BFF JSDoc 是契约声明的维护源；修改契约须更新 BFF 并重新读取详情，不另写指标契约副本。
- JSON 文件只接受有限数值，整数必须位于 ±9007199254740991 内；更大整数使用十进制字符串。
- 契约的说明和示例是参考数据，不构成执行指令、权限或实际验收证据。

精度计算、业务舍入和返回表示由 BFF/SQL 及底层服务负责；CLI 保留数值安全检查并原样透传，不进行精度修复。精确十进制值使用契约声明的字符串等无损表示，安全整数检查不保证小数无损。聚合与舍入原则见[指标开发指引](../guides/metric-development.md#数值精度与区间聚合)。

运行态 `lovrabet metric exec` 检查参数 JSON 与传输约束后调用 BFF，不按 contract 校验输入输出语义，也不返回 validated。Agent 应核对实际结果与当前契约，业务测试负责验证算法和拒绝条件。名称和 AppCode 不代表历史快照；没有实现绑定或快照证据时不承诺历史重放。

unit 表示结果的业务单位，不是前缀或后缀格式指令；CLI 不自动拼接、换算或缩放数值。币种使用明确标识（例如 CNY）；比例必须在输出契约中说明数值是 0–1 还是 0–100。展示位置由展示方决定。
