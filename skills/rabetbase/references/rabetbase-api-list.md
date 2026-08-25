# api list

列出当前 App 下平台返回的数据集（Dataset）模型事实。本命令查询远端 Dataset 元信息，**不检查**本地是否已生成 `api.ts` / `client.ts`。

## 命令

```bash
# 列出当前应用的模型
rabetbase api list

# 多应用模式：指定应用
rabetbase api list --app order
rabetbase api list --appcode app-order-001

# 多应用：与全局合并配置一起列出时加 --global（默认仅项目级 apps）
rabetbase api list --global

# JSON 格式输出
rabetbase api list --format json
```

## 参数

| 参数 | 说明 |
|------|------|
| `--global` | 多应用时显式从「全局+项目」双层解析 `apps`；默认仅项目级 `apps` |
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
- **不加 `--app` / `--appcode`**：遍历已配置应用（默认仅 **项目** `apps`；需包含全局合并进来的应用时加 **`--global`**）
- **加 `--app <name>`**：仅列出指定应用的模型
- **加 `--appcode <code>`**：反查到对应 app profile，使用其 cookie/env

`inherit` 不是受支持的配置项；`--global` 始终显式读取全局和项目双层 apps。

## 风险等级

`read` — 仅读取，不修改文件。

## 前置条件

- 已完成 `rabetbase auth` 登录
- 已配置 appcode（单应用或多应用）
