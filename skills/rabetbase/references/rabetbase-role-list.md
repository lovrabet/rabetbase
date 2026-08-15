# rabetbase role（list / detail / update / delete）

管理当前应用的开发角色。角色类型：`ADMIN` / `DEV` / `OWNER`（内置）与已有 `CUSTOM`。所有命令输出包含 `scope: "dev"`。当前支持 list、detail、update、delete，不提供 `role create`。

**内置角色守卫**：`update` / `delete` 仅限 `CUSTOM`；内置角色会被 validation 拒绝。成员增删不允许操作 OWNER，见 [`user-add / user-remove`](rabetbase-role-user-add.md)。业务人员角色与资源权限归属 `lovrabet`；若当前 `lovrabet` Help/Schema 尚未提供对应命令，应明确告知能力暂不可用，不得臆造命令、使用 rabetbase 代替或跨端双写。

写操作默认输出 `compress`，先 `--dry-run` 确认再 `--yes`。

## 命令

```bash
rabetbase role list --format compress
rabetbase role list --type DEV --format compress
rabetbase role detail --id 12 --format compress
rabetbase role update --id 12 --name 销售组-华东 --dry-run
rabetbase role delete --id 12 --dry-run
rabetbase role delete --id 12 --yes
```

## list / detail

`list` 列出开发角色，`--type` 可过滤 `ADMIN|DEV|OWNER|CUSTOM`；`--page` 默认 `1`，`--pagesize` 默认 `100` 且范围为 `1-100`。输出关注 `data.scope` 与 `data.roles[]`：`id`、`roleName`、`roleType`、`remark`、`userCount`、`permitCount`、`builtin`。

`detail --id` 按返回对象的 `id` 精确匹配。名称引用会覆盖应用内全部角色，避免遗漏后续分页。

## update

仅改 `CUSTOM` 角色的 `--name` / `--remark`（至少一个）。ADMIN / DEV / OWNER 被拒绝。输出 `data.before` / `data.after` 为角色投影。

## delete

`high-risk-write`；仅 `CUSTOM`。dry-run 会在角色仍有成员时给出 `warnings`。正式执行需 `--yes`。

## 输出协议（写命令）

`update` / `delete` 在 `data` 内返回稳定字段，Agent 读内层而非顶层 envelope：

```json
{
  "scope": "dev",
  "operation": "delete",
  "selector": { "appCode": "app-xxx", "roleId": 12, "roleName": "销售组" },
  "before": { "id": 12, "roleName": "销售组", "roleType": "CUSTOM", "builtin": false },
  "after": null,
  "dryRun": true,
  "warnings": []
}
```

不含合成 `protocol` key。Agent 只依赖上述稳定业务字段判断目标和变更，不读取底层请求信息。

## 用例编排

- 用例 1（把小张加入开发者）：`role user-resolve --name 小张` → `role list --type DEV` 取 devId → [`role user-add`](rabetbase-role-user-add.md)。

## 参考

- [role user-resolve](rabetbase-role-user-resolve.md)
- [role user-add / user-remove](rabetbase-role-user-add.md)
- [SKILL.md](../SKILL.md)
