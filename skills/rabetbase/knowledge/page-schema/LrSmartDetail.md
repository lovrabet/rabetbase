---
name: LrSmartDetail
description: LrSmartDetail 详情组件。单表/引用表详情的 getOne 数据源绑定、引用表依赖链（isSync + dataHandler）。
tags: [detail, page-schema, dataSource, isSync]
---

# LrSmartDetail

## 1. 组件定位

- **组件名**：`LrSmartDetail`
- **用途**：只读展示单条记录详情
- **适用场景**：单表详情、引用表多区域详情
- **不适用场景**：需要编辑提交 → `LrSmartCreate` / `LrSmartUpdate`

## 2. 边界与依赖

- **与 LrSmartCreate 的关系**：共享 items/events/form 配置，Detail 专属部分仅为 getOne 数据绑定和引用表依赖链
- **依赖文档**：items 配置 → `LrSmartCreate.md`；通用语法 → `syntax-reference.md`；布局 → `YtPage.md`

## 3. 核心属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `_bindDataSource` | string | `"<datasetId>_getOne"` 或 `"<relatedDatasetCode>_getOne"` |
| `defaultValue` | JSExpression | `"this.state['<同_bindDataSource的id>']"` |
| `preview` | boolean | 固定 `true` |
| `title` | string | 引用表区域标题（主表不需要） |
| `sub_info` | object | 主子表子表区域；由标准页生成器写入主表详情组件，字段结构见 `LrSmartCreate.md` |

> 通用表单项配置与 `LrSmartCreate` 一致；本页只补充详情页数据绑定与引用表依赖链。

## 4. 运行机制

### 4.1 单表详情

- **机制**：urlParams 提供 `dataid` → getOne 用 `dataid` 查主表 → defaultValue 绑定结果
- **参数**：`this.state.urlParams.dataid` 来自列表页跳转 URL `?dataid=<id>`
- **示例**：

```json
{
  "type": "fetch", "id": "<datasetId>_getOne", "isInit": true, "source": "dataset",
  "options": {
    "method": "POST",
    "uri": "/api/<appCode>/<datasetCode>/getOne",
    "params": {"id": {"type": "JSExpression", "value": "this.state.urlParams.dataid"}},
    "requiredParams": ["id"]
  }
}
```

```json
{
  "componentName": "LrSmartDetail",
  "props": {
    "_bindDataSource": "<datasetId>_getOne",
    "defaultValue": {"type": "JSExpression", "value": "this.state['<datasetId>_getOne']"},
    "preview": true,
    "items": [...]
  }
}
```

### 4.2 引用表依赖链

- **机制**：引用表 getOne 的 `params.id` 依赖主表 getOne 返回值中的外键。必须用 `isSync: true` 串行加载 + 主表 `dataHandler` 同步写 state，否则并行触发时主表数据还没返回，引用表 `requiredParams` 校验失败被跳过
- **参数**：
  - 主表外键值：`this.state['<datasetId>_getOne']?.<fromField>`
  - `isSync: true`：让引擎按 list 顺序 `await` 逐个加载
  - `dataHandler`：主表请求完成后同步直写 `this.state[key] = response`（与 urlParams 同步直写模式一致），确保后续 isSync 数据源能立即读到
- **示例**（dataSource.list 关键片段）：

```json
[
  {"type": "urlParams", "id": "urlParams", "isInit": true, "source": "system", "options": {}},
  {
    "type": "fetch", "id": "<datasetId>_getOne", "isInit": true, "isSync": true, "source": "dataset",
    "dataHandler": {
      "type": "JSFunction",
      "value": "function(response) { if (this.state) { this.state['<datasetId>_getOne'] = response; } return response; }"
    },
    "options": {
      "uri": "/api/<appCode>/<datasetCode>/getOne",
      "params": {"id": {"type": "JSExpression", "value": "this.state.urlParams.dataid"}},
      "requiredParams": ["id"]
    }
  },
  {
    "type": "fetch", "id": "<relatedDatasetCode>_getOne", "isInit": true, "isSync": true, "source": "dataset",
    "options": {
      "uri": "/api/<appCode>/<relatedDatasetCode>/getOne",
      "params": {"id": {"type": "JSExpression", "value": "this.state['<datasetId>_getOne']?.<fromField>"}},
      "requiredParams": ["id"]
    }
  }
]
```

引用表 LrSmartDetail 组件：

```json
{
  "componentName": "LrSmartDetail",
  "props": {
    "_bindDataSource": "<relatedDatasetCode>_getOne",
    "defaultValue": {"type": "JSExpression", "value": "this.state['<relatedDatasetCode>_getOne']"},
    "title": "<引用表显示名>",
    "preview": true,
    "items": [...]
  }
}
```

### 4.3 主子表详情（sub_info）

主子表详情使用主表 `LrSmartDetail.props.sub_info` 展示一对多的子记录，不需要为每条子记录新增独立的 `LrSmartDetail`，也不应套用 4.2 的“引用表 getOne 依赖链”。

`sub_info` 只承接 PageSchema 展示结果，不能包含或替代数据集关系字段、CLI 创建参数。关系的 `bizRelationType: "main_sub"`、基数和展示字段应先按 [dataset-relations.md](dataset-relations.md) 写入并回查。

- 当主表协议的 `extend` 提供 `subDatasetCode`、`subTableName`，且 `relatedTablesJson` 中存在对应子表协议时，标准页生成/同步会为主表详情组件写入 `sub_info`。
- 子表记录随主表详情的默认值传入；运行时会规范化记录并写入 `__subFormData[datasetCode]`。该字段路径为内部状态，不是数据集字段。
- `preview: true` 下，子表记录以卡片汇总展示；用户只能通过查看按钮打开只读抽屉。不会显示新增、编辑、删除或抽屉提交按钮。
- 子表卡片默认显示前三个可见、非分组字段。子表字段的 `isShown`、`hidden`、`group`、`formSpan` 和 `readonly` 仍然生效。

| 场景 | 正确配置 |
|------|----------|
| 主表引用一条客户、部门等记录 | 使用 4.2 的关联表 `getOne` 数据源和独立详情区域 |
| 一条订单包含多条订单明细 | 使用主表 `sub_info`，由主子表标准页生成器维护 |

完整的 `sub_info` 字段、生成条件及 Create/Update 行为见 [LrSmartCreate.md](LrSmartCreate.md)；关系字段方向见 [dataset-relations.md](dataset-relations.md)。

## 5. 高频变更点

- 增删 items 字段
- 新增引用表区域：dataSource.list 加 getOne（`isSync: true`） + YtRglContainer.children 加 LrSmartDetail + RGLlayout 加布局块
- 修改引用表区域标题（`title`）
- 变更主子表字段或布局：调整子表数据集与主表关系配置后重新执行标准页生成/同步，不要为明细列表新增引用表 getOne 链

## 6. 限制与误用

- ❌ 引用表 getOne 不加 `isSync: true` → 与主表并行触发，params 为 undefined，控制台报 `"should not fetch"`
- ❌ 有引用表时主表 getOne 不加 `dataHandler` 同步写 state → `isSync` 仅保证 await 顺序，但 setState 是异步的，下一个数据源读不到值
- ❌ `defaultValue` 的 state key 与 dataSource id 不匹配 → 无数据
- ❌ 将一对多主子表按引用表方式逐条配置 getOne → 会丢失 `sub_info` 的记录卡片、只读抽屉和统一内部值结构

## 7. 交叉校验

- **已验证事实**：
  - isSync + dataHandler 同步写 state 组合可解决引用表 getOne 依赖链问题（线上已验证，最大 3 个引用表场景）
  - 无引用表时主表 getOne 不需要 isSync 和 dataHandler，保持默认异步
  - `_options` 数据源不加 isSync，与 getOne 链并行加载，互不影响
- **未完全验证**：超过 3 个引用表时的串行加载性能
