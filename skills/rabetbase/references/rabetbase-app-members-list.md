# rabetbase app members-list

列出当前解析应用的完整人员清单，并将服务端角色分组整理为按 userId 去重的人员视图。只读，需要认证和 App Code。

## 命令

```bash
rabetbase app members-list --format compress
rabetbase app members-list --appcode app-example --format compress
```

应用选择遵循 CLI 统一规则：显式 `--appcode`、`--app` 或当前工作区默认应用。命令返回全量人员，不提供分页、关键字、状态或角色筛选。

## 结构化输出

```json
{
  "ok": true,
  "command": "rabetbase app members-list",
  "risk": "read",
  "data": {
    "appCode": "app-example",
    "items": [
      {
        "userId": 100,
        "username": "alice",
        "displayName": "Alice",
        "status": "ACTIVE",
        "roles": [
          {
            "roleId": 2,
            "roleName": "开发者",
            "roleType": "DEV",
            "membershipStatus": "ACTIVE"
          }
        ]
      }
    ],
    "totalCount": 1
  }
}
```

- `membershipStatus` 为 `ACTIVE`（正式成员）或 `PENDING`（待加入成员）。
- 同一用户只要存在一个 ACTIVE 角色，顶层 `status` 即为 `ACTIVE`；全部角色待加入时为 `PENDING`。
- 人员按 `userId` 升序；角色按 `roleId` 升序，同角色下 ACTIVE 排在 PENDING 前。
- `username`、`displayName` 缺失时为 `null`；不会用 userId 伪造。
- 不输出头像、手机、邮箱、Cookie、Token 或 AccessKey。

## 失败路径

- 未登录：先执行 `rabetbase auth login`。
- 缺少 App Code：在工作区绑定应用，或显式传 `--appcode` / `--app`。
- 当前账号无应用权限：保留服务端权限错误，不把错误解释为空列表。
- 网络异常或服务端错误：保留 CLI 结构化错误。

合法空应用返回成功，`items: []` 且 `totalCount: 0`。

## 参考

- [租户人员列表](rabetbase-tenant-members-list.md)
- [应用列表](rabetbase-app-list.md)
- [角色列表](rabetbase-role-list.md)
- [SKILL.md](../SKILL.md)
