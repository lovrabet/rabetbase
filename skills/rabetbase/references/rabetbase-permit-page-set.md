# rabetbase permit page-get / page-set

按菜单配置页面权限：菜单可见、数据操作（create/update/delete/detail/export）、行级可见（`dataPermit` = DATA_ROW）。角色字面量支持 `SELF` / `ALL` / 角色 id / 角色名（解析自 `get-permit-list`，含特殊组）。`SELF`（行级）**只允许**用于 `--row-roles`。

`page-set` 是 `high-risk-write`：服务端 `page/save` 回写完整 `IPagePermit`，CLI 读取完整快照后只改传入的槽位，其余原样回写；只能设置 `page-get` 中已存在的槽位。

## 命令

```bash
rabetbase permit page-get --menu-id 55 --format compress
rabetbase permit page-set --menu-id 55 --row-roles SELF --dry-run
rabetbase permit page-set --menu-id 55 --row-roles SELF --yes
rabetbase permit page-set --menu-id 55 --menu-roles ALL,销售组 --yes
```

## page-get

返回 `data.page`：`menuId`、`appCode`，以及 `menuPermit` / `dataCreate` / `dataUpdate` / `dataDelete` / `dataDetail` / `dataExport` / `dataPermit` 各槽的 `{ resourceCode, roles[] }`（roles 为 `{id, roleName, roleType}`）。槽位缺失表示该页面无此权限维度。

## page-set 参数

| Flag | 槽位 | 允许 SELF |
|------|------|-----------|
| `--menu-roles` | menuPermit（目录可见） | 否 |
| `--create-roles` | dataCreate | 否 |
| `--update-roles` | dataUpdate | 否 |
| `--delete-roles` | dataDelete | 否 |
| `--detail-roles` | dataDetail | 否 |
| `--export-roles` | dataExport | 否 |
| `--row-roles` | dataPermit（行级可见） | 是（SELF = 只看自己） |

每个值是逗号分隔的角色字面量。至少传一个槽位 flag。传入不存在的槽位会报错。

## 用例 5（请假列表只看自己）

行级 `SELF` 打在 `dataPermit`，**不要** revoke 菜单：

```bash
rabetbase menu list                              # 定位请假列表 menuId
rabetbase permit page-get --menu-id <id> --format compress
rabetbase permit page-set --menu-id <id> --row-roles SELF --dry-run
rabetbase permit page-set --menu-id <id> --row-roles SELF --yes
```

## 输出协议

```json
{
  "operation": "page-set",
  "selector": { "appCode": "app-xxx", "menuId": 55, "slots": ["dataPermit"] },
  "before": { "dataPermit": { "resourceCode": "ROW_1", "roles": [{ "id": -1, "roleName": "ALL", "roleType": "ALL" }] } },
  "after": { "dataPermit": { "resourceCode": "ROW_1", "roles": [{ "id": 0, "roleName": "SELF", "roleType": "SELF" }] } },
  "dryRun": true,
  "backend": { "method": "POST", "path": "/smartapi/permit/page/save" },
  "warnings": []
}
```

不含合成 `protocol`。校验读 `data.after.<slot>.roles`。

## 运行态验证交接

`page-set --row-roles SELF` 只写入权限配置。要验证「登录后 filter 是否只返回自己的行」，交给 `lovrabet data filter` 在运行态执行验证。

## 参考

- [permit role-menus-set / role-apis-set](rabetbase-permit-role-menus-set.md)
- [menu list](rabetbase-menu-list.md)
- [SKILL.md](../SKILL.md)
