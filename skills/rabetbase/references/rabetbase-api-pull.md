# api pull

拉取当前 App 下所有数据集（Dataset）的元信息，生成 TypeScript API 客户端代码到 `src/api/` 目录。

生成的代码包含每个 Dataset 的 Model 类，可直接用于 SDK 查询，无需手动调用 HTTP 接口。

## 命令

```bash
# 拉取并生成 API 代码（默认 ./src/api/）
rabetbase api pull

# 指定输出目录
rabetbase api pull --output ./src/generated/api/

# 多应用模式：指定某个应用
rabetbase api pull --app order
rabetbase api pull --appcode app-order-001
```

## 参数

| 参数 | 说明 |
|------|------|
| `--output <dir>` | 输出目录，默认 `./src/api/` |
| `--app <name>` | 多应用模式下，指定应用名称 |
| `--appcode <code>` | 直接指定 appcode，跳过配置查找 |

## 多应用过滤

多应用模式下：
- **不加 `--app` / `--appcode`**：遍历所有已配置应用，逐个拉取
- **加 `--app <name>`**：仅拉取指定应用
- **加 `--appcode <code>`**：反查到对应 app profile，使用其 cookie/env/apiDir

## 风险等级

`write` — 修改本地文件系统，生成/覆盖 TypeScript 文件。

## 输出文件

| 文件 | 说明 |
|------|------|
| `<prefix>-api.ts` | 数据模型定义（datasetCode → Model 映射） |
| `<prefix>-client.ts` | SDK 客户端初始化代码 |

默认 `prefix` 为空（单应用），多应用非 default 应用使用应用名作为 prefix。

## 前置条件

- 已完成 `rabetbase auth` 登录
- 已配置 appcode（单应用或多应用）

## 示例

```bash
# 单应用：生成到 ./src/api/
rabetbase api pull

# 多应用 order：生成到 ./src/api/ 并使用 order- 前缀
rabetbase api pull --app order

# 独立输出目录
rabetbase api pull --output ./src/generated/api/
```
