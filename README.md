# Rabetbase

面向 [skills](https://www.npmjs.com/package/skills) 开放生态的 **Rabetbase** Agent Skill 发布仓库。

[Lovrabet](https://www.lovrabet.com)（中文名云兔）是企业级 AI-Native 业务系统生成平台，通过首创的数据库逆向分析引擎，自动理解业务模型，实现端到端的业务系统自动开发，将企业沉睡的数据资产直接转化为可生长、可智能交互的 AI 驱动能力。

**Rabetbase** 是 Lovrabet 的 CLI 开发工具和 Agent Skill 体系，让 Cursor、Claude Code 等 AI 助手能够直接通过命令行管理数据集、SQL 查询、BFF 脚本和代码生成。

## 为什么需要这个 Skill

在 Lovrabet 项目里，开发任务不只是"写一段代码"，而是需要同时遵守平台约束、数据结构、CLI 工作流和多人协作规范。这个 Skill 把关键经验沉淀成可复用的工作流，让 AI 助手在真实项目里少走弯路。

核心覆盖：

* 通过 `rabetbase` CLI 管理数据集（Dataset）、自定义 SQL、BFF 脚本
* SQL 创建、校验（`sql validate`）、保存（`sql save`）与冲突处理
* BFF 脚本编写规范与安全保存流程
* Lovrabet TypeScript SDK 的正确使用方式（filter / aggregate / sql.execute / bff.execute）
* 多应用配置与切换（`app add` / `app use` / `--app`）
* 团队协作中的冲突检测与防覆盖规范

## 安装

推荐在项目根目录使用 `skills` CLI 安装：

```bash
npx skills add lovrabet/rabetbase -g -y
```

安装后，Skill 位于 `skills/rabetbase/` 下，遵循标准技能包结构。

## 前置条件

使用 Skill 前，确保已安装 `rabetbase` CLI：

```bash
npm install -g @lovrabet/rabetbase-cli
```

并完成认证和项目初始化：

```bash
rabetbase auth
rabetbase project init
```

## 仓库结构

```
README.md           ← 本说明
LICENSE
skills/rabetbase/
  SKILL.md          ← Skill 入口（意图路由 + 命令索引 + Agent 规则）
  references/       ← 每条 CLI 命令的详细参考文档
  guides/           ← 深入主题指南（SDK、SQL、BFF、冲突检测等）
```

## 适用场景

如果你在做这些事情，这个 Skill 会特别有帮助：

* 在 Lovrabet 项目中查询或修改数据集
* 编写自定义 SQL 并通过 CLI 校验、保存和测试
* 创建或更新 BFF (Backend Function) 脚本
* 在多成员协作场景下避免覆盖他人的 SQL / BFF 脚本
* 让 AI 助手按 Lovrabet 规范生成更可靠的前端或服务端代码
* 管理多应用项目配置（订单系统、商品系统等多应用协同）

## 进一步了解

* 官网：[www.lovrabet.com](https://www.lovrabet.com)
* 开发文档：[open.lovrabet.com](https://open.lovrabet.com/)
* CLI 文档：`rabetbase --help`
* Skill 详细说明：[`skills/rabetbase/SKILL.md`](skills/rabetbase/SKILL.md)

## License

[MIT](LICENSE)
