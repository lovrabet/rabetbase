# api list

列出当前 App 下已生成的所有数据集（Dataset）模型，查看 API 客户端代码中有哪些可用 Model。

## 命令

```bash
# 列出当前应用的模型
rabetbase api list

# 多应用模式：指定应用
rabetbase api list --app order
rabetbase api list --appcode app-order-001

# JSON 格式输出
rabetbase api list --format json
```

## 参数

| 参数 | 说明 |
|------|------|
| `--app <name>` | 多应用模式下，指定应用名称 |
| `--appcode <code>` | 直接指定 appcode |
| `--format json` | JSON 格式输出（用于脚本解析） |

## 输出说明

列出每个 Dataset 的：

| 字段 | 说明 |
|------|------|
| `id` | Dataset ID |
| `name` | Dataset 名称 |
| `key` | Dataset Key |
| `datasetCode` | Dataset Code（用于 API 调用） |

## 多应用过滤

多应用模式下：
- **不加 `--app` / `--appcode`**：遍历所有已配置应用，依次列出
- **加 `--app <name>`**：仅列出指定应用的模型
- **加 `--appcode <code>`**：反查到对应 app profile，使用其 cookie/env

## 风险等级

`read` — 仅读取，不修改文件。

## 前置条件

- 已完成 `rabetbase auth` 登录
- 已运行过 `rabetbase api pull` 生成过 API 代码（用于确认本地已有哪些模型）
