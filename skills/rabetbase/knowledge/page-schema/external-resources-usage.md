---
name: PageSchema-external-resources-usage
description: 外部资源（自定义SQL、BFF脚本、权限点）在PageSchema中的消费规范，Skill只引用不生成。先验知识：低代码引擎搭建协议、Ant Design 5 组件知识。
tags: [page-schema, external-resources, custom-sql, bff, permission]
---

# 外部资源使用规范

> **先验知识**：默认已具备低代码引擎搭建协议和 Ant Design 5 组件知识。

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

## 4. 权限点使用规范

当前标准页面权限由 `LrSmartTable` 运行时内置逻辑处理，Agent 只维护 `LrSmartTable` schema，不生成权限字段。

### 4.1 当前实现

表格会在运行时过滤无权限按钮：

- `toolbar.globalButtons`：按新增动作 `create` 校验。
- `toolbar.exportButtons.buttons`：按按钮上的 `actionType` 校验，常见值为 `export` / `delete`。
- `operateButtons`：按数组下标映射为 `update` / `detail` / `delete` 校验。

hard rule：`operateButtons` 当前按数组下标硬编码权限动作，顺序必须严格保持 `[update, detail, delete]`。不要调换、插入额外按钮或省略中间位置；第 4 个及之后按钮不会进入当前权限过滤。

### 4.2 当前权限链路

运行时读取 `_context.appHelper.yt.pointPermitMap`：

```text
datasetCode -> {
  DATA_CREATE?: boolean,
  DATA_UPDATE?: boolean,
  DATA_DELETE?: boolean,
  DATA_DETAIL?: boolean,
  DATA_EXPORT?: boolean
}
```

当表格配置了 `datasetCode` 时，使用该数据集对应权限；未配置 `datasetCode` 时，使用 `pointPermitMap` 中第一组权限。

动作映射：

| actionType | 权限 key |
|------------|----------|
| `create` | `DATA_CREATE` |
| `update` | `DATA_UPDATE` |
| `delete` | `DATA_DELETE` |
| `detail` | `DATA_DETAIL` |
| `export` | `DATA_EXPORT` |

### 4.3 Agent 规则

- 不生成任何权限字段。
- 批量导出 / 批量删除等 `exportButtons.buttons[]` 需要明确动作语义时，可配置 `actionType`。

---

## 5. 总结

| 资源类型 | 允许操作 | 禁止操作 |
|---------|---------|---------|
| 自定义 SQL | 引用 sqlCode | 生成 SQL 语句 |
| BFF 脚本 | 调用 scriptName | 生成脚本代码 |
| 权限点 | 使用当前表格内置权限过滤 | 生成权限字段或创建权限点 |

**用户需要新建资源时**：引导用户到平台手动配置，获取 code/name 后帮助配置到 Schema。
