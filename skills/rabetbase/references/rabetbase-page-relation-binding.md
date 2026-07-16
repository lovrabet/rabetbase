# page relation binding

页面关系绑定用于校准 Smart List Page 中选择类字段的 options 来源。执行前先读取数据集关系事实，再审计页面 schema，最后通过本地 schema 工作流做可回滚修改。

## 执行流程

```bash
rabetbase dataset relations --appcode <appCode> --datasetcode <datasetCode> --format compress
rabetbase page relation-audit --appcode <appCode> --datasetcode <datasetCode> --format compress
rabetbase page pull --appcode <appCode> --id <pageId> --format json
rabetbase page push --appcode <appCode> --id <pageId> --dry-run --format compress
```

`dataset relations` 是关系事实来源；底层默认读取 D0/DO V2，不需要在主流程显式传版本参数。不要靠页面 schema 反推出关系。展示字段以 `dataset relations` 的 `to.labelField` 为准，关系基数以 `relation.cardinality` 为准。`page relation-audit` 只读输出差异，不修改平台。修改页面前先 `page pull` 建立本地基线，提交前必须先跑 `page push --dry-run`。

Agent 不得假设存在自动修复命令，也不得把 audit 结果直接转成平台写入。修复必须走本地 schema 工作流：先 `page pull`，只改审计命中的本地字段节点和对应数据源，再用 `page push --dry-run` 复核差异，最后由明确授权的 `page push` 提交。

## 审计到处理边界

- `relation_options` 通常表示页面绑定与数据集关系匹配，不应作为错误处理。
- `custom_options` 表示字段或页面当前使用自定义选项，不一定是错误；先确认业务是否期望数据集关联选项。
- `wrong_label` / `wrong_code` / `wrong_state` / `missing` 只说明审计发现差异，不能直接承诺 `page sync` 自动修复。
- 若 `dataset relations` 关系事实本身不符合预期，先修正数据集关系并重新审计；若关系事实正确但页面绑定仍不一致，再按本地 schema 工作流或页面同步流程处理。
- `page sync` 是已有页面同步动作，不是页面 schema 审计结果的通用自动修复器。

## relation options 绑定规则

`optionsType=relation` 来自平台数据集关系时，使用 Page 层 `dataSource.list`，字段层绑定同名 state：

- `dataSource.id` 应稳定指向当前字段的 options 数据源，例如 `<field>_options` 或已有页面中的等价 id。
- 请求目标使用 `/api/<appCode>/<toDatasetCode>/getSelectOptions`。
- 值字段写在 `options.params.code`，数组参数形态写在 `paramsArr[name=code].value`。
- 展示字段写在 `options.params.label`，数组参数形态写在 `paramsArr[name=label].value`。
- 字段组件通过 `this.state.<dataSource.id>` 读取 options。

`optionsRequest` 只用于外部 options 源。平台数据集的 `getSelectOptions` 不放在字段级 `optionsRequest`，避免页面配置绑定到具体域名或环境。

## custom options

字段自身已有 `options` / `selectItems` 枚举数组时，服务端字段扩展为 `optionsType=custom`，审计结果可以是 `custom_options`。这类字段通常来自数据集枚举，不需要强制改成关系数据源。

## From 字段元数据

`page relation-audit` 会输出 `fromField`，包含 From 数据集字段的 `name`、`doType` / `type`、`optionsType` 和 `hasOptions`。其中 `fromField.optionsType` 与服务端字段扩展 `optionsType` 对齐：

- `relation_options`：页面字段使用 `getSelectOptions` / `JSExpression`，绑定匹配关系事实；对应服务端 `optionsType=relation`。
- `custom_options`：页面字段使用枚举数组，且 From 字段自身存在 `options` / `selectItems`；对应服务端 `optionsType=custom`。

`fromField.doType` / `fromField.type` 只作为审计上下文输出，不通过本地硬编码类型白名单拦截有效关系绑定。不同业务可能使用字符串、数字或其他类型保存关联值，判断应以关系事实和页面绑定为准。

## 结构化修改边界

只修改审计命中的字段节点和对应 `dataSource.id`。不要做全局字符串替换，也不要因为某个 label 字段变化就同时改其他数据源。

检查顺序：

1. 以 `page relation-audit` 的 `field`、`pageId`、`currentBinding.dataSourceId` 定位 schema。
2. 校准 `options.params.code` / `paramsArr[name=code].value`。
3. 校准 `options.params.label` / `paramsArr[name=label].value`。
4. 确认字段层 state key 与 `dataSource.id` 一致。
5. 运行 `page push --dry-run`，确认请求体只包含预期页面。

## 输出判断

- `relation_options`：字段使用 `getSelectOptions` / `JSExpression`，且绑定与数据集关系一致。
- `wrong_label`：展示字段不一致，优先修正 label。
- `wrong_code`：值字段不一致，需要确认关系方向后再修正。
- `wrong_state`：字段绑定的 state key 与数据源 id 不一致。
- `missing`：关系字段没有找到可识别绑定。
- `custom_options`：字段使用枚举数组，且 From 字段自身存在 `optionsType=custom`。
- `pageStatus=main_sub_missing`：`relation.bizRelationType=main_sub` 命中 CREATE / UPDATE / DETAIL 页面，且 `LrSmartCreate` / `LrSmartDetail` / `LrSmartUpdate` 中未识别到对象类型的 `props.sub_info`；`status` 仍保留原有字段绑定检查结果。
