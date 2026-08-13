# rabetbase kb

管理开发者侧当前解析应用下的 `company` 知识库。所有命令都通过标准工作区、`--app <name>` 或 `--appcode <code>` 解析目标应用，并在读写前后精确校验 `scope=company`、appCode 和 ID。

> 边界：`lovrabet kb` 使用运行态 AK/当前应用/当前用户，面向个人知识库和可见知识搜索；`rabetbase kb` 使用开发者登录态，面向公司知识库管理。不要用其中一个替代另一个。

## 命令

```sh
rabetbase kb list --format compress
rabetbase kb list --appcode app-xxxxx --title "业务规则" --format compress
rabetbase kb detail --id 60 --format compress
rabetbase kb create --title "业务规则" --file ./knowledge.md --dry-run
rabetbase kb create --title "业务规则" --file ./knowledge.md
rabetbase kb update --id 60 --file ./knowledge.md --expected-version 2 --dry-run
rabetbase kb update --id 60 --file ./knowledge.md --expected-version 2
rabetbase kb delete --id 60 --dry-run
rabetbase kb delete --id 60 --yes
```

## 内容文件与安全输出

- `create --file` 必填，`update --file` 可选；输入必须是可读的普通 UTF-8 文本或 Markdown 文件。
- CLI 保留解码后的文本，不解析 JSON、不重排段落，也不把路径映射转换成其他格式。
- 空白文件、目录、无效 UTF-8、缺失或不可读文件都会在网络请求前拒绝。
- 结构化输出不包含正文，只展示 `contentByteSize`、`contentHash`、`snapshotHash` 等安全元数据。

## list / detail

- `list` 自动读取全部分页，并强制过滤为精确 company scope 与当前 appCode；服务端的 appCode/title LIKE 结果不能扩大客户端作用域。
- `detail` 使用正安全整数 ID 精确读取，并验证返回 ID、scope 和 appCode。
- 服务端 Long 字段即使序列化为十进制字符串，CLI 也只接受能安全转换的整数；超出 JavaScript 安全整数范围时失败关闭。

## create

- `create/update` 为 `write`，正式执行不要求 `--yes`；仍应先审阅 dry-run。
- 只需 `--title`、`--file`；应用由标准全局应用解析得到。
- 写前按 `(company, appCode, title)` 精确检查重复。
- 审阅 dry-run 后，使用相同参数移除 `--dry-run` 正式执行；请求最多提交一次。
- 正常响应使用服务端返回 VO/ID 并按 ID 回读。只有底层响应不确定时，才按同一 company/app/title 三元组做只读恢复；零条或多条都返回 outcome unknown，绝不自动重提。
- 授权、权限、参数和其他确定性服务端错误直接失败，不能用旧状态回读伪装成成功。

## update

- `--id` 必填，`--title`、`--file` 至少传一个；服务端合同不支持 `--remark`。
- 标题单独更新只发送 `{id,title}`，不会合并并回写未修改的 content，也不强求版本递增。
- 内容更新发送 `{id,content}` 与可选 title；回读必须匹配内容哈希且版本高于写前版本。
- `--expected-version` 是非原子的 read-before-write 断言；服务端请求没有 version 条件，不能把它描述为 compare-and-set。
- 响应不确定时只按同一 ID 回读；证据不足返回 outcome unknown，不自动重提。

## delete

- 已验证服务端合同：`POST /admin/knowledge-base/delete`，请求体 `{id}`，成功为 `Result<Void>`。
- `delete` 保持 `high-risk-write`；正式执行必须显式 `--yes`，并建议先审阅 dry-run。
- 写前读取目标，校验 ID、company scope、appCode、可选 expected-version 和安全快照；`--expected-version` 仍是非原子断言，dry-run 明确输出 `atomicCompareAndSet: false`。
- 目标明确不存在时返回 no-op 且不发请求；只有服务端 `DATA_NOT_EXIST (306)` 可识别为不存在，`PARAM_INVALID (103)`、权限、认证或传输错误不能降级为 no-op。
- 正式删除最多提交一次，再按同一 ID 回读；明确不存在用于验证数据库软删除。
- 服务端在事务提交后异步清理 RAG。CLI 不验证 RAG 文件/索引清理，不宣称物理删除。

## 环境与授权

- 使用 `--env daily` 或 `--env production` 选择 Profile 环境，不通过 URL 参数切换。
- Production 写操作必须单独取得授权；需求、dry-run 或命令可用不等于写入授权。
- 输出和错误不得包含 Cookie、Token、AccessKey 或知识正文。
