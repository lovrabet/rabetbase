---
name: PageSchema-syntax-reference
description: PageSchema 平台扩展语法规范。先验知识：低代码引擎搭建协议、Ant Design 5 组件知识。
tags: [page-schema, syntax, reference]
---

# PageSchema 语法参考（平台扩展）

> **先验知识**：默认已具备低代码引擎搭建协议和 Ant Design 5 组件知识。本文档仅记录平台特有扩展。

---

## 1. JSFunction vs JSExpression 关键差异

| 特性 | JSFunction | JSExpression |
|-----|-----------|-------------|
| **解析方式** | `new Function("return " + value)()` | `new Function("$scope", "with($scope) { return " + value + " }")(self)` |
| **简写支持** | ❌ 不支持 `state.xxx` | ✅ 支持 `state.xxx` 简写 |
| **适用场景** | 函数定义（按钮、校验、render） | 动态表达式（属性值、条件判断） |

```json
// ❌ 错误：JSFunction 不支持 state 简写
{"type": "JSFunction", "value": "() => state.filterValues"}  // state 未定义

// ✅ 正确：JSFunction 通过参数访问
{"type": "JSFunction", "value": "(config) => config._context.state.filterValues"}

// ✅ 正确：JSExpression 支持简写
{"type": "JSExpression", "value": "state.filterValues"}
```

**state key 限制**：`this.state.xxx` 中的 key（如 `filterValues`）由组件初始化时自动生成，Agent 不可修改或重命名。只能读取和引用已有的 state key。

---

## 2. 按钮配置（optType + options）

**禁止使用 onClick**，必须使用 `optType + options` 结构。

### 顶层按钮 optType 类型

| optType | 说明 | options 关键字段 |
|---------|------|-----------------|
| `jsAction` | 执行 JavaScript 函数 | `jsFunction` |
| `flowAction` | 执行流程 | `flowList` |

### flowAction 的 flowList 子动作 optType 类型

| optType | 说明 | options 关键字段 |
|---------|------|-----------------|
| `request` | 发送请求 | `_bindDataSource`, `_bindDataSourceParams`, `confirmConfig` |
| `submit` | 表单提交 | `_bindForm`, `_bindDataSource` |
| `reset` | 表单重置 | `_bindForm` |
| `jsAction` | 执行 JavaScript 函数 | `jsFunction` |
| `message` | 消息提示 | `type`, `content`, `duration` |

### 示例

```json
// jsAction：页面跳转
{
  "title": "更新",
  "optType": "jsAction",
  "options": {
    "jsFunction": {
      "type": "JSFunction",
      "value": "(config) => config._context?.appHelper?.yt?.smartPageHistory?.push({ pageTagName: 'UPDATE', datasetId: '<datasetId>', dataid: config.extraParams?.rowKey })"
    }
  }
}

// flowAction + request：删除
{
  "title": "删除",
  "optType": "flowAction",
  "options": {
    "flowList": [{
      "active": true,
      "optType": "request",
      "options": {
        "_bindDataSource": "<datasetId>_delete",
        "_bindDataSourceParams": {
          "type": "JSFunction",
          "value": "(extraParams) => { return { id: extraParams.rowKey?.toString() }; }"
        },
        "confirmConfig": { "enable": true, "title": "确认删除", "content": "确认删除该条数据吗？" }
      }
    }]
  }
}

// flowAction + message：提示消息
{
  "title": "保存成功提示",
  "optType": "flowAction",
  "options": {
    "flowList": [{
      "active": true,
      "optType": "message",
      "options": {
        "type": "success",
        "content": "操作成功",
        "duration": 3
      }
    }]
  }
}
```

---

## 3. 事件处理（events）

### Agent 生成边界

当前运行时不消费字段组件上的 `events.onChange` / `events.onMount`，也没有字段级 context 联动方法实现。

Agent 不应为 `LrSmartCreate` / `LrSmartUpdate` / `LrSmartDetail` / `LrSmartFilter` 的字段项生成字段级 `events` 联动 schema。

### 字段联动决策表

| 组件场景 | Agent 应该怎么生成 | 不要生成 |
|----------|--------------------|----------|
| 表单型组件字段联动 | 在组件 props 顶层生成 `fieldReactions`；返回“字段名 → 字段属性覆盖”的对象。当前适用于 `LrSmartCreate` / `LrSmartUpdate`，详情展示控制同理遵循 Form 机制 | `items[].events.onChange`、字段级 context 方法 |
| `LrSmartFilter` 筛选搜索 | 生成宿主级 `events.onSearch`，用于刷新消费 filter state 的数据源 | 筛选项之间的实时字段联动、`fieldReactions`、`items[].events.onChange` |
| 字段静态隐藏 | 在字段 item 上设置 `hidden` 或 `isShown` | 通过字段级事件动态 setFieldVisible |

`fieldReactions` 只用于表单型组件，不用于 `LrSmartFilter`。返回对象的 key 是被联动字段 `name`，value 是要覆盖的字段属性，例如 `isShown`、`hidden`、`disabled`、`rules`、`options`。具体示例见 `LrSmartCreate.md`。

### 当前可生成的事件入口

| 入口 | 适用场景 | Agent 生成规则 |
|------|----------|----------------|
| `optType + options` | 按钮点击、提交、请求、跳转、提示消息 | 标准按钮协议，见 § 2 |
| 宿主级 `events` | 表单提交、默认值加载完成、筛选搜索、表格行/分页事件 | 仅在组件文档明确给出事件名时生成 |
| `fieldReactions` | 表单型组件字段显隐、禁用、选项等联动覆盖 | 仅用于表单型组件，不用于 Filter；具体见 `LrSmartCreate.md` |

### 禁止生成模式

不要为字段项生成字段级 `events` 联动 schema，也不要假设存在字段级 context 联动方法。表单型组件字段联动使用 `fieldReactions`；Filter 字段联动当前不作为 Agent 默认生成能力。

---

## 4. 数据绑定（平台扩展字段）

以下划线 `_` 开头的属性是平台扩展字段：

| 属性 | 说明 |
|------|------|
| `_bindDataSource` | 绑定数据源 ID（如 `"<datasetId>_filter"`） |
| `_bindForm` | 绑定表单 ID |
| `_bindDataSourceParams` | 数据源参数（JSFunction） |
| `_autoBindDataSourceParams` | 自动绑定表单参数（`true`） |

```json
{
  "_bindDataSource": "<datasetId>_delete",
  "_bindDataSourceParams": {
    "type": "JSFunction",
    "value": "function(extraParams) {\n  return { id: extraParams.rowKey };\n}"
  }
}
```

> `_bindDataSourceParams` 仅用于操作型数据源（如 `_delete`、`_create`、`_update`）。filter 查询数据源的 where/orderBy/select 由组件属性自动同步，不可通过 `_bindDataSourceParams` 注入。

**数据源唯一性原则**：

- `datasource.params` 和 `datasource.willFetch` 由平台根据组件属性自动同步生成
- Agent 只需修改组件属性（如 `initialValue`、`defaultSortOrder`），不应直接修改 `datasource` 配置
- 示例：筛选器默认值通过 `items[].initialValue` 配置，自动同步到 `datasource.params.filter`

---

## 5. config 对象结构（按钮 JSFunction 参数）

| 属性 | 说明 |
|------|------|
| `config._context` | 组件上下文（含 appHelper、state 等） |
| `config.extraParams` | 额外参数（如行数据 `rowKey`） |
| `config.getFlowActionInfo()` | 获取流程执行结果 |

### JSFunction 作用域

JSFunction 通过 `new Function("return " + value)()` 执行，作用域中只有：

| 可用对象 | 来源 | 说明 |
|---------|------|------|
| `config` | 函数参数 | `_context`、`extraParams` 等 |
| `window.antd` | UMD externals | antd 全部组件（Modal、message 等） |
| `window.__GLOBAL__` | 平台注入 | 用户信息等，见 § 6 |
| `window.*` | 浏览器原生 | `window.open`、`console` 等 |

不可直接使用裸名（如 `Modal`、`Button`），必须从 `window.antd` 解构：

```javascript
// ✅ 正确
(config) => { const { Modal } = window.antd; Modal.info({ title: '提示', content: '内容' }); }

// ❌ 错误：Modal is not defined
(config) => { Modal.info({ title: '提示', content: '内容' }); }
```

> 与 LrSmartJsx 的区别：LrSmartJsx 的 `render.source` 支持 `import { Modal } from 'antd'`（编译时转译为 `const { Modal } = window.antd`），JSFunction 不经过编译，必须手动解构。

---

## 6. 平台全局对象（window.__GLOBAL__）

### userInfo 当前用户信息

**类型定义**：

```typescript
window.__GLOBAL__.userInfo: {
  userId: string;      // 用户ID
  nickname: string;    // 用户昵称
  username: string;    // 用户登录账号
  mobile: string;      // 用户手机号码
  email?: string;      // 邮箱（可选）
  image?: string;      // 用户头像（可选）
}
```

**高频业务场景**：

| 场景 | 字段 | 用途 | 示例 |
|------|------|------|------|
| 快捷过滤按钮 | `userId` | 筛选"我创建的"、"我负责的"数据 | `{ creator_id: { $in: [userId] } }` |
| 表单默认值 | `userId` | 自动填充创建人、负责人 | `initialValue: window.__GLOBAL__.userInfo.userId` |
| 联系方式回填 | `mobile` / `email` | 自动填充联系人信息 | `initialValue: window.__GLOBAL__.userInfo.mobile` |

**访问方式**：

```javascript
// JSFunction 中访问
const currentUserId = window.__GLOBAL__?.userInfo?.userId;

// JSExpression 中不支持（需通过 JSFunction 传递）
```

**注意事项**：

- `window.__GLOBAL__` 是平台运行时注入的全局对象，仅在浏览器环境可用
- 访问时建议使用可选链 `?.` 避免未定义错误
- `userId` 为字符串类型（如 `"80"`），与数据库 ID 类型保持一致

---

## 7. 点号字段（引用表字段）

引用表字段使用 `关联表名.字段名` 的点号语法，如 `customer.customer_name`。

**底层支持**：仅 filter 接口支持点号字段，可用于 `select`、`where`、`orderBy`。

```typescript
// filter 接口示例
await client.models.dataset_xxx.filter({
  select: ["opportunity_name", "customer.customer_name"],
  where: { "customer.customer_type": { "$eq": "企业" } },
  orderBy: [{ "customer.customer_name": "asc" }]
})
```

**限制**：
- 仅支持一级关联（`relation.field`），不支持 `a.b.c`
- 关联关系必须在数据集 `relations` 中定义
- create / update / getOne 等其他接口不支持点号语法

数据集关系的主子表声明、字段方向和验收规则见 [数据集关系声明](dataset-relations.md)。

---

## 8. 数据集关系：主子表

主子表是数据集关系元数据，用于表示一条主记录对应多条明细记录。通过 Rabetbase CLI 写入时，关系基数使用 `--cardinality ONE_TO_MANY`，业务类型使用 `--biz-relation-type main_sub`，并保证主表被引用字段与子表关联字段真实匹配。

页面通过 `relation.field` 消费已定义的关系；它不会因为声明主子表而自动生成子表布局或交互。不要将数据集内部关系对象直接复制到 CLI 请求或 PageSchema props；完整字段说明、示例与校验规则见 [数据集关系声明](dataset-relations.md)。
