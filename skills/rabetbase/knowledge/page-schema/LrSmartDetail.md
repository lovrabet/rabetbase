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

> `_doVersion`、`_context`、`_uniqueId`、`ref`、`items` 与 LrSmartCreate 一致，不再赘述。

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

## 5. 高频变更点

- 增删 items 字段
- 新增引用表区域：dataSource.list 加 getOne（`isSync: true`） + YtRglContainer.children 加 LrSmartDetail + RGLlayout 加布局块
- 修改引用表区域标题（`title`）

## 6. 限制与误用

- ❌ 引用表 getOne 不加 `isSync: true` → 与主表并行触发，params 为 undefined，控制台报 `"should not fetch"`
- ❌ 有引用表时主表 getOne 不加 `dataHandler` 同步写 state → `isSync` 仅保证 await 顺序，但 setState 是异步的，下一个数据源读不到值
- ❌ `defaultValue` 的 state key 与 dataSource id 不匹配 → 无数据

## 7. 交叉校验

- **已验证事实**：
  - isSync + dataHandler 同步写 state 组合可解决引用表 getOne 依赖链问题（线上已验证，最大 3 个引用表场景）
  - 无引用表时主表 getOne 不需要 isSync 和 dataHandler，保持默认异步
  - `_options` 数据源不加 isSync，与 getOne 链并行加载，互不影响
- **未完全验证**：超过 3 个引用表时的串行加载性能
