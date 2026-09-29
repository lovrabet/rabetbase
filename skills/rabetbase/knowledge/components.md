# 公共组件用法

本文件基于 `@lovrabet/components` 最新版本的 README 和包根入口公开类型，登记可复用的 UI 组件规范。生成或修改页面前，先确认项目使用最新版本；生成或修改自定义页面时还需阅读 [`generation-standards.md`](custom-page/generation-standards.md)。

## 使用边界

- 自定义页面可导入的组件包仅限 [`generation-standards.md`](custom-page/generation-standards.md)“依赖白名单”中的包及其子路径
- `@lovrabet/components` 使用包根入口的公开具名导出，不导入内部实现路径
- 组件的用户可见文案必须使用 `$i18n.t("key")`
- 所有组件均支持 Ant Design 主题、多语言和响应式自适应；Ant Design 文案继承外层 `ConfigProvider.locale`，YT 文案跟随 `@lovrabet/i18n` 全局语言
- `env` 已废弃，后续不再透出，也不属于向用户展示或由用户选择的能力；新代码不得传入该参数
- 组件样式优先复用 Ant Design token 和 CSS 变量，不写死主题色
- 未登记的业务组件、组件属性或行为不得猜测

## `@lovrabet/components` 组件

组件、属性和行为以 `@lovrabet/components` 最新版本包根入口的公开导出为准。页面代码从包根入口导入：

```ts
import {
  YTAIButton,
  YTAIInput,
  YtAddressPicker,
  YtCodeEditor,
  YTRichEditor,
  YTRichEditorPreview,
  YtUpload,
  YtUserSelect,
  type YTUserSelectCode,
  type YtUploadFile,
} from "@lovrabet/components";
```

### 请求能力

组件只消费已传入的请求实例，不负责创建或配置请求实例。`apiFetch` 用于 `YTAIButton`、`YTAIInput` 等 AI 组件；`runtimeFetch` 用于 `YtUserSelect`、`YtUpload`、`YTRichEditor` 等用户、文件和默认图片上传能力。

面向新代码，请求实例的消费优先级为：组件属性传入的实例优先于 `YtConfigProvider` 注入的实例；两者均未传入时，组件使用内置的线上默认实例。页面中多个相关组件共用同一实例时，在根部通过 `YtConfigProvider` 注入；仅单个组件需要使用另一实例时，直接传入该组件的 `apiFetch` 或 `runtimeFetch` 属性。

`apiFetch` 与 `runtimeFetch` 分别独立解析，不会相互替代：未传入 `runtimeFetch` 时不会使用 `apiFetch`，反之亦然。已废弃的 `env` 不属于新代码的请求实例选择规则。

推荐按以下顺序使用：

1. 页面内多个组件共用实例时，优先通过 `YtConfigProvider` 在根部传入。
2. 仅个别组件需要不同实例时，再通过该组件属性覆盖根部实例。
3. 不需要传入实例时，直接使用组件，组件会回退到内置的线上默认实例。

根部共享实例时，其下未显式传入实例的相关组件都会消费对应实例：

```tsx
import { YtConfigProvider } from "@lovrabet/components";

<YtConfigProvider apiFetch={apiFetch} runtimeFetch={runtimeFetch}>
  <YtUserSelect appCode={appCode} />
  <YtUpload appCode={appCode} />
  <YTAIButton promptId={promptId} userPrompt={userPrompt} />
</YtConfigProvider>;
```

单个组件传入的实例会覆盖根部同类实例，适合该组件有独立请求需求的场景：

```tsx
<YtUpload appCode={appCode} runtimeFetch={fileRuntimeFetch} />

<YTAIButton apiFetch={aiApiFetch} promptId={promptId} userPrompt={userPrompt} />
```

未传入 `YtConfigProvider` 或组件级实例时，无需额外配置：

```tsx
<YtUserSelect appCode={appCode} />

<YTRichEditor appCode={appCode} defaultValue={content} />
```

请求实例的创建、Domain 配置和注入位置由上层应用处理；组件文档不约定这些配置方式。

| 组件 | 适用场景 | 关键使用方式 |
| --- | --- | --- |
| `YTAIButton` | 通过已配置的提示词或 Skill 发起 AI 调用 | 配置 `promptId` 或 `skillId` 与必填 `userPrompt`，处理成功和失败回调 |
| `YTAIInput` | 需要输入内容后发起 AI 调用 | 配置 AI 调用参数，并使用 `inputType`、`userPromptTemplate` 和提交前校验控制输入流程 |
| `YtUserSelect` | 多选、编辑或只读展示应用用户 | 交互值使用用户 code 编号数组，使用 `preview` 切换只读展示 |
| `YtAddressPicker` | 选择或只读展示地址层级 | 受控传入地址 value 路径；需要自定义地址时传入 `options` 或 `optionsUrl` |
| `YtUpload` | 上传、展示、预览或下载附件 | 使用 Ant Design 原生 `fileList` 和 `onChange(info)` 管理完整文件列表；默认上传传入当前有效的 `appCode`，自定义上传使用 `customRequest` |
| `YTRichEditor` | 编辑富文本内容 | 用 `defaultValue` 初始化，通过 `onChange` 接收 HTML；使用默认图片上传时必须显式传入 `appCode`，组件不再自动注入，也可通过 `upload` 使用自定义上传 |
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

使用 `skillId` 替代 `promptId` 时，同时传入 `appCode`。Skill 调用需要展示流式结果时可传入 `showPopover`，默认值为 `false`。需要在调用前校验页面状态时，传入 `onClick` 并在不满足条件时返回 `false`。

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

```tsx
const [userCodes, setUserCodes] = useState<YTUserSelectCode[]>([]);

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

```tsx
const [files, setFiles] = useState<YtUploadFile[]>([]);

<YtUpload
  appCode={appCode}
  accept="image/*"
  maxCount={maxFiles}
  maxSizeMB={maxFileSizeMB}
  fileList={files}
  onChange={({ fileList }) => setFiles(fileList)}
/>
```

展示已有附件时，将完整的 `YtUploadFile[]` 传给 `fileList`；每项至少提供稳定 `uid` 和标准 `name`。只读预览、下载和列表展示时传入 `preview`。

自定义请求、响应转换、校验和表单绑定约束见下方 `YtUpload` 章节。

### 富文本编辑与预览

```tsx
const [content, setContent] = useState<string | null>(null);

<YTRichEditor
  appCode={appCode}
  defaultValue={content}
  onChange={setContent}
  onImageUploadError={setError}
/>

<YTRichEditorPreview content={content} />
```

未传 `upload` 时组件使用 Lovrabet 默认图片上传，此时必须由页面显式传入有效的 `appCode`。组件不再从页面配置、运行环境或其他全局信息中自动注入 `appCode`。调用方无需配置或感知内部环境参数。需要使用自定义上传时，传入接收 `File` 并返回可访问图片 URL 的 `upload(file) => Promise<string>`，此时不需要 `appCode`。`upload` 和 `appCode` 均未配置时，文字编辑和内容预览仍可使用，但图片上传功能不可用。图片上传和内容持久化仍需遵循已确认的服务契约。

富文本的自定义 `upload` 每次只处理一张图片，返回值必须是可直接作为图片 `src` 使用的非空最终 URL。失败时应抛出或返回 rejected Promise，由 `onImageUploadError` 统一处理；不得返回临时 Blob URL，也不得返回包含 URL 的对象。`maxImageSize` 和 `imageAccept` 只提供编辑器侧选择与体积限制，上传服务仍需校验文件内容、类型、权限和大小。该方法只负责获得图片地址，富文本内容仍通过 `onChange` 返回并由页面业务逻辑持久化。

### 代码编辑

```jsx
const [code, setCode] = useState("");

<YtCodeEditor
  language="json"
  value={code}
  onChange={setCode}
  wordWrap="on"
/>
```

`YtCodeEditor` 默认使用 JSON 语言和 `240px` 高度。只读展示时使用 `readOnly`，按需使用 `minimap`、`lineNumbers`、`wordWrap`、`fontSize`、`tabSize` 或 `options` 配置编辑器；这些显式属性优先于 `options` 中的同名配置。当前公开类型不提供 `placeholder` 属性。

## 主题、多语言与自适应

- 所有组件继承外层 Ant Design `ConfigProvider` 的主题色以及浅色、深色算法，不需要为组件单独写死颜色
- YT 自身文案跟随页面由 `@lovrabet/i18n` 管理的全局语言，组件不再提供独立 `locale` 属性
- Ant Design 内部文案继承外层 `ConfigProvider.locale`；切换语言时由上层同时更新全局语言和 `ConfigProvider`
- 所有组件均支持响应式自适应；页面仍应提供可收缩的容器，避免固定宽度阻止组件适配小屏

### AI 调用组件

#### `YTAIButton`

- `userPrompt` 必填
- `promptId` 与 `skillId` 二选一，组件优先使用 `promptId`
- 使用 `skillId` 时传入当前 `appCode`；`showPopover` 只在 Skill 场景生效，用于展示流式结果，默认值为 `false`
- `gradient` 默认开启；需要使用普通 Ant Design 按钮样式时传入 `gradient={false}`
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
- 组件固定为多选模式；交互场景使用用户 code 数组，`onChange` 返回 `YTUserSelectCode[]`，编号可以是字符串或数字
- 读取时兼容字符串或数字类型的用户编号、历史 `{ code }[]` 以及用于旧数据展示的单个字符串或数字；新写入使用组件返回的 `YTUserSelectCode[]`，不转换成对象数组
- 需要只读展示时传入 `preview`
- 查询应用用户列表时传入当前 `appCode`

#### `YtAddressPicker`

- `value` 使用 `string[] | null`，`onChange` 返回仅保留非空字符串的地址层级路径
- 传入 `options` 时使用调用方提供的级联选项；未传时使用组件内置地址数据，也可通过 `optionsUrl` 配置地址选项来源
- 只读展示时传入 `preview`
- 自定义选项的 `label` 为用户可见内容时，必须来自已翻译词条或已确认的业务数据

#### `YtUpload`

- `YtUploadProps` 以 Ant Design `UploadProps<T>` 为主线，使用 `fileList`、`defaultFileList` 和原生 `onChange(info)`；不再使用历史 `value`、`onChange(files)` 或 JSON 字符串协议
- `YtUploadFile<T>` 等同于 Ant Design `UploadFile<T>`；已有附件使用稳定 `uid`、标准 `name` 和 `status`，文件服务路径放在已确认的 `response` 数据结构中
- 使用默认上传时传入当前有效的 `appCode`；仅传 `action` 时使用原生请求，传 `customRequest` 时由调用方完整接管请求
- `dataProcess` 可同步或异步把内置请求、`action` 或 `customRequest` 的成功响应转换为最终文件 URL；未配置时保留原始响应
- 自定义 `customRequest` 成功时调用 `onSuccess`，失败时调用 `onError`，需要取消时返回带 `abort` 的对象；不得把仅当前会话有效的 Blob URL 持久化为附件地址
- `maxCount` 遵循 Ant Design 列表规则，不保证上传请求数量上限；需要严格限制请求数量或类型时在 `beforeUpload` 和服务端共同校验
- 未提供自定义 `beforeUpload` 时，`maxSizeMB` 才接管文件大小校验；`accept` 只限制文件选择器
- `preview` 是可覆盖的只读展示预设，不是权限控制；自定义 `showUploadList`、`children`、`onPreview` 或 `onDownload` 时使用调用方行为
- 表单使用 `valuePropName="fileList"` 和 `getValueFromEvent={(info) => info.fileList}`；提交时只转换成功附件，不把筛选后的列表回写 `fileList`
- 文件持久化、提交和删除由页面业务逻辑处理；先确认数据集 SDK 或其他已确认服务契约，再写入文件字段

### 内容编辑组件

#### `YTRichEditor` 与 `YTRichEditorPreview`

- `YTRichEditor` 的 `defaultValue` 支持 HTML 字符串或 JSON 内容，`onChange` 返回 HTML 字符串，空内容返回 `null`
- 未传 `upload` 时使用默认图片上传，调用方必须显式传入有效的 `appCode`；组件不会再自动注入 `appCode`
- 传入 `upload` 时使用自定义图片上传，不需要 `appCode`；方法签名为 `(file: File) => Promise<string>`，必须返回非空、稳定且可直接作为图片 `src` 使用的最终 URL
- 自定义图片上传失败时抛出错误或返回 rejected Promise，由 `onImageUploadError` 处理；不返回响应对象或临时 Blob URL
- `upload` 和 `appCode` 均未配置时，文字编辑和内容预览仍可使用，图片上传功能不可用
- 使用 `maxImageSize` 和 `imageAccept` 限制图片，使用 `onImageUploadError` 处理上传或校验失败
- 通过 `disabled` 控制编辑状态；工具栏和错误提示跟随页面全局语言，不传入独立 `locale`
- 使用 `YTRichEditorPreview` 或 `YTRichEditor.Preview` 展示 HTML，且必须传入 `content`
- 需要自定义加载占位时使用 `loadingFallback`，不得猜测组件内部资源地址

#### `YtCodeEditor`

- 使用 `value` 与 `onChange` 管理编辑内容；空值按空字符串处理
- `language`、`theme`、`minimap`、`lineNumbers`、`wordWrap`、`fontSize`、`tabSize` 和 `options` 用于配置 Monaco 编辑器
- 需要展示不可编辑内容时使用 `readOnly` 或 `disabled`
- 当前公开类型不支持 `placeholder`，需要空态提示时由页面在编辑器外层呈现
- 未传 `theme` 时组件跟随外层 Ant Design 主题；页面不自行猜测 Monaco 运行资源或加载方式

### 非组件公开能力

- 包根入口公开 `useUserList`，用于自定义用户选择交互；默认优先使用 `YtUserSelect`，仅在已确认 Hook 返回结构时直接调用
- `queryLLM` 未从 `@lovrabet/components` 包根入口公开，不导入内部路径；需要 AI 调用时使用 `YTAIButton` 或 `YTAIInput`

### 使用检查

- 组件名称与属性以公开类型声明为准，不使用 README、Storybook 或包内部文件中未公开的实现路径
- `YTRichEditor` 和 `YtUpload` 使用默认上传时，确认代码显式传入当前有效的 `appCode`，不得依赖历史默认注入行为
- `YtUpload` 使用 `fileList`、原生 `onChange(info)` 与 `customRequest`，不得生成已移除的 `value`、`upload` 或组件级 `locale` 属性
- 所有用户可见字符串，包括按钮文本、占位文本和自定义选项 label，均使用 `$i18n.t("key")` 或来自已确认的业务数据
- 组件的加载、空态、失败和禁用状态与页面交互一致
- 自定义数据读取或写入仍遵循 [`generation-standards.md`](custom-page/generation-standards.md) 中的数据客户端文档流程
