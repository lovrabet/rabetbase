# bff save（已退出主工作流）

> `bff save` 不再作为当前 CLI 的主工作流命令使用。
>
> Agent 看到旧文档、旧脚本或用户口头提到 `bff save` 时，统一迁移到下面这套流程：
>
> - 本地创建：`rabetbase bff create`
> - 查看状态：`rabetbase bff status`
> - 预览上传：`rabetbase bff push --dry-run`
> - 推送远端：`rabetbase bff push --yes`
> - 拉取远端：`rabetbase bff pull`
> - 删除脚本：`rabetbase bff delete --yes --target ...`

## 迁移规则

### 旧“新建并保存”

旧思路：

```bash
rabetbase bff save --file ./scripts/getUserInfo.js --name getUserInfo --format json
```

迁移为：

```bash
rabetbase bff create --type ENDPOINT --name getUserInfo --format json
# 编辑 .rabetbase/bff/<appCode>/ENDPOINT/getUserInfo.js
rabetbase bff push --dry-run --format json
rabetbase bff push --yes --format json
```

### 旧“更新已有脚本”

旧思路：

```bash
rabetbase bff save --file ./scripts/getUserInfo.js --id 42 --yes --format json
```

迁移为：

```bash
rabetbase bff pull --format json
# 编辑同步目录中的本地文件
rabetbase bff status --format json
rabetbase bff push --dry-run --format json
rabetbase bff push --yes --format json
```

## Agent 规则

- 不要再基于 `bff save` 设计新流程
- 如果用户只是沿用旧术语，先解释“当前主路径是 `bff create + bff push`”，然后直接按新流程执行
- 如果用户贴出历史脚本，优先帮他改造成 `create/push/pull/status/delete`

## 参考

- [SKILL.md](../SKILL.md)
- [bff-creation-workflow.md](../guides/bff-creation-workflow.md)
- [conflict-detection.md](../guides/conflict-detection.md)
- [rabetbase-bff-create.md](./rabetbase-bff-create.md)
- [rabetbase-bff-push.md](./rabetbase-bff-push.md)
