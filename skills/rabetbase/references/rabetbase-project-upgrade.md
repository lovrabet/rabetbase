# project upgrade

升级已有项目的 SDK 依赖与应用源码结构。新项目继续使用 `project create`；Dataset 事实刷新继续使用 `api pull`。

`.lovrabet.json`、`.lovrabet/` 与 Lovrabet Skill 属于 Lovrabet 运行态 CLI，可与 Rabetbase 项目配置和源码共存。该命令不会读取、迁移、改写或删除这些运行态资产。

## 命令

```bash
# 交互模式（先分析 → 展示报告 → 确认 → 执行）
rabetbase project upgrade

# 只查看完整迁移计划，不写文件
rabetbase project upgrade --dry-run

# 自动模式（跳过确认）
rabetbase project upgrade --yes
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--yes` | boolean | 否 | `false` | 跳过确认提示，直接执行 |
| `--dry-run` | boolean | 否 | `false` | 只输出依赖和源码升级计划 |

## 执行流程

两阶段模式：先分析 → 展示报告 → 确认 → 执行 2 步：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 升级 SDK 依赖 | 将已有 `@lovrabet/sdk` 依赖更新到当前维护版本 |
| 2 | 升级应用源码 | 迁移稳定 API 入口、模型注册表、运行时选择器和项目 Domain 路由 |

## 源码迁移安全规则

- 先执行 `rabetbase project upgrade --dry-run`；确认 `apiArchitecture.groups[].files[]` 的动作后再正式执行。
- 缺失文件直接创建；可识别的旧 CLI 生成文件先备份到 `.rabetbase/project-upgrade/api-model-registry-v2/backups/` 再替换。
- 发现业务定制的 `api.ts` / `client.ts` 时不覆盖原文件，在 `candidates/` 下写入候选文件，并返回 `manualMergeRequired: true`。
- 旧 `api.ts` 中可识别的 Dataset 事实迁入 `models.generated.ts`，迁移后再运行 `rabetbase api pull --format compress` 刷新远端事实。
- 多个未配置 `apiGroup` 且共用同一 `apiDir` 的历史 profile 保留 `[name-]api.ts` / `[name-]client.ts`，不会被静默归并。
- 命令可重复执行；已是最新稳定入口时不重复改写。

## 分析报告

执行前会扫描并展示：
- `@lovrabet/sdk` 是否需要更新
- 每个 API 目录的 profile、文件动作、备份或候选文件路径

不会把 `.lovrabet.json`、`.lovrabet/`、Lovrabet Skill 或 Lovrabet 的 IDE 配置列为待清理项。

## 输出

- 每步显示 OK / FAIL 和详情
- 最终汇总表：所有步骤的执行结果
- 全部成功：`Upgrade completed successfully!`
- 部分失败：提示检查 summary

## 触发方式

- 手动执行：`rabetbase project upgrade`
- 已有应用需要采用当前源码架构：先 `rabetbase project upgrade --dry-run`，再正式升级

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase config init](rabetbase-init.md)
