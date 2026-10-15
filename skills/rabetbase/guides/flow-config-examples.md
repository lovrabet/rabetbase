# 完整 FlowConfig 示例

以下两份文件用于说明可保存的完整图，均需先用 `flow validate` 校验。页面 ID `101/102` 和用户 ID `REPLACE_WITH_RUNTIME_USER_ID` 是示例值，提交前必须按 [资源发现](../references/rabetbase-flow-resources.md) 替换为真实查询结果，不直接执行示例资源。

## 多页面表单审批

两个 CREATE 页面共用一个审批定义，首个审批人由 `assignee` 指定。实际发起只传本次页面的单个 `pageId`，并按 [表单启动](form-flow-start.md) 先保存记录再启动。此文件不绑定数据集、不配置自定义页面导航。

```json
{
  "flowName": "申请审批",
  "flowDesc": "两个申请页面共用主管审批",
  "flowType": "FORM_FLOW",
  "pageIds": [101, 102],
  "flowJson": {
    "nodes": [
      { "id": "start", "type": "START", "name": "提交", "position": { "x": 80, "y": 180 } },
      {
        "id": "manager", "type": "APPROVAL", "name": "主管审批",
        "position": { "x": 300, "y": 180 },
        "taskMode": "APPROVAL", "approvalMode": "SINGLE",
        "assignee": { "strategy": "FIXED", "userIds": ["REPLACE_WITH_RUNTIME_USER_ID"] }
      },
      { "id": "approved", "type": "END", "name": "通过", "result": "APPROVED", "position": { "x": 560, "y": 80 } },
      { "id": "rejected", "type": "END", "name": "拒绝", "result": "REJECTED", "position": { "x": 560, "y": 280 } }
    ],
    "edges": [
      { "id": "submit", "source": "start", "target": "manager", "conditionType": "ALWAYS" },
      { "id": "accept", "source": "manager", "target": "approved", "conditionType": "APPROVED" },
      { "id": "decline", "source": "manager", "target": "rejected", "conditionType": "REJECTED" }
    ]
  }
}
```

## 独立工作流：办理后审批

此图先由办理人补充材料，再由审批人决策。两个节点的示例用户应分别替换为确认的实际办理人。`HANDLE` 只有一条 `ALWAYS` 出边；需要按业务变量分支时，在其后增加 CONDITION 或 SCRIPT → CONDITION。

从零创建时可先保存以下不带导航的合法图，取得真实 `id/flowCode`，再生成业务页面；页面发布后回读 `runtimePageUrl`，补 `startPath` 与所需 `APPROVAL/END.path`，update 并 publish。宿主 SDK 必须具备 [Flow SDK](custom-page-flow-sdk.md) 能力；发起独立流程不要求先创建业务记录。

```json
{
  "flowName": "材料办理与审批",
  "flowType": "INDEPENDENT_FLOW",
  "flowJson": {
    "pageMode": "CUSTOM_PAGE",
    "nodes": [
      { "id": "start", "type": "START", "name": "发起", "position": { "x": 80, "y": 180 } },
      {
        "id": "prepare", "type": "APPROVAL", "name": "补充材料",
        "position": { "x": 280, "y": 180 },
        "taskMode": "HANDLE", "approvalMode": "SINGLE",
        "assignee": { "strategy": "FIXED", "userIds": ["REPLACE_WITH_RUNTIME_USER_ID"] }
      },
      {
        "id": "review", "type": "APPROVAL", "name": "审批",
        "position": { "x": 480, "y": 180 },
        "taskMode": "APPROVAL", "approvalMode": "SINGLE",
        "assignee": { "strategy": "FIXED", "userIds": ["REPLACE_WITH_RUNTIME_USER_ID"] }
      },
      { "id": "approved", "type": "END", "name": "通过", "result": "APPROVED", "position": { "x": 700, "y": 80 } },
      { "id": "rejected", "type": "END", "name": "拒绝", "result": "REJECTED", "position": { "x": 700, "y": 280 } }
    ],
    "edges": [
      { "id": "submit", "source": "start", "target": "prepare", "conditionType": "ALWAYS" },
      { "id": "prepared", "source": "prepare", "target": "review", "conditionType": "ALWAYS" },
      { "id": "accept", "source": "review", "target": "approved", "conditionType": "APPROVED" },
      { "id": "decline", "source": "review", "target": "rejected", "conditionType": "REJECTED" }
    ]
  }
}
```

按 [创建工作流](approval-flow-development-workflow.md) 继续资源核对、dry-run、保存、发布和运行验收。START → END 的占位图与模板名都不能替代实际人工节点。
