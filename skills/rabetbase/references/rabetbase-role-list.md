# rabetbase role（list / detail / create / update / delete）

管理当前应用的用户组（角色）。事实源是平台角色管理面。角色类型：`ADMIN` / `DEV` / `USER`（内置）与 `CUSTOM`（用户创建）。

**内置角色守卫**：`update` / `delete` 仅限 `CUSTOM`；对 `DEV` / `ADMIN` 会被 validation 拒绝。修改权限矩阵见 [`permit`](rabetbase-permit-role-menus-set.md)。成员增删对所有角色可用，见 [`user-add / user-remove`](rabetbase-role-user-add.md)。

写操作默认输出 `compress`，先 `--dry-run` 确认再 `--yes`。

## 命令

```bash
rabetbase role list --format compress
rabetbase role list --type DEV --format compress
rabetbase role detail --id 12 --format compress
rabetbase role create --name 销售组 --dry-run
rabetbase role create --name 销售组 --remark "华东销售" --format compress
rabetbase role update --id 12 --name 销售组-华东 --dry-run
rabetbase role delete --id 12 --dry-run
rabetbase role delete --id 12 --yes
```

## list / detail

`list` 列出用户组，`--type` 可过滤 `ADMIN|DEV|USER|CUSTOM`；`--page` 默认 `1`，`--pagesize` 默认 `100` 且范围为 `1-100`。输出关注 `data.roles[]`：`id`、`roleName`、`roleType`、`remark`、`userCount`、`permitCount`、`builtin`。

`detail --id` 通过单条接口返回同结构 `data.role`。名称引用会在 CLI 内部遍历所有角色分页，避免遗漏第 100 条之后的角色。

## create

创建 CUSTOM 用户组（服务端固定 `roleType=CUSTOM`）。服务端 `role/create` 返回创建后的完整角色对象；CLI 直接填充 `data.selector.roleId` 与 `data.after`，不再按名称回查。

## update

仅改 `CUSTOM` 角色的 `--name` / `--remark`（至少一个）。DEV / ADMIN 被拒绝。输出 `data.before` / `data.after` 为角色投影。

## delete

`high-risk-write`；仅 `CUSTOM`。dry-run 会在角色仍有成员时给出 `warnings`。正式执行需 `--yes`。

## 输出协议（写命令）

`create` / `update` / `delete` 在 `data` 内返回稳定字段，Agent 读内层而非顶层 envelope：

```json
{
  "operation": "delete",
  "selector": { "appCode": "app-xxx", "roleId": 12, "roleName": "销售组" },
  "before": { "id": 12, "roleName": "销售组", "roleType": "CUSTOM", "builtin": false },
  "after": null,
  "dryRun": true,
  "backend": { "method": "POST", "path": "/smartapi/permit/role/delete" },
  "warnings": []
}
```

不含合成 `protocol` key。dry-run 会在同一 `data` 上附加不可枚举的 `method` / `url`，仅用于请求预览。

## 用例编排

- 用例 1（把小张加入开发者）：`role user-resolve --name 小张` → `role list --type DEV` 取 devId → [`role user-add`](rabetbase-role-user-add.md)。
- 用例 2（创建销售组）：`role create --name 销售组 --yes`，从 `data.selector.roleId` 取新 id。

## 参考

- [role user-resolve](rabetbase-role-user-resolve.md)
- [role user-add / user-remove](rabetbase-role-user-add.md)
- [permit role-menus-set / role-apis-set](rabetbase-permit-role-menus-set.md)
- [SKILL.md](../SKILL.md)
