---
name: PageSchema-external-resources-usage
description: 外部资源（自定义SQL、BFF脚本、权限点）在PageSchema中的消费规范，Skill只引用不生成。先验知识：低代码引擎协议、antd 5+组件知识。
tags: [page-schema, external-resources, custom-sql, bff, permission]
---

# 外部资源使用规范

> **先验知识**：GLM-5 已具备低代码引擎协议和 antd 5+ 组件知识。

## 1. 核心原则

**Skill 职责边界**：

- ✅ **只消费**：引用已生成的自定义 SQL、BFF 脚本、权限点
- ❌ **不生成**：不负责创建新的自定义 SQL、BFF 脚本、权限点

## 2. 自定义 SQL 使用规范

### 2.1 两种聚合查询方式对比

| 维度 | aggregate API | custom SQL |
|------|--------------|------------|
| API 路径 | `POST /api/{appCode}/{datasetCode}/aggregate` | `POST /api/custom/executeSql` |
| 跨表支持 | **单表**（基于 datasetCode 对应的一张表） | **任意跨表**（自定义 SELECT + JOIN） |
| SQL 编写者 | 平台自动生成 SQL | 开发者手写 SQL（MyBatis 语法） |

**选择策略**：

- **单表聚合**：优先使用 aggregate API（平台自动生成 SQL）
- **跨表聚合**：必须使用 custom SQL（支持 JOIN、复杂条件）

### 2.2 aggregate API 接口

**接口路径**：`POST /api/{appCode}/{datasetCode}/aggregate`

**请求参数**：

```json
{
  "where": { "status": { "$eq": "ACTIVE" } },
  "aggregate": [
    { "type": "SUM", "column": "amount", "alias": "totalAmount" },
    { "type": "COUNT", "column": "id", "alias": "recordCount", "distinct": true }
  ],
  "groupBy": ["company_id"],
  "having": { "totalAmount": { "$gt": 10000 } },
  "orderBy": [{ "totalAmount": "desc" }],
  "currentPage": 1,
  "pageSize": 20
}
```

**聚合函数类型**：`SUM` / `AVG` / `COUNT` / `MAX` / `MIN`

**在 PageSchema 中的配置方式**：

```json
{
  "dataSource": {
    "type": "aggregate",
    "datasetCode": "customer_dataset_code",
    "aggregate": [{ "type": "SUM", "column": "amount", "alias": "totalAmount" }],
    "groupBy": ["status"]
  }
}
```

### 2.3 custom SQL 接口

**接口路径**：`POST /api/custom/executeSql`

**请求参数**：

```json
{
  "sqlCode": "xxxx-xxxxx",
  "params": { "customerId": "123", "startDate": "2025-01-01" }
}
```

**在 PageSchema 中的配置方式**：

```json
{
  "dataSource": {
    "type": "customSql",
    "sqlCode": "query_customer_with_level",
    "params": { "customerId": "123" }
  }
}
```

- `type: "customSql"` 表示使用自定义 SQL 数据源
- `sqlCode` 是在平台创建的自定义 SQL 的唯一标识
- `params` 是传递给 SQL 的参数（对应 SQL 中的 `#{paramName}` 占位符）

### 2.4 禁止的操作

- ❌ 生成新的自定义 SQL 语句
- ❌ 创建新的 sqlCode
- ❌ 修改已有的自定义 SQL 内容

---

## 3. BFF 脚本使用规范

### 3.1 BFF 脚本类型

| 类型 | 说明 | 消费方式 | PageSchema 是否需要配置 |
|------|------|---------|---------------------|
| **HOOK 脚本** | 绑定到数据集 API（Before/After） | 平台自动执行 | ❌ 否（平台配置） |
| **ENDPOINT 脚本** | 独立业务端点 | 通过 dataSource 配置 | ✅ 是（需要配置 dataSource） |

**职责边界**：

- `skill_edit_smart_page_dov2` 只消费 ENDPOINT 类型的 BFF 脚本
- HOOK 类型的 BFF 脚本由平台自动执行，不需要在 PageSchema 中引用

### 3.2 ENDPOINT 脚本消费方式

**API 路径**：`POST /api/{appCode}/endpoint/{scriptName}`

**步骤 1：在 dataSource 中配置 ENDPOINT API**：

```json
{
  "dataSource": {
    "list": [{
      "id": "convertToCustomer",
      "type": "fetch",
      "options": {
        "uri": "/api/<appCode>/endpoint/convertToCustomer",
        "params": {}
      }
    }]
  }
}
```

**步骤 2：在按钮配置中调用**：

```json
{
  "title": "转化为客户",
  "optType": "flowAction",
  "options": {
    "flowList": [{
      "active": true,
      "optType": "request",
      "options": {
        "_bindDataSource": "convertToCustomer",
        "_bindDataSourceParams": {
          "type": "JSFunction",
          "value": "function(extraParams) {\n  return { id: extraParams.rowKey };\n}"
        }
      }
    }]
  }
}
```

### 3.3 HOOK 脚本说明

HOOK 脚本绑定到数据集 API 上，在 API 执行前或执行后自动触发。Skill 不需要处理 HOOK 脚本，PageSchema 中无需引用。

### 3.4 禁止的操作

- ❌ 生成新的 BFF 脚本代码（HOOK 或 ENDPOINT）
- ❌ 创建新的 scriptName
- ❌ 在 PageSchema 中引用 HOOK 类型的 BFF 脚本

---

## 4. 权限点使用规范（待开发）

### 4.1 当前状态

- 🚧 **待开发能力**：`_permission` 协议暂未作为当前标准页面的现行可用能力
- ✅ 可提前了解协议设计，用于后续能力上线后的兼容规划
- ❌ 当前标准页面编辑不应生成、修改或依赖 `_permission` 配置

**上线后适用范围**：仅覆盖权限系统**原生支持的判断模式**

- ✅ **角色权限**（身份判断）：用户角色（管理员、试用期员工）、用户身份标识（部门、职级等离散属性）
- ✅ **行级权限**（归属判断）：数据所有权（只能编辑/删除自己创建的）、数据范围（全部/本部门/仅本人）
- ❌ 不属于权限：运行时计算（入职时长、日期比较）、数据状态、时间规则、业务流程、UI 交互逻辑

### 4.2 权限协议语法（_permission，预留参考）

> 以下内容为协议预留参考，便于后续能力上线时统一实现；当前标准页面编辑暂不直接使用。

**标准结构**：

```json
{
  "_permission": {
    "code": "customer_page_delete",
    "policy": {
      "type": "JSFunction",
      "value": "function(hasPermission, data, ref) { return hasPermission; }"
    }
  }
}
```

**字段说明**：

- `code`：权限码（必需），引用平台创建的权限点
- `policy`：权限策略函数（可选），用于 UI 优化

**两层配置位置（能力上线后）**：

- 组件级：配置在组件 `props._permission`，控制整个组件显示/隐藏
- 操作级：配置在按钮对象的 `_permission`，控制按钮显示/隐藏/禁用

```json
// 组件级：控制整个组件
{
  "componentName": "LrSmartTable",
  "props": {
    "_permission": { "code": "<pageId>_<componentId>_component_view" }
  }
}

// 操作级：控制按钮
{
  "operateButtons": [{
    "title": "编辑",
    "_permission": { "code": "<pageId>_<componentId>_operation_update" }
  }]
}
```

### 4.3 code 命名规范（预留）

`<pageId>_<componentId>_<层级>_<action>[_<suffix>]`

| 字段 | 说明 | 示例 |
|------|------|------|
| pageId | 页面 ID | 从页面配置获取 |
| componentId | 组件 ID | 从组件配置获取 |
| 层级 | operation/component | operation |
| action | 操作类型 | delete/update/create/view |
| suffix | 可选后缀 | batch/special |

### 4.4 policy 函数签名（预留）

```typescript
function(
  hasPermission: boolean,  // 后端权限验证结果（必需）
  data?: any,              // 数据参数（可选）
  ref?: Ref                // 组件/按钮引用（可选）
): boolean
```

**policy 函数职责**：

- ✅ 基于 hasPermission 控制显示/隐藏/禁用
- ✅ 基于用户角色/属性提供不同提示（与人相关）
- ❌ 不判断数据状态（如 `row.status === 'processing'`）
- ❌ 不判断时间规则（如"最近三个月"）

### 4.5 禁止使用的字段

❌ rowFilter、actions、permissionCode、fallbackBehavior、itemResourceCode

---

## 5. 总结

| 资源类型 | 允许操作 | 禁止操作 |
|---------|---------|---------|
| 自定义 SQL | 引用 sqlCode | 生成 SQL 语句 |
| BFF 脚本 | 调用 scriptName | 生成脚本代码 |
| 权限点 | 协议预留，待能力上线后消费 | 创建权限点 |

**用户需要新建资源时**：引导用户到平台手动配置，获取 code/name 后帮助配置到 Schema。
