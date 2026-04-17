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
| fieldReactions | string | 字段联动规则（函数字符串形式） |

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
| `hidden: true` | ❌ | ✅ | 隐藏但仍提交值（antd Form.Item 规范） |

| 属性 | 类型 | 说明 |
|------|------|------|
| name | string | 字段名（仅主表字段，不支持点号字段） |
| label | string | 标签 |
| indexType | string | 索引类型（`primary` / `unique` / `index`），系统自动生成，无需手动设置 |
| componentName | string | 组件类型 |
| formSpan | string | 占宽：`full` / `half` |
| initialValue | any | 默认值（与 antd Form.Item initialValue 一致） |
| rules | Rule[] | 校验规则 |
| options | array | 静态选项 |
| events | object | 事件配置 |
| readonly | boolean | 只读（Detail 默认 true） |

### 组件类型

表单项可用的 `componentName`、平台专属组件、数据格式与动态选项约束，统一参考 `field-components.md`。本文档仅补充 Form 场景特有的模式、校验、提交与联动行为。

### 校验规则

| 规则 | 示例 |
|------|------|
| 必填 | `{required: true, message: "不能为空"}` |
| 手机号 | `{type: "phone", message: "手机号格式错误"}` |
| 邮箱 | `{type: "email", message: "邮箱格式错误"}` |
| 正则 | `{type: "regex", pattern: "^\\d{11}$", message: "格式错误"}` |

---

## events 与上下文

- 事件签名、通用 `context` 方法、按钮 `optType + options` 协议：统一参考 `syntax-reference.md`
- `form` 方法属于表单组件特有能力，集中在本页维护
- Form 场景分工：
  - 字段值读写使用 `form`
  - 字段显隐、禁用、规则、选项等状态控制使用 `context`

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

### 字段联动（fieldReactions）

**推荐方式**：使用 `fieldReactions` 实现多字段联动，相比单个字段的 onChange 事件更简洁高效。

- **机制**：表单值变化时自动执行联动函数，返回各字段的属性覆盖映射
- **优势**：集中管理联动逻辑，避免在多个字段的 onChange 中重复代码
- **支持属性**：`disabled`、`required`、`rules`、`isShown`、`hidden` 等所有 Item 属性

**函数签名**：

```javascript
({ changedValues, allValues, form }) => {
  // 返回对象，key 为字段名，value 为要覆盖的属性
  return {
    fieldName: { disabled: true, required: false, isShown: true, ... }
  }
}
```

**示例 1：状态联动必填规则**

商机表单中，当状态为"已赢单"时，产品相关字段变为必填；当状态为"已输单"时，隐藏预计成交日期。

```json
{
  "componentName": "LrSmartCreate",
  "props": {
    "fieldReactions": "({ allValues }) => { const isWon = allValues.status === 2; return { product_id: { disabled: false, required: isWon, rules: isWon ? [{ type: 'required', message: '请输入产品ID' }] : [] }, product_name: { disabled: false, required: isWon, rules: isWon ? [{ type: 'required', message: '请输入产品名称' }] : [] }, product_num: { disabled: false, required: isWon, rules: isWon ? [{ type: 'required', message: '请输入产品数量' }] : [] }, expected_date: { isShown: allValues.status !== 3 } }; }",
    "items": [
      {
        "name": "status",
        "componentName": "YtSingleSelect",
        "label": "状态",
        "options": [
          {"label": "进行中", "value": 1},
          {"label": "已赢单", "value": 2},
          {"label": "已输单", "value": 3}
        ]
      },
      {"name": "product_id", "componentName": "YtInputNumber", "label": "产品ID"},
      {"name": "product_name", "componentName": "YtInput", "label": "产品名称"},
      {"name": "product_num", "componentName": "YtInputNumber", "label": "产品数量"},
      {"name": "expected_date", "componentName": "YtDatePicker", "label": "预计成交日期"}
    ]
  }
}
```

**示例 2：客户类型联动显示**

根据客户类型显示不同字段：企业客户显示等级，个人客户显示联系人。

```json
{
  "fieldReactions": "({ allValues }) => { const isEnterprise = allValues.customer_type === '企业'; return { level_id: { isShown: isEnterprise }, contact_id: { isShown: !isEnterprise } }; }"
}
```

**注意事项**：

- fieldReactions 中的函数以字符串形式配置，运行时通过 `new Function` 解析执行
- 返回的属性会覆盖 items 中的原始配置
- 联动逻辑应保持纯函数，避免副作用
- 复杂联动建议使用 fieldReactions，简单的单字段联动可使用 onChange 事件

### 条件显隐（onChange 方式）

通过 onChange 事件调用 `context.setFieldVisible()` 控制其他字段显隐。适用于简单的单字段联动场景。

```json
{
  "name": "customer_type",
  "componentName": "YtSingleSelect",
  "options": [{"label": "企业", "value": "企业"}, {"label": "个人", "value": "个人"}],
  "events": {
    "onChange": {
      "type": "JSFunction",
      "value": "async (value, form, context) => {\n  const isEnterprise = value === '企业';\n  context.setFieldVisible('level_id', isEnterprise);\n  context.setFieldVisible('contact_id', !isEnterprise);\n}"
    }
  }
}
```

### 动态选项加载

- **机制**：options 数据来自关联表，需两步配置：Page 层请求数据 → formItem 层绑定 state。state key 必须与 dataSource id 一致，确保数据正确传递
- **参数**：

| Placeholder | Value Source | Description |
|-------------|--------------|-------------|
| `<fieldName>_options` | Field name + `_options` suffix | dataSource id, must match state key |
| `<appCode>` | Application code | From current app config |
| `<datasetCode>` | Related dataset code | From dataset relations |
| `<valueField>` | Related table value field | Usually `id` or `code` |
| `<labelField>` | Related table display field | Usually `name` or `title` |

- **示例**：

```json
// Page.dataSource.list
{
  "type": "fetch",
  "id": "level_id_options",
  "isInit": true,
  "options": {
    "uri": "/api/<appCode>/<datasetCode>/getSelectOptions",
    "params": {"code": "<valueField>", "label": "<labelField>"}
  }
}

// formItem
{
  "name": "level_id",
  "componentName": "YtSingleSelect",
  "options": {
    "type": "JSExpression",
    "value": "this.state.level_id_options"
  }
}
```

> `formItem.optionsRequest` 配置 `{url, dataProcess}` 后组件自动请求并填充 options，详见 `field-components.md` § options 数据获取方式。

---

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 表单不显示 | items 为空 | 配置 items |
| 联动不生效 | onChange 配置错误或 fieldReactions 语法错误 | 使用 JSFunction 或检查 fieldReactions 函数语法 |
| fieldReactions 不执行 | 函数字符串格式错误 | 确保返回对象格式正确，检查控制台错误 |
| 联动后字段状态不更新 | 返回的字段名拼写错误 | 确保返回对象的 key 与 item.name 完全一致 |
| 下拉选项为空 | options 未配置，或 Page 层未配置 getSelectOptions 请求 | 配置静态 options，或按标准方案配置 Page 层 dataSource + state 绑定 |
| 提交失败 | 必填字段未填 | 检查 rules 或 fieldReactions 中的 required 配置 |

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
