# cli-skill install

从当前 npm 包内安装、重装或刷新同版本 Rabetbase CLI Built-in Skill。命令通过 `npx --yes skills@latest` 使用包内本地路径作为安装源，由官方 Skills CLI 识别已安装的 Agent 并选择兼容目录；不会从外部 Skill 内容源下载文件。

## 命令

```bash
rabetbase cli-skill install
```

## 参数

无。

## 行为

- 安装源固定为当前 npm 包内的 `skills/rabetbase/`。
- 安装目录和多 Agent 兼容规则由最新版官方 Skills CLI 管理，Rabetbase 不自行选择或写入某个固定 Agent 目录。
- 每次安装请求都交给当前最新版 Skills CLI；若安装前后内容均为当前版本，结果标记为 `already current`。
- 安装前校验 npm 包内 Skill 的版本和整体摘要，安装后复验实际生效内容。
- 失败会返回稳定错误码和修复命令；不会回退到外部 Skill 内容源，也不自动修改 npm、Git 或网络配置。

业务 Skill 安装属于 `lovrabet skill install`，不要迁移到 `rabetbase`。

## 输出

- 首次安装或替换：输出 Skill 版本和目标路径。
- 已是当前版本：输出 `is already current`。
- 失败：输出本地错误码，并提示重新执行 `rabetbase cli-skill install`。

## 提示

- 安装或升级 `@lovrabet/rabetbase-cli` 时，npm `postinstall` 会自动安装包内同版本 Skill。
- `rabetbase update` 的语义是 CLI 与 Built-in Skill 一体升级。
- Skill 丢失、被修改或 npm 安装时跳过 scripts 后，可用本命令修复；需要 npm registry 可访问，以获取最新版 `skills` 安装器。
- 修复后重新读取 `SKILL.md` 与所需 reference，再继续当前任务。

## 参考

- [SKILL.md](../SKILL.md)
- [update](rabetbase-update.md)
- [doctor](rabetbase-doctor.md)
