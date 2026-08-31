# SDK 编码规则与 API 约束

> 目标：约束 AI 在前端与 Node 环境中生成 Lovrabet SDK 代码时的用法，杜绝凭借通用经验瞎猜 API 结构。
>
> 适用范围：前端 / Node 环境中通过 `@lovrabet/sdk` 初始化后的 `client` 调用。
> Backend Function 脚本中的 `context.client` 由平台注入，API 子集与前端 SDK 有差异；编写 Backend Function 时以 `backend-function.md` 为准。

## 何时使用

当任务涉及以下动作时，必须阅读并遵守本指南：
* 初始化 Lovrabet Client
* 编写模型/数据集的查询与写入代码
* 在前端或中台服务中执行自定义 SQL
* 在前端调用 Backend Function (BF)
* 处理 API 返回的错误或业务异常

不要把本指南里的 `createClient`、`registerModels` 等前端 / Node SDK 初始化能力套用到 Backend Function。Backend Function 内只使用平台注入的 `context.client`。

生成或更新项目里的 `src/api/api.ts` / `client.ts` 时，先 `rabetbase api pull --format compress`，再遵守 [`sdk-client-generation.md`](sdk-client-generation.md)。浏览器子应用的默认 client 是 Cookie + `...LOVRABET_SDK_CONFIG`，不要把下面的服务端 `accessKey` 示例写进 `src/api/client.ts`。

## 初始化规则

必须使用 `createClient` 命名导出，禁止使用 `new LovrabetClient()`：

```typescript
import { createClient } from "@lovrabet/sdk";

const client = createClient({
  appCode: "your-app-code",
  authMode: "client-ak", // 必须显式声明；否则一律走 cookie 模式，accessKey 会被忽略
  accessKey: process.env.RABETBASE_ACCESS_KEY, // 仅在服务端使用
  models: [
    { tableName: "users", datasetCode: "39f758e7b38c476b8bb3996771a601a1", alias: "users" }
  ],
});
```

认证模式必须显式声明,不会按字段自动推断:

* 仅 `accessKey` → `authMode: "client-ak"`
* `accessKey`(+可选 `secretKey`)签名，或已配对的预计算 `token` + `timestamp` → `authMode: "openapi"`（凭据通过 `X-Token` / `X-Time-Stamp` 请求头传递，不是 Authorization Bearer；用 `token` 时必须同时提供配对的 `timestamp`，否则首次请求报 `timestamp is required`）
* 浏览器 Cookie 环境 → 省略 `authMode`(默认 cookie)

## 1. 模型查询 (Filter API)

这是操作模型（表）的**最高优** API。

### 强制参数名称
绝不允许用错以下参数名：
* ❌ `fields` -> ✅ `select`
* ❌ `sort` -> ✅ `orderBy`
* ❌ `page` / `limit` -> ✅ `currentPage` / `pageSize`

### 强制操作符
`where` 条件中**禁止直接写值**，必须使用操作符：
* ❌ `where: { status: 'active' }`
* ✅ `where: { status: { $eq: 'active' } }`

支持的操作符：`$eq`, `$ne`, `$gt`, `$lt`, `$gte`（或兼容旧写法 `$gteq`）, `$lte`（或兼容旧写法 `$lteq`）, `$contain`, `$startWith`, `$endWith`, `$in`, `$notNull`。`$gteq` / `$lteq` 是后端保留的别名，与 `$gte` / `$lte` 映射到同一 SQL 比较，新代码推荐用 `$gte` / `$lte`。

逻辑组合：
```typescript
where: {
  $and: [
    { age: { $gte: 18 } },
    { $or: [{ status: { $eq: "pending" } }, { status: { $eq: "processing" } }] }
  ]
}
```

### 多表关联（自动 JOIN）
* 仅支持 1:1 或 N:1
* 关联字段引用必须使用**表名**（如 `profile.username`），不能用 datasetCode。
* 只有提前通过 CLI 命令分析出有外键关联的，才能这么写。

```typescript
// 示例：查询文章及其关联的作者信息
const result = await client.models.article.filter({
  select: ["id", "title", "profile.username"],
  where: { "profile.is_signed": { $eq: true } }
});
```

### 写入与删除操作（SDK >= 1.2.0）

从 SDK v1.2.0 开始，update 和 delete 支持对象模式（推荐，与 Backend Function 一致）和兼容模式两种写法，单次最多 **1000 条**记录。

#### 更新（Update）

**接口**：
```typescript
client.models[`dataset_${code}`].update({
  id: number | string | (number | string)[]; [key: string]: any
})
```

**示例**：
```typescript
// 单条更新
await client.models.customer.update({ id: 1001, status: 'active' });

// 批量更新
await client.models.customer.update({
  id: [1, 2, 3, 4, 5],
  status: 'active',
  updateTime: new Date().toISOString()
});
```

**注意事项**：
- 单次最多更新 **1000 条**
- 原子操作：要么全部成功，要么全部失败
- 不能批量修改主键字段
- 不能将必填字段设置为空值

#### 删除（Delete）

**接口**：
```typescript
client.models[`dataset_${code}`].delete({ id: number | string | (number | string)[] })
```

**示例**：
```typescript
// 单条删除
await client.models.customer.delete({ id: 1001 });

// 批量删除
const inactiveUsers = await client.models.customer.filter({
  where: { lastLoginTime: { $lt: '2026-01-01' } },
  select: ['id']
});

// filter() 返回 ListResponse（{ tableData, paging, tableColumns }），列表数据在 tableData
await client.models.customer.delete({
  id: inactiveUsers.tableData.map(u => u.id)
});
```

**注意事项**：
- 单次最多删除 **1000 条**
- 删除操作不可逆，建议先备份
- 如果有外键约束，需要先删除关联数据

#### 分批处理超过 1000 条的数据

当数据量超过 1000 时，需要手动分批：

```typescript
async function updateInBatches(ids: number[], batchSize = 1000) {
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    await client.models.customer.update({ id: batch, status: 'processed' });
    console.log(`已处理 ${i + batch.length} / ${ids.length}`);
  }
}
```

### 别名模式（Alias Pattern）

在前端 / Node SDK 中给数据集配置 `alias` 后，可用别名访问模型，批量操作同样支持。此能力不适用于 Backend Function 的 `context.client`。

别名在初始化时配置，最常见的是直接写进 `createClient` 的 `models`：

```typescript
import { createClient } from "@lovrabet/sdk";

const client = createClient({
  appCode: "your-app-code",
  models: [
    { tableName: "orders", datasetCode: "abc123", alias: "primary" },
    { tableName: "order_items", datasetCode: "def456", alias: "detail" },
  ],
});

// 使用别名批量操作
await client.models.primary.update({ id: [1, 2, 3], status: "active" });
```

`registerModels` 是 `@lovrabet/sdk` 的独立命名导出（不是 client 实例方法），用于把一份完整 `ModelsConfig`（`{ appCode, models }`）注册到全局配置表，再由 `createClient` 按配置名引用（CLI 生成的 `src/api/*.ts` 就是这样自动注册的）：

```typescript
import { createClient, registerModels } from "@lovrabet/sdk";

registerModels(
  {
    appCode: "your-app-code",
    models: [
      { tableName: "orders", datasetCode: "abc123", alias: "primary" },
      { tableName: "order_items", datasetCode: "def456", alias: "detail" },
    ],
  },
  "prod",
);

const client = createClient("prod"); // 或 createClient({ apiConfigName: "prod", authMode: "openapi", token, timestamp, env })（用 token 时必须显式 authMode 且带配对 timestamp）
await client.models.primary.update({ id: [1, 2, 3], status: "active" });
```

## 2. 自定义 SQL (SQL API)

### 强制返回值处理
SDK 中的 SQL API 返回的是**业务数据层**，包含 `execSuccess` 和 `execResult`：
* **必须**判断 `execSuccess`，不能直接读结果。
* 这与 Backend Function 环境中直接返回数组（不带 execResult）有本质区别，**不要混用**。

```typescript
// ✅ 前端/Node SDK 调用 SQL：
const data = await client.sql.execute<MyRowType>({
  sqlCode: "fc8e7777-06e3847d",
  params: { userId: "123" },
});

if (data.execSuccess && data.execResult) {
  console.log(data.execResult); // T[]
} else {
  console.error("业务级SQL执行失败");
}
```

## 3. 前端 / Node 调用 Backend Function (BF) API

### 强制返回值处理
调用 Backend Function 返回的直接是你在脚本中 `return` 的业务数据对象。
* 这里**没有** `execSuccess` 或 `execResult`。

```typescript
// ✅ 调用 Backend Function
const dashboard = await client.bff.execute<DashboardData>({
  scriptName: "getUserDashboard",
  params: { userId: "123" },
});

// 直接使用业务数据
console.log(dashboard.userCount);
```

## 4. 前端 / Node 调用 Personal Backend Function API

Personal Backend Function 通过数值型 `scriptId` 定位，`params` 必须是业务契约允许的对象。返回值是个人函数直接 `return` 的业务数据，不带 `execSuccess` 或 `execResult` 包装。

```typescript
interface DashboardData {
  userCount: number;
}

const personalBff = client?.personal?.bff;
if (typeof personalBff?.execute !== "function") {
  throw new Error("client.personal.bff.execute is not available");
}

const dashboard = await personalBff.execute<DashboardData>({
  scriptId: 123,
  params: { userId: "123" },
});

if (!dashboard || typeof dashboard.userCount !== "number") {
  throw new Error("Unexpected Personal Backend Function response");
}
```

认证边界：

- 浏览器 client 使用当前登录 Cookie。不要把 Cookie、AccessKey、SecretKey 或 token 写进前端源码、构建变量、页面配置或日志。
- Node 服务使用 Client AK 时，凭据只保存在服务端，并显式设置 `authMode: "client-ak"`。
- Personal Backend Function 不支持 `authMode: "openapi"`。

兼容与 `undefined` 安全规则：

- 可选链只用于读取能力：`const personalBff = client?.personal?.bff`。
- 禁止调用 `client.personal?.bff?.execute?.(...)`。方法缺失时该写法会静默返回 `undefined`，无法与业务函数的空返回可靠区分。
- 能力不存在时立即抛出明确错误，提示升级到包含 `client.personal.bff.execute` 的 SDK 版本。
- 检测通过后通过 `personalBff.execute(...)` 调用；不要提取成 `const execute = personalBff.execute` 后裸调用，以免丢失方法上下文。
- 不以 `result === undefined` 作为能力检测；根据个人函数已确认的返回契约校验必需字段和空态。
- 接入页面前，先通过 `lovrabet personal-bff exec --id <id> --params '<json>' --format compress` 核对同一 `scriptId` 的字段、空态和错误形状。

## 异常处理底线

所有通过 `client` 发起的网络调用都可能抛出 HTTP 级别的错误，AI 必须养成使用 `try...catch` 包裹代码的习惯，并识别 `LovrabetError`。

```typescript
import { LovrabetError } from "@lovrabet/sdk";

try {
  const result = await client.models.users.getOne({ id });
} catch (error) {
  if (error instanceof LovrabetError) {
    console.error("HTTP/框架级错误:", error.message, error.code);
  } else {
    console.error("其他运行时错误:", error);
  }
}
```

## AI 自检清单
每次生成 SDK 相关代码时，AI 应隐式自问：
* [ ] 是否把 `select` 写成了 `fields`？
* [ ] 是否把 `orderBy` 写成了 `sort`？
* [ ] `where` 对象里是否老老实实带了 `$eq` 等操作符？
* [ ] 处理 SQL 的返回值时，判断了 `execSuccess` 吗？
* [ ] 处理 Backend Function 的返回值时，是不是直接使用了业务数据？
* [ ] 调 Personal Backend Function 前，是否显式校验了 `client.personal.bff.execute`，且没有使用可选调用？
* [ ] 浏览器代码是否只依赖登录 Cookie，没有写入 Client AK 或其他凭据？
* [ ] 是否按已确认契约校验 Personal Backend Function 的返回形状，而不是用 `undefined` 猜测能力状态？
* [ ] 加入了 `try...catch` 块防止整个应用崩溃吗？
