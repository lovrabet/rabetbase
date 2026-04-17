---
name: LrSmartTable
description: LrSmartTable 数据表格组件完整配置
tags: [table, page-schema, select, orderBy, isFetched, defaultSortOrder, sortable]
---

# LrSmartTable

数据表格组件，用于展示列表数据。

---

## 禁止事项

| 禁止 | 替代方案/说明 |
|------|---------------|
| `meta.handleProps` | 纯 JSON props |
| `hidden` 字段 | `isShown` |
| `customProps.tableFormat` | `column.format` |
| 按钮 `onClick` | `optType + options.jsFunction` |
| 字段改名/类型/字典 | 需在数据集管理中操作 |

---

## Props 属性

平台自定义属性与 antd Table 原生属性并存；原生 TableProps 统一放到 `otherTableProps`。

| 属性 | 类型 | 说明 |
|------|------|------|
| _doVersion | string | 数据集版本标识，必须保留原值（如 `"V2"`），不可删除或修改 |
| columns | Column[] | 列配置 |
| operateButtonsEnable | boolean | 操作列显示 |
| operateButtons | Button[] | 操作列按钮 |
| toolbar.enable | boolean | 工具栏显示 |
| toolbar.globalButtons | Button[] | 全局按钮 |
| toolbar.selectedButtons | Button[] | 批量操作功能被 exportButtons 取代，先不配置 |
| toolbar.exportButtons | BatchButton[] | 批量操作 |
| rowSelection.enable | boolean | 行选择 |
| pagination.enable | boolean | 分页 |
| columnSetting.enable | boolean | 列设置 |
| columnSetting.storageKey | string | 列设置缓存 key |
| otherTableProps | Record<string, any> | 其他 antd Table 的原生属性 |

---

## Column 列属性

V2 页面生成时已为数据集所有字段全量生成 columns，通过 `isShown` 控制显示。用户"添加列" = 将已有 column 的 `isShown` 设为 `true`；仅当字段在 columns 中完全不存在时才需插入新对象。

`isShown` 控制列是否渲染，`isFetched` 控制字段是否参与 API 请求，两者独立。具体组合见下方 isShown + isFetched 组合表。

列属性兼容 antd TableColumnProps，并补充平台扩展字段。

| 属性 | 类型 | 说明 |
|------|------|------|
| title | string | 列标题 |
| dataIndex | string | 字段路径，支持 `relation.field` |
| doType | string | 字段数据类型，来自 dataset_detail_tool，用于二次更新时参考 |
| align | string | 对齐方式 |
| format | string | 格式化类型 |
| formatOptions | object | 格式化配置 |
| isShown | boolean | 是否显示该列 |
| isFetched | boolean | 是否从 API 请求该字段数据 |
| width | number | 列宽 |
| fixed | string | 列是否固定，可选 `'start'`、`'end'` |
| tooltip | string | 列标题旁的提示信息（antd Table 原生属性） |
| sortable | boolean | 可排序，点击表头更新 sort state → filter 请求 orderBy |
| defaultSortOrder | string | 初始排序方向（`ascend`/`descend`），列的扁平属性，初始化时写入 sort state → orderBy |
| ellipsis | boolean | antd Table 列级省略（td 宽度截断），与 `formatOptions.ellipsis` 无关，两者独立生效 |
| render | JSFunction | 自定义渲染 |
| ... | ... | 其余 TableColumnProps |

### sortable + defaultSortOrder 组合

两者独立控制不同生命周期，可自由组合：

| 组合 | 效果 |
|------|------|
| `sortable: true` + 无 `defaultSortOrder` | 用户可交互排序，首次加载无默认排序 |
| `defaultSortOrder: "descend"` + 无 `sortable` | 首次加载有默认排序，用户不可交互切换 |
| `sortable: true` + `defaultSortOrder: "descend"` | 首次加载有默认排序，且用户可交互切换（最常用） |

### isShown + isFetched 组合

通过组合 `isShown` 和 `isFetched`，可以精确控制字段的显示和请求行为：

| 组合 | 说明 | 典型场景 |
|------|------|---------|
| `isShown: true, isFetched: true` | 显示且请求 | 普通业务字段 |
| `isShown: false, isFetched: true` | 不显示但请求 | render 派生字段的数据源（如 `customer_id` 不显示，但被其他列 render 引用） |
| `isShown: false, isFetched: false` | 不显示不请求 | 完全隐藏的字段 |
| `isShown: true, isFetched: false` | ❌ 无效组合 | 显示但无数据，不应使用 |

**派生字段模式**：当 render 函数需要引用其他字段时，被引用字段必须设置 `isFetched: true`（确保数据被请求），可设置 `isShown: false`（不单独显示列）。

**主键字段约束**：`pkField: true` 的列（通常是 `id`）隐藏时必须保留 `isFetched: true`，行操作按钮（编辑、详情、删除）依赖主键值。正确做法：`isShown: false, isFetched: true`。

```json
// 示例：「客户信息」列通过 render 拼接 customer_name 和 customer_id
[
  {"title": "客户ID", "dataIndex": "customer_id", "isShown": false, "isFetched": true},
  {
    "title": "客户信息", "dataIndex": "customer_name", "isShown": true, "isFetched": true,
    "render": {"type": "JSFunction", "value": "(value, record) => `${record.customer_name}（${record.customer_id}）`"}
  }
]
```

### format 类型

| format | formatOptions | 效果 |
|--------|---------------|------|
| text | `{ellipsis: boolean}` | 纯文本（`formatOptions.ellipsis` 是渲染层省略，不替代列级 `ellipsis`） |
| link | 基于 antd 5.x 的 Typography.Link 的属性 | 可点击链接跳转 |
| user | - | 系统账户字段，通过平台用户表渲染头像+姓名（仅限 systemRetain 标记字段） |
| enum | `{options: [{label, value}]}` | 枚举样式（value→label 映射） |
| tag | `{options: [{label, value}], colorMapping: {"<value>": "<color>"}}` | 标签样式（options 做 value→label 映射，colorMapping 做颜色映射） |
| datetime | `{format: "<dayjs格式>"}` | 日期/日期时间（如 `YYYY-MM-DD`、`YYYY-MM-DD HH:mm:ss`） |
| number | `{precision: 2, prefix: "¥"}` | 数字 |
| progress | 基于 antd 5.x 的 Progress 的属性 | 进度条展示 |
| percent | `{precision: 1, showSymbol: true}` | 百分比 |
| address | - | 地址级联展示 |
| upload | `{showPreview: true}` | 上传附件 |
| image | 基于 antd 5.x 的 Image 的属性 | 图片 |
| file | - | 在线文件 |

**tag vs enum 选型**：字段名包含 status/state/level/grade → `tag`；有 options 但非状态类 → `enum`。用户要求颜色展示时，无论是否说"枚举"，统一使用 `tag`（唯一支持 colorMapping 的 format）。

**tag/enum 共性**：`formatOptions.options` 来自数据集字段的枚举定义，tag 额外通过 `colorMapping` 指定颜色（key 必须与 options value 一致）。

```json
// tag 示例
{"format": "tag", "formatOptions": {"options": [{"label": "进行中", "value": "in_progress"}], "colorMapping": {"in_progress": "#1890ff"}}}
// enum 示例
{"format": "enum", "formatOptions": {"options": [{"label": "高", "value": "high"}, {"label": "低", "value": "low"}]}}
// datetime 示例
{"format": "datetime", "formatOptions": {"format": "YYYY-MM-DD"}}
```

### filter 数据源与表格贡献

表格列表数据来自 filter 数据源，请求体遵循 filter 接口协议。**filter 接口完整协议**（请求体结构、where 操作符、逻辑符、willFetch 合并逻辑等）见 `LrSmartFilter.md`。

**表格对 filter 的贡献**：

| 表格来源 | 对应 filter 字段 | 说明 |
|----------|------------------|------|
| sort | orderBy | 点击表头排序时更新 `yttable_xxx_sort` → orderBy |
| 列 dataIndex（isFetched: true） | select | 聚合为 select 数组 |
| pagination | currentPage、pageSize | 分页参数 |

表格消费 filter 数据源返回结果进行渲染。

---

### 数据源唯一性与自动同步

filter 请求体的三个核心参数各有唯一数据源。Agent 只修改组件属性，不直接修改 datasource 配置：

| 参数 | 唯一数据源 | 同步方式 |
|------|-----------|---------|
| select | `columns[].isFetched` | 静态同步：schema 合并时自动重建 |
| where | `LrSmartFilter.items[].initialValue` | 动态同步：组件初始化时写入 state |
| orderBy | `columns[].defaultSortOrder` | 动态同步：组件初始化时写入 state |

**select 同步细节**：

**机制**：`tableColumns[].isFetched` 是 select 的唯一数据源。`datasource.params.select` 和 `paramsArr[select].value` 是派生数据，由系统自动从 `isFetched === true` 的列聚合生成。Agent 只需修改 `props.columns[].isFetched`，无需手动修改 datasource。

**参数**：

| Source | Target (auto-sync) | Description |
|--------|-------------------|-------------|
| `dataIndex` collection where `columns[].isFetched: true` | `{datasetId}_filter` → `params.select` | filter query fields |
| Same as above | `{datasetId}_filter` → `paramsArr[name=select].value` | filter params array |
| Same as above | `{datasetId}_excelExport` → `params.select` | export query fields |

大文本字段（`doType` 为 TEXT/LONGTEXT/MEDIUMTEXT/TINYTEXT）首次生成时默认 `isFetched: false`，避免列表查询大字段。用户可通过 Agent 更新或手动配置修改，但不推荐。

**示例**：修改 `description` 列为不查询

```json
{
  "props": {
    "columns": [
      {"dataIndex": "customer_name", "isFetched": true},
      {"dataIndex": "description", "isFetched": false}
    ]
  }
}
```

系统自动同步后的 datasource（无需手动修改）：

```json
{
  "id": "1002663_filter",
  "options": {
    "params": {"select": ["customer_name"]},
    "paramsArr": [{"name": "select", "value": ["customer_name"]}]
  }
}
```

---

### orderBy 排序（sort → orderBy）

**机制**：排序配置分两个生命周期阶段。

| 阶段 | 配置属性 | 作用 |
|------|---------|------|
| 初始化 | `defaultSortOrder` | 页面首次加载时的默认排序方向，写入 sort state → orderBy |
| 交互 | `sortable: true` | 用户点击表头切换正序/倒序，覆盖 sort state → orderBy |

组件初始化时通过 useLayoutEffect 从 columns 中提取有 `defaultSortOrder` 的列，写入 `context.state[sortKey]`，确保首次 datasource.load() 能读到 orderBy。用户点击表头后，sort state 被覆盖为新的排序。

**参数**：

| Source | Target | Description |
|--------|--------|-------------|
| `columns[].defaultSortOrder` | `context.state[sortKey].orderBy` | Auto-converted on initialization |
| `"ascend"` | `"ASC"` | antd value → API value |
| `"descend"` | `"DESC"` | antd value → API value |

`defaultSortOrder` 必须是列的扁平属性，不可嵌套在 `extendProps` 中。

**示例**：

```json
{
  "dataIndex": "updated_at",
  "title": "更新时间",
  "format": "datetime",
  "sortable": true,
  "defaultSortOrder": "descend"
}
```

> 首次加载时 API 请求 orderBy 包含 `[{ "updated_at": "DESC" }]`。

**❌ 错误**：`defaultSortOrder` 嵌套在 `extendProps` 中，antd Table 无法识别。

```json
{ "sortable": true, "extendProps": { "defaultSortOrder": "ascend" } }
```

`<fieldName>` 从 dataset_detail_tool 获取（如 `isModifyDate=true` 列对应更新时间字段）。

---

## 按钮配置

表格按钮遵循平台通用按钮协议：`optType + options`。

- 通用顶层按钮 `optType`、`flowAction.flowList` 子动作类型、平台扩展字段：见 `syntax-reference.md`
- 本文档仅补充 LrSmartTable 的按钮挂载位置、行操作上下文和表格场景示例

---

## operateButtons 行操作按钮

基于 antd 5.x 的 ```ButtonProps```，ButtonProps 属性加到下列表格后

### 按钮属性

| 属性 | 类型 | 说明 |
|------|------|------|
| title | string | 按钮文本（必填） |
| optType | string | 按钮功能类型（必填） |
| options | object | 按钮配置项（必填） |
| type | string | 按钮样式：`default` / `primary` / `text` / `link` / `dashed` |
| size | string | 按钮尺寸：`large` / `medium` / `small` |
| danger | boolean | 危险按钮样式（红色） |
| icon | string | 按钮图标 |
| disabled | boolean | 禁用状态 |
| hidden | boolean | 隐藏状态 |
| _permission | object | 权限点配置 `{code: string, policy?: JSFunction}`，权限协议预留字段，当前标准页面编辑暂不作为现行可用能力 |

### operateButtons 的 optType 类型

行操作按钮沿用平台通用按钮协议：

- 顶层按钮常用 `optType`：`jsAction`、`flowAction`
- `request`、`submit`、`reset`、`message` 等属于 `flowAction.flowList` 子动作
- 具体枚举和字段说明见 `syntax-reference.md`

### options 配置项

表格场景中最常用的 `options` 字段：

| 字段 | 说明 |
|------|------|
| jsFunction | 跳转详情/编辑、执行表格场景 JS 逻辑 |
| flowList | 串联请求、提示消息等流程动作 |
| _bindDataSource | 表格相关数据源 ID（常见于删除、批量操作） |
| _bindDataSourceParams | 基于行数据或选中数据生成请求参数 |
| confirmConfig | 删除、批量操作等场景的确认弹窗配置 |

### 行操作按钮示例

```json
{
  "title": "编辑", // antd Button 属性
  "type": "link", // antd Button 属性
  "size": "medium", // antd Button 属性
  "optType": "jsAction", // 自定义属性
  "options": { // 自定义属性
    "jsFunction": {
      "type": "JSFunction",
      "value": "(config) => config._context?.appHelper?.yt?.smartPageHistory?.push({ pageTagName: 'UPDATE', datasetId: '<datasetId>', dataid: config.extraParams?.rowKey })"
    }
  }
}
```

### 注意事项

- `extraParams` 会自动注入 `{rowKey, row}`，行按钮可通过 `config.extraParams` 访问当前行数据
- `type` 建议使用 `"link"`，保持操作列视觉一致性
- 删除等危险操作建议设置 `danger: true`
- `_permission` 为待开发协议预留能力，当前标准页面编辑暂不直接生成

---

## globalButtons 全局按钮

工具栏全局按钮，属性与 operateButtons 一致（同 Button 协议），区别：全局按钮无行上下文（无 `extraParams.rowKey/row`）。

arrayName = `tableGlobalButtons`，不受数据集约束，直接 INSERT。

```json
{"title": "新建", "type": "primary", "optType": "jsAction", "options": {"jsFunction": {"type": "JSFunction", "value": "(config) => config._context?.appHelper?.yt?.smartPageHistory?.push({ pageTagName: 'CREATE', datasetId: '<datasetId>' })"}}}
```

---

## exportButtons 批量操作

| 属性 | 类型 | 说明 |
|------|------|------|
| mainButton | BatchConfig | 主按钮 |
| buttons | BatchConfig[] | 下拉按钮组 |
| buttonsDisabled | boolean / JSExpression | 下拉按钮组是否禁用 |
| loading | boolean / JSExpression | 批量操作请求中的加载态 |

### 按钮属性

- `mainButton`：主按钮，属性与 `operateButtons` 基本一致；可直接承载默认批量动作，也可作为批量操作入口，常见用途是批量编辑入口
- `buttons`：下拉按钮组，在 `operateButtons` 基础上增加 `actionType`
- `actionType`：常见值为 `export` / `delete`
- `buttonsDisabled`：常与 `rowSelection.isDisabled` 或页面状态联动，支持 `JSExpression`
- `loading`：常绑定批量导出等动作数据源的 loading state，支持 `JSExpression`

### 运行机制

- 使用批量操作前，必须开启 `rowSelection.enable`
- `mainButton` 可直接承载默认批量动作，也可作为批量操作入口；`buttons` 定义下拉批量动作
- `actionType` 用于标识动作语义，常见为 `export` / `delete`
- 选中项相关动作依赖表格选中态；导出类通常绑定 `<datasetId>_excelExport`，删除类通常绑定 `<datasetId>_delete`
- 生成器默认先提供“导出选中 / 导出全部”能力，真实页面可继续扩展为“批量编辑 / 删除选中 / 删除全部”等批量动作容器
- 删除类动作通常需要 `confirmConfig`，成功后通常需要刷新表格数据源

### 当前实现中的常见参数约定

- 行按钮场景：`config.extraParams` 常见为 `{ rowKey, row }`
- `selectedRowKeys`：表格选中主键列表，当前实现中常用于批量编辑入口或按钮禁用判断
- `selectKeys`：导出接口常用的选中主键参数，当前实现中通常由 `_autoBindDataSourceParams: true` 自动传递
- `extraParams.selectId`：当前批量删除选中项实现中常用的 ID 集合
- `extraParams.allIds`：当前批量删除全部实现中常用的 ID 集合
- 批量导出场景：选中导出通常配合 `_autoBindDataSourceParams: true`，全量导出通常直接请求导出数据源

### 批量操作示例

```json
{
  "mainButton": {
    "title": "批量操作",
    "optType": "jsAction",
    "options": {
      "jsFunction": {
        "type": "JSFunction",
        "value": "(config) => config._context?.appHelper?.yt?.smartPageHistory?.push({ pageTagName: 'UPDATE', datasetId: <datasetId>, dataid: config._context?.state['yttable_215eiu2svrr_selectedRowKeys'], type: 'batch' })"
      }
    }
  },
  "buttons": [
    {
      "title": "导出选中项",
      "actionType": "export",
      "optType": "flowAction",
      "options": {
        "flowList": [{
          "optType": "request",
          "options": {
            "_bindDataSource": "<datasetId>_excelExport",
            "_autoBindDataSourceParams": true
          }
        }]
      }
    },
    {
      "title": "导出全部",
      "actionType": "export",
      "optType": "flowAction",
      "options": {
        "flowList": [{
          "optType": "request",
          "options": {
            "_bindDataSource": "<datasetId>_excelExport"
          }
        }]
      }
    },
    {
      "title": "删除选中项",
      "actionType": "delete",
      "danger": true,
      "optType": "flowAction",
      "options": {
        "flowList": [
          {
            "optType": "request",
            "options": {
              "_bindDataSource": "<datasetId>_delete",
              "_bindDataSourceParams": {
                "type": "JSFunction",
                "value": "(extraParams) => ({ id: extraParams.selectId })"
              },
              "confirmConfig": {"enable": true, "title": "确认删除选中项？"}
            }
          },
          {
            "optType": "jsAction",
            "options": {
              "jsFunction": {
                "type": "JSFunction",
                "value": "(config) => config._context.reloadDataSource()"
              }
            }
          }
        ]
      }
    }
  ]
}
```

## context 方法

context 对象在 **JSFunction** 中通过 `config._context` 访问，用于状态管理和数据刷新。

### API 列表

| 方法 | 说明 |
|------|------|
| `_context.reloadDataSource()` | 重新加载数据源（刷新表格） |
| `_context.setState(state)` | 设置状态（浅合并） |
| `_context.state` | 读取当前状态 |

常见用途：

- 删除、更新后调用 `_context.reloadDataSource()` 刷新表格
- 使用 `_context.state` / `_context.setState()` 读写表格相关状态

### 权限控制（待开发）

`_permission` 为声明式权限协议预留能力，控制按钮显示，当前标准页面编辑暂不直接生成；完整协议参考 `external-resources-usage.md`。

### 消息提示

`message` 属于 `flowAction.flowList` 子动作，完整协议参考 `syntax-reference.md`；表格场景通常用于请求完成后的提示消息。

---

## 关联字段引用

在 `dataIndex` 中使用点号路径访问关联表字段（底层通过 filter 接口的 select 实现）：

```json
{"title": "客户等级", "dataIndex": "customer_level.level_name"}
{"title": "负责人", "dataIndex": "employee.employee_name"}
```

**限制**：

- 仅支持一级关联（不支持 `a.b.c`）
- 关联关系必须在数据集 `relations` 中定义

---

## 派生字段（自定义渲染）

通过 `render` JSFunction 拼接多个字段，实现自定义列显示。派生列不来自数据集，arrayName = `tableColumns`，直接 INSERT，不受数据集约束。

### INSERT element 最小结构（派生列）

| 字段 | 必填 | 说明 |
|------|:----:|------|
| title | ✅ | 列标题 |
| dataIndex | ✅ | 虚拟标识（如 `custom_info`），不对应数据集字段 |
| isShown | ✅ | `true` |
| isFetched | ✅ | `false`（无真实字段可查询） |
| render | ✅ | JSFunction，通过 `record` 访问其他已 fetch 的字段 |

```json
{"title": "客户信息", "dataIndex": "custom_info", "isShown": true, "isFetched": false, "render": {"type": "JSFunction", "value": "(value, record) => `${record.customer_name}（${record.customer_id}）`"}}
```

> 被 render 引用的字段（如 `customer_id`）必须 `isFetched: true`，参见 isShown + isFetched 组合表。

```json
{
  "title": "客户信息",
  "dataIndex": "customer_name",
  "render": {
    "type": "JSFunction",
    "value": "(value, record, index) => `${record.customer_name}（${record.customer_id}）`"
  }
}
```

**参数**：`value`(当前值), `record`(行数据), `index`(行索引)

### 敏感字段脱敏

对于手机号、身份证号、银行卡号等敏感字段，通过 `render` JSFunction 实现前端脱敏显示。

**脱敏规则**：手机号中间4位星号（138****5678）、身份证号中间10位星号、银行卡号中间星号、邮箱@前保留3位。

**实现方式**：使用 `render` + `replace` 正则替换或 `substring` 截取拼接。

```json
// 手机号脱敏示例
{
  "dataIndex": "phone",
  "render": {
    "type": "JSFunction",
    "value": "(value) => value ? value.replace(/^(\\d{3})\\d{4}(\\d{4})$/, '$1****$2') : '-'"
  }
}
```

**注意**：脱敏仅影响前端显示，不影响数据源和导出。后续 BFF 成熟后将下沉到 filter 接口。脱敏字段仍需 `isFetched: true`。

---

## 权限配置（待开发）

以下为 `_permission` 协议预留示例，当前标准页面编辑暂不直接生成：

```json
{
  "operateButtons": [{
    "title": "删除",
    "_permission": {"code": "<pageId>_<componentId>_operation_delete"}
  }]
}
```

**code 命名规范**：`<pageId>_<componentId>_<层级>_<action>[_<suffix>]`

| 字段 | 说明 | 必需 | 示例 |
|------|------|------|------|
| pageId | 页面 ID | ✅ | 从页面配置获取 |
| componentId | 组件 ID（带 node_ 前缀） | ✅ | 从组件配置获取 |
| 层级 | `component` / `operation` | ✅ | operation |
| action | 操作类型 | ✅ | view / create / update / delete / approve |
| suffix | 可选后缀 | ❌ | custom / batch |

**示例**：

- `<pageId>_<componentId>_component_view` - 组件查看权限
- `<pageId>_<componentId>_operation_delete` - 删除操作权限
- `<pageId>_<componentId>_operation_update_batch` - 批量更新权限