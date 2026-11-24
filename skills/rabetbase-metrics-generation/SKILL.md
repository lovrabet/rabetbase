---
name: rabetbase-metrics-generation
displayName: 企业指标开发与交付
version: 2.6.1
description: "Use when 用户要求创建、修改或交付应用级业务指标，或已明确的业务目标需要补齐应用级统计能力。也适用于核验已有指标是否满足需求。不用于仅查询或消费已有指标、仅因报告缺数自动开发、权限提权或替企业决定未知业务口径。"
metadata:
  requires:
    bins: ["rabetbase"]
    node: ">=24.0.0"
---

# 企业指标开发与交付

指标体系是应用级的，定义与计算实现属于企业的具体应用，以 AppCode 和指标名定位；登记或发布不改变企业资产归属，不产生跨应用或跨企业访问授权。指标体系按需建设，不是运营报告的必选项。任务决定工具，当前账号在目标应用的实际操作权限决定能否执行，不以岗位为触发或准入条件。本 Skill 负责指标开发与验收，任务管理、协作和会话上下文获取由 Agent 与宿主负责。把明确的业务问题交付为可引用、可验证的统计能力。已有同口径能力满足需求时核验并返回引用，不重复开发。独立研发任务可直接进入，不要求先有报告任务。

开发指标时，使用随本次所用 CLI 提供的 `rabetbase` built-in CLI skill，读取其中的 `guides/metric-development.md` 和 `references/rabetbase-metric.md`。通过同一 CLI 入口运行 `schema` 核对命令参数，并执行后续开发命令。缺少所需工具时，见[按需准备](references/workflow.md#按需准备)。

1. 按[开发与交付流程](references/workflow.md)核对口径、授权及已有能力。
2. 按[验证清单](references/verification.md)取得分项证据；不把发布成功当作业务验收。
3. 使用[交付摘要](templates/statistics-delivery.md)返回可读取的云端入口、适用范围、验证证据和限制。

业务定义交企业负责人决定，权限缺口交管理员；超出授权的结构变更单列。云端当前定义是执行事实来源，本地 JSON 仅为编辑辅助，不要求用户拥有源码仓。两个 CLI 分别核验认证和实际操作权限，同一用户无需切换账号；不传递凭证或以管理员结果代替当前账号结果。
