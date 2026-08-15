# rabetbase role user-resolve

把当前租户中的昵称/用户名解析为稳定 `userId`，用于 `user-add` / `user-remove` 前确认唯一人员。

## 命令

```bash
rabetbase role user-resolve --name 小张 --format compress
```

## 行为

- 精确匹配 `nickname` 或 `username`（大小写不敏感）。
- 命中 0 个：报 validation error，提示核对或改用 `--user <id>`。
- 命中多个：报歧义并列出候选 `nickname(userId)`，要求用 id 消歧。
- 输出 `data.user`：`userId`、`username`、`nickname`、`matchedBy`（`id` / `name`）。

## 边界

- 未登录或当前账号无法读取租户成员时会失败，应先恢复 rabetbase 登录状态。
- 只做只读解析，不写任何角色成员；实际加入用 [`role user-add`](rabetbase-role-user-add.md)。

## 参考

- [role user-add / user-remove](rabetbase-role-user-add.md)
- [role list](rabetbase-role-list.md)
- [SKILL.md](../SKILL.md)
