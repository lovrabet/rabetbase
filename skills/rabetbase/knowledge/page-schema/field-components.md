---
name: field-components
description: Filter/Form 字段 componentName 速查及平台扩展属性。先验知识：低代码引擎搭建协议、Ant Design 5 组件知识。
tags: [filter, form, field-components, page-schema]
---

# 字段 componentName 速查

> **先验知识**：默认已具备低代码引擎搭建协议和 Ant Design 5 组件知识。本文档仅记录平台扩展属性。
>
> `LrSmartFilter` 与 `LrSmartCreate` / `LrSmartUpdate` / `LrSmartDetail` 共享部分字段组件能力，但并非完全等同。Filter 偏查询条件组件，Form 偏录入/编辑组件；具体支持范围以各组件文档为准。Ant Design 5 原生 props 可直接用于 `extendProps`。


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
| 代码录入 | `YtCodeEditor` | JSON、JavaScript、TypeScript、SQL、CSS、HTML、Markdown 等代码/配置文本 |
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

自动加载系统用户列表，值格式需按场景区分：

- 业务字段（YtUserSelect 录入）：数组 `["80"]` / `["80","81"]`
- 系统字段（creator_id / updater_id）：字符串 `"80"`
- Filter 查询场景必须使用数组值，如 `["80"]`，运行时只对数组生成 `$in`

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
| YtCodeEditor | string | `""` / `null` | 代码/配置文本 | `""` | JSON / JS / SQL 字符串 |
| YtSingleSelect | string/number | `null` / `undefined` | 选中值 | `undefined` | `1` / `"vip"` |
| YtMultiSelect | array | `[]` | `[value1, value2]` | `[]` | `[1, 2, 3]` |
| YtRadioGroup | string/number | `null` | 选中值 | `null` | `"enterprise"` |
| YtCheckboxGroup | array | `[]` | `[value1, value2]` | `[]` | `["phone", "email"]` |
| YtSwitch | boolean | `false` | `true` / `false` | `false` | `true` |
| YtDatePicker | number | `null` | 时间戳 | `null` | `1773014400000` |
| YtRangeDatePicker | number[] | `[]` / `null` | `[startTimestamp, endTimestamp]` | `[]` / `null` | `[1767225600000, 1773014400000]` |
| YtCascader | array | `[]` | `[level1, level2, ...]` | `[]` | `["浙江省", "杭州市", "西湖区"]` |
| YtUpload | array | `[]` | `[{uid, name, url, ...}]` | `[]` | 文件对象数组 |
| YtUserSelect | array / string | `null` | `["80"]` / `["80","81"]` | `null` | 业务字段数组 `["80"]`，系统字段字符串 `"80"` |
| YtAddressPicker | array | `[]` | `[省, 市, 区]` | `[]` | `["浙江省", "杭州市", "西湖区"]` |

### 关键格式说明

- **YtSingleSelect**：初始值 `null` / `undefined`（不是空字符串）
- **YtMultiSelect**：始终是数组，清空为 `[]`
- **YtDatePicker**：运行时提交值为时间戳 number；`extendProps.format` 只控制展示格式
- **YtRangeDatePicker**：运行时提交值为时间戳数组 `[startTimestamp, endTimestamp]`
- **YtUserSelect**：业务字段为数组 `["80"]` / `["80","81"]`，系统字段存储可为字符串 `"80"`；Filter 查询值必须是数组
- **YtCodeEditor**：只用于 Form 录入代码/配置文本，值为 string；Agent 必须显式生成 `language`，不要依赖默认值

---

## 基础示例

```json
// YtInput
{"name": "customer_name", "componentName": "YtInput", "label": "客户名称", "extendProps": {"maxLength": 100}}

// YtInputNumber
{"name": "amount", "componentName": "YtInputNumber", "label": "金额", "extendProps": {"min": 0, "precision": 2}}

// YtTextArea
{"name": "remark", "componentName": "YtTextArea", "label": "备注", "extendProps": {"rows": 4, "maxLength": 500}}

// YtCodeEditor（Form 场景代码/配置文本录入）
{"name": "config_json", "componentName": "YtCodeEditor", "label": "配置内容", "language": "json", "height": 240, "placeholder": "请输入配置内容"}

// YtSingleSelect（静态选项）
{"name": "status", "componentName": "YtSingleSelect", "label": "状态", "options": [{"label": "活跃", "value": 1}, {"label": "流失", "value": 0}]}

// 选择类动态选项（YtSingleSelect / YtMultiSelect 等，需配套 Page.dataSource.list getSelectOptions）
{"name": "<FromRelationField>", "componentName": "<selectComponentName>", "label": "<fieldLabel>", "options": {"type": "JSExpression", "value": "this.state.<FromDatasetCode>_<FromRelationField>_options"}}

// YtMultiSelect（静态选项）
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
| placeholder, options, disabled, readonly | precision, min, max, format, showTime, picker, 其他 Ant Design 5 控件原生 props |

禁止在 extendProps 中重复写顶层已有的属性。

### options 数据获取方式

| options 来源 | 配置位置 | Agent 生成规则 |
|--------------|---------|----------------|
| 平台数据集 / 关联关系 | `Page.dataSource.list` + 字段 `options` | 必须配置 `getSelectOptions` 数据源，并通过 `this.state.<dataSourceId>` 绑定 |
| 三方链接 / 外部接口 / 外部静态资源 | 字段级 `optionsRequest` | 仅外部 options 源可用；禁止用于 `/getSelectOptions` |

> hard rule：`getSelectOptions` 是 Page 数据源能力，Agent 必须生成 Page 层 `dataSource.list` + state 绑定，不生成 `optionsRequest`。即使 `optionsRequest` 技术上能请求完整接口，也会把字段配置耦合到具体域名或部署环境，客户私有化部署、域名切换时会产生迁移成本。
>
> `YtSingleSelect`、`YtMultiSelect` 等选择类组件使用平台数据集 / 关联关系动态 options 时，加载机制一致：都必须生成 Page 层 `dataSource.list` 的 `getSelectOptions`，并通过字段 `options` 绑定同名 state；差异仅在字段值是单值还是数组。`<selectComponentName>` 按字段类型选择具体组件，如 `YtSingleSelect` 或 `YtMultiSelect`。

```json
// ✅ 平台数据集关联关系：Page 层预配置 getSelectOptions
{
  "type": "fetch",
  "id": "<FromDatasetCode>_<FromRelationField>_options",
  "isInit": true,
  "source": "dataset",
  "options": {
    "method": "POST",
    "uri": "/api/<appCode>/<ToDatasetCode>/getSelectOptions",
    "params": {"code": "<ToValueField>", "label": "<ToLabelField>"}
  }
}

// Form / Filter 字段绑定 state（YtSingleSelect / YtMultiSelect 等选择类组件通用）
{
  "name": "<FromRelationField>",
  "componentName": "<selectComponentName>",
  "options": {
    "type": "JSExpression",
    "value": "this.state.<FromDatasetCode>_<FromRelationField>_options"
  }
}

// ✅ 三方链接 / 外部接口：字段级 optionsRequest
{
  "name": "external_level",
  "componentName": "YtSingleSelect",
  "optionsRequest": {
    "url": "https://example.com/options/levels.json",
    "dataProcess": {
      "type": "JSFunction",
      "value": "(data) => data.map(item => ({ label: item.name, value: item.id }))"
    }
  }
}

// 说明：dataProcess 是函数型字段。Agent 生成 PageSchema 时使用 JSFunction；
// 若绕过 PageSchema 运行时直接传给物料组件，则应传入真实函数。

// ❌ 错误：使用 optionsRequest 调 getSelectOptions
// 运行时会按当前页面域名解析相对 /api 路径，不会使用 Page 数据源的接口域名；
// 页面域名与接口域名不一致时会请求错地址。
{
  "optionsRequest": {
    "url": "/api/<appCode>/<ToDatasetCode>/getSelectOptions?code=id&label=name"
  }
}

// ❌ 错误：把完整域名写进 getSelectOptions
// 即使当前环境可用，也会把 schema 耦合到某个租户域名或部署环境。
{
  "optionsRequest": {
    "url": "https://<tenant-domain>/api/<appCode>/<ToDatasetCode>/getSelectOptions?code=id&label=name"
  }
}
```

#### 关联关系 From / To 规则

- `dataSource.id = <FromDatasetCode>_<From表关联字段名>_options`，其中 `FromDatasetCode` 为 From 表字段所属数据集的 code
- `dataSource.isInit = true`
- `dataSource.source = "dataset"`
- `uri = /api/<appCode>/<ToDatasetCode>/getSelectOptions`
- `params.code = <To表被关联字段，通常是 id>`
- `params.label = <To表展示字段>`
- 字段 `options` 绑定 `this.state.<FromDatasetCode>_<From表关联字段名>_options`
- 自关联父子关系中，`FromDatasetCode` 与 `ToDatasetCode` 可以相同；state key 仍按 `<FromDatasetCode>_<FromRelationField>_options` 命名，URI 仍使用 To 侧 options 数据源。

---

## 字段组件选型规则

本节按 Agent 生成时的选型优先级和运行时依赖记录组件。生成 `componentName` 时必须使用真实运行时注册名，不要按相似名称猜测组件名。

### 强平台依赖组件

| 组件 | 依赖 / 定制点 | Agent 生成约束 |
|------|---------------|----------------|
| `YtUserSelect` | 读取当前应用 `appCode`，请求平台用户接口 `/api/dept/queryAppUser` | 用户字段用数组值；Filter 查询值必须是数组 |
| `YtUpload` | 依赖平台上传接口 `/api/common/uploadFile`、运行时域名、当前应用 `appCode` 和平台文件预览能力 | 仅用于 Form；值为文件对象数组 |
| `YtAddressPicker` | 默认加载 Lovrabet 地址资源 `https://www.lovrabet.com/external/regions.json` | 值为地址路径数组；不用于关联表 `getSelectOptions` |
| `YtRichEditor` | 动态加载 Lovrabet CDN 富文本 UMD 资源 | 仅用于 Form；值为 HTML 字符串 |

### 推荐优先使用的字段组件

| 组件 | 适用语义 | Agent 生成约束 |
|------|----------|----------------|
| `YtCodeEditor` | 代码、脚本、SQL、JSON、配置内容 | 仅用于 Form；必须显式生成 `language` |
| `YtAmountInput` | 金额、价格、费用 | 值为 number |
| `YtCurrencySelect` | 币种 | 必须使用真实运行时注册名 `YtCurrencySelect` |

### 低频语义组件

以下组件只有在用户需求或字段语义明确命中时使用，不作为默认选型；不要因为它们以 `Yt` 开头就优先使用。

- `YtInputPassword`
- `YtRate`
- `YtTransfer`
- `YtSingleTreeSelect`
- `YtMultiTreeSelect`
- `YtTimePicker`
- `YtRangeTimePicker`

---
