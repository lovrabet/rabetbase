# cli-skill install

安装、重装或刷新 Rabetbase CLI Built-in Skill。

> **风险等级：write** — 全局安装 Skill 包。

## 命令

```bash
rabetbase cli-skill install

# 等价的显式非交互命令
npx skills add lovrabet/rabetbase -g -y
```

## 参数

无。

## 行为

默认执行 `npx skills add lovrabet/rabetbase -g -y`，全局安装或刷新 Rabetbase CLI Built-in Skill。

该命令只负责 CLI Built-in Skill 安装或刷新；业务 Skill 安装属于 `lovrabet skill install`。该命令不会自动修复本地 npm、网络、Git 或权限问题。

若设置了环境变量 `RABETBASE_SKIP_NPX_SKILLS=1`，则跳过安装（假定 skill 已存在）。

## 输出

- 成功：`CLI Built-in Skill installed`
- 跳过：`Skipped npx (RABETBASE_SKIP_NPX_SKILLS=1); assuming CLI Built-in Skill is already present.`
- 失败：报错并提示检查网络，或手动执行 `npx skills add lovrabet/rabetbase -g -y`

## 提示

- `rabetbase update` 默认会在更新后 best-effort 刷新 CLI Built-in Skill
- 手动执行适用于 skill 丢失、需要重装、或发现本地 skill 描述与当前仓库命令不一致时
- 若判断安装的全局 skill 已过期，先执行本命令，再重新读取 `skills/rabetbase/SKILL.md` 与所需 reference
- `rabetbase skill install` 与 `rabetbase skills` 已移除；刷新 Rabetbase CLI Built-in Skill 请使用本命令
- 安装业务 Skill 请使用 `lovrabet skill install`，不要迁移到 `rabetbase`

## 参考

- [SKILL.md](../SKILL.md)
- [update](rabetbase-update.md)
