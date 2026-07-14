# rabetbase permit role-apis-set

按 `datasetCode` 收回角色的**数据集 API** 访问（`roleType: API`）。`high-risk-write`。一期以 `--revoke` 为主满足用例 4（销售组不能访问财务数据接口）。

`datasetCode → resourceCode` 映射与当前全量 API 权限状态来自内部读取的 `get-api-list`（服务端标 `@Deprecated`，CLI 内部读、不做公开命令）。CLI 读取全量后只把指定 datasets 的 resourceCodes 置为 `0`，其余保持当前动作后全量回写。

## 命令

```bash
rabetbase permit role-apis-set --role 销售组 --revoke --datasets ds_finance --dry-run
rabetbase permit role-apis-set --role 销售组 --revoke --datasets ds_finance,ds_invoice --yes
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--role <id\|name>` | 是 | 目标角色（DEV/ADMIN 拒绝） |
| `--revoke` | 是 | 一期语义固定为收回；缺失会报 validation |
| `--datasets <code,...>` | 是 | 逗号分隔 datasetCode；该角色 API 资源里不存在的 code 报错 |
| `--appcode <code>` | 否 | 覆盖当前 app |

## 输出协议

```json
{
  "operation": "api-revoke",
  "selector": { "appCode": "app-xxx", "roleId": 12, "roleName": "销售组", "datasetCodes": ["ds_finance"] },
  "before": [{ "resourceCode": "API_FIN_1", "datasetCode": "ds_finance", "action": "access" }],
  "after": [{ "resourceCode": "API_FIN_1", "datasetCode": "ds_finance", "action": "0" }],
  "dryRun": true,
  "backend": { "method": "POST", "path": "/smartapi/permit/role/update", "roleType": "API" },
  "warnings": []
}
```

不含合成 `protocol`。校验读 `data.after`（应全部为 `action: "0"`）。

## 用例 4（完整：菜单 + API 双侧收紧）

```bash
rabetbase dataset list                       # 定位财务域 datasetCode
rabetbase permit role-menus-set --role <salesId> --revoke --menus <财务菜单> --dry-run|--yes
rabetbase permit role-apis-set  --role <salesId> --revoke --datasets <财务datasetCode> --dry-run|--yes
```

菜单不可见 + 数据接口不可访问，两侧都要收紧。真实「登录后是否读不到财务行」用 `lovrabet` 运行态验证。

## 参考

- [permit role-menus / role-menus-set](rabetbase-permit-role-menus-set.md)
- [dataset list](rabetbase-dataset-list.md)
- [SKILL.md](../SKILL.md)
