# DATABASE

> 仅记录当前应用特有、稳定，且无法从实时数据库结构、约束、索引、字段注释及通用数据库约定可靠推断的业务事实。删除不适用条目；完成后的文件建议不超过 5000 个 Unicode 字符，重要规则优先。物理结构以实时元数据为准，业务语义以已确认业务合同为准。

## BUSINESS_DOMAIN

- 系统定位：{一句话说明系统服务的业务与核心目标}。
- 业务边界：{明确包含与不包含的业务范围}。
- 数据隔离：{租户、组织、门店或其他隔离维度及判定字段}。
- 权威来源：{主数据、交易事实或配置分别以哪些表或外部系统为准}。
- 业务分组：{表前缀、模块或领域之间的稳定映射}。

## ENTITY_DICTIONARY

- `<table>`：业务对象={名称}；类型={coreEntity/dictionaryTable/config/other}；职责={该表保存什么事实}；主记录判定={条件或字段}。
- `<child_table>`：从属于 `<parent_table>`；生命周期={随主对象创建、独立维护或历史保留}；一条记录表示={业务含义}。
- `<snapshot_or_log_table>`：类型={snapshot/log/history}；固化时点={事件}；用途={审计、展示或计算}；不作为={不应承担的当前事实来源}。

## COLUMN_SEMANTIC

- `<table>.<column>`：含义={业务含义}；值域/单位={枚举、金额单位、时区等}；空值={未设置、未知或不适用}。
- 系统字段：创建时间={field}；修改时间={field}；创建人={field}；修改人={field}；逻辑删除={field, 正常值/删除值}。
- 人员标识：`<table>.<column>` 表示 {employee/account/customer/external id}，关联目标={table.column 或外部系统}。
- 状态流：`<table>.<status_column>` 的关键状态={value: meaning}；终态={value}；允许回退={条件}。
- 冗余或快照字段：`<table>.<column>` 来源={table.column/计算规则}；刷新时机={事件}；分析时优先级={源字段或快照字段}。

## RELATION_RULE

- `<source_table>.<source_column> -> <target_table>.<target_column>`；关系={many-to-one/one-to-one}；唯一性依据={PK/unique index}；展示字段={target.column}。
- `<source_table>.(<column_a>, <column_b>) -> <target_table>.(<column_x>, <column_y>)`；关系={复合关联含义}；成立条件={过滤条件}。
- `<source_table>.<code_column> -> <dictionary_table>.<code_column>`；字典分类={类型字段及取值}；展示字段={name_column}。

### 禁止关联

- `<table>.<column>` 是多态引用；由 `<type_column>` 判定目标：`<value> -> <target_table>`；缺少类型条件时不得直接关联。
- `<table>.<column>` 是 {快照文本/ID 列表/外部标识}，不得按普通外键关联；正确解释={规则}。
