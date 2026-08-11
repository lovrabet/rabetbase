# update

从 npm 更新 rabetbase CLI，并让 CLI 与 Built-in Skill 一体升级。

## 命令策略

- 默认升级入口是 `rabetbase update`。
- 预览版本使用 `rabetbase update --beta`；精确版本使用 `rabetbase update --version <version>`。
- 内部发布或故障复现可使用 `npm install -g @lovrabet/rabetbase-cli@<version>`。

## 命令

```bash
rabetbase update
rabetbase update --beta
rabetbase update --version 2.1.8-beta.1
rabetbase update --help
```

## 行为

1. 显示当前版本。
2. 默认解析 npm `latest`；`--beta` 解析 `beta`；`--version` 使用指定完整 semver。
3. 通过当前包管理器全局安装目标 npm 包。
4. 新 npm 包的 `postinstall` 调用最新版官方 Skills CLI，以包内本地路径安装同版本 Built-in Skill。
5. 安装目录由官方 Skills CLI 按已安装 Agent 决定，发起更新的进程验证实际生效路径和版本。
6. CLI 已是目标版本时，仍以当前 npm 包内文件检查并修复 Skill。

## Flags

| Flag | 说明 |
|------|------|
| `--beta` | 安装 npm `beta` dist-tag 指向的版本 |
| `--version <version>` | 安装指定完整 semver，如 `2.1.8` 或 `2.1.8-beta.1` |

`--beta` 与 `--version` 互斥。CLI 与 Skill 不提供拆分升级模式。

## 前置条件

- CLI 通过 npm 或 bun 全局安装。
- 能访问 npm registry，以安装 CLI 并获取最新版 `skills` 安装器；Skill 内容仍来自当前 Rabetbase npm 包，不访问外部 Skill 内容源。

## 修复

若 npm 包安装成功但 Skill 校验失败，执行：

```bash
rabetbase cli-skill install
rabetbase doctor
```

如 update 命令本身失败，可精确安装 npm 包；其 `postinstall` 仍会自动安装同版本 Skill：

```bash
npm install -g @lovrabet/rabetbase-cli@latest
```
