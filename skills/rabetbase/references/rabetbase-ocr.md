# ocr recognize

使用平台 OCR 服务提取票证类业务材料中的文字与结构化字段。发票、票据、证照等是典型示例，不是票证类型的封闭白名单。

## 使用边界

图片本身不是 OCR 的触发条件。先判断用户处理的是否为票证类业务材料，再判断需要的是文字或结构化字段，还是通用画面理解。

| 任务 | 处理方式 |
|------|----------|
| 提取票证类材料的票面全文、关键字段或表格 | 使用 OCR；不要求用户必须说出“OCR” |
| 用户明确要求对票证材料执行 OCR 或文字识别 | 使用 OCR |
| 理解普通照片、UI 截图、商品图、设计稿或图表 | 当前模型支持多模态时由 Agent 直接处理，不调用 OCR |
| 当前模型不能读取通用图片，但用户只要求提取可见文字 | 仅将 OCR 作为明确的文字提取降级能力，并说明限制 |
| 当前模型不能读取图片，任务依赖物体、颜色、布局或视觉关系 | 不用 OCR 冒充视觉理解，向用户说明能力限制 |
| 图片只是附件，用户没有要求分析或提取 | 不调用 OCR |

按业务对象和任务输出路由，不按文件名、扩展名、MIME、某组关键词或 `scene` 枚举路由。已明确需要精确票证提取时可以直接使用专业 OCR，不必先做一次重复的多模态识别。`general` 是兼容的通用文字识别 scene，可用于未被专用 scene 覆盖的票证材料或用户明确要求的纯文字降级；它不是通用图片理解入口。

## 风险边界

`ocr recognize` 的风险等级为 `read`：命令不创建、更新或删除业务数据。服务端仍可能记录审计、调用第三方 OCR 服务并产生计费；如需将识别结果写入 Dataset，后续写入命令继续按其 `write` 或 `high-risk-write` 规则执行。

本地文件模式会把文件上传到目标应用并生成未绑定业务记录的文件对象。执行前需要核对 `appCode`、文件敏感性和账号权限；最终答复不回显 Cookie 或签名 URL。

## 命令

```bash
# 远程 URL
rabetbase ocr recognize \
  --scene invoice \
  --image-url 'https://files.example.com/invoice.png' \
  --appcode app-demo \
  --format compress

# 本地文件
rabetbase ocr recognize \
  --scene invoice \
  --image-file ./invoice.png \
  --appcode app-demo \
  --format compress
```

## Flags

| Flag | 必填 | 说明 |
|------|------|------|
| `--scene <scene>` | 是 | `invoice`、`general`、`form` 或 `idCard` |
| `--image-url <url>` | 二选一 | 公开可访问的图片或文件 URL |
| `--image-file <path>` | 二选一 | 本地图片或 PDF，普通文件且不超过 50 MB |
| `--appcode <code>` | 取决于项目上下文 | 目标应用；项目未配置默认应用时显式传入 |
| `--dry-run` | 否 | 只预览调用链，不上传、不查询 URL、不调用 OCR |

`--image-url` 与 `--image-file` 必须且只能选择一个。

## Scene 映射

| scene | 服务端 type | 用途 |
|-------|-------------|------|
| `invoice` | `Invoice` | 发票 |
| `general` | `General` | 票证材料的通用文字识别；非通用图片理解 |
| `form` | `Table` | 表格 |
| `idCard` | `IdCard` | 身份证 |

## 调用链

- URL 模式：直接调用平台 OCR 接口。
- 本地文件模式：`file upload` → `file query-url`（短效 URL）→ OCR。
- OCR 中间 URL 不使用 `--long-term`，输出的 `sourceFile.filePath` 可用于后续文件引用。

正式执行前可先预览：

```bash
rabetbase ocr recognize \
  --scene form \
  --image-file ./table.png \
  --appcode app-demo \
  --dry-run \
  --format compress
```

## 输出

稳定输出包含 `scene`，并透传服务端可用的 OCR 事实，例如 `text`、`lines`、`type`、`kvData`、`width`、`height` 和 `requestId`。本地文件模式额外返回 `sourceFile.fileName/filePath`。

不要假设所有识别类型都会返回结构化 `kvData`，也不要伪造积分或计费字段。

## 失败处理

| 情况 | 处理 |
|------|------|
| 缺少输入或同时提供两种输入 | 只保留 `--image-url` 或 `--image-file` 之一 |
| scene 不支持 | 改用 `invoice/general/form/idCard` |
| 本地文件不存在、不是普通文件或超过 50 MB | 修正文件路径或缩小文件后重试 |
| 上传结果缺少 `filePath` | 报告上传响应异常，不继续查询 URL 或 OCR |
| URL 查询缺少 `fileUrl` | 报告链接响应异常，不调用 OCR |
| OCR 权限不足 | 核对目标 `appCode` 和当前 rabetbase 登录账号的应用权限 |

## 相关参考

- [file upload / query-url](rabetbase-file.md)
