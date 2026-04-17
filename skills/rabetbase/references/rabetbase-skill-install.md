# skill install

安装、重装或刷新官方 rabetbase skill 包。

> **风险等级：write** — 全局安装 npm 包。

## 命令

```bash
rabetbase skill install

# 别名
rabetbase skills

# 等价的显式非交互命令
npx -y skills add lovrabet/rabetbase --global -y
```

## 参数

无。

## 行为

默认执行 `npx -y skills add lovrabet/rabetbase --global`，全局安装 rabetbase skill。

若需要在文档、脚本或 AI 引导里显式表达“无交互刷新”，可使用：

`npx -y skills add lovrabet/rabetbase --global -y`

若设置了环境变量 `RABETBASE_SKIP_NPX_SKILLS=1`，则跳过安装（假定 skill 已存在）。

## 输出

- 成功：`rabetbase skill installed`
- 跳过：`Skipped npx (RABETBASE_SKIP_NPX_SKILLS=1); assuming skill is already present.`
- 失败：报错并提示检查网络

## 提示

- 通常在 `rabetbase project upgrade` 的第 6 步自动执行
- 手动执行适用于 skill 丢失、需要重装、或发现本地 skill 描述与当前仓库命令不一致时
- 若判断安装的全局 skill 已过期，先执行本命令，再重新读取 `skills/rabetbase/SKILL.md` 与所需 reference
- `rabetbase skills` 是 deprecated alias

## 参考

- [SKILL.md](../SKILL.md)
- [project upgrade](rabetbase-project-upgrade.md)
