# 智能列表页（Smart List Page，page）工作流

前置知识：已完成 `rabetbase auth` 登录，当前上下文已有 `appcode`，并已理解 `knowledge/page-schema/` 是 PageSchema 组件知识，不是 `rabetbase page` 命令说明。

## 适用范围

本工作流只适用于 **智能列表页（Smart List Page，DOV2 `smart_page`）** 路径：

- 基于数据集首次生成智能列表页
- 数据集字段变更后同步已有智能列表页
- 将智能列表页 schema 拉到本地进行 IDE 修改，再推回平台

不适用于：

- 云端研发态 WebCode 页面
- 本地主子应用开发（使用 `rabetbase project`）
- 只凭“数据集缺页”就批量生成所有页面的场景

评论、附件、历史记录等子资源，默认不优先作为独立主页面；若业务目标不是标准智能列表页（Smart List Page），先停下来确认是否还应继续走 `page` 工作流。

## 核心原则

1. 智能列表页是“标准后台”方案，不是所有数据集的默认答案。
2. 平台是页面真源；本地 schema 开发也必须通过 `pull -> IDE 编辑 -> push` 闭环。
3. 新编排与自动化链路统一按 async-first 契约编排：任务状态查询与页面事实查询分离。
4. 生产环境先做 `--dry-run`，确认目标、页面组和风险后再执行真实写操作。

## 工作流

```text
确认业务目标 → 校验数据集与关联页 → 判断 generate-start 还是 sync → [按需]pull → IDE 编辑 → push --dry-run → push
```

### 1. 确认业务目标

先确认这次目标是不是智能列表页最擅长的事情：

- 列表 / 新建 / 编辑 / 详情型智能列表页（Smart List Page）
- 标准管理后台
- 单数据集或轻度关联数据集

如果目标是工作台、看板、时间轴、复杂协作视图，说明它不属于本地 `rabetbase page` 工作流；本地研发工程师应改走 `rabetbase project` 主子应用开发路径。

### 2. 校验数据集与关联页

执行以下命令拿事实，不要凭经验判断：

```bash
rabetbase dataset detail --code <datasetCode> --format json
rabetbase dataset links --format json
```

重点确认：

- 数据集是否存在
- 字段名、字段类型、关联关系是否符合预期
- `relatedPages` 中是否已有未删除智能列表页

### 3. 判断用 `generate-start / sync`

规则如下：

- **没有未删除关联页**
  - 用 [`page generate-start`](../references/rabetbase-page-generate-start.md)
- **已有未删除关联页** → 用 [`page sync`](../references/rabetbase-page-sync.md)

推荐先做预演：

```bash
rabetbase page standard-page-status --datasetcode <datasetCode> --format json
rabetbase page generate-start --datasetcode <datasetCode> --dry-run --format json
rabetbase page sync --datasetcode <datasetCode> --dry-run --format json
```

不要在已有页面时继续强行 `generate-start`，也不要在完全没页时直接 `sync`。

### 4. 拉取页面 schema 到本地

需要本地开发 formal schema 时，执行 [`page pull`](../references/rabetbase-page-pull.md)：

```bash
rabetbase page pull --id <pageId> --format json
rabetbase page pull --datasetcode <datasetCode> --format json
```

规则：

- `--id` 用于单页
- `--datasetcode` / `--alias` 用于按数据集批量
- 同一命令里 `--id` 与 `--datasetcode` / `--alias` 互斥
- 批量模式下若同一数据集存在多组页面且 `versionTag` 不唯一，必须显式传 `--version-tag`

本地文件位置：

- 页面文件：`.rabetbase/page/<appCode>/<pageId>-<TYPE>.json`
- lock 文件：`.rabetbase/page.lock.json`

### 5. 在 IDE 中修改 schema

本地修改时遵循两条底线：

1. 先 `pull` 建立基线，不要手工从零伪造页面文件
2. 如需理解 PageSchema 组件语义，再按需查看 `knowledge/page-schema/`；那是组件知识，不是 CLI 命令 reference

### 6. 推送前先做 `push --dry-run`

真实推送前，先执行 [`page push`](../references/rabetbase-page-push.md) 的预演：

```bash
rabetbase page push --id <pageId> --dry-run --format json
rabetbase page push --datasetcode <datasetCode> --version-tag <tag> --dry-run --format json
```

预演阶段重点看：

- 本地是否存在基线
- 远端页面版本是否已变化
- 批量模式下目标页面组是否正确

若命令提示先重新 `pull`，不要盲推覆盖。

### 7. 推送回平台

确认无误后再执行真实推送：

```bash
rabetbase page push --id <pageId> --format json
rabetbase page push --datasetcode <datasetCode> --version-tag <tag> --format json
```

推送成功后，CLI 会自动回拉远端 canonical schema 覆盖本地文件和 lock。这个行为是预期的，不要把它当作“本地改动丢失”；它的目的是让本地重新回到平台真源状态。

## 常见分支

### A. 首次补齐智能列表页

```text
dataset detail → standard-page-status → page generate-start --dry-run → page generate-start → page generate-status → page standard-page-status
```

适用于：

- 主业务对象还没有任何智能列表页
- 目标是快速补齐标准智能列表页（Smart List Page）

### B. 字段变更后同步已有智能列表页

```text
dataset detail → page sync --dry-run → page sync
```

适用于：

- 数据集字段已变化
- 关联智能列表页已经存在

### C. 本地 schema 开发

```text
page pull → IDE 编辑 → page push --dry-run → page push
```

适用于：

- 智能列表页已经生成
- 需要在 formal schema 层做本地定制

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-page-generate-start.md](../references/rabetbase-page-generate-start.md)
- [rabetbase-page-generate-status.md](../references/rabetbase-page-generate-status.md)
- [rabetbase-standard-page-status.md](../references/rabetbase-standard-page-status.md)
- [rabetbase-page-sync.md](../references/rabetbase-page-sync.md)
- [rabetbase-page-pull.md](../references/rabetbase-page-pull.md)
- [rabetbase-page-push.md](../references/rabetbase-page-push.md)
