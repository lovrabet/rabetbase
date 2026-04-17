---
name: field-components
description: Filter/Form 字段 componentName 速查及平台扩展属性。先验知识：低代码引擎协议、antd 5+组件知识。
tags: [filter, form, field-components, page-schema]
---

# 字段 componentName 速查

> **先验知识**：GLM-5 已知 antd 5+ 组件属性。本文档仅记录平台扩展属性。
>
> `LrSmartFilter` 与 `LrSmartCreate` / `LrSmartUpdate` / `LrSmartDetail` 共享部分字段组件能力，但并非完全等同。Filter 偏查询条件组件，Form 偏录入/编辑组件；具体支持范围以各组件文档为准。antd 原生 props 可直接用于 `extendProps`。


## 按宿主组件选型

### LrSmartFilter 常用字段组件

| 场景 | 推荐 componentName | 说明 |
|------|--------------------|------|
| 文本查询 | `YtInput` | 模糊搜索、关键字查询 |
| 单选筛选 | `YtSingleSelect` | 枚举值或关联表单选 |
| 多选筛选 | `YtMultiSelect` | 标签、多状态并集筛选 |
| 小选项集单选 | `YtRadioGroup` | 选项较少时更直观 |
| 小选项集多选 | `YtCheckboxGroup` | 选项较少的多选 |
| 单日期 | `YtDatePicker` | 精确到某一天的筛选 |
| 日期范围 | `YtRangeDatePicker` | 时间区间筛选的主力组件 |
| 数字单值 | `YtInputNumber` | 单个数值条件 |
| 数字范围 | `YtRangeInputNumber` | 区间数值筛选 |
| 级联选择 | `YtCascader` | 层级分类筛选 |
| 用户筛选 | `YtUserSelect` | 平台用户相关筛选 |
| 地址筛选 | `YtAddressPicker` | 地址维度筛选 |

**不推荐/不适用**：
- `YtRichEditor`：不用于筛选场景
- `YtUpload`：不用于筛选场景
- `YtTextArea`：一般不作为筛选主组件
- `YtSwitch`：仅在极少数布尔筛选场景使用，通常优先用选择类组件表达

### LrSmartCreate / LrSmartUpdate / LrSmartDetail 常用字段组件

| 场景 | 推荐 componentName | 说明 |
|------|--------------------|------|
| 单行录入 | `YtInput` | 名称、编号、邮箱、电话等 |
| 数字录入 | `YtInputNumber` | 金额、数量、价格等 |
| 多行录入 | `YtTextArea` | 备注、说明、简短描述 |
| 富文本录入 | `YtRichEditor` | 详细内容、正文、复杂说明 |
| 单选录入 | `YtSingleSelect` | 枚举值或关联表单选 |
| 多选录入 | `YtMultiSelect` | 标签、多选枚举 |
| 小选项集单选 | `YtRadioGroup` | 少量单选项 |
| 小选项集多选 | `YtCheckboxGroup` | 少量多选项 |
| 布尔录入 | `YtSwitch` | 是否启用、是否标品等 |
| 单日期 | `YtDatePicker` | 日期/时间选择 |
| 级联选择 | `YtCascader` | 层级分类录入 |
| 文件上传 | `YtUpload` | 附件、图片、材料上传 |
| 用户选择 | `YtUserSelect` | 平台用户选择 |
| 地址录入 | `YtAddressPicker` | 地址信息录入 |

**按场景谨慎使用**：
- `YtRangeDatePicker`：更偏筛选语义，Form 中仅在明确需要录入时间区间时使用
- `YtRangeInputNumber`：更偏筛选语义，Form 中通常不作为常规录入组件

自动加载系统用户列表，值格式与存储格式一致：

- 业务字段（YtUserSelect 录入）：数组 `["80"]` / `["80","81"]`
- 系统字段（creator_id / updater_id）：字符串 `"80"`
- 查询时统一使用 `$in`，服务端兼容两种存储格式

```json
{
  "name": "employee_id",
  "componentName": "YtUserSelect",
  "label": "负责人"
}
```

### YtAddressPicker

自动加载省市区数据

```json
{
  "name": "address",
  "componentName": "YtAddressPicker",
  "label": "联系地址"
}
```

---

## 数据格式速查

| 组件 | 类型 | 初始值 | 用户输入 | 清空值 | 示例 |
|------|------|--------|---------|--------|------|
| YtInput | string | `""` / `null` | `"hello"` | `""` / `null` | `"客户名称"` |
| YtInputNumber | number | `null` | `123` | `null` | `100.50` |
| YtTextArea | string | `""` / `null` | `"多行文本"` | `""` / `null` | 备注内容 |
| YtRichEditor | string | `""` / `null` | `"<p>...</p>"` | `""` / `null` | HTML 字符串 |
| YtSingleSelect | string/number | `null` / `undefined` | 选中值 | `undefined` | `1` / `"vip"` |
| YtMultiSelect | array | `[]` | `[value1, value2]` | `[]` | `[1, 2, 3]` |
| YtRadioGroup | string/number | `null` | 选中值 | `null` | `"enterprise"` |
| YtCheckboxGroup | array | `[]` | `[value1, value2]` | `[]` | `["phone", "email"]` |
| YtSwitch | boolean | `false` | `true` / `false` | `false` | `true` |
| YtDatePicker | string | `null` | `"2026-03-09"` | `null` | `"2026-03-09"` |
| YtRangeDatePicker | array | `[]` / `null` | `["start", "end"]` | `[]` / `null` | `["2026-01-01", "2026-03-09"]` |
| YtCascader | array | `[]` | `[level1, level2, ...]` | `[]` | `["浙江省", "杭州市", "西湖区"]` |
| YtUpload | array | `[]` | `[{uid, name, url, ...}]` | `[]` | 文件对象数组 |
| YtUserSelect | array / string | `null` | `["80"]` / `["80","81"]` | `null` | 业务字段数组 `["80"]`，系统字段字符串 `"80"` |
| YtAddressPicker | array | `[]` | `[省, 市, 区]` | `[]` | `["浙江省", "杭州市", "西湖区"]` |

### 关键格式说明

- **YtSingleSelect**：初始值 `null` / `undefined`（不是空字符串）
- **YtMultiSelect**：始终是数组，清空为 `[]`
- **YtRangeDatePicker**：返回 `[start, end]` 数组
- **YtUserSelect**：业务字段为数组 `["80"]` / `["80","81"]`，系统字段为字符串 `"80"`。查询时统一使用 `$in`

---

## 基础示例

```json
// YtInput
{"name": "customer_name", "componentName": "YtInput", "label": "客户名称", "extendProps": {"maxLength": 100}}

// YtInputNumber
{"name": "amount", "componentName": "YtInputNumber", "label": "金额", "extendProps": {"min": 0, "precision": 2}}

// YtTextArea
{"name": "remark", "componentName": "YtTextArea", "label": "备注", "extendProps": {"rows": 4, "maxLength": 500}}

// YtSingleSelect（静态选项）
{"name": "status", "componentName": "YtSingleSelect", "label": "状态", "options": [{"label": "活跃", "value": 1}, {"label": "流失", "value": 0}]}

// YtSingleSelect（动态选项 - 当前标准方案：Page 层 getSelectOptions + state 绑定）
{"name": "level_id", "componentName": "YtSingleSelect", "label": "客户等级", "options": {"type": "JSExpression", "value": "this.state.level_id_options"}}

// YtMultiSelect
{"name": "tags", "componentName": "YtMultiSelect", "label": "标签", "options": [{"label": "重点", "value": "key"}, {"label": "潜在", "value": "potential"}]}

// YtSwitch
{"name": "is_active", "componentName": "YtSwitch", "label": "是否启用"}

// YtDatePicker（仅日期；placeholder 用表单项顶层，勿放 extendProps）
{"name": "expected_date", "componentName": "YtDatePicker", "label": "预计日期", "placeholder": "请选择日期", "extendProps": {"format": "YYYY-MM-DD", "picker": "date"}}

// YtDatePicker（日期+时间，用于 datetime 类型字段）
{"name": "created_at", "componentName": "YtDatePicker", "label": "创建时间", "extendProps": {"showTime": {"format": "HH:mm"}, "format": "YYYY-MM-DD HH:mm", "picker": "date"}}

// YtRangeDatePicker
{"name": "visit_time", "componentName": "YtRangeDatePicker", "label": "拜访时间"}

// YtUserSelect
{"name": "employee_id", "componentName": "YtUserSelect", "label": "负责人"}
```

## YtDatePicker 时间配置

### 机制

`showTime` 决定是否启用时间选择，`format` 控制显示格式，`picker` 指定基础选择器类型。datetime 类型字段需要 `showTime` + `format: "YYYY-MM-DD HH:mm"`，date 类型字段无需 `showTime`。

### 参数

| 属性 | 类型 | 说明 |
|------|------|------|
| showTime | object \| boolean | 启用时间选择，`{format: "HH:mm"}` 控制时间格式 |
| format | string | 日期时间显示格式，datetime 用 `"YYYY-MM-DD HH:mm"`，date 用 `"YYYY-MM-DD"` |
| picker | string | 基础选择器类型：`date` / `week` / `month` / `quarter` / `year` |

### 配置位置

`picker` / `format` / `showTime` 写在 `extendProps`；`placeholder` 写在表单项顶层，禁止放入 `extendProps`。

## 分层设计原则

| 层级 | 职责 |
|------|------|
| **Page 层** | dataSource.list 配置数据请求 |
| **formItem 层** | 绑定数据、字段元数据 |
| **组件层** | 消费数据、渲染 |

### 属性位置规则

| 写在 formItem 顶层 | 写在 extendProps |
|-------------------|-----------------|
| placeholder, options, disabled, readonly | precision, min, max, format, showTime, picker, 其他 antd 控件原生 props |

禁止在 extendProps 中重复写顶层已有的属性。

### options 数据获取方式

| 方式 | 配置位置 | 说明 |
|------|---------|------|
| **Page 层预配置** | `dataSource.list` | 配置 `getSelectOptions` 请求，formItem 层通过 state 绑定 |
| **formItem 层定制** | formItem.`optionsRequest` | 配置 `{url, dataProcess}`，组件自动请求并填充 options |

```json
// ✅ 方式1：Page 层预配置 getSelectOptions
// Page.dataSource.list 中配置
{
  "type": "fetch",
  "id": "customer_id_options",
  "isInit": true,
  "options": {
    "uri": "/api/<appCode>/<datasetCode>/getSelectOptions",
    "params": {"code": "id", "label": "cust_name"}
  }
}

// formItem 层绑定 state
{
  "name": "customer_id",
  "componentName": "YtSingleSelect",
  "label": "客户名称",
  "options": {
    "type": "JSExpression",
    "value": "this.state.customer_id_options"
  }
}

// ✅ 方式2：formItem 层 optionsRequest
{
  "name": "level_id",
  "componentName": "YtSingleSelect",
  "label": "客户等级",
  "optionsRequest": {
    "url": "/api/<appCode>/<datasetCode>/getSelectOptions?code=id&label=level_name"
  }
}

// ❌ 错误：optionsRequest 放在组件层（extendProps）
{
  "name": "level_id",
  "componentName": "YtSingleSelect",
  "extendProps": {"optionsRequest": {...}}  // 错误位置，应放 formItem 顶层
}
```

---

## 平台专属组件

### YtUserSelect

---
