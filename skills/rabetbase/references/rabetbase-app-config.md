# rabetbase app-config

管理当前应用的运行态 app-config key。事实源是平台 app-config 管理面，不是本地 `.rabetbase.json`；本地配置仍用 `rabetbase config` / `rabetbase app` 管理。

`value` 默认按敏感内容处理：`list` 和默认 `get` 不输出明文，`set --dry-run` / `delete --dry-run` 也不输出明文。不要把 app-config value 写入本地配置、缓存、日志或命令参数转发给业务 BFF。

## 命令

```bash
rabetbase app-config list --format compress
rabetbase app-config get <key> --format compress
rabetbase app-config get <key> --reveal --format compress
rabetbase app-config set <key> --from-env ENV_NAME --dry-run
rabetbase app-config set <key> --from-env ENV_NAME --yes
rabetbase app-config delete <key> --dry-run
rabetbase app-config delete <key> --yes
```

## list

```bash
rabetbase app-config list --format compress
rabetbase app-config list --tags webhook --format compress
rabetbase app-config list --key vectorengine.apiKey --format compress
```

输出关注：

- `data.configs[]`
- `key`
- `tags`
- `hasValue`
- `valueRedacted`
- `gmtModified`

不会输出 `value`，也不展示服务端预留的 `encrypted` 字段。

## get

```bash
rabetbase app-config get vectorengine.apiKey --format compress
```

默认只用于确认 key 是否存在、是否有值、路由 appCode 是否正确。

只有明确需要查看单个 key 明文时才使用：

```bash
rabetbase app-config get vectorengine.apiKey --reveal --format compress
```

`--reveal` 只支持单 key，不存在批量 reveal。

## set

`set` 以 key 为主路径，CLI 会先按 key 查询：存在则 update，不存在则 create。用户不需要传服务端 id。

value 来源必须且只能选一种：

```bash
printf '%s' "$VECTORENGINE_API_KEY" | rabetbase app-config set vectorengine.apiKey --stdin --dry-run
rabetbase app-config set vectorengine.apiKey --from-env VECTORENGINE_API_KEY --dry-run
rabetbase app-config set vectorengine.apiKey --file ./secret.txt --dry-run
```

确认 dry-run 的 `operation`、`selector.appCode`、`selector.key`、`before`、`after` 后，再执行正式写入：

```bash
rabetbase app-config set vectorengine.apiKey --from-env VECTORENGINE_API_KEY --yes
```

显式位置参数 `<value>` 可用，但不推荐用于敏感值，因为 shell history 可能记录明文。

## delete

```bash
rabetbase app-config delete vectorengine.apiKey --dry-run
rabetbase app-config delete vectorengine.apiKey --yes
```

删除同样以 key 定位，CLI 内部解析服务端 id 后调用 delete。缺失 key 会报 validation error。

## 安全边界

- `list` 永不批量输出明文 value。
- 默认 `get` 不输出明文 value。
- `set/delete --dry-run` 不输出明文 value，也不写入。
- `set` 的成功/失败输出只报告 key、appCode、operation 和脱敏上下文。
- 不依赖、不展示、不判断 `encrypted`。
- 不把运行态 app-config 混入 `rabetbase config`。
- 当前不把该能力描述为 KMS、密钥托管、自动轮换或后端加密能力。

## 业务消费

rabetbase 负责维护运行态 app-config 管理面。业务代码需要敏感配置时，应由服务端 BFF 上下文通过 `context.appConfig.get(...)` 读取并消费；不要让 Agent 先 `--reveal` value，再把明文作为 `lovrabet bff exec` 参数或脚本参数传入。
