# rabetbase role user-add / user-remove

把用户加入/移出角色。服务端 `update-user-list` 是**全量回写所有用户组成员**，CLI 内部做 read-merge-write：只改目标角色的 `userList`，其余角色原样带回，避免误删其它角色成员。`high-risk-write`，正式执行需 `--yes`。

## 命令

```bash
rabetbase role user-add --role 12 --user 小张 --dry-run
rabetbase role user-add --role 销售组 --user 1001 --yes
rabetbase role user-remove --role 12 --user 1001 --dry-run
rabetbase role user-remove --role 12 --user 1001 --yes
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--role <id\|name>` | 是 | 目标角色；纯数字按 id，否则按角色名精确匹配（重名报错） |
| `--user <id\|name>` | 是 | 目标用户；纯数字按 userId，否则按昵称/用户名解析（见 user-resolve） |
| `--expect-user-count <n>` | 否 | 防漂移：目标角色当前成员数不等于 n 即中止 |
| `--appcode <code>` | 否 | 覆盖当前 app |

## 行为与幂等

- `user-add`：用户已在该角色 → 不写、`data.warnings` 提示、`before == after`。
- `user-remove`：用户不在该角色 → 不写、`data.warnings` 提示。
- 有变更时才调用 `update-user-list`。

## 输出协议

```json
{
  "operation": "user-add",
  "selector": { "appCode": "app-xxx", "roleId": 12, "roleName": "销售组", "userId": 1001 },
  "before": { "userIds": [200] },
  "after": { "userIds": [200, 1001] },
  "dryRun": true,
  "backend": { "method": "POST", "path": "/smartapi/permit/user-role/update-user-list" },
  "warnings": []
}
```

Agent 校验时读 `data.before.userIds` / `data.after.userIds`，不要读顶层 envelope。dry-run 会附加不可枚举的 `method` / `url`。

## 用例编排

- 用例 1：`role user-resolve --name 小张` → `role list --type DEV` → `role user-add --role <devId> --user <userId> --yes`。
- 用例 3：`role user-resolve --name 小陈` → `role user-add --role <salesId> --user <userId> --yes`。

## 参考

- [role user-resolve](rabetbase-role-user-resolve.md)
- [role list](rabetbase-role-list.md)
- [SKILL.md](../SKILL.md)
