# Dataset design

使用随 CLI 分发、由使用方显式安装的 `rabetbase-dataset-generation` Skill，由调用方 Agent 生成结构化 DO 与关联清单。

从已安装 CLI 包的真实位置安装，并确认 Agent 能发现此 Skill：

```sh
npx skills add /path/to/rabetbase-cli/skills/rabetbase-dataset-generation -g -y
```

CLI postinstall、`cli-skill install` 与 `doctor` 只管理主 `rabetbase` Skill，此独立 Skill 需要显式安装或更新。

1. design-context：取得真实部署语言、版本及能力边界；引用已有实体前读取detail/relations。
2. design-validate --file raw.json --output normalized.json：服务端确定性校验，无模型调用、不创建业务产物。
3. design-start --file normalized.json --checkpoint operation.json：创建前保存恢复记录。
4. design-status --checkpoint operation.json：查询同一操作并核对创建集合。
5. 成功后使用dataset detail/relations/operations回读字段、枚举、主键和关系。

单表与批次使用同一kind=dataset-design/schemaVersion=1/doVersion=v2清单，类型与字段以context为准。已有历史单表任务可通过design-status --operation-id原ID查询，输出dataset_created/createdDataset；新清单返回datasets_created/createdDatasets与createdRelations。

未确认结果不自动重POST或换ID。checkpoint绑定节点/应用/主体/设计；同步失败通过design-sync-status/retry恢复现有datasetCode，不重新建模。dry-run不提交、不写文件或checkpoint。

完整生成规则和示例在同版本配套 Skill 的 references/workflow.md；不把页面创建或业务行数据初始化混入数据集保存结果。

PENDING / PROCESSING 都是非终态，继续查询同一操作；不得自动重提。
