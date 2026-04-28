# update

检查并更新 CLI 到指定 npm dist-tag 或版本，并刷新官方 rabetbase skill。

## 命令

```bash
# 检查并更新
rabetbase update

# 安装 npm beta dist-tag 指向的版本
rabetbase update --beta

# 安装指定版本
rabetbase update --version 2.1.8-beta.1

# 只更新 CLI，跳过官方 skill 刷新
rabetbase update --no-skills

# 查看帮助
rabetbase update --help
```

## 行为

1. 显示当前版本
2. 从 npm registry 解析目标版本：
   - 默认使用 `latest` dist-tag
   - `--beta` 使用 `beta` dist-tag
   - `--version` 使用显式版本，不请求 dist-tag
3. 比较当前版本与目标版本
4. 如需安装，自动执行安装命令（npm 或 bun，依据当前 CLI 运行时检测）
5. 安装完成后自动刷新官方 skill，除非传了 `--no-skills`
6. 完成后提示重启终端

## Flags

| Flag | 说明 |
|------|------|
| `--beta` | 安装 npm `beta` dist-tag 指向的版本。即使当前稳定版 semver 高于 beta 预览版，只要版本不完全相同，也会安装 beta |
| `--version <version>` | 安装指定 semver 版本，如 `2.1.8` 或 `2.1.8-beta.1` |
| `--no-skills` | 更新 CLI 后跳过官方 rabetbase skill 刷新 |

`--beta` 与 `--version` 互斥。

## 升级渠道

默认从 npm registry 安装的包更新。如使用 `bun install -g @lovrabet/rabetbase-cli` 安装，命令会自动检测并使用 `bun` 升级。

## 风险等级

`high-risk-write` — 执行全局包安装命令。

## 前置条件

- 全局安装（`npm install -g` 或 `bun install -g`）
- 网络连接 npmjs.org
- 若不传 `--no-skills`，还需要能执行 `npx skills add lovrabet/rabetbase -g -y`

## 示例

```bash
$ rabetbase update
Current version: 2.0.2+65
⠋ Checking for updates...
ℹ Update available: 2.0.2+65 → 2.1.0
⠋ Updating via npm...
✔ Updated to v2.1.0
  Restart your terminal to use the new version.
Checking official skill package...
  Official skill package is up to date.
```

```bash
$ rabetbase update --beta
Current version: 2.1.8 (62e252d, 2026-04-28)
ℹ Beta version: 2.1.8 (62e252d, 2026-04-28) → 2.1.8-beta.1
⠋ Updating via npm...
✔ Updated to v2.1.8-beta.1
  Restart your terminal to use the new version.
```

## 手动升级

如命令失败，可手动执行：

```bash
npm install -g @lovrabet/rabetbase-cli@latest
# 或
bun install -g @lovrabet/rabetbase-cli@latest
```
