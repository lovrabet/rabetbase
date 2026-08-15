# db update

**增量更新**已有 dblink：只传需要改的 flag，未传的字段保持原值。`write`。

## 何时用

- 换库地址、改账号、轮换密码
- 改描述便于他人识别

## 命令

```bash
rabetbase db update --id 10157 --dburl new-host:3306 --format compress
rabetbase db update --id 10157 --password '***' --format compress
```

## 参数

| Flag | 必填 | 说明 |
|------|------|------|
| `--id` | **是** | dblink id（`db list`） |
| `--dbname` / `--dburl` / `--username` / `--password` / `--dbparam` / `--dbdesc` | 否 | 仅传要覆盖的项；**不传 `--password` 则不修改密码**（空字符串行为以后端为准） |

写操作支持 **`--dry-run`**：合并当前配置与更新项后预览（密码脱敏）。

非 dry-run 默认先测试合并后的完整配置，失败时不提交更新。只有明确传 `--skip-connection-test` 才跳过。已保存密码为掩码且未显式传真实 `--password` 时，在预检和保存前都会中止。

## 参考

- [database-connection-workflow.md](../guides/database-connection-workflow.md)
- [SKILL.md](../SKILL.md)
