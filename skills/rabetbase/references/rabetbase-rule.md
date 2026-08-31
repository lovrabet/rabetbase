# rabetbase rule

管理开发者侧当前解析应用的固定 Agent context 规则文件 `RULES.md` 与 `DATABASE.md`。目标应用通过工作区、`--app <name>` 或 `--appcode <code>` 解析。

- `RULES.md`：页面、API 等研发时 Agent 使用的全局规则，`contextScope=page-api-development`；DB Agent 不消费。
- `DATABASE.md`：数据库分析时 DB Agent 使用的规则，`contextScope=database-analysis`；页面、API 等研发 Agent 不消费。
- 两份规则都在各自对应流程中全程参与 context，但不会跨消费者叠加。
- 两者统一属于应用级规则库，管理入口保持为单数命名空间 `rabetbase rule`；不要为 `DATABASE.md` 复制一套 `rabetbase db` 写入口。

## 服务端合同边界

- 只使用现有 KB 域接口：`GET /smartapi/knowledge-base/get-by-app` 与 `POST /smartapi/knowledge-base/save`。
- 文件名映射为 `scope=rules` 下的规范标题：`rules -> RULES`、`database -> DATABASE`。
- 不开放任意 scope、标题、重命名或删除，也不调用 admin 列表接口模拟规则管理。
- CLI 验证持久化结果；规则运行时只在互斥的 context scope 中消费。DB Agent 只消费 `DATABASE.md`、不消费 `RULES.md`；页面、API 等研发 Agent 只消费 `RULES.md`、不消费 `DATABASE.md`。

## 命令

```sh
rabetbase rule list --format compress
rabetbase rule get --type rules --format compress
rabetbase rule get --type database --format compress
rabetbase rule set --type rules --file ./RULES.md --dry-run
rabetbase rule set --type rules --file ./RULES.md --expected-version 2
rabetbase rule set --type database --file ./DATABASE.md --dry-run
```

## list / get

- `list` 固定精确读取两个类型，未配置项显示 `enabled: false`；输出包含 `contextScope`、版本、大小、哈希、时间等元数据，不包含正文。
- `get --type rules|database` 返回一个文件的完整正文；未配置时返回 `app_rule_not_found`。
- 响应必须与请求的 appCode、`scope=rules`、规范标题完全一致，否则失败关闭。

## set

- `--type rules|database` 与 `--file` 必填。
- 输入必须是可读普通文件、有效 UTF-8、非空且不超过服务端 5 MiB 限制；CLI 逐字发送解码后的内容。
- `set` 是普通 `write`，建议先 dry-run；正式执行不要求额外 `--yes`。
- 写前精确回读当前状态；`--expected-version` 可选，只是非原子 read-before-write 断言。服务端保存合同没有 version 条件，不能描述为 compare-and-set。
- 请求最多提交一次。正常或不确定响应之后都只做一次精确回读；正文哈希必须一致，已有记录还必须证明版本推进。
- 响应丢失但回读可证明结果时返回 verified 并标记恢复；证据不足时返回 outcome unknown，禁止自动重提。
- dry-run 和正式结构化输出都不包含新正文，只包含 `contentByteSize`、`contentHash` 等安全元数据。

## DATABASE.md 结构模板

创建数据库规则时，优先复制 [DATABASE.md 结构模板](./rabetbase-database-template.md)，再用当前应用中无法从 DDL、索引、约束和注释可靠推断的事实替换占位符。模板按 DB Agent 可识别的四类语义组织：

- `BUSINESS_DOMAIN`：系统定位、业务边界、数据隔离和权威来源。
- `ENTITY_DICTIONARY`：表对应的业务对象、职责、类型和生命周期。
- `COLUMN_SEMANTIC`：系统字段、人员标识、枚举状态、单位和快照语义。
- `RELATION_RULE`：明确的关联方向、唯一性依据、复合关系及禁止误关联的例外。

四个二级标题已经提供分类语义，不要在每条规则前重复 `[BUSINESS_DOMAIN]`、`[ENTITY_DICTIONARY]`、`[COLUMN_SEMANTIC]` 或 `[RELATION_RULE]` 标签。准入标准是“当前应用特有且无法可靠推断”，不以某个 LLM 版本是否声称知道该知识为判断依据。

完整 `DATABASE.md` 建议不超过 5000 个 Unicode 字符，并把影响范围最大的规则放在前面。这是控制 Agent context 的编写建议，不是 `rule set` 的硬校验；CLI 仍遵循服务端现有的 5 MiB 文件限制。字符数也不应换算为固定 token 数，因为不同语言和模型的分词结果不同。

不要重复普通主外键、字段类型等可直接推断的信息，也不要写入凭证、连接串、样例业务数据或短期排障结论。`one-to-one` 必须有主键或唯一索引依据；多态引用、ID 列表、快照文本和外部标识应放入 `RELATION_RULE` 下的“禁止关联”小节，明确排除普通外键推断。

## 最佳实践

1. 先执行 `rule list`，确认目标文件是否存在及当前版本。
2. 已存在时优先带 `--expected-version` 做写前漂移检查。
3. 使用相同参数先执行 `--dry-run`，检查 appCode、type、大小、哈希和非原子警告。
4. 正式执行后以返回的 `verification.status=verified` 为准；若为 outcome unknown，不要再次 set，先执行精确 `rule get` 人工核对。
