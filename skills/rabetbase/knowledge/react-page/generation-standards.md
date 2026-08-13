# React 自定义页面生成规范

本规范用于编写或修改 React 自定义页面的完整 `codeContent`。执行命令流程、参数和发布规则以 [`react-custom-page-workflow.md`](../../guides/react-custom-page-workflow.md) 为准。

## 使用方式

在生成 JSX、CSS 或词包前，先阅读本规范，并按需加载以下文档：

- 选择或组合 UI 组件时，阅读 [`components.md`](components.md)
- 访问数据集时，先确认数据集事实，并按“数据客户端文档”拉取目标数据集的 SDK 契约

未在这些文档中登记的规范、组件或方法，不得假设存在。

## 固定实现约束

| 范畴     | 约束                                                                                                                      |
| -------- | ------------------------------------------------------------------------------------------------------------------------- |
| 提交内容 | `page react-update` 提交完整页面文件，未提交的已有文件会被删除                                                            |
| 推荐结构 | 页面入口仍为 `src/app/index.jsx`；其余文件和组件目录可按页面需要组织，`src/app/index.css`、`src/locales/index.js`、`src/components/*.jsx` 仅为示例 |
| 组件复用 | 优先查阅并复用已有组件库和页面已有组件；仅当现有能力无法满足需求时才创建新组件 |
| 依赖     | 仅使用“依赖白名单”中的包及其子路径                                                                                       |
| 文案     | 用户可见文案通过 `$i18n.t("key")` 获取，并在词包中提供翻译                                                                |
| 路由     | 使用 `navigate` 跳转、使用 `location` 读取地址，不使用 `window.location`                                                  |
| 样式     | 使用 Ant Design CSS 变量，不写死颜色或其他样式常量                                                                        |
| 数据访问 | `client` 支持当前页面可用的全部 `@lovrabet/sdk` 能力，先确认数据集、SQL 或 BFF 的事实与 SDK 调用方式，再使用 `client` |

### 依赖白名单（严格）

- 在所有代码和文件中，包括 `src/**`、配置文件和脚本，导入仅限以下包及其子路径：
  - `react@18`
  - `react-dom@18`
  - `lodash@4`
  - `dayjs@1`
  - `antd@5`
  - `@ant-design/icons@5`
  - `echarts@5`
  - `@lovrabet/components`
- 禁止引入、建议引入或暗示使用任何未在白名单中的依赖

## 生成前信息清单

生成前应确认以下信息。缺失且会影响页面行为时，先向用户确认。

| 信息     | 已确认内容                         | 待补充 |
| -------- | ---------------------------------- | ------ |
| 页面目标 | 页面解决的问题、目标用户、完成标准 |        |
| 页面身份 | 新建页面或已有 `pageId`            |        |
| 布局     | 区域划分、响应式要求、视觉约束     |        |
| 交互     | 查询、编辑、跳转、提交、确认等行为 |        |
| 数据     | 数据集、字段、操作、权限和错误语义 |        |
| 文案     | 语言范围、词条命名和默认文案       |        |

## 页面实现分层

### 入口与布局

- `src/app/index.jsx` 负责页面入口、页面级状态和主布局
- 复杂或可复用区域可按页面需要拆分到合适的相对路径，`src/components/` 仅为示例
- 每个组件只接收其实际需要的数据和回调，避免把页面全部状态逐层透传

### `src/app/index.jsx` 模板

初始化创建自定义页面时，必须以此模板生成 `src/app/index.jsx`。将 `{{pageName}}` 替换为页面名称；保留词包注册、上下文 Hook 和样式导入，再按需求扩展布局、状态与数据调用。通过 `page react-create --page-content` 初始化创建时，传入的 `src/app/index.jsx` 也必须从此模板派生。

更新已有页面时，以 `page react-detail` 返回的最新 `codeContent` 为基线；不强制套用或补齐本模板，应仅按更新需求修改页面内容。

```jsx
import React from "react";
import {
  useSdkClient,
  useNavigate,
  useLocation,
  useI18n,
} from "@/context/app-context";

const App = () => {
  const client = useSdkClient(); // 用于访问后端数据和服务
  const $i18n = useI18n(); // 国际化多语言实例
  const navigate = useNavigate(); // 路由跳转实例
  const location = useLocation(); // 浏览器 location 实例

  return <div className="page-container">{/* 自定义页面内容 */}</div>;
};

export default App;
```

### 数据与交互

- 在调用 `client` 前按“数据客户端文档”拉取目标数据集的 SDK 契约
- 根据业务需求选择数据访问方式，不要为了简单查询引入 SQL 或 BFF
- 明确加载中、空结果、失败和成功后的页面行为
- 写操作前明确校验、确认、提交中和失败恢复策略

#### 数据访问方式选型

| 方式 | 优先考虑的场景 | 使用边界 |
| --- | --- | --- |
| 单一数据集请求 | 只涉及一个数据集的列表查询、详情读取、分页、筛选、创建、更新或删除，且业务逻辑主要由数据集模型能力完成 | 优先使用 API-doc 中确认过的 `client.models` 方法；不要为简单的数据集操作改用 SQL 或 BFF，也不要猜测未在 API-doc 中出现的字段和参数 |
| 自定义 SQL | 需要一次查询组合多个数据集，或需要数据库完成明确的关联、聚合、分组、排序、计算和复杂筛选 | 先确认 SQL 资源、参数和实际返回结构，再按 SDK 规范调用；不要在页面中拼接 SQL、把 SQL 当作通用业务逻辑容器，或据 CLI 的 `data.rows` 推断运行时返回值 |
| BFF | 需要在数据操作生命周期中注入业务校验、转换、加密、条件分支、多步编排，或需要调用外部服务 | 将这类业务逻辑放在已确认的 BFF 中，由页面通过 SDK 调用；不要用 BFF 包装本可直接完成的简单 CRUD，也不要在页面中实现应由 BFF 负责的敏感逻辑 |

选择顺序：先判断单一数据集请求是否能直接满足需求；仅在数据组合或数据库计算是主要问题时使用自定义 SQL；当核心问题是业务规则、数据处理流程或外部服务协同时使用 BFF。三种方式的具体方法、参数、返回值和异常形态，均以当前资源事实和 `@lovrabet/sdk` 使用规范为准。

### 国际化与样式

- 先定义词条，再在组件中使用词条 key
- 所有用户可见文案必须使用 `$i18n.t("key")` 获取，并在 `src/locales/index.js` 中提供有效翻译，禁止硬编码字符串
- 所有 CSS 样式必须使用 Ant Design 提供的 CSS 变量，例如 `color: var(--ant-color-primary);`，禁止直接写死色值或其他样式常量
- 页面状态、禁用态和反馈文案与交互行为保持一致

## 页面上下文内置能力

### 使用边界

- 仅使用本节已登记的内置能力和已确认的调用方式
- 数据访问前先确认数据集字段、操作和 SDK 契约
- 调用内置能力时保持组件生命周期、加载状态和错误反馈清晰
- 未登记的方法、参数、返回结构或副作用不得猜测

### 页面上下文入口

页面可从 `@/context/app-context` 导入以下能力：

```js
import {
  useSdkClient,
  useI18n,
  useNavigate,
  useLocation,
} from "@/context/app-context";
```

| 能力           | 已确认用途               | 使用前应确认                          | 详细规则       |
| -------------- | ------------------------ | ------------------------------------- | -------------- |
| `useSdkClient` | 获取当前页面的数据客户端 | 目标数据集、字段、操作和 SDK 调用方式 | 与 `@lovrabet/sdk` 客户端行为一致（见“数据客户端”） |
| `useI18n`      | 获取页面国际化实例       | 词条 key 和词包翻译                   | 见“国际化”     |
| `useNavigate`  | 获取页面路由跳转方法     | 目标页面 code 和跳转方式              | 见“导航”       |
| `useLocation`  | 获取当前地址信息         | 需要读取的地址信息                    | 见“导航”       |

### 国际化

```js
const $i18n = useI18n();
$i18n.addLocale(locales);
const title = $i18n.t("pageTitle");
```

- 页面加载词包后再使用词条
- 所有用户可见文案必须使用 `$i18n.t("key")` 获取，禁止硬编码字符串
- 每个 `$i18n.t("key")` 的 `key` 必须在 `src/locales/index.js` 中提供有效翻译
- 词条命名、默认值和多语言回退规则在此处补充后再作为统一约定使用

### 数据客户端

```js
const client = useSdkClient();
```

`client` 是运行时注入的 SDK 客户端实例，语义上与 `@lovrabet/sdk` 生成的 `createClient()` 返回对象一致，支持当前页面可用的全部 `@lovrabet/sdk` 能力。

- 数据集模型调用（如 `models.<dataset>`）的调用标识、方法、参数、返回值和异常形态，严格以当前数据集的 API-doc 返回文档为准，不得自行推测或自由发挥
- SQL、BFF 及其他非数据集能力的可用方法、参数、返回值和异常形态，均以 `@lovrabet/sdk` 的使用规范为准
- 页面调用 SQL 前，先用 `rabetbase sql list` 与 `rabetbase sql detail --sqlcode <sqlCode>` 确认目标资源
- 生成使用自定义 SQL 的页面代码前，使用代表性参数执行 `rabetbase sql exec --sqlcode <sqlCode> --params <json> --format json`。若返回 `data.error`，先解决执行错误；成功后再根据 `data.rows` 与 `data.rowCount` 确认实际数据结构
- `sql exec` 仅用于确认 SQL 可执行及返回数据结构。页面代码按 `@lovrabet/sdk` 的使用规范处理 `client.sql` 的返回值，不使用 CLI 输出字段 `data.rows`
- 页面调用 BFF 前，先用 `rabetbase bff list` 与 `rabetbase bff detail --id <id>` 确认目标资源
- 数据操作需要注入自定义业务逻辑，例如校验、转换或加密时，优先考虑使用 BFF；页面运行时通过 `client` 按 `@lovrabet/sdk` 规范调用已确认的 BFF
- CLI 只用于读取 SQL 与 BFF 的资源事实，页面运行时仍通过 `client` 调用，不在页面中拼接 CLI 命令或平台接口
- 不在文档中出现的 dataset 字段、参数名、返回形态、方法名不得猜测
- `client` 只是页面内可直接调用的入口，页面侧不在本地重新创建 `createClient` 或重复注册模型

#### 数据客户端文档

在生成或修改任何包含数据访问的页面代码前，获取目标数据集当前返回的 API-doc。该文档仅用于确认页面生成阶段的 SDK 契约，不得写入页面的 `codeContent`，也不得通过 `client` 在页面运行时再次请求。

- 从 API-doc 确认查询、写入等数据方法的参数、响应结构和示例，并严格按该使用规范实现 `client` 调用
- 数据集、字段、操作或权限发生变化时，重新获取 API-doc，不复用其他数据集或历史页面的调用假设
- API-doc 的获取方式、环境地址和请求参数由当前页面平台能力提供，页面代码中不得固化这些请求细节

#### 示例：API-doc 返回片段的消费方式（联系人）

以下为 `联系人` 文档样例中的可用调用。示例仅用于该文档对应数据集；其他数据集必须按其文档替换签名与参数。

- 目标 app：`app-c53b7ce2`
- 数据集：`d1c3af3347d543a7a9aa3311caf7af59_75_3c7120aa`
- SDK key：`dataset_d1c3af3347d543a7a9aa3311caf7af59_75_3c7120aa`

```javascript
// API-doc 示例可用的调用签名（完整以文档为准）:
// filter(params)
// getOne(id)
// create(payload)
// update(id, data)
// delete(id)
// getSelectOptions(params)
// aggregate(params)
// excelExport(params)

const CONTACTS_DATASET_CODE = "dataset_d1c3af3347d543a7a9aa3311caf7af59_75_3c7120aa";
const CONTACTS_DATASET_ALIAS = "crmContact";

export const queryContactList = async (client, searchKeyword) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  const result = await model.filter({
    where: {
      name: searchKeyword ? { $contain: searchKeyword } : undefined,
    },
    select: ["id", "customer_id", "name", "sex", "position"],
    orderBy: [{ id: "DESC" }],
    currentPage: 1,
    pageSize: 20,
  });
  return Array.isArray(result?.tableData) ? result.tableData : [];
};

export const queryContactDetail = async (client, id) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.getOne(id);
};

export const queryContactSelectOptions = async (client, fieldName) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.getSelectOptions({
    code: fieldName,
    label: fieldName,
  });
};

export const createContact = async (client, contactPayload) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.create(contactPayload);
};

export const updateContact = async (client, id, contactPayload) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.update(id, contactPayload);
};

export const deleteContact = async (client, id) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.delete(id);
};

export const aggregateContact = async (client, params) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.aggregate(params);
};

export const exportContact = async (client, params) => {
  const model = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
  return model.excelExport(params);
};

// 以下为文档中给出的注册信息示例（页面运行时不需要重复调用 createClient）：
// import { registerModels, createClient } from "@lovrabet/sdk";
// registerModels({
//   appCode: "app-c53b7ce2",
//   models: [
//     {
//       tableName: "crm_contact",
//       datasetCode: "d1c3af3347d543a7a9aa3311caf7af59_75_3c7120aa",
//       alias: "crmContact",
//     },
//   ],
// });
// const client = createClient();
```

#### 常见返回体处理（示例）

`filter`、`aggregate` 返回常见 `paging`、`tableData`、`tableColumns`：

```js
const contactsModel = client.models.crmContact || client.models.dataset_d1c3af3347d543a7a9aa3311caf7af59_75_3c7120aa;
const { paging, tableData, tableColumns } = await contactsModel.filter({ currentPage: 1, pageSize: 20 });
console.log("total", paging?.totalCount);
console.log("rows", tableData?.length ?? 0);
console.log("columns", tableColumns?.length ?? 0);
```

其中：
- `filter`、`aggregate` 返回字段包括 `paging.pageSize`、`paging.totalCount`、`paging.currentPage`
- 成功返回的 `tableData` 为行数据，`tableColumns` 为列描述
- `create`、`update`、`delete` 成功时返回 `null`
- `excelExport` 成功时返回导出文件链接字符串

### 示例：React 组件中的数据加载与错误处理

```jsx
import React, { useEffect, useState } from "react";
import { message, Table, Button, Input } from "antd";
import { useSdkClient, useI18n } from "@/context/app-context";

const CONTACTS_DATASET_CODE = "dataset_d1c3af3347d543a7a9aa3311caf7af59_75_3c7120aa";
const CONTACTS_DATASET_ALIAS = "crmContact";

const App = () => {
  const client = useSdkClient();
  const $i18n = useI18n();
  const [keyword, setKeyword] = useState("");
  const [rows, setRows] = useState([]);
  const [loading, setLoading] = useState(false);
  const [selected, setSelected] = useState(null);

  const loadRows = async () => {
    setLoading(true);
    try {
      const contactsModel = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
      const result = await contactsModel.filter({
        where: keyword ? { name: { $contain: keyword } } : undefined,
        select: ["id", "name", "mobile", "email", "position"],
        currentPage: 1,
        pageSize: 20,
      });
      setRows(result?.tableData ?? []);
    } catch (err) {
      message.error($i18n.t("common.loadFailed"));
      console.error("load contacts failed", err);
    } finally {
      setLoading(false);
    }
  };

  const loadDetail = async (record) => {
    try {
      const contactsModel = client.models[CONTACTS_DATASET_ALIAS] || client.models[CONTACTS_DATASET_CODE];
      const detail = await contactsModel.getOne(record.id);
      setSelected(detail);
    } catch (err) {
      message.error($i18n.t("common.loadDetailFailed"));
      console.error("load contact detail failed", err);
    }
  };

  useEffect(() => {
    loadRows();
  }, [keyword]);

  return (
    <div>
      <Input
        placeholder={$i18n.t("contact.searchPlaceholder")}
        onChange={(event) => setKeyword(event.target.value)}
      />
      <Button loading={loading} onClick={loadRows}>
        {$i18n.t("contact.search")}
      </Button>
      <Table
        dataSource={rows}
        rowKey="id"
        onRow={(record) => ({
          onDoubleClick: () => void loadDetail(record),
        })}
      />
    </div>
  );
};

export default App;
```

**注意事项**

- 使用 `client` 前请确认该数据集当前 API-doc 的 `操作列表` 与字段，尤其是 `filter/getOne/getSelectOptions/update/create/delete` 的请求参数签名。
- 运行时错误统一走 `try/catch`，并显示明确错误信息。
- `dataset_d...` 这种标准 SDK key 为通用写法；若文档返回 `alias`，可改用 `client.models.<alias>`。
- 所有示例文案仍以 `$i18n.t("...")` 作为文案入口。

### 导航

```js
const navigate = useNavigate();
const location = useLocation();
```

- 跳转使用 `navigate`，当前地址信息使用 `location`
- 目标页面与当前页面关联时，优先使用 `navigate("/pagecode", { onePage: true })`，在当前页面基础上横向追加新页面
- `"/pagecode"` 必须是以 `/` 开头的目标页面完整路径，可以携带参数和 hash
- 仅当目标页面与当前页面无关联时，使用普通 `navigate("/pagecode")`
- 不使用 `window.location`

## 交付前检查

- [ ] 完整页面文件包含全部需要保留的文件
- [ ] 导入均在依赖白名单内
- [ ] 每个本地相对导入均指向提交内容中的文件，导入名称与该文件的导出名称一致
- [ ] 每个非全局标识符均由 import、组件参数、Hook 返回值或同一作用域内声明提供，不保留未定义引用
- [ ] 用户可见文案均有有效词条和翻译
- [ ] 每个数据集调用的调用标识、方法、参数、返回值和异常处理均严格遵循当前 API-doc 返回文档，未自行推测或自由发挥
- [ ] 数据集、SQL、BFF 与其他 SDK 调用的资源事实、参数和返回结构均有依据
- [ ] 路由、状态和错误反馈符合已确认的交互
- [ ] CSS 未写死颜色或其他样式常量
- [ ] 已先执行 `page react-create --dry-run` 或 `page react-update --dry-run` 审阅完整提交内容，并通过 CLI 的基础 JS/JSX/TS/TSX 语法校验

CLI 的基础校验只确认页面包含 `.jsx` 或 `.tsx` 文件，且页面内的 JS、JSX、TS、TSX 文件能够解析。变量引用、相对模块导出、数据调用契约和运行时行为必须由本清单和页面发布后的实际使用继续确认
