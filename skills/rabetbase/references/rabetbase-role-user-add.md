# rabetbase role user-add / user-remove

把开发协作者加入或移出一个开发角色。命令只改变目标角色，不会同步业务人员角色。业务人员角色应使用 `lovrabet`；若对应命令当前不可用，应明确告知，不得改用 rabetbase 或跨端双写。

成员变更属于 `high-risk-write`，正式执行必须显式传 `--yes`。OWNER 成员不能通过这两个命令变更；所有权变更应走平台专用流程。

## 推荐顺序

1. 用 `role user-resolve` 把姓名解析为唯一 `userId`。
2. 按 SKILL 的“应用决议指引”确认应用；只有明确的 `app-...` 才能直接作为 appcode，不能把业务名或本地别名改写成 appcode。
3. 用 `role list` 取得目标开发角色。
4. 执行 `user-add` 或 `user-remove --dry-run`。
5. 检查 `data.before.userIds`、`data.after.userIds` 和 `data.warnings`。
6. 用户已授权时，复用相同参数并增加 `--yes` 正式执行。
7. 再用 `app members-list` 回读成员事实。

## 命令

```bash
rabetbase role user-resolve --name 小张 --format compress
rabetbase role list --type DEV --format compress

rabetbase role user-add --role 12 --user 1001 --dry-run
rabetbase role user-add --role 12 --user 1001 --yes

rabetbase role user-remove --role 12 --user 1001 --dry-run
rabetbase role user-remove --role 12 --user 1001 --yes
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--role <id\|name>` | 是 | 目标开发角色；纯数字按 id，否则按角色名精确匹配，重名时报错 |
| `--user <id\|name>` | 是 | 目标开发协作者；推荐先解析为唯一 userId |
| `--expect-user-count <n>` | 否 | 目标角色当前成员数不是 n 时中止，防止基于过期快照写入 |
| `--appcode <code>` | 否 | 覆盖当前应用 |

## 行为与幂等

- `user-add`：用户已在目标角色时不写入，`before == after`，并返回 warning。
- `user-remove`：用户不在目标角色时不写入，并返回 warning。
- `user-remove`：发现同一用户存在重复成员关系时会一并清理，并在 warning 中说明数量。
- 目标角色为 OWNER 时直接失败，不执行成员写入。
- 每次操作只影响目标开发角色；不得同时调用 lovrabet 做补偿或同步。

## 输出协议

```json
{
  "scope": "dev",
  "operation": "user-add",
  "selector": {
    "appCode": "app-xxx",
    "roleId": 12,
    "roleName": "开发者",
    "userId": 1001
  },
  "before": { "userIds": [200] },
  "after": { "userIds": [200, 1001] },
  "dryRun": true,
  "warnings": []
}
```

Agent 只读取 `data.scope`、`data.selector`、`data.before`、`data.after` 和 `data.warnings` 判断业务结果，不向产品用户展开底层请求、工程仓库或迁移过程。

## 常见失败

- 姓名命中多人：先执行 `role user-resolve`，再使用明确 userId。
- 找不到角色：执行 `role list`，不要猜角色 ID 或名称。
- 应用只给了业务名或别名：先按应用决议确认，不要把它直接传给 `--appcode`。
- OWNER 被拒绝：停止操作并说明需要平台所有权变更流程。
- `expect-user-count` 不一致：重新读取成员事实并重新生成 dry-run，不要强行跳过。
- lovrabet 暂无对应业务角色命令：如实告知当前不可用，不得用 rabetbase 代替。

## 参考

- [role user-resolve](rabetbase-role-user-resolve.md)
- [role list](rabetbase-role-list.md)
- [app members-list](rabetbase-app-members-list.md)
- [SKILL.md](../SKILL.md)
