# 数据接口访问规范

> **目标**：统一前后端数据接口访问方式，确保性能最优、数据准确
>
> **适用范围**：前端页面开发、Backend Function 编写、SQL 查询编写

---

## 核心原则

| 原则 | 说明 |
|------|------|
| **先获取数据集详情** | 使用 CLI 命令获取字段信息和依赖关系 |
| **分析主外键关系** | 理解表间关联，正确处理下拉框数据 |
| **使用真实接口** | 不使用 mock 数据，直接调用数据集接口 |
| **避免循环访问** | 批量查询替代循环单个查询 |
| **性能优先** | 单次接口调用次数 < 50 次 |

---

## 编写前强制检查清单

在编写任何数据访问代码前，**必须先完成以下检查**：

- [ ] 使用 `rabetbase dataset detail --code <数据集编码> --verbose --format json` 获取数据集完整字段信息
- [ ] 核对字段名（区分大小写）、类型、是否必填
- [ ] 分析数据集依赖关系（主外键约束）
- [ ] 识别外键字段，确定下拉框数据来源

**注意**：系统自动维护字段（id、create_time 等）的处理方式，详见 `backend-function.md`。

---

## 步骤 1: 获取数据集详情

**使用 CLI 命令获取字段信息和依赖关系**：

```bash
rabetbase dataset detail --code orders --format json
```

**返回信息包含**：

| 信息项 | 路径 | 说明 | 用途 |
|-------|------|------|------|
| 数据集基础信息 | `data.dataset` / `data.modelId` | 数据集 ID、UUID code、名称 | 接口路径中的 modelCode 即为 UUID |
| 字段列表 | `data.fields[]` | 列名、展示名、类型、主键标志、系统字段标志 | 表单字段校验、查询条件 |
| 依赖关系 | `data.relations[]` | 主外键关系（无关联时为空数组） | 下拉框数据来源、关联查询 |
| 数据库配置 | `data.dbtableConfig` | dbId、tableName、pkField、系统字段名 | SQL 编写时使用 |

---

### CLI 命令调用示例

```bash
# 获取数据集详情
rabetbase dataset detail --code orders --format json
```

**返回数据结构**（以"迭代"数据集为例）：

```json
{
  "data": {
    "modelId": 1004496,
    "modelCode": "0b59efad4b184f3181470156ec19bf52",
    "name": "迭代",
    "dataset": {
      "datasetId": 1004496,
      "datasetCode": "0b59efad4b184f3181470156ec19bf52",
      "datasetName": "迭代",
      "tableName": "iterations",
      "sourceType": "DB_TABLE"
    },
    "fields": [
      {
        "id": 724353,
        "name": "id",
        "displayName": "迭代ID",
        "dbType": "BIGINT",
        "doType": "NUMBER",
        "pkField": true,
        "autoIncrement": true,
        "required": true,
        "systemRetain": null,
        "options": null
      },
      {
        "id": 724354,
        "name": "project_id",
        "displayName": "项目ID",
        "dbType": "BIGINT",
        "doType": "NUMBER",
        "pkField": false,
        "required": true,
        "systemRetain": null,
        "options": null
      },
      {
        "id": 724359,
        "name": "status",
        "displayName": "迭代状态",
        "dbType": "ENUM",
        "doType": "SELECT",
        "pkField": false,
        "required": true,
        "systemRetain": null,
        "options": [
          { "label": "planning", "value": "planning", "children": null },
          { "label": "in_progress", "value": "in_progress", "children": null }
        ]
      },
      {
        "id": 724361,
        "name": "created_at",
        "displayName": "创建时间",
        "dbType": "DATETIME",
        "doType": "DATETIME",
        "pkField": false,
        "required": true,
        "systemRetain": "createTime",
        "options": null
      },
      {
        "id": 724362,
        "name": "updated_at",
        "displayName": "更新时间",
        "dbType": "DATETIME",
        "doType": "DATETIME",
        "pkField": false,
        "required": true,
        "systemRetain": "updateTime",
        "options": null
      }
    ],
    "relations": [],
    "dbtableConfig": {
      "dbId": 10065,
      "tableName": "iterations",
      "pkField": "id",
      "createTimeField": { "fieldName": "created_at", "fieldType": "DATETIME" },
      "updateTimeField": { "fieldName": "updated_at", "fieldType": "DATETIME" },
      "creatorIdField": { "fieldName": null },
      "creatorNameField": { "fieldName": null }
    }
  }
}
```

### 字段结构说明

| 字段 | 含义 | 关键用途 |
|------|------|---------|
| `name` | **列名**（即接口参数中使用的字段名） | 查询/写入时用这个名字 |
| `displayName` | 人类可读展示名 | 表单 label、错误提示 |
| `dbType` | 数据库存储类型（BIGINT / VARCHAR / ENUM / DATE / DATETIME / TEXT） | SQL 编写时参考 |
| `doType` | **API 层类型**（NUMBER / TEXT / SELECT / DATETIME） | 接口调用时的类型约束 |
| `pkField` | `true` 表示主键字段 | 标识主键，不要在 create 时传入 |
| `autoIncrement` | `true` 表示自增主键 | create 时绝对不传 |
| `required` | 是否必填 | 表单校验、写入前检查 |
| `systemRetain` | 系统自动维护字段标志：`"createTime"` / `"updateTime"` / `null` | 非 `null` 的字段由系统自动填写，**create/update 时不传** |
| `options` | SELECT/ENUM 字段的枚举值数组 `{label, value, children}[]`；普通字段为 `null` | 下拉框直接使用，不需要额外请求 |

### 识别系统字段的规则

```
fields[].systemRetain 的值：
  "createTime"  → 创建时间字段，系统自动写入，create/update 不传
  "updateTime"  → 更新时间字段，系统自动写入，create/update 不传
  null          → 普通业务字段，需要手动传入
```

> 详见 `backend-function.md` 中的系统字段处理规范。

### 数据库配置（`dbtableConfig`）关键字段

| 路径 | 含义 | 用途 |
|------|------|------|
| `dbtableConfig.dbId` | 数据库 ID | SQL 编写时确定数据库 |
| `dbtableConfig.tableName` | 实际数据库表名 | SQL 中的表名 |
| `dbtableConfig.pkField` | 主键字段名（如 `"id"`） | JOIN 条件 |
| `dbtableConfig.createTimeField.fieldName` | 创建时间字段名 | 筛选、排序 |
| `dbtableConfig.updateTimeField.fieldName` | 更新时间字段名 | 筛选、排序 |
| `dbtableConfig.creatorIdField.fieldName` | 创建人 ID 字段（`null` 表示未配置） | 权限过滤 |

### 错误处理

**CLI 命令执行失败时**：

检查以下常见问题：
1. 数据集编码是否正确
2. 是否有权限访问该数据集
3. CLI 是否已正确认证（`rabetbase auth`）

**Backend Function 中的错误处理**：

```javascript
export default async function createOrder(params, context) {
  const customerResult = await context.client.models.customers.filter({
    where: { id: { $eq: params.customer_id } },
    select: ['id', 'customer_name']
  });

  if (!customerResult.tableData || customerResult.tableData.length === 0) {
    throw new Error(`客户 ${params.customer_id} 不存在`);
  }

  // 继续业务逻辑...
}
```

---

## 步骤 2: 分析字段依赖关系

### 主外键关系分析

**从数据集详情中获取依赖关系**，重点关注：

```
订单数据集依赖关系示例：
1. customer_id → customers.id (外键)
   - 创建订单时 customer_id 必须存在
   - 下拉框数据来自 customers 表
   - 需要显示 customer_name，存储 customer_id

2. product_id → products.id (外键)
   - 下拉框数据来自 products 表
   - 需要显示 product_name，存储 product_id

3. status (无外键，doType 为 SELECT)
   - 枚举字段，直接使用 fields[].options 数组
   - 不需要额外接口请求
```

### 下拉框数据来源推理

| 字段类型 | 识别方式 | 数据来源 | 处理方式 |
|---------|---------|---------|---------|
| **外键字段** | `relations[]` 中有对应记录 | 关联的数据集 | 调用关联数据集的 filter 接口 |
| **枚举字段** | `doType === "SELECT"` 且 `options` 非空 | `fields[].options` | 直接使用，无需额外请求 |
| **级联选择** | 父字段决定子字段 | 父字段变化时动态加载 | 监听父字段变化，动态加载子选项 |

**前端示例**：

```tsx
// 分析：status 字段 doType === "SELECT"，options 已有枚举值
// 直接从字段定义取，不需要接口请求
const statusOptions = field.options.map(o => ({ label: o.label, value: o.value }));

// 分析：customer_id 是外键，关联 customers 表
// 需要获取客户列表作为下拉框数据
const [customers, setCustomers] = useState([]);

useEffect(() => {
  client.models.customers.filter({
    select: ['id', 'customer_name'],
    where: { status: { $eq: 'active' } }
  }).then(result => {
    setCustomers(result.tableData || []);
  });
}, []);

// 下拉框配置
const customerOptions = customers.map(c => ({
  label: c.customer_name,
  value: c.id,
}));
```

**Backend Function 示例**：

```javascript
// 分析：订单表 customer_id 是外键
// 创建订单前需要校验客户是否存在

const customer = await customerDS.getOne({ id: params.customer_id });
if (!customer) {
  throw new Error(`客户 ${params.customer_id} 不存在`);
}
```

---

## 步骤 3: 使用真实接口

### 禁止使用 Mock 数据

```tsx
// ❌ 错误：使用 mock 数据
const CUSTOMER_OPTIONS = [
  { label: '客户A', value: 1 },
  { label: '客户B', value: 2 },
];

// ✅ 正确：使用真实接口
const [customers, setCustomers] = useState([]);
useEffect(() => {
  client.models.customers.filter({
    select: ['id', 'customer_name'],
  }).then(result => {
    setCustomers(result.tableData || []);
  });
}, []);
```

### 标准接口调用方式

**前端**：

```tsx
import { createClient } from '@lovrabet/sdk';

const client = createClient({ appCode: 'your-app-code' });

// 查询列表
const result = await client.models.dataset_XXXXXXXXXX.filter({
  where: { status: { $eq: 'active' } },
  select: ['id', 'name'],
  pageSize: 100,
});

const data = result.tableData || [];
```

**Backend Function**：

```javascript
// 使用 context.client 访问数据集
const models = context.client.models;
const TABLES = {
  customers: 'dataset_XXXXXXXXXX', // 数据集: 客户 | 数据表: customers
};

const result = await models[TABLES.customers].filter({
  where: { status: { $eq: 'active' } },
  select: ['id', 'customer_name'],
});
```

---

## 步骤 4: 性能优化（避免循环访问）

### 场景识别

代码中出现**循环内调用接口**时，必须优化：

```tsx
// ❌ 性能灾难：N 次接口调用
for (const order of orders) {
  const customer = await getCustomer(order.customer_id);
}
```

### 优化方案选择

```
循环访问接口（N 次调用）
    │
    ├─ 关联类型是 1:1 或 N:1？
    │   │
    │   ├─ 是 → 使用 filter 多表关联查询（推荐）
    │   │        一次查询，自动 JOIN
    │   │
    │   └─ 否 → 使用批量查询（$in）
    │        一次查询，手动映射
```

### 方案 1：多表关联查询（推荐）

<span style={{fontSize: '0.9em', color: '#888'}}>v1.2.0+</span>

**适用于**：1:1 或 N:1 关联（订单→客户、员工→部门）

```tsx
// ✅ 一次查询，自动 JOIN
const result = await client.models.orders.filter({
  select: [
    "id", "order_no",
    "customer.name",     // 关联表字段
    "customer.level"     // 关联表字段
  ],
  where: {
    status: { $eq: "pending" },
    "customer.level": { $eq: "VIP" }
  },
});

// result.tableData[0].customer.name ← 直接可用
```

### 方案 2：批量查询

**适用于**：复杂场景或不支持多表关联时

```tsx
// ✅ 一次查询，$in 批量获取
const customerIds = [...new Set(orders.map(o => o.customer_id))];
const customers = await client.models.customers.filter({
  where: { id: { $in: customerIds } },
  select: ['id', 'name']
});
const customerMap = Object.fromEntries(
  customers.tableData?.map(c => [c.id, c.name]) || []
);
orders.forEach(o => o.customerName = customerMap[o.customer_id]);
```

### 性能对比

| 方案 | 调用次数 | 耗时 |
|------|---------|------|
| 循环单条（100 条） | 100 次 | ~10 秒 |
| 多表关联 | 1 次 | ~0.1 秒 |
| 批量查询 | 1 次 | ~0.1 秒 |

---

## 步骤 5: 批量操作优化

### 大量数据写入场景

**场景**：批量复制 100 条订单记录

```javascript
// ❌ 错误：循环创建（100 次接口调用）
for (const order of orders) {
  await orderDS.create(order);
}

// ✅ 正确：使用自定义 SQL（1 次接口调用）
await context.client.sql.execute({
  sqlCode: "batch-copy-orders",
  params: { sourceOrderId: sourceId, targetOrderId: targetId }
});
```

**自定义 SQL 创建流程**：

1. 使用 CLI 命令 `rabetbase sql save --file <sql文件路径> --format json` 创建 SQL
2. 获得返回的 `sqlCode`
3. 在代码中调用 `context.client.sql.execute({ sqlCode, params })`

详见：`sql-creation-workflow.md`

---

## 检查清单

### 前端页面开发

- [ ] **已获取数据集详情**：使用 CLI 命令获取字段和依赖关系
- [ ] **外键字段已识别**：下拉框数据来自关联数据集接口
- [ ] **枚举字段已处理**：直接使用 `fields[].options`，`doType === "SELECT"` 的字段无需额外接口
- [ ] **未使用 mock 数据**：所有下拉框数据来自真实接口
- [ ] **无循环访问**：优先使用多表关联查询，或使用批量查询
- [ ] **字段名正确**：使用 `fields[].name`（列名），区分大小写，与数据集定义一致
- [ ] **系统字段已识别**：`systemRetain` 非 null 的字段（创建/更新时间等）不传入
- [ ] **关联表使用表名**：多表关联时使用 `tableName.fieldName` 格式

### Backend Function 开发

- [ ] **已获取数据集详情**：使用 CLI 命令获取字段和依赖关系
- [ ] **主外键关系已分析**：理解表间关联关系
- [ ] **外键校验已处理**：创建/更新前校验外键有效性
- [ ] **系统字段未设置**：`pkField: true`、`autoIncrement: true`、`systemRetain` 非 null 的字段不传入，详见 `backend-function.md`
- [ ] **无循环访问**：使用 filter + $in 批量查询
- [ ] **批量操作已优化**：大量数据操作使用自定义 SQL

### SQL 开发

- [ ] **已获取数据集详情**：通过 `dbtableConfig.tableName` 确认表名，通过 `dbtableConfig.dbId` 确认数据库
- [ ] **必填字段已确认**：INSERT 包含所有必填字段（`required: true` 且 `systemRetain` 为 null 的字段）
- [ ] **主外键关系已分析**：JOIN 条件使用 `dbtableConfig.pkField`
- [ ] **SQL 已验证**：使用 `rabetbase sql validate` 验证
- [ ] **SELECT 已测试**：使用 `rabetbase sql exec` 测试

---

## 相关指南

- **前端页面开发**：`frontend-development.md`
- **SQL 创建工作流**：`sql-creation-workflow.md`
- **Backend Function 规范**：`backend-function.md`
