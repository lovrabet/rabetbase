---
name: dataset-relations
description: 数据集关系模型参考，包含关系方向、字段语义、关系基数、主子表语义与页面配置边界
tags: [page-schema, dataset, relation]
---

# 数据集关系声明

数据集关系是数据模型元数据，用于描述两个数据集的字段如何关联。它不是某个页面组件的专属配置，也不等同于数据库外键：关系既可以表示物理表之间的引用，也可以表示字典选项源、METADATA 自关联或主子表业务归属。

页面中的 `relation.field`、关联字段展示和筛选，都依赖已定义且有效的数据集关系。

## 关系事实字段

服务端返回的关系事实嵌入在来源数据集的 `relations` 中。下面保留服务端原始字段名；来源数据集本身由外层数据集对象确定，因此关系对象内不重复提供 `fromDatasetCode`。

数据集 `relations` 中的普通关联关系示例：

```json
{
  "relationId": 9074,
  "fromField": "create_by",
  "toDatasetCode": "d7c1ad8525ce4823bf5f7e4f4edab30b",
  "toDatasetId": 48426,
  "toDatasetName": "员工",
  "toField": "id",
  "toFieldLabel": "user_name",
  "toTableName": "crm_user",
  "sourceType": "DB_TABLE",
  "joinType": "MANY_TO_ONE",
  "cardinality": "MANY_TO_ONE",
  "bizRelationType": null,
  "mainColumnName": null,
  "subColumnName": null,
  "subDatasetCode": null,
  "subTableName": null
}
```

示例将 `joinType` 和 `cardinality` 并列展示，便于说明字段对应关系；实际接口会按各自定义使用其中一个字段名，两者表示的关系基数必须一致。

| 字段 | 含义 |
|------|------|
| `relationId` | 关系记录标识 |
| `fromField` | 当前数据集中的关联字段 |
| `sourceType` | 主表的数据源类型：`DB_TABLE` 表示数据库类型，`METADATA` 表示文生类型 |
| `toDatasetCode` / `toDatasetId` / `toDatasetName` | 目标数据集的编码、标识和名称 |
| `toField` | 目标数据集中的关联字段 |
| `toFieldLabel` | 目标数据集的展示字段名，例如 `user_name`；不是中文标题 |
| `toTableName` | 目标物理表名称；非物理表数据集时可为空 |
| `joinType` | 关系对象中的关系基数字段，可选 `ONE_TO_ONE`、`ONE_TO_MANY`、`MANY_TO_ONE`、`MANY_TO_MANY` |
| `cardinality` | 关系接口中的关系基数字段，与 `er-config` 中的 `joinType` 表示同一个内容；只是接口字段名不同，取值范围完全一致 |
| `bizRelationType` | 业务关系类型；普通关联为 `null`，主子表为 `main_sub` |
| `mainColumnName` / `subColumnName` | 主子表关联时使用的主表列和子表列 |
| `subDatasetCode` / `subTableName` | 主子表关联时使用的子表数据集编码和子表名称 |

一条关系从当前数据集的 `fromField` 指向 `toDatasetCode.toField`。目标数据集名称、物理表名和展示字段是关系的补充信息，不能代替数据集编码和真实字段名判断关系。

## 关系方向

关系方向始终从当前数据集指向目标数据集。方向决定字段含义、关系基数的观察视角，以及页面从哪个字段加载目标数据。

| From | To | 是否支持 | 常见语义 |
|------|----|:---:|----------|
| `DB_TABLE` | `DB_TABLE` | 是 | 物理表数据集之间的引用或主子关系 |
| `DB_TABLE` | `METADATA` | 是 | 业务字段消费 METADATA 字典或选项源 |
| `METADATA` | `METADATA` | 是 | 配置数据集关系或自关联父子关系 |
| `METADATA` | `DB_TABLE` | 否 | 当前关系模型不支持该方向 |

当当前数据集编码与 `toDatasetCode` 相同时，该关系是自关联。自关联仍需明确 `fromField`、`toField` 和基数，不能仅凭数据集编码相同推断父子语义。

## 关系基数

服务端不同接口对同一关系基数使用了不同的字段表达：`er-config` 关系对象使用 `joinType`，其他关系接口使用 `cardinality`。两者本质上是同一个属性，不是两项需要分别维护的数据。CLI 调用的是使用 `cardinality` 的关系接口，因此对应的参数名为 `cardinality`（命令行参数为 `--cardinality`）。

关系基数以当前数据集记录到目标数据集记录的方向解释：

| 值 | 含义 |
|----|------|
| `ONE_TO_ONE` | 一条来源记录对应一条目标记录 |
| `ONE_TO_MANY` | 一条来源记录对应多条目标记录 |
| `MANY_TO_ONE` | 多条来源记录可以指向同一条目标记录 |
| `MANY_TO_MANY` | 来源与目标两端都可以关联多条记录 |

基数必须与真实数据约束一致。不能因为字段名看起来像外键，就直接判断为 `MANY_TO_ONE`；也不能因为页面要展示列表，就把普通引用关系改成 `ONE_TO_MANY`。

## 普通关系与主子表

普通关系表示两个相对独立的数据集之间存在引用，例如订单引用客户、项目引用负责人。主子表则表示明确的业务归属：一条主记录拥有一组随其维护的明细记录，例如订单与订单明细、合同与付款计划。

| 业务关系 | `joinType` | `bizRelationType` |
|----------|------------|-------------------|
| 普通引用 | 按真实基数设置 | `null` |
| 主子表 | `ONE_TO_MANY` | `main_sub` |

判断是否属于主子表时，应同时确认：

1. 打开一条主记录时，是否天然需要查看或维护它下面的一组明细。
2. 明细是否独立归属于该主记录，而不是多个业务记录共享的独立对象。
3. 两端字段能否真实关联到同一条主记录，字段类型和值域是否兼容。
4. 子记录是否应随主记录的业务生命周期一起维护。

普通引用、枚举映射、字典选项和只用于展示的 LOOKUP 都不应标记为 `main_sub`。元数据只能描述结构，不能证明业务归属；最终仍需结合真实记录验证。

## 完整性规则

有效的数据集关系应满足：

- 当前和目标数据集都存在，且 `toDatasetCode` 指向真实数据集。
- `fromField`、`toField` 都存在，字段类型和值域能够匹配。
- `toFieldLabel` 是目标数据集的真实字段名，并适合向用户展示。
- 关系方向处于支持范围内。
- `joinType` 与真实数据约束和业务语义一致。
- 同一组来源、目标和字段不存在重复关系。
- `main_sub` 只用于真实的一对多业务归属关系。

验收主子表时，应至少准备一条主记录和多条子记录：当前主记录只能读取归属于它的子记录，切换主记录后不能继续出现上一条记录的明细。

## 与页面配置的边界

- 数据集关系只定义数据模型语义；关系存在不代表页面已经生成关联区域、明细布局或新增编辑交互。
- `LrSmartCreate`、`LrSmartUpdate` 和 `LrSmartDetail` 的 `props.sub_info` 是 PageSchema 展示与交互配置，不是数据集关系对象。
- 标准页生成器使用的 `extend.subDatasetCode`、`extend.subTableName`、可选 `extend.subColumnName` 和 `relatedTablesJson` 属于页面生成协议；即使与关系元数据存在同名字段，它们也不是数据集 `relations` 对象本身。
- 表单、筛选器和表格中的动态选项配置只负责加载候选数据，不能替代数据集关系声明。

排查页面关联异常时，应先确认数据集关系本身的方向、字段、展示字段、基数和业务类型，再检查页面的数据源与组件绑定。
