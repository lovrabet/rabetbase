# page standard-page-status

查询某个数据集当前标准页事实快照。

## 命令

```bash
rabetbase page standard-page-status --datasetcode 097b7361b76c42bcb12b923fa5a08861 --format json
rabetbase page standard-page-status --alias order --format json
```

## 行为说明

- 该命令只返回**页面事实**：
  - 标准页组
  - 是否完整
  - 残留页
  - 菜单事实
- 它不返回异步任务的执行进度，也不负责表达 job 状态。
- 这是 `page generate-start` 的前置判定与终态回查真源，属于事实接口，不是任务状态接口。

## 适用场景

- 生成前判断是否允许继续 `generate-start`
- 生成完成后确认是否真的形成完整标准页组
- 排查为什么当前数据集被判定为 `blocked_*`

## 参考

- [rabetbase-page-generate-start.md](rabetbase-page-generate-start.md)
- [rabetbase-page-generate-status.md](rabetbase-page-generate-status.md)
