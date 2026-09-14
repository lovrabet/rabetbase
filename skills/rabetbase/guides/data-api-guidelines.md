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
| **有界读取** | 按接口限制与业务预算设定调用和读取上限；超限不能截断结果并冒充完整数据 |

---

## 编写前强制检查清单

在编写任何数据访问代码前，**必须先完成以下检查**：

- [ ] 使用 `rabetbase dataset detail --code <数据集编码> --format json` 获取数据集完整字段与操作信息
- [ ] 核对字段名（区分大小写）、类型、是否必填、枚举选项
- [ ] 分析数据集依赖关系（主外键约束）
- [ ] 识别外键字段，确定下拉框数据来源

**注意**：系统自动维护字段（id、create_time 等）的处理方式，详见 `backend-function.md`。

**CLI 信封、`data.*` 键位、平台 `get-driven-data` 与 CLI 归一化对照**：单一权威见 [`references/rabetbase-dataset-detail.md`](../references/rabetbase-dataset-detail.md)。

**不要复制示例字段名。** 本指南里的表名、字段名、枚举值只用于说明形态；真实代码的字段与连接事实来自当前项目的 `rabetbase dataset detail` 输出。同库关系读取 `data.relations[]`，跨库登记事实单独使用 `dataset cross-relation-list` 核对。

---

## 先判定数据访问路线

在决定写 SQL、Backend Function 还是直接查数据前，先看 `rabetbase dataset detail --code <数据集编码> --format json` 的 `source`：

| `source` | 建议路线 |
|----------|----------|
| `METADATA` | **不要走 SQL / aggregate**；优先 `filter` / `getOne` / `create` / `update` / `delete` 等平台返回的标准操作；Backend Function HOOK 可挂载，operation 以后端返回为准 |
| `CUSTOM` / `DB` | 可继续评估 SQL、Backend Function 或标准数据接口 |

---

## 元数据取值规则

字段事实以 `rabetbase dataset detail` 的归一化输出为准：

| 目标 | 路径 | 规则 |
|------|------|------|
| 字段名 | `data.fields[].name` | 查询、写入、SQL 列名均使用它，不猜通用字段名 |
| 聚合列名 | `aggregate[].column` | SDK aggregate 定义使用 `column`，不要写旧别名 `field` |
| 必填字段 | `data.fields[].required` | `true` 表示创建/写入时需处理，平台自动维护字段除外 |
| 枚举/选择值 | `data.fields[].options[].value` | 写入持久化 `value`，不要写展示用的 `label` |
| 同库关系 | `data.relations[]` / `dataset relations` | 确认端点、基数和查询能力，不按同名字段推断关系 |
| 跨库登记关系 | `dataset cross-relation-list` | 与业务合同分别核对，不用同库关系列表替代 |

复合键 BFF 可依据已确认的业务合同和两端字段事实实现，不要求先拆成单字段 Relation；平台登记关系不等于执行能力。完整键、查询顺序和授权边界见 [跨库 BFF 查询与拼接](cross-database-bff.md)。

常用投影：

```bash
rabetbase dataset detail --code <数据集编码> --format compress \
  --jq '.data.fields[] | {name, displayName, type, required, options}'
```

---

## 真实行数据：交接给 `lovrabet`

不需要真实行数据时，`lovrabet` CLI 可以不装；本 skill 的结构/发布主路径始终基于 **`rabetbase`**（`dataset detail`、`sql exec` 验证已发布 SQL 等）。

一旦要验证真实业务行数据，必须交接给 **`lovrabet data filter`** / **`lovrabet data getOne`**（与 `@lovrabet/sdk` 相同语义）。`rabetbase` 与 `lovrabet` Skill 不互斥：不可用时**报告阻断**并提示安装，不要静默安装或修复运行态 CLI，也不要把 `rabetbase sql exec` 当成行数据查询的通用替代。

若本机已安装 **Lovrabet 运行时 CLI**（npm 包 **`@lovrabet/lovrabet-cli`**，命令名 **`lovrabet`**，须 **≥ 2.0**），可在终端对照调试前端 / Backend Function 里的 `filter`、`getOne`。低于 2.0 请先升级：`npm install -g @lovrabet/lovrabet-cli@^2.0.0`。自检：`lovrabet --version`。

**注意：**

| 点 | 说明 |
|----|------|
| **结构 vs 行数据** | 结构用 `rabetbase dataset detail`。真实行数据必须用 `lovrabet data filter` / `data getOne`。 |
| **未安装时** | 报告阻断，并提示安装 `lovrabet` Skill（`npx skills add lovrabet/lovrabet-cli`）和 CLI。不要由本 Skill 静默安装或修复。**`rabetbase sql exec` 只验证已发布 SQL 的可执行性与结果结构。** |
| **≥ 2.0** | 本节 `data` 子命令以 **2.0+** 为准；版本不符时先升级。 |
| **配置与认证** | `lovrabet` 与 `rabetbase` 的配置项、鉴权方式可能不完全相同，按各自 CLI 文档配置。 |
| **详细用法** | 以 `lovrabet data --help`、`lovrabet data filter --help` 为准（参数多为 `--code` + `--params` JSON）。 |

示例（仅作形态参考，需本机已安装且已登录/配置）：

```bash
lovrabet data filter --code <数据集code> --params '{"where":{...},"currentPage":1,"pageSize":10}' --format json
lovrabet data getOne --code <数据集code> --params '{"id":123}' --format json
```

---

## 步骤 1: 获取数据集详情

```bash
rabetbase dataset detail --code <数据集编码> --format json
```

字段表、归一化规则、`dbtable`、jq 示例：**见 [`references/rabetbase-dataset-detail.md`](../references/rabetbase-dataset-detail.md)**。

做行数据验证时，补一个现实预期：

- **不要默认 `filter()` 返回完整字段**。列表接口常为展示做裁剪，某些字段可能缺失。
- 需要确认“某条记录的完整字段”或依赖关键字段时，优先 `getOne({ id })`。
- 遇到 `USER` 类型字段时，留意同名的 `_label` 扩展对象（如 `creator_id_label`、`assignee_id_label`），很多展示信息已在其中，无需立刻反查 SQL 或额外写 Backend Function。

### 错误处理

**CLI 命令执行失败时**：

检查以下常见问题：
1. 数据集编码是否正确
2. 是否有权限访问该数据集
3. CLI 是否已正确认证（`rabetbase auth`）

**Backend Function 中的错误处理**：

```javascript
export default async function validateRelatedRecord(params, context) {
  const result = await context.client.models["dataset_0123456789abcdef0123456789abcdef"].filter({
    where: { id: { $eq: params.related_id } },
    select: ['id', '<displayField>']
  });

  if (!result.tableData || result.tableData.length === 0) {
    throw new Error(`关联记录 ${params.related_id} 不存在`);
  }

  // 继续业务逻辑...
}
```

---

## 步骤 2: 分析字段依赖关系

### 主外键关系分析

**从数据集详情中获取依赖关系**，重点关注：

```
主数据集依赖关系示例：
1. related_id → related_table.id (外键)
   - 创建主记录时 related_id 必须存在
   - 下拉框数据来自关联数据集
   - 需要显示关联记录名称，存储 related_id

2. item_id → item_table.id (外键)
   - 下拉框数据来自另一关联数据集
   - 需要显示条目名称，存储 item_id

3. status (无外键，`type` 为 SELECT)
   - 枚举字段，直接使用 `data.fields[].options` 数组
   - 不需要额外接口请求
```

### 下拉框数据来源推理

| 字段类型 | 识别方式 | 数据来源 | 处理方式 |
|---------|---------|---------|---------|
| **外键字段** | `relations[]` 中有对应记录 | 关联的数据集 | 调用关联数据集的 filter 接口 |
| **枚举字段** | `type === "SELECT"` 且 `options` 非空 | `data.fields[].options` | 写入 `option.value`，展示 `option.label` |
| **级联选择** | 父字段决定子字段 | 父字段变化时动态加载 | 监听父字段变化，动态加载子选项 |

枚举/选择字段的 `label` 只用于展示，写入 Backend Function、SDK 或 SQL 参数时使用对应 `value`。`value` 的类型以数据集详情为准，可能是字符串、数字或其它平台约定类型。

**前端示例**：

```tsx
// 分析：status 字段 type === "SELECT"，options 已有枚举值
// 直接从字段定义取，不需要接口请求
const statusOptions = field.options.map(o => ({ label: o.label, value: o.value }));

// 分析：related_id 是外键，关联另一个数据集
// 需要获取关联记录列表作为下拉框数据
const [relatedRecords, setRelatedRecords] = useState([]);

useEffect(() => {
  client.models.related.filter({
    select: ['id', '<displayField>'],
    where: { status: { $eq: 'active' } }
  }).then(result => {
    setRelatedRecords(result.tableData || []);
  });
}, []);

// 下拉框配置
const relatedOptions = relatedRecords.map(record => ({
  label: record["<displayField>"],
  value: record.id,
}));
```

**Backend Function 示例**：

```javascript
// 分析：related_id 是外键
// 创建记录前需要校验关联记录是否存在

const related = await relatedDS.getOne({ id: params.related_id });
if (!related) {
  throw new Error(`关联记录 ${params.related_id} 不存在`);
}
```

---

## 步骤 3: 使用真实接口

### 禁止使用 Mock 数据

```tsx
// ❌ 错误：使用 mock 数据
const STATIC_OPTIONS = [
  { label: '示例A', value: 1 },
  { label: '示例B', value: 2 },
];

// ✅ 正确：使用真实接口
const [relatedRecords, setRelatedRecords] = useState([]);
useEffect(() => {
  client.models.related.filter({
    select: ['id', '<displayField>'],
  }).then(result => {
    setRelatedRecords(result.tableData || []);
  });
}, []);
```

### 标准接口调用方式

**前端**：

```tsx
import { createClient } from '@lovrabet/sdk';

const client = createClient({ appCode: 'your-app-code' });

// 查询列表
const result = await client.models.dataset_0123456789abcdef0123456789abcdef.filter({
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
  primary: 'dataset_0123456789abcdef0123456789abcdef', // 数据集: <displayName> | 数据表: <tableName>
};

const result = await models[TABLES.primary].filter({
  where: { status: { $eq: 'active' } },
  select: ['id', '<fieldName>'],
});
```

---

## 步骤 4: 性能优化（避免循环访问）

### 场景识别

代码中出现**按主记录逐条调用接口的 N+1 查询**时，必须优化；按接口合同执行的有界分批和分页循环属于批量读取：

```tsx
// ❌ 性能灾难：N 次接口调用
for (const row of rows) {
  const related = await getRelatedRecord(row.related_id);
}
```

### 优化方案选择

```
循环逐条访问接口（N 次调用）
    │
    ├─ 不同数据库连接 → BFF 分库批量读取，按完整键拼接
    │
    └─ 同一连接
        ├─ 1:1/N:1 且已确认支持关联查询 → filter 多表关联查询
        └─ 其他场景 → 有界批量查询，或已确认支持的库内 Custom SQL
```

先核对数据源及查询能力，再选择方案；基数或平台存在 Relation 不能单独证明可自动 JOIN。跨库场景必须阅读 [跨库 BFF 查询与拼接](cross-database-bff.md)，分别处理展示补充、关联筛选、全局排序和统计。

### 方案 1：多表关联查询（推荐）

<span style={{fontSize: '0.9em', color: '#888'}}>v1.2.0+</span>

**适用于**：同一数据库连接内、已确认支持关联查询的 1:1 或 N:1 关系（主记录→关联记录、成员→组织等）。以下示例不能用于推断跨连接 JOIN 能力。

```tsx
// ✅ 一次查询，自动 JOIN
const result = await client.models.primary.filter({
  select: [
    "id", "record_no",
    "related.name",     // 关联表字段
    "related.level"     // 关联表字段
  ],
  where: {
    status: { $eq: "pending" },
    "related.level": { $eq: "important" }
  },
});

// result.tableData[0].related.name ← 直接可用
```

### 方案 2：批量查询

**适用于**：不支持多表关联或需要分开读取时；跨库拼接在可信 BFF 中完成。

以下步骤用于向已选定主记录补充展示字段。外表参与筛选、排序或统计时，先按 [跨库 BFF 查询与拼接](cross-database-bff.md)确定完整查询顺序，不能先取主表当前页后套用本流程。

1. 从已授权主记录提取有效完整键并去重；空键集合跳过查询，合法的 `0` 不用真值过滤排除。
2. 单字段键可用 `$in` 批量过滤；复合键保留字段配对，不能用两个独立 `$in` 替代联合条件。
3. 按目标接口限制分批并读取所有匹配页，只取拼接和输出必需的已授权字段。批量查询不保证一次调用返回完整数据；结果不完整或请求失败时默认返回错误。部分结果只按已有接口合同输出，规则见 [跨库 BFF 查询与拼接](cross-database-bff.md)。
4. 完整结果按完整键和基数组装；单目标关系出现重复键时报告冲突，一对多按合同收集为集合，不能静默覆盖。
5. 只有完整查询成功才判断未匹配；允许缺失的单目标展示关系保留主记录并返回 `null`，必需关系缺失按业务合同失败。

具体执行顺序、复合键示例与失败语义见 [跨库 BFF 查询与拼接](cross-database-bff.md)。

### 性能对比

| 方案 | 调用量取决于 | 使用前提 |
|------|-------------|----------|
| 循环单条 | 主记录数量 | 容易产生 N+1，应优先批量化 |
| 库内关联 | 接口分页与查询计划 | 同连接且查询能力已确认 |
| 批量查询 | 去重键数量、批次和匹配结果页数 | 完整读取并在预算内组装 |

调用次数不能直接换算为固定耗时；按真实查询计划、数据量和接口限制验证。

---

## 步骤 5: 批量操作优化

### 大量数据写入场景

**场景**：批量复制 100 条记录

```javascript
// ❌ 错误：循环创建（100 次接口调用）
for (const row of rows) {
  await model.create(row);
}

// ✅ 正确：使用自定义 SQL（1 次接口调用）
await context.client.sql.execute({
  sqlCode: "batch-copy-records",
  params: { sourceId, targetId }
});
```

**自定义 SQL 创建流程**：

1. 优先使用 `rabetbase sql create` 创建 SQL（此时平台端已创建记录），再在本地同步目录中编辑并通过 `sql push` 更新
2. 获得返回或已存在的 `sqlCode`
3. 在代码中调用 `context.client.sql.execute({ sqlCode, params })`

详见：`sql-creation-workflow.md`

---

## 检查清单

### 前端页面开发

- [ ] **已获取数据集详情**：使用 CLI 命令获取字段和依赖关系
- [ ] **外键字段已识别**：下拉框数据来自关联数据集接口
- [ ] **枚举字段已处理**：展示使用 `options[].label`，写入使用 `options[].value`
- [ ] **未使用 mock 数据**：所有下拉框数据来自真实接口
- [ ] **避免 N+1**：同连接且能力已确认时使用多表关联；跨连接交由 BFF 有界批量读取
- [ ] **字段名正确**：使用 `data.fields[].name`（列名），区分大小写，与数据集定义一致
- [ ] **系统字段已识别**：结合 `data.dbtable` 与 `backend-function.md`，创建/更新时间等按平台约定不传
- [ ] **关联表使用表名**：多表关联时使用 `tableName.fieldName` 格式

### Backend Function 开发

- [ ] **已获取数据集详情**：使用 CLI 命令获取字段和依赖关系
- [ ] **主外键关系已分析**：理解表间关联关系
- [ ] **必填字段已覆盖**：`data.fields[].required === true` 的业务字段已提供值
- [ ] **枚举字段写入正确**：写入 `options[].value`，不是展示 `label`
- [ ] **外键校验已处理**：创建/更新前校验外键有效性
- [ ] **系统字段未设置**：主键自增、系统时间等按平台约定不手工传入，详见 `backend-function.md`
- [ ] **批量读取完整**：按完整键去重、分批和分页；查询失败、不完整或唯一关系重复匹配均明确报告
- [ ] **批量操作已优化**：大量数据操作使用自定义 SQL

### SQL 开发

- [ ] **已获取数据集详情**：通过 `data.dbtable.tableName` 确认表名，通过 `data.dbtable.dbId` 确认数据库
- [ ] **必填字段已确认**：INSERT 包含所有业务必填字段（`data.fields[].required === true` 中需由调用方传入的列）
- [ ] **主外键关系已分析**：JOIN 条件使用 `data.dbtable.pkField` 等
- [ ] **SQL 已验证**：使用 `rabetbase sql validate` 验证
- [ ] **SELECT 已测试**：使用 `rabetbase sql exec` 测试

---

## 相关指南

- **前端页面开发**：`frontend-development.md`
- **SQL 创建工作流**：`sql-creation-workflow.md`
- **Backend Function 规范**：`backend-function.md`
