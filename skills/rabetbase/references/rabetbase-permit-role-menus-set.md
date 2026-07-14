# rabetbase permit role-menus / role-menus-set

查看与设置某角色的**菜单**权限矩阵（`roleType: PAGE`）。`role-menus-set` 是 `high-risk-write`：CLI 读取角色全量菜单资源，只翻转 `--menus` 指定的 `resourceCode`，其余保持当前动作后全量回写 `permits`。

**内置角色守卫**：DEV / ADMIN 的权限矩阵不可改，会被 validation 拒绝。

## 命令

```bash
rabetbase permit role-menus --role 销售组 --format compress
rabetbase permit role-menus-set --role 销售组 --revoke --menus MENU_FIN_1,MENU_FIN_2 --dry-run
rabetbase permit role-menus-set --role 销售组 --grant --menus MENU_SALES_1 --yes
```

## role-menus（读）

返回 `data.menus[]`：`resourceCode`、`resourceName`、`allowed`（是否 `access`）、`action`。用它拿到可用的 `resourceCode` 再去 set。

## role-menus-set 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--role <id\|name>` | 是 | 目标角色（DEV/ADMIN 拒绝） |
| `--grant` / `--revoke` | 是（二选一） | 授予（`access`）或收回（`0`） |
| `--menus <code,...>` | 是 | 逗号分隔的菜单 `resourceCode`；未知 code 报错 |
| `--appcode <code>` | 否 | 覆盖当前 app |

## 输出协议

```json
{
  "operation": "menu-revoke",
  "selector": { "appCode": "app-xxx", "roleId": 12, "roleName": "销售组", "resourceCodes": ["MENU_FIN_1"] },
  "before": [{ "resourceCode": "MENU_FIN_1", "action": "access" }],
  "after": [{ "resourceCode": "MENU_FIN_1", "action": "0" }],
  "dryRun": true,
  "backend": { "method": "POST", "path": "/smartapi/permit/role/update", "roleType": "PAGE" },
  "warnings": []
}
```

## 用例 4（菜单侧：销售组不看财务菜单）

```bash
rabetbase dataset list                      # 定位财务域
rabetbase permit role-menus --role <salesId>
rabetbase permit role-menus-set --role <salesId> --revoke --menus <财务菜单resourceCodes> --dry-run
rabetbase permit role-menus-set --role <salesId> --revoke --menus <...> --yes
```

一期不做自动 suggest，必须显式 `--menus`，避免误伤非财务菜单。API 侧收紧见 [`role-apis-set`](rabetbase-permit-role-apis-set.md)。

## 参考

- [permit role-apis-set](rabetbase-permit-role-apis-set.md)
- [permit page-get / page-set](rabetbase-permit-page-set.md)
- [SKILL.md](../SKILL.md)
