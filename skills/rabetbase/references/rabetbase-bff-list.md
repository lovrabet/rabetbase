# bff list

列出当前 App 下的 BFF 脚本。

## 命令

```bash
rabetbase bff list --format json
rabetbase bff list --type COMMON --format json
rabetbase bff list --name getUserInfo --format json
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--type <type>` | string | 否 | `ENDPOINT` | 脚本类型：`ENDPOINT` / `COMMON` |
| `--name <name>` | string | 否 | — | 按脚本名过滤 |
| `--verbose` | boolean | 否 | — | 返回完整脚本对象 |
| `--format <fmt>` | string | 否 | `pretty` | 输出格式 |

## 输出

默认返回精简列表（id, functionName, description, scriptType）。`--verbose` 返回完整对象。

## BFF 脚本类型

| 类型 | 用途 | 说明 |
|------|------|------|
| ENDPOINT | 独立业务端点，前端通过 `client.bff.execute()` 调用 | 默认类型 |
| COMMON | 公共函数，供其他 BFF 脚本 import 复用 | 需 `--type COMMON` 查询 |
| HOOK | 挂在标准数据接口前后执行 | 通过数据集详情查看 |

## 提示

- 写 BFF 前先查公共函数列表（`--type COMMON`），看是否有可复用函数
- 修改已有 BFF 前必须先用此命令找到脚本 ID

## 参考

- [SKILL.md](../SKILL.md)
- [bff-creation-workflow.md](../guides/bff-creation-workflow.md)
