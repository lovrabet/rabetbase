---
name: LrSmartFilter
description: LrSmartFilter 筛选器组件完整配置
tags: [filter, page-schema, where, initialValue]
---

# LrSmartFilter

筛选器组件，用于配置查询条件。

---

## 禁止事项

| 禁止 | 替代方案/说明 |
|------|---------------|
| `meta.handleProps` | 纯 JSON props |
| 字段改名/类型/字典 | 需在数据集管理中操作 |

---

## Props 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| _doVersion | string | 数据集版本标识，必须保留原值（如 `"V2"`），不可删除或修改 |
| items | Item[] | 筛选项配置 |
| events | Event[] | 事件配置（onSearch 等） |

> 运行时约束：`LrSmartFilter` 不消费 `layoutMode` / `cols`，PageSchema 中不要生成这两个字段。

---

## Item 筛选项属性

V2 页面生成时已为数据集所有字段全量生成 items，通过 `isShown` 控制显示。用户"添加筛选项" = 将已有 item 的 `isShown` 设为 `true`；仅当字段在 items 中完全不存在时才需插入新对象。

### INSERT element 最小结构

arrayName = `filterItems`，字段必须存在于数据集（先 dataset_detail_tool 确认）。工具层自动为选择类组件补 `options: []` 骨架。

输入类：

```json
{"componentName": "YtInput", "label": "<fieldLabel>", "name": "<fieldName>", "isShown": true}
```

选择类（静态枚举）：

```json
{"componentName": "YtSingleSelect", "label": "<fieldLabel>", "name": "<fieldName>", "isShown": true, "options": [{"label": "<显示文本>", "value": "<枚举值>"}]}
```

选择类（动态选项）：需同时在 Page 层配置 dataSource，参见下方「筛选项 options 数据配置 § 方式 2」。

| 属性 | 渲染 | 参与提交 | 用途 |
|------|:----:|:-------:|------|
| `isShown: false` | ❌ | ❌ | 完全不渲染，等同于不存在 |
| `hidden: true` | ❌ | ❌ | Filter 中会被跳过，不参与 where |

| 属性 | 类型 | 说明 |
|------|------|------|
| componentName | string | 组件类型 |
| label | string | 标签 |
| name | string | 字段名，支持 `relation.field` |
| placeholder | string | 占位符 |
| initialValue | any | 默认值（与 Ant Design 5 Form.Item initialValue 一致），同时作为首次 API 请求 where 条件的数据源 |
| options | array | 选项列表（Select/Cascader） |
| showSearch | boolean | 可搜索（Select） |
| filterOption | boolean | 本地过滤（远程搜索时设 false） |
| events | object | 禁止默认生成；当前运行时不消费字段级 `events` |

### 组件类型

筛选项可用的 `componentName`、字段组件选型规则、数据格式与动态选项约束，统一参考 `field-components.md`。本文档仅补充 Filter 场景特有的 options 数据配置、关联字段引用和与 Table 的联动方式。

---

## 筛选项 options 数据配置

`YtSingleSelect`、`YtMultiSelect`、`YtCascader` 必须配置 options 数据来源，否则下拉列表为空。

### 方式 1：静态选项（枚举值已知）

直接在筛选项配置中提供 options 数组。

```json
{
  "componentName": "YtSingleSelect",
  "label": "客户状态",
  "name": "status",
  "options": [{"label": "活跃", "value": 1}, {"label": "流失", "value": 0}]
}
```

### 方式 2：动态选项（关联表数据）

- **机制**：options 数据来自关联表，需两步配置：Page 层请求 To 表 options 数据 → 筛选项绑定 state。state key 必须与 dataSource id 一致
- **参数**：

| Placeholder | Value Source | Description |
|-------------|--------------|-------------|
| `dataset_<FromDatasetCode>_<FromRelationField>_options` | `dataset_` prefix + From 表字段所属 datasetCode + From 表关联字段名 + `_options` suffix | dataSource id, must match state key |
| `<appCode>` | Application code | From current app config |
| `<ToDatasetCode>` | To 表 datasetCode | From dataset relations |
| `<ToValueField>` | To 表被关联字段 | Usually `id` or `code` |
| `<ToLabelField>` | To 表展示字段 | Usually `name` / `title` / location field |

> 自关联父子关系中，`FromDatasetCode` 与 `ToDatasetCode` 可以相同；state key 仍按 `dataset_<FromDatasetCode>_<FromRelationField>_options` 命名，URI 仍使用 To 侧 options 数据源。

- **示例**：

步骤 1：Page 层配置 getSelectOptions 请求

```json
{
  "dataSource": {
    "list": [{
      "id": "dataset_<FromDatasetCode>_<FromRelationField>_options",
      "type": "fetch",
      "isInit": true,
      "source": "dataset",
      "options": {
        "uri": "/api/<appCode>/<ToDatasetCode>/getSelectOptions",
        "params": {"code": "<ToValueField>", "label": "<ToLabelField>"}
      }
    }]
  }
}
```

步骤 2：筛选项通过 JSExpression 绑定 state

```json
{
  "componentName": "YtSingleSelect",
  "label": "<filterLabel>",
  "name": "<FromRelationField>",
  "options": {
    "type": "JSExpression",
    "value": "this.state.dataset_<FromDatasetCode>_<FromRelationField>_options"
  }
}
```

> `getSelectOptions` 禁止使用 `optionsRequest`。即使字段级请求技术上能访问完整接口，也会把筛选项耦合到具体域名或部署环境；Agent 必须使用 Page 层 `dataSource.list` 的相对 URI，并通过 `this.state.dataset_<FromDatasetCode>_<FromRelationField>_options` 绑定。`optionsRequest` 仅用于三方链接、外部接口或外部静态资源 options。

---

## initialValue 与 where 条件初始化

### 机制

where 条件配置分两个生命周期阶段。

| 阶段 | 配置属性 | 作用 |
|------|---------|------|
| 初始化 | `initialValue` | 页面首次加载时的默认筛选条件，写入 filter state → where |
| 交互 | 用户选择/输入筛选项 | onValuesChange / onSearch 更新 filter state → where |

`initialValue` 是首次 where 条件的唯一数据源。组件初始化时会将默认筛选值合并为 `initialValues`，再写入 `context.state[filterKey]`，确保首次 datasource.load() 的 JSExpression 求值能读到 where 条件。不配置 initialValue 的筛选项不参与首次 where。

维护说明：

- `LrSmartFilter` 会将 `items[].initialValue/defaultValue` 合并到顶层 `initialValues` 后进入运行时筛选逻辑。
- 顶层 `initialValues` 优先级高于 `items[].initialValue/defaultValue`。
- `initialValues` 与 `defaultValue` 不应同时生成；常规默认筛选条件优先使用 `items[].initialValue`。
- 常规新增/更新筛选默认值时，优先维护对应 `items[].initialValue`；只有批量初始化或已有 schema 已使用顶层 `initialValues` 时，才维护顶层 `initialValues`。

### 参数

| Source | Target | Description |
|--------|--------|-------------|
| `items[].initialValue` | `context.state[filterKey].where.$and` | Auto-converted on component initialization |
| YtSingleSelect initialValue | `{ fieldName: { $eq: value } }` | Single select → $eq |
| YtMultiSelect initialValue | `{ fieldName: { $in: [...] } }` | Multi-select array → $in |
| YtRangeDatePicker initialValue | `{ fieldName: { $between: [start, end] } }` | Date range → `$between` |
| YtUserSelect initialValue | `{ fieldName: { $in: ["value1", "value2", ...] } }` | User selector → `$in`，值必须是数组 |

> **YtUserSelect 查询规范**：业务字段存储可为数组，系统字段存储可为字符串；但在 `LrSmartFilter` 中，`YtUserSelect` 查询值必须是数组，如 `["80"]`，运行时只对数组值生成 `$in`。

### 示例

```json
{
  "componentName": "YtSingleSelect",
  "label": "客户状态",
  "name": "status",
  "initialValue": 1,
  "options": [{"label": "活跃", "value": 1}, {"label": "流失", "value": 0}]
}
```

> 首次加载时 API 请求 where 包含 `{ "status": { "$eq": 1 } }`。

---

## events 与上下文

- 宿主级事件、按钮 `optType + options` 协议：统一参考 `syntax-reference.md`
- Filter 场景只生成宿主级 `onSearch`，用于触发表格或其他消费 filter 接口的数据源刷新。
- Filter 没有 `fieldReactions`。当前运行时不消费筛选项字段级 `events.onChange` / `events.onMount`，也没有字段级 context 联动方法。Agent 不应生成筛选项显隐、选项更新和值联动类字段级 events。
- `LrSmartFilter` 不生成筛选项之间的实时动态联动。筛选项展示由静态 `items`、`hidden`、`isShown` 决定；筛选值变化后的数据刷新只通过宿主级 `onSearch` 完成。

---

## 关联字段引用

在 `name` 中使用点号路径访问关联表字段（底层通过 filter 接口的 where 条件实现）：

```json
{"name": "customer_level.level_name"}
{"name": "employee.employee_name"}
```

**限制**：仅支持一级关联（不支持 `a.b.c`）

---

## filter 接口协议

Filter 数据源请求遵循 filter 接口协议，LrSmartFilter 写入的 `ytfilter_xxx_filter` state 作为 `where` 参与请求体构建。完整协议见下。

### 请求体结构

> 以下为 filter 接口的完整请求体结构（供理解协议用）。各参数由不同组件属性控制，Agent 不直接构造此请求体。

```json
{
  "select": ["field1", "relation.field2"],
  "where": { "$and": [{ "field": { "$eq": value } }] },
  "orderBy": [{ "fieldName": "asc"|"desc" }],
  "currentPage": 1,
  "pageSize": 10
}
```

### where 筛选条件

**格式**：`where: { "$and": [...] }` 或 `where: { "$or": [...] }`，支持嵌套组合。字段支持主表字段或 `relation.field`。

**操作符列表**：

| 操作符 | 适用场景 | 示例（CRM 客户信息） |
|--------|----------|----------------------|
| `$eq` | 枚举、状态、精确匹配 | `{ status: { $eq: 1 } }` |
| `$ne` | 排除某个值 | `{ status: { $ne: 0 } }` |
| `$gte` | 数值/日期下界（含） | `{ create_time: { $gte: "2024-01-01" } }` |
| `$lte` | 数值/日期上界（含） | `{ create_time: { $lte: "2024-12-31" } }` |
| `$gt` | 数值/日期下界（不含） | `{ create_time: { $gt: "2024-01-01" } }` |
| `$lt` | 数值/日期上界（不含） | `{ create_time: { $lt: "2024-12-31" } }` |
| `$contain` | 文本模糊搜索 | `{ customer_name: { $contain: "阿里" } }` |
| `$startWith` | 前缀匹配 | `{ customer_name: { $startWith: "阿里" } }` |
| `$endWith` | 后缀匹配 | `{ customer_name: { $endWith: "集团" } }` |
| `$in` | 多选枚举、批量 ID | `{ source: { $in: ["官网", "推荐"] } }` |

**逻辑操作符**：

```json
// $and：多条件同时满足
{ "$and": [{ "status": { "$eq": 1 } }, { "source": { "$eq": "官网" } }] }

// $or：满足其中之一
{ "$or": [{ "status": { "$eq": 1 } }, { "status": { "$eq": 2 } }] }

// 嵌套：$and 内嵌 $or
{ "$and": [{ "status": { "$eq": 1 } }, { "$or": [{ "source": { "$eq": "官网" } }, { "source": { "$eq": "推荐" } }] }] }
```

**LrSmartFilter 贡献**：用户选择筛选项后，LrSmartFilter 写入 `ytfilter_xxx_filter` state，格式 `{ where: { $and: [...] } }`。

### 数据源 params → filter 请求体映射（系统内部机制，Agent 不可直接修改）

> 以下为系统内部同步机制说明。Agent 禁止直接修改 datasource params，只能通过组件属性间接控制（见"数据源唯一性原则"）。

| DataSource param | Source state | Filter field | Description |
|-----------------|--------------|--------------|-------------|
| filter | `ytfilter_xxx_filter` | where | Filter conditions from LrSmartFilter |
| sort | `yttable_xxx_sort` | orderBy | Sort config, format: `{ orderBy: [{ dataIndex: "ASC"\|"DESC" }] }` |
| select | Static array (aggregated from column dataIndex) | select | Return fields, main table or `relation.field` format |
| currentPage | `yttable_xxx_currentPage` | currentPage | Page number |
| pageSize | `yttable_xxx_pageSize` | pageSize | Items per page |

### 数据源唯一性原则

**机制**：filter 请求体的三个核心参数（select、where、orderBy）各有唯一数据源。Agent 只修改组件属性，不直接修改 datasource 配置，同步由系统自动完成。

| Parameter | Unique source | Sync method | Agent modification target |
|-----------|--------------|-------------|--------------------------|
| select | `LrSmartTable.columns[].isFetched` | Static sync: auto-rebuild `params.select` during schema merge | `columns[].isFetched` |
| where | `LrSmartFilter.items[].initialValue` | Dynamic sync: write to state on component init → `params.filter` | `items[].initialValue` |
| orderBy | `LrSmartTable.columns[].defaultSortOrder` | Dynamic sync: write to state on component init → `params.sort` | `columns[].defaultSortOrder` |

> 详见 `LrSmartTable.md` 的"数据源唯一性与自动同步"（select）和"orderBy 排序"（orderBy）章节。

### willFetch 合并逻辑（系统自动生成，Agent 禁止修改）

`({ filter, sort, ...rest }) => ({ ...rest, ...filter, ...sort })`。即 `rest`（select、currentPage、pageSize）+ `filter`（where）+ `sort`（orderBy）→ 最终请求体。

> 默认筛选条件请使用 `items[].initialValue`，默认排序请使用 `columns[].defaultSortOrder`，不可通过修改 willFetch 实现。

---

## Filter 联动 Table

Filter 搜索后触发 Table 刷新：

```json
{
  "componentName": "LrSmartFilter",
  "props": {
    "events": [{
      "name": "onSearch",
      "optType": "flowAction",
      "flowList": [{
        "optType": "request",
        "options": {
          "_bindDataSource": "<datasetId>_filter",
          "_autoBindDataSourceParams": true
        }
      }]
    }]
  }
}
```

Table 绑定数据源：

```json
{
  "componentName": "LrSmartTable",
  "props": {
    "dataSource": {"type": "JSExpression", "value": "state['<datasetId>_filter']"}
  }
}
```

---

## 定制快捷过滤按钮

> 此模式属于交互阶段（用户点击按钮后动态修改 where 条件）。初始化阶段的默认筛选条件必须使用 `items[].initialValue`，不可通过 setState 实现。

### 机制

快捷过滤按钮不直接改 Table，而是改写 Filter 对应 state 中的 `where.$and`，再触发 filter 数据源 `load()`。这样按钮与普通筛选共用同一套查询协议，改法稳定，且不需要改组件/引擎。

### 参数

| Parameter | Value source | Description |
|-----------|--------------|-------------|
| `filterStateKey` | `ytfilter_` + `meta.datasetId` + `_filter` | Filter state key |
| `dataSourceId` | `id` in `dataSource.list` filter request item | Used for `dataSourceMap[dataSourceId].load()` |
| `currentUserId` | `window.__GLOBAL__.userInfo.userId` | Current logged-in user, see syntax-reference.md § 6 |
| `fieldName` | Business field | e.g., `creator_id`, `assignee_id` |
| `excludeFields` | Current button field + mutually exclusive fields | Used to generate `baseConditions` |
| `buttonLabel` | Business label | e.g., "My created requirements", "My assigned requirements" |
| `conditionValue` | Determined by field storage structure | YtUserSelect fields uniformly use `$in` |

字段取值规则：

- YtUserSelect 筛选字段（无论业务字段还是系统字段）：`{ [fieldName]: { $in: [currentUserId] } }`

### 示例

LrSmartJsx 的 `render` 这里应写 `source`，不写编译后的 `value`。知识库中按多行 React 源码理解。

1. 从 `filterStateKey` 取出当前 `where.$and`
2. 用 `excludeFields` 过滤出 `baseConditions`
3. 判断当前按钮是否已激活
4. 未激活时追加当前按钮条件，已激活时回退到 `baseConditions`
5. `setState` 后调用 `dataSourceMap[dataSourceId].load()`

```jsx
import React from 'react';
import { Button } from 'antd';

function render(props) {
  const _context = props.env;
  const filterStateKey = '<filterStateKey>';
  const dataSourceId = '<dataSourceId>';
  const fieldName = '<fieldName>';
  const buttonLabel = '<buttonLabel>';
  const currentUserId = window.__GLOBAL__?.userInfo?.userId;
  const conditionValue = <conditionValue>;
  const whereAndConditions = _context.state?.[filterStateKey]?.where?.$and || [];
  const baseConditions = whereAndConditions.filter((conditionItem) => <excludePredicate>);
  const isActive = whereAndConditions.some((conditionItem) => <activePredicate>);

  const handleClick = () => {
    if (!currentUserId) return;
    const stateUpdate = {};
    stateUpdate[filterStateKey] = {
      where: {
        $and: isActive ? baseConditions : baseConditions.concat([{ [fieldName]: conditionValue }])
      }
    };
    _context.setState(stateUpdate);
    setTimeout(() => {
      _context.dataSourceMap?.[dataSourceId]?.load();
    }, 0);
  };

  return (
    <Button type={isActive ? 'primary' : 'default'} onClick={handleClick}>
      {buttonLabel}
    </Button>
  );
}
```

替换点：

| 占位 | 含义 |
|------|------|
| `filterStateKey` | Filter 对应的 state key |
| `dataSourceId` | filter 数据源 id |
| `fieldName` | 当前按钮控制的字段 |
| `buttonLabel` | 按钮文案 |
| `conditionValue` | 按“字段取值规则”替换 |
| `excludePredicate` | 排除 `excludeFields` |
| `activePredicate` | 判断“当前按钮条件已存在” |

落盘说明：真正写入配置时，把这段源码放进 `render.source`，并保持 `source` 字符串 + `\n` 换行转义。`source` 组件拿到的是 `props`，页面上下文在 `props.env`。

改法：

- 换 `filterStateKey`
- 换 `dataSourceId`
- 换 `fieldName`
- 换 `excludeFields`
- 换 `buttonLabel`
- 换 `conditionValue` / `activePredicate`

---

## 常见场景示例

### 条件显隐

`LrSmartFilter` 目前不支持筛选项之间的实时字段联动。`hidden` / `isShown` 只作为静态渲染控制：`hidden: true` 或 `isShown: false` 的筛选项不会渲染，也不会参与 where 条件。

Agent 规则：

- 不为 `LrSmartFilter.items[]` 生成字段级 `events.onChange` / `events.onMount`。
- 不生成 `fieldReactions`，该能力只用于 Form。
- 需要搜索时生成宿主级 `events.onSearch`，让 filter state 刷新后触发表格数据源请求。

---

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 筛选项不显示 | `hidden: true` 或 `isShown: false` | 设置静态 `isShown: true`，不要生成字段级联动 |
| 选项列表为空 | options 未配置 | 配置静态 options，或按标准方案配置 Page 层 getSelectOptions + state 绑定 |
| 联动不生效 | 字段级 events 未被消费 | 不生成 Filter 字段级联动 schema |
| 远程搜索不生效 | filterOption 未禁用 | 设置 `filterOption: false` |
| 默认值不生效 | initialValue 未配置 | 在对应 item 上配置 `initialValue` |

---

## 完整示例

```json
{
  "componentName": "LrSmartFilter",
  "props": {
    "_doVersion": "V2",
    "items": [
      {"componentName": "YtInput", "label": "客户名称", "name": "customer_name", "placeholder": "请输入"},
      {"componentName": "YtSingleSelect", "label": "客户状态", "name": "status", "initialValue": 1, "options": [{"label": "活跃", "value": 1}, {"label": "流失", "value": 0}]},
      {"componentName": "YtRangeDatePicker", "label": "创建时间", "name": "create_time_range"}
    ],
    "events": [{"name": "onSearch", "optType": "flowAction", "flowList": [{"optType": "request", "options": {"_bindDataSource": "<datasetId>_filter", "_autoBindDataSourceParams": true}}]}]
  }
}
```
