# rabetbase deployment sync

将当前应用中指定数据库连接下的少量研发元数据同步到独立部署目标。命令只同步显式给出的业务编码，支持 `dataset`、`bff`、`sql`。

## 使用前确认

- 已完成 rabetbase 认证，并明确目标应用的 `appCode`。
- 从可信事实中取得基础 dblink／环境连接组 ID 和业务编码，不猜测 ID 或编码。
- `--target-env` 只选择服务端维护的环境连接映射；CLI 不保存数据库 URL、账号、密码或环境到连接 ID 的映射。
- 服务端同步接口会检查目标 appCode 的一键部署开关；未开启时正常跳过，不属于异常或无权限。
- 同一批最多 100 个编码；需要分批时逐批确认结果，不并发执行。

## 先预览

环境部署：

```sh
rabetbase deployment sync \
  --appcode <appCode> \
  --db-id <dbLinkId> \
  --target-env test \
  --type dataset \
  --codes <code1,code2> \
  --dry-run \
  --format compress
```

直连部署：

```sh
rabetbase deployment sync \
  --appcode <appCode> \
  --db-id <dbLinkId> \
  --direct \
  --type bff \
  --codes <code1> \
  --dry-run \
  --format compress
```

检查输出中的 `data.selector.targetMode`、`data.selector.targetEnv`、`data.selector.bizType`、`data.selector.bizCodes` 和 `data.selector.dbLinkId`。dry-run 不发起同步请求。

## 正式执行

复用已审阅的参数，移除 `--dry-run` 并追加 `--confirm`。CI 或非交互环境还需要全局 `--yes`：

```sh
rabetbase deployment sync \
  --appcode <appCode> \
  --db-id <dbLinkId> \
  --target-env test \
  --type dataset \
  --codes <code1,code2> \
  --confirm \
  --yes \
  --format compress
```

## 参数约束

- `--db-id`：正整数基础 dblink／环境连接组 ID；不是某个环境的最终连接 ID。
- `--target-env dev|test|pre|prod`：服务端环境连接映射选择器；与 `--direct` 必须且只能选择一个。
- `--type dataset|bff|sql`：同步资源类型。
- `--codes`：逗号分隔；命令 trim、过滤空项并按首次出现顺序去重，最多 100 个。
- 全局 `--env production|daily` 只决定 CLI 服务域，不是部署目标；不要用它替代 `--target-env`。

## 结果与失败处理

- 退出码 `0`：服务端没有返回 `FAILED`；`SKIPPED`、`UNCHANGED` 等 no-op action 或空 items 均表示无需执行，不作为异常。
- 退出码 `1`：至少一个条目为 `FAILED`。读取结构化输出的 `data.items[]`，保留成功项和失败项的完整事实，不要把整批误报为失败或成功，也不要自动重提整批。
- 退出码 `2`：写请求响应丢失或响应无法证明结果。同步可能已经生效；先人工核对目标部署，再决定是否处理未完成项，禁止直接重试。

Dataset 同步会先确保目标数据库连接存在，再逐项同步 Dataset，不是跨资源原子事务。命令当前不提供目标端只读回查；部署后的业务可用性仍需单独验收。

## 日常 BFF / SQL push 自动同步

只有用户明确要求将日常开发变更持续同步到独立部署测试环境时，才写入 app profile：

```sh
rabetbase workspace use --app <appName> --appcode <appCode> \
  --deployment-db-id <dbLinkId> --deployment-target-env dev
```

- 自动目标仅允许 `dev`、`test`、`pre`；生产和直连仍用显式命令并要求确认。
- 每个 app profile 只保存一个明确的自动目标，不根据 Git 分支、CLI 服务域或登录状态推断环境。
- 存在实际变更且主 push 成功后调用一次同步接口，由服务端检查 appCode 开关并按 `(dbLinkId, targetEnv)` 解析真实连接；未开启时正常 no-op，环境映射缺失则是配置错误。
- `bff push` 只同步本次上传成功且完成缓存清理的 function name。
- `sql push` 只同步本次 pushed sqlCode；未变化项不触发同步。
- Dataset 没有统一 push 入口，暂不自动同步。
- dry-run 展示配置目标和精确 code，但不发请求。
- 主 push 成功但后置同步失败时，不声称回滚主操作；只有同步请求已提交后才按 `data.deploymentSync.recoveryCommand` 人工处理。
- `partial_failure`、`failed`、`outcome_unknown` 均不得自动重试。
