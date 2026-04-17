---
name: PageSchema-syntax-reference
description: PageSchema 平台扩展语法规范。先验知识：低代码引擎协议、antd 5+组件知识。
tags: [page-schema, syntax, reference]
---

# PageSchema 语法参考（平台扩展）

> **先验知识**：GLM-5 已具备低代码引擎协议和 antd 5+ 组件知识。本文档仅记录平台特有扩展。

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

### 字段型标准组件事件签名

| 宿主组件 | 事件 | 函数签名 | 说明 |
|---------|------|---------|------|
| `LrSmartCreate` / `LrSmartUpdate` / `LrSmartDetail` | `onChange` | `function(value, form, context)` | 字段值变化 |
| `LrSmartCreate` / `LrSmartUpdate` / `LrSmartDetail` | `onMount` | `function(form, context)` | 组件挂载 |
| `LrSmartFilter` | `onChange` | `function(value, context)` | 筛选项值变化 |
| `LrSmartFilter` | `onMount` | `function(context)` | 组件挂载 |
| `LrSmartFilter` | `onSearch` | `function(keyword, context)` | Select 搜索时触发 |

### LrSmartFilter / LrSmartCreate / LrSmartUpdate / LrSmartDetail 事件上下文方法

以下 `context` 方法适用于上述字段型标准组件的事件处理场景，不适用于 `LrSmartTable`、`LrSmartJsx` 等其他组件上下文。

| 方法 | 说明 |
|------|------|
| `setFieldOptions(name, options)` | 设置字段选项列表 |
| `setFieldVisible(name, visible)` | 设置字段显隐 |
| `setFieldDisabled(name, disabled)` | 设置字段禁用 |
| `setFieldRules(name, rules)` | 设置字段校验规则 |
| `setFieldValue(name, value)` | 设置字段值 |
| `getFieldValue(name)` | 获取字段值 |
| `showMessage(type, text)` | 显示提示消息 |

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
