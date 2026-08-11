# rabetbase tenant members-list

按精确 `tenantCode` 列出当前登录账号所属租户的完整人员清单。只读，不要求 App Code。

## 命令

```bash
rabetbase tenant members-list \
  --tenant-code tenant-example \
  --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--tenant-code <code>` | 是 | 精确租户编码；不支持名称猜测 |
| `--format <compress\|json\|pretty>` | 否 | Agent 优先使用 `compress` |

命令返回全量人员，不提供分页、关键字、状态或角色筛选。

## 安全边界

命令先查询当前账号所属的启用租户关系。只有 `tenantCode` 精确命中时，才继续读取租户人员；未命中会报错，且不请求成员详情。

输出不会包含手机、邮箱、头像、Cookie、Token 或 AccessKey。

## 结构化输出

```json
{
  "ok": true,
  "command": "rabetbase tenant members-list",
  "risk": "read",
  "data": {
    "tenantCode": "tenant-example",
    "items": [
      {
        "userId": 100,
        "username": "alice",
        "displayName": "Alice",
        "role": "ADMIN",
        "status": "ENABLE"
      }
    ],
    "totalCount": 1
  }
}
```

- `role` 为 `ADMIN` 或 `MEMBER`。
- 返回服务端目录中的全部状态，包括 `ENABLE` 与 `DISABLE`。
- `username`、`displayName`、`status` 缺失时为 `null`；不会用 userId 伪造。
- 结果按 `userId` 升序，同一 userId 只保留一个人员条目。

## 失败路径

- 未登录：先执行 `rabetbase auth login`。
- 缺少 `--tenant-code`：补充精确租户编码。
- 租户不属于当前账号：核对编码或切换登录账号；不要尝试枚举其他租户。
- 租户不存在、网络异常或服务端错误：保留 CLI 结构化错误，不把错误解释为空列表。

合法空租户返回成功，`items: []` 且 `totalCount: 0`。

## 参考

- [应用人员列表](rabetbase-app-members-list.md)
- [角色查人](rabetbase-role-user-resolve.md)
- [SKILL.md](../SKILL.md)
