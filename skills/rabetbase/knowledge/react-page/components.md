# React 自定义页面组件用法

本文件登记 React 自定义页面可复用的 UI 组件规范。生成或修改页面前先阅读 [`generation-standards.md`](generation-standards.md)。

## 使用边界

- 可导入的组件包仅限 [`generation-standards.md`](generation-standards.md)“依赖白名单”中的包及其子路径
- `@lovrabet/components` 使用包根入口的公开具名导出，不导入内部实现路径
- 组件的用户可见文案必须使用 `$i18n.t("key")`
- 组件样式使用 Ant Design CSS 变量，不写死颜色或其他样式常量
- 未登记的业务组件、组件属性或行为不得猜测

## `@lovrabet/components` 组件

本节基于组件库 README、公开类型声明和 Storybook 用例整理。组件、属性和行为以当前发布版本的公开导出为准。页面代码从包根入口导入：

```js
import {
  YTAIButton,
  YTAIInput,
  YtAddressPicker,
  YtCodeEditor,
  YTRichEditor,
  YTRichEditorPreview,
  YtUpload,
  YtUserSelect,
} from "@lovrabet/components";
```

| 组件 | 适用场景 | 关键使用方式 |
| --- | --- | --- |
| `YTAIButton` | 通过已配置的提示词或 Skill 发起 AI 调用 | 配置 `promptId` 或 `skillId` 与必填 `userPrompt`，处理成功和失败回调 |
| `YTAIInput` | 需要输入内容后发起 AI 调用 | 配置 AI 调用参数，并使用 `inputType`、`userPromptTemplate` 和提交前校验控制输入流程 |
| `YtUserSelect` | 选择、编辑或只读展示应用用户 | 受控传入用户 code，使用 `preview` 切换只读展示 |
| `YtAddressPicker` | 选择或只读展示地址层级 | 受控传入地址 value 路径；需要自定义地址时传入 `options` 或 `optionsUrl` |
| `YtUpload` | 上传、展示、预览或下载附件 | 传入当前 `appCode`，以 `value` 和 `onChange` 管理已完成的业务文件 |
| `YTRichEditor` | 编辑富文本内容 | 用 `defaultValue` 初始化，通过 `onChange` 接收 HTML，并为图片上传提供 `upload` |
| `YTRichEditorPreview` | 只读展示富文本 HTML | 传入必填 `content` |
| `YtCodeEditor` | 编辑或只读展示代码、JSON 等文本 | 以 `value` 和 `onChange` 受控，按需设置 `language`、`readOnly` 和编辑器选项 |

## 使用 Demo

以下示例基于组件库 Storybook 用例调整而来。示例中的 `appCode`、提示词或 Skill 标识、数据和回调均由页面上下文提供；所有用户可见文案使用 `$i18n.t("key")`。示例均假设已通过 `useI18n()` 获取 `$i18n`，并按需从 React 导入 `useState`。

### AI 按钮

```jsx
<YTAIButton
  promptId={promptId}
  userPrompt={$i18n.t("ai.userPrompt")}
  onSuccess={setGeneratedContent}
  onError={setError}
>
  {$i18n.t("ai.generate")}
</YTAIButton>
```

使用 `skillId` 替代 `promptId` 时，同时传入 `appCode`。需要在调用前校验页面状态时，传入 `onClick` 并在不满足条件时返回 `false`。

### AI 输入框

```jsx
<YTAIInput
  inputType="textarea"
  promptId={promptId}
  userPromptTemplate={$i18n.t("ai.userPromptTemplate")}
  placeholder={$i18n.t("ai.placeholder")}
  buttonText={$i18n.t("ai.generate")}
  onBeforeSubmit={validateInput}
  onSuccess={setGeneratedContent}
  onError={setError}
/>
```

`userPromptTemplate` 中使用 `{input}` 引用输入内容。`validateInput` 返回 `false` 时不会调用 AI。

### 用户选择

```jsx
const [userCodes, setUserCodes] = useState([]);

<YtUserSelect
  appCode={appCode}
  value={userCodes}
  onChange={setUserCodes}
  placeholder={$i18n.t("userSelect.placeholder")}
/>
```

只读展示时使用 `<YtUserSelect appCode={appCode} value={userCodes} preview />`。

### 地址选择

```jsx
const [address, setAddress] = useState([]);

<YtAddressPicker
  options={addressOptions}
  value={address}
  onChange={setAddress}
/>
```

`addressOptions` 的用户可见 `label` 必须使用词条或已确认的业务数据。只读展示时传入 `preview`。

### 附件上传

```jsx
const [files, setFiles] = useState([]);

<YtUpload
  appCode={appCode}
  accept="image/*"
  max={maxFiles}
  maxSizeMB={maxFileSizeMB}
  value={files}
  onChange={setFiles}
/>
```

展示已有附件时，将已确认的 `YtUploadFile[]` 传给 `value`；只读预览、下载和列表展示时传入 `preview`。

### 富文本编辑与预览

```jsx
const [content, setContent] = useState(initialContent);

<YTRichEditor
  defaultValue={initialContent}
  upload={uploadImage}
  onChange={setContent}
  onImageUploadError={setError}
/>

<YTRichEditorPreview content={content} />
```

`uploadImage` 接收 `File` 并返回可访问图片 URL。图片上传服务和内容持久化的实现必须遵循已确认的服务契约。

### 代码编辑

```jsx
const [code, setCode] = useState("");

<YtCodeEditor
  language="json"
  value={code}
  onChange={setCode}
  placeholder={$i18n.t("codeEditor.placeholder")}
  wordWrap="on"
/>
```

只读展示时使用 `readOnly`，按需使用 `minimap`、`lineNumbers`、`tabSize` 或 `options` 配置编辑器。

### AI 调用组件

#### `YTAIButton`

- `userPrompt` 必填
- `promptId` 与 `skillId` 二选一，组件优先使用 `promptId`
- 使用 `skillId` 时传入当前 `appCode`；`showPopover` 只在 Skill 场景生效
- `onClick` 返回 `false` 可阻止本次调用；通过 `onSuccess`、`onResponse` 和 `onError` 分别处理结果、完整响应和失败
- `options` 仅用于传入已确认的 `temperature`、`maxTokens` 等 AI 调用选项
- 提示词标识、权限和可用环境未确认时，不生成调用代码

#### `YTAIInput`

- 继承 `YTAIButton` 的 AI 调用配置和回调，且不直接传入 `userPrompt`
- 使用 `inputType="input"` 或 `inputType="textarea"` 选择输入形态，使用 `display="inline"` 或 `display="block"` 控制按钮布局
- `userPromptTemplate` 中的 `{input}` 会替换为当前输入内容；需要特殊拼接时使用 `formatUserPrompt`
- 使用 `onBeforeSubmit` 校验输入，返回 `false` 时阻止调用；使用 `onInputChange` 同步页面状态
- `placeholder`、`buttonText` 和其他用户可见字符串必须传入 `$i18n.t("key")` 的结果

### 选择与上传组件

#### `YtUserSelect`

- 当前用户列表由组件加载，页面不自行猜测或复刻用户查询接口
- `value` 支持用户 code 数组、单个 code，以及兼容历史数据的 `{ code }` 数组；`onChange` 始终返回用户 code 数组
- 需要只读展示时传入 `preview`
- 传入当前 `appCode` 以明确查询上下文；`env` 只在需要覆盖当前环境识别时设置

#### `YtAddressPicker`

- `value` 与 `onChange` 使用 `string[]` 形式的地址层级路径
- 传入 `options` 时使用调用方提供的级联选项；未传时使用组件内置地址数据，也可通过 `optionsUrl` 配置地址选项来源
- 只读展示时传入 `preview`
- 自定义选项的 `label` 为用户可见内容时，必须来自已翻译词条或已确认的业务数据

#### `YtUpload`

- 传入当前 `appCode`，不依赖页面自行读取宿主全局变量
- `value` 支持 `YtUploadFile[]` 或兼容的 JSON 字符串，`onChange` 返回上传完成的业务文件列表
- 使用 `max`、`maxSizeMB` 和 `accept` 限制上传数量、体积和文件类型；需要只读展示、预览和下载时传入 `preview`
- 文件持久化、提交和删除由页面业务逻辑处理；先确认数据集 SDK 或其他已确认服务契约，再写入文件字段

### 内容编辑组件

#### `YTRichEditor` 与 `YTRichEditorPreview`

- `YTRichEditor` 使用 `defaultValue` 初始化内容，`onChange` 返回 HTML 字符串或空值
- 启用图片上传时，`upload` 必须返回可访问图片 URL；用 `onImageUploadError` 和 `onLoadError` 处理上传或资源加载失败
- 通过 `disabled` 控制编辑状态；通过 `locale` 设置工具栏和错误提示语言
- 使用 `YTRichEditorPreview` 或 `YTRichEditor.Preview` 展示 HTML，且必须传入 `content`
- 需要覆盖运行资源或加载占位时，使用已确认的 `resourceUrl` 和 `loadingFallback`，不得猜测资源地址

#### `YtCodeEditor`

- 使用 `value` 与 `onChange` 管理编辑内容；空值按空字符串处理
- `language`、`theme`、`minimap`、`lineNumbers`、`wordWrap`、`fontSize`、`tabSize` 和 `options` 用于配置 Monaco 编辑器
- 需要展示不可编辑内容时使用 `readOnly` 或 `disabled`
- 未传 `theme` 时组件跟随外层 Ant Design 主题；页面不自行猜测 Monaco 运行资源或加载方式

### 非组件公开能力

- `queryLLM` 是 `YTAIButton` 对应的直接 AI 调用方法。仅在已确认提示词或 Skill 标识、调用权限和错误处理策略时使用
- `useUserList` 是 `YtUserSelect` 对应的用户列表 Hook。默认优先使用 `YtUserSelect`；仅在需要自定义交互且已确认返回结构时使用

### 使用检查

- 组件名称与属性以公开类型声明为准，不使用 README、Storybook 或包内部文件中未公开的实现路径
- 所有用户可见字符串，包括按钮文本、占位文本和自定义选项 label，均使用 `$i18n.t("key")` 或来自已确认的业务数据
- 组件的加载、空态、失败和禁用状态与页面交互一致
- 自定义数据读取或写入仍遵循 [`generation-standards.md`](generation-standards.md) 中的数据客户端文档流程
