# 表单创建与显式启动

适用于 `FORM_FLOW` 的页面运行逻辑。独立自定义页面工作流继续使用 [Flow SDK 指南](custom-page-flow-sdk.md)，不能把 `client.flow().start()` 的独立流程合同用于表单审批。

## 页面与流程绑定

- 设计态通过 `pageIds` 为一个流程绑定多个 CREATE 页面，`flow detail` 返回完整数组和兼容 CSV `pageId`。
- 运行态使用本次实际页面的详情 `/api/page/detail` 返回的 `hasApprovalFlow` 和 `flowCode`。同一数据集不同页面可以绑定不同流程，不能按 datasetCode 猜流程或选数组首项。
- `hasApprovalFlow=false` 时只保存业务数据；如果页面业务明确要求审批，则提示未配置可用流程，不伪造流程编码。
- 多个匹配、无权限或配置异常应显示错误；不得当作没有流程而静默跳过。

## 提交流程

执行前确认目标 Runtime 已支持显式启动合同、流程已发布且启用、实际页面在完整 `pageIds` 中并有有效菜单路由。本指南只适用于 `FORM_FLOW`；`INDEPENDENT_FLOW` / `CUSTOM_PAGE` 继续遵循 Flow SDK 合同。

DB_TABLE 业务表需要原主键和 `process_instance_id` 文本字段（长度至少 64）；METADATA 使用内置 PID 和索引保存流程关联。两种数据集均由 Runtime 绑定业务记录与流程实例，不再要求或维护 `instance_status`、`running_node`、`node_start_time`、`node_process_user`、`flow_status` 五个默认快照字段；业务模型中已配置的同名字段仍按真实业务契约使用。缺少必要字段或索引时先完成平台配置，不在页面中手工补系统字段。

1. 使用页面已初始化的 SDK 模型 `create(fields)` 保存业务记录，保存其返回的单个 `dataId`。支持该合同的 Runtime 数据创建不会自动启动审批；未核实目标版本时先确认行为，避免对旧自动发起结果再次 start。
2. 对有审批流程的实际页面，使用当前浏览器 Cookie 登录态调用 `POST /api/flow/start`。最小请求体是 `{ appCode, pageId, dataId }`；`flowCode`、`datasetCode` 可选。省略 `datasetCode` 时由服务端按实际页面解析主标准表单，已有调用也可显式传入已确认的正确编码。此可选规则仅针对 `FORM_FLOW` 启动，不代表普通 `create` 可以缺少实际目标模型。
3. `pageId` 是本次实际使用的单个正整数，即使只绑定一个页面也必传；启动不再创建业务数据。页面的 `mainDatasetId` / `mainDatasetCode` 在成功解析 CREATE、UPDATE、DETAIL 的标准主表单绑定后有值；绑定无法确定或页面为 FTP 列表等情况可能返回 `null`，不能据此判断没有流程。无法从实际页面解析主标准表单时，修正页面配置，或使用已确认正确的 `datasetCode`，不得猜测。
4. 成功读取响应 `data.processInstanceId`、`data.dataId` 和 `data.dataCreated`（固定为 `false`）；不要把整个 `data` 当标量业务 ID。

`formData`、`variables` 按该流程的业务变量契约传入；最小请求示例不代表可以省略审批条件或脚本实际需要的业务值。

使用项目已有的浏览器请求封装和已解析的 runtime 地址。SDK 未提供 FORM_FLOW 启动方法时，在页面 API 模块中封装该浏览器请求；不得改用 OpenAPI、AK、伪造用户或 `client.flow().start()`。下面是可供页面 API 模块复用的调用骨架；`client` 必须来自页面已初始化的 Cookie client，`input` 来自当前页面详情和业务数据创建结果：

```typescript
async function startFormApproval(client, input) {
  const { appCode } = client.getConfig();
  const response = await fetch(`${client.getBaseUrl().replace(/\/$/, "")}/api/flow/start`, {
    method: "POST",
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ ...input, appCode }),
  });
  const result = await response.json();
  if (!response.ok || result.success !== true) {
    throw new Error(result.errorMsg || result.msg || "Flow start failed");
  }
  return result.data;
}

// 在业务 create 成功后，仅将它返回的真实 id 作为 dataId。
const started = await startFormApproval(client, {
  flowCode: pageDetail.flowCode, // optional
  pageId: currentPageId,
  dataId: createdDataId,
  formData: submittedFields,
  variables: {},
  // datasetCode: confirmedDatasetCode, // only when page resolution cannot supply it
});
// started: { processInstanceId, dataId, dataCreated: false, ... }
```

启动成功后以响应中的 `processInstanceId` 作为流程关联事实；后端会同步 PID 到业务记录，页面不得再用普通 `update` 自行写入 PID。启动状态必须按该 PID 回读真实流程查询，PID 存在不等于流程已 `completed`。

## 失败和恢复

创建与启动是两次请求，创建成功后启动明确失败不会自动撤销已有业务记录。保留已返回的 dataId，提示“已保存，审批未发起”；修正明确失败原因后再对同一记录发起，不重新 create。启动响应丢失、超时或结果未知时，先按已有 dataId 回读业务记录的 PID，再用该 PID 查询真实流程；尚未确认结果时提示“已保存，审批状态待确认”，不得重新 create 或盲目重试。只有真实流程查询确认终态后才能展示审批完成、拒绝等结果，不能把 PID 存在当作 `completed`。页面提交时禁用重复点击，不能用本地防重代替服务端状态判断。

验证第二个绑定页面发起、同数据集不同页面流程、缺失/未绑定页面、非法或其他数据集的业务 ID、重复发起及创建成功但启动失败的恢复。普通创建成功只证明数据保存，不能描述为审批已启动。
