---
name: LrSmartCreate
description: LrSmartCreate/LrSmartUpdate/LrSmartDetail 表单组件完整配置
tags: [form, page-schema]
---

# LrSmartCreate / LrSmartUpdate / LrSmartDetail

表单组件，用于数据录入/编辑/详情展示。

| 组件 | 用途 | 差异 |
|------|------|------|
| LrSmartCreate | 新建表单 | 默认空表单 |
| LrSmartUpdate | 编辑表单 | 加载已有数据 |
| LrSmartDetail | 详情页 | 所有字段只读 |

**配置结构相似度**: 90%+

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
| items | Item[] | 表单项配置 |
| layoutMode | string | 布局：`multiple` / `inline` |
| cols | number | 每行字段数（multiple 模式） |
| valuesKey | string | 数据加载状态 key（Update/Detail） |
| fieldReactions | JSFunction（PageSchema 写法） | 表单型组件字段联动函数，返回“字段名 → 字段属性覆盖”对象 |

---

## Item 表单项属性

V2 页面生成时已为数据集所有字段全量生成 items，通过 `isShown` 控制显示。用户"添加表单项" = 将已有 item 的 `isShown` 设为 `true`；仅当字段在 items 中完全不存在时才需插入新对象。

### INSERT element 最小结构

arrayName = `formItems`，字段必须存在于数据集（先 dataset_detail_tool 确认）。工具层自动默认 `readonly: true`，仅当用户明确要求可编辑时设 `readonly: false`。

```json
{"name": "<fieldName>", "label": "<fieldLabel>", "componentName": "<组件类型>", "isShown": true}
```

> 组件类型参考 `field-components.md`。选择类组件需同时配置 options（静态枚举或动态数据源），参见下方「动态选项加载」。

**能力边界**：

- ✅ 支持：显示数据集中已有但表单未展示的字段（将 `isShown: false` 改为 `true`，或在 items 中新增引用）
- ✅ 支持：修改已有字段的展示方式（组件类型、校验规则、标签等）
- ❌ 不支持：添加数据集中不存在的字段（需要持久化的字段必须先在 Lovrabet 平台的数据集管理中创建）
- 原因：表单提交的字段必须对应数据集中的实际字段，否则提交会失败

| 属性 | 渲染 | 参与提交 | 用途 |
|------|:----:|:-------:|------|
| `isShown: false` | ❌ | ❌ | 完全不渲染，等同于不存在 |
| `hidden: true` | ❌ | ✅ | 隐藏但仍提交值（Ant Design 5 Form.Item 规范） |

| 属性 | 类型 | 说明 |
|------|------|------|
| name | string | 字段名（仅主表字段，不支持点号字段） |
| label | string | 标签 |
| indexType | string | 索引类型（`primary` / `unique` / `index`），系统自动生成，无需手动设置 |
| componentName | string | 组件类型 |
| formSpan | string | 占宽：`full` / `half` |
| initialValue | any | 默认值（与 Ant Design 5 Form.Item initialValue 一致） |
| rules | Rule[] | 校验规则 |
| options | array | 静态选项 |
| events | object | 禁止默认生成；当前运行时不消费字段级 `events`，字段联动请使用 Form 顶层 `fieldReactions` |
| readonly | boolean | 只读（Detail 默认 true） |

### 组件类型

表单项可用的 `componentName`、字段组件选型规则、数据格式与动态选项约束，统一参考 `field-components.md`。本文档仅补充 Form 场景特有的模式、校验、提交与联动行为。

### 校验规则

| 规则 | 示例 |
|------|------|
| 必填 | `{required: true, message: "不能为空"}` |
| 手机号 | `{type: "phone", message: "手机号格式错误"}` |
| 邮箱 | `{type: "email", message: "邮箱格式错误"}` |
| 正则 | `{type: "regex", pattern: "^\\d{11}$", message: "格式错误"}` |

---

## events 与上下文

- 宿主级事件、按钮 `optType + options` 协议：统一参考 `syntax-reference.md`
- 当前运行时不消费字段级 `events.onChange` / `events.onMount`，Agent 不应新增此类字段级事件 schema
- 表单型组件字段联动使用组件 props 顶层 `fieldReactions`，不要写在 `items[]` 内
- `form` 方法属于表单组件特有能力，集中在本页维护

### fieldReactions（字段联动）

`fieldReactions` 是表单型组件字段联动的默认方式，写在组件 props 顶层，用于集中描述多字段之间的显隐、禁用、校验和选项覆盖逻辑。函数返回一个“字段名 → 字段属性覆盖”的对象，运行时会把返回值按字段 `name` 合并到对应 item 上。当前适用于 `LrSmartCreate` / `LrSmartUpdate`；`LrSmartDetail` 如需展示控制也遵循同一机制。

Agent 规则：

- 联动触发源字段不用写事件；表单值变化后运行时会重新执行 `fieldReactions`。
- 返回对象的 key 必须是被联动字段的 `name`。
- value 是要覆盖的 item 属性，常见为 `isShown`、`hidden`、`disabled`、`rules`、`options`。
- `isShown: false` 表示不渲染该字段；`hidden: true` 表示隐藏但保留在 Form 中。
- 读取值优先使用 `allValues`；不要用 `changedValues` 判断本次变化字段，因为当前运行时 `changedValues` 与 `allValues` 同值。
- 只有需要命令式读写值时才使用 `form`。
- 联动函数应保持纯函数：根据 `allValues` 返回覆盖对象，不在函数内发请求、改 schema 或写页面状态。
- PageSchema 中统一生成 `{"type": "JSFunction", "value": "..."}`，不要生成裸函数字符串。

### form（管理字段值）

| 方法 | 说明 |
|------|------|
| `form.getFieldValue(name)` | 获取字段值 |
| `form.setFieldValue(name, value)` | 设置字段值 |
| `form.setFieldsValue({...})` | 批量设置 |
| `form.resetFields([names])` | 重置字段 |

---

## 提交按钮配置

按钮协议本身是平台通用语法；Form 场景的关键点是 `submit` 动作需要同时绑定表单与数据源。

```json
{
  "title": "提交",
  "type": "primary",
  "optType": "flowAction",
  "options": {
    "flowList": [{
      "optType": "submit",
      "options": {
        "_bindForm": "form_unique_id",
        "_bindDataSource": "<datasetId>_create",
        "_autoBindDataSourceParams": true
      }
    }]
  }
}
```

| 操作 | `_bindDataSource` |
|------|-------------------|
| 新建 | `<datasetId>_create` |
| 更新 | `<datasetId>_update` |

### 多步流程（提交后刷新 + 跳转）

- **机制**：`flowList` 数组中的动作按顺序串行执行。提交成功后常需刷新列表页数据源（跨页面通过 `__rendererInstanceRegistry` 访问）+ 跳转到结果页
- **参数**：

| Placeholder | Value Source | Description |
|-------------|--------------|-------------|
| `<form_uniqueId>` | `props._uniqueId` | Form unique identifier |
| `<datasetId>_update` | Dataset id + operation suffix | Use `_create` for create, `_update` for update |
| `<datasetId>_filter` | List page filter dataSource id | For cross-page list refresh |
| `pageTagName` | Page identifier | `RESULT` = detail page, `UPDATE` = edit page |
| `dataid` | `_context.state.urlParams.dataid` | Current record id |

- **示例**：

```json
{
  "title": "提交",
  "type": "primary",
  "optType": "flowAction",
  "options": {
    "flowList": [
      {
        "optType": "submit",
        "options": {
          "_bindForm": "<form_uniqueId>",
          "_bindDataSource": "<datasetId>_update",
          "_autoBindDataSourceParams": true,
          "responseConfig": {
            "enableSuccessToast": true,
            "successToastMsg": "更新成功！"
          }
        }
      },
      {
        "optType": "jsAction",
        "options": {
          "jsFunction": {
            "type": "JSFunction",
            "value": "(config) => {\n  window.__rendererInstanceRegistry?.instances.values().forEach((item) => item.instance.dataSourceMap['<datasetId>_filter']?.load());\n}"
          }
        }
      },
      {
        "optType": "jsAction",
        "options": {
          "jsFunction": {
            "type": "JSFunction",
            "value": "(config) => {\n  const { _context } = config;\n  const dataid = _context?.state?.urlParams?.dataid;\n  _context?.appHelper?.yt?.smartPageHistory?.push({ pageTagName: 'RESULT', datasetId: '<datasetId>', dataid: dataid });\n}"
          }
        }
      }
    ]
  }
}
```

---

## 常见场景

### 条件显隐与禁用

通过 `fieldReactions` 根据当前表单值覆盖字段属性，控制其他字段显隐或禁用。不要生成字段级 `events` 联动 schema。

`fieldReactions` 是函数型字段。Agent 生成 PageSchema 时使用 `JSFunction`。函数入参使用 `{ allValues, form }`：`allValues` 用于读取完整表单值，`form` 用于命令式读写表单。

```json
{
  "fieldReactions": {
    "type": "JSFunction",
    "value": "({ allValues }) => {\n  const isEnterprise = allValues.customer_type === 'enterprise';\n  return {\n    company_name: { isShown: isEnterprise },\n    personal_name: { isShown: !isEnterprise },\n    tax_no: { disabled: !isEnterprise }\n  };\n}"
  },
  "items": [
    {"name": "customer_type", "label": "客户类型", "componentName": "YtSingleSelect", "options": [{"label": "企业", "value": "enterprise"}, {"label": "个人", "value": "personal"}]},
    {"name": "company_name", "label": "企业名称", "componentName": "YtInput", "isShown": false},
    {"name": "personal_name", "label": "个人姓名", "componentName": "YtInput", "isShown": true},
    {"name": "tax_no", "label": "税号", "componentName": "YtInput"}
  ]
}
```

### 动态选项加载

- **机制**：options 数据来自关联表，需两步配置：Page 层请求 To 表 options 数据 → formItem 层绑定 state。state key 必须与 dataSource id 一致，确保数据正确传递
- **参数**：

| Placeholder | Value Source | Description |
|-------------|--------------|-------------|
| `<FromDatasetCode>_<FromRelationField>_options` | From 表字段所属 datasetCode + From 表关联字段名 + `_options` suffix | dataSource id, must match state key |
| `<appCode>` | Application code | From current app config |
| `<ToDatasetCode>` | To 表 datasetCode | From dataset relations |
| `<ToValueField>` | To 表被关联字段 | Usually `id` or `code` |
| `<ToLabelField>` | To 表展示字段 | Usually `name` / `title` / location field |

> 自关联父子关系中，`FromDatasetCode` 与 `ToDatasetCode` 可以相同；state key 仍按 `<FromDatasetCode>_<FromRelationField>_options` 命名，URI 仍使用 To 侧 options 数据源。

- **示例**：

```json
// Page.dataSource.list
{
  "type": "fetch",
  "id": "<FromDatasetCode>_<FromRelationField>_options",
  "isInit": true,
  "source": "dataset",
  "options": {
    "uri": "/api/<appCode>/<ToDatasetCode>/getSelectOptions",
    "params": {"code": "<ToValueField>", "label": "<ToLabelField>"}
  }
}

// formItem
{
  "name": "<FromRelationField>",
  "componentName": "YtSingleSelect",
  "options": {
    "type": "JSExpression",
    "value": "this.state.<FromDatasetCode>_<FromRelationField>_options"
  }
}
```

> `getSelectOptions` 禁止使用 `optionsRequest`。即使字段级请求技术上能访问完整接口，也会把表单字段耦合到具体域名或部署环境；Agent 必须使用 Page 层 `dataSource.list` 的相对 URI，并通过 `this.state.<FromDatasetCode>_<FromRelationField>_options` 绑定。`optionsRequest` 仅用于三方链接、外部接口或外部静态资源 options。

---

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 表单不显示 | items 为空 | 配置 items |
| 联动不生效 | 使用了字段级 events/onChange | 表单型组件字段联动使用 `fieldReactions`，不要生成字段级 events 联动 schema |
| 下拉选项为空 | options 未配置，或 Page 层未配置 getSelectOptions 请求 | 配置静态 options，或按标准方案配置 Page 层 dataSource + state 绑定 |
| 提交失败 | 必填字段未填 | 检查 rules |

---

## 完整示例

```json
{
  "componentName": "LrSmartCreate",
  "props": {
    "_doVersion": "V2",
    "layoutMode": "multiple",
    "cols": 2,
    "items": [
      {"name": "customer_name", "componentName": "YtInput", "label": "客户名称", "formSpan": "full", "rules": [{"required": true, "message": "不能为空"}]},
      {"name": "level_id", "componentName": "YtSingleSelect", "label": "客户等级", "options": [{"label": "A类", "value": "A"}, {"label": "B类", "value": "B"}]},
      {"name": "status", "componentName": "YtSingleSelect", "label": "状态", "initialValue": 1, "options": [{"label": "活跃", "value": 1}, {"label": "流失", "value": 0}]}
    ]
  }
}
```
