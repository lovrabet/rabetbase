---
name: LrSmartEcharts
description: LrSmartEcharts 图表组件完整配置
tags: [echarts, chart, page-schema]
---

# LrSmartEcharts

图表组件，用于数据可视化。

---

## 禁止事项

| 禁止 | 说明 |
|------|------|
| 中文别名 | SQL 字段名和 AS 别名必须使用英文 |
| 编造字段 | 只能使用数据集字段列表里的 `code` |
| 无 LIMIT | SQL 必须包含 `LIMIT 1000` |
| 无 GROUP BY | 有聚合函数时必须 GROUP BY 所有非聚合字段 |

---

## 数据源配置

图表数据需要在 `page.dataSource.list` 中定义，组件通过 JSExpression 绑定：

```json
// Page 层：dataSource.list 配置（source/method/isCors/timeout 等平台自动补全，无需手写）
{
  "id": "chart_data_<nodeId>",
  "type": "fetch",
  "isInit": true,
  "options": {
    "uri": "/api/custom/execPreparedSql",
    "params": {
      "dbId": "<dbId>",
      "sqlContent": "SELECT source, COUNT(*) as customer_count FROM <tableName> WHERE 1=1 GROUP BY source LIMIT 1000",
      "params": {}
    }
  }
}

// Component 层：绑定 state
{
  "dataSource": {"type": "JSExpression", "value": "this.state.chart_data_<nodeId>"}
}
```

> `<dbId>` 从 dataset_detail_tool 获取，`<tableName>` 为数据集对应的物理表名，`<nodeId>` 为组件节点 ID。

---

## Props 属性

```json
{
  "dataSource": {"type": "JSExpression", "value": "this.state.chart_data_<nodeId>"},
  "cardItem": {
    "title": "图表标题",
    "query_config": {"sqlContent": "...", "datasetId": "<datasetId>"},
    "echarts_config": {
      "chartType": "bar",
      "dataMapping": {...},
      "option": {...},
      "theme": "default"
    }
  }
}
```

---

## 图表类型

| chartType | 用途 | dataMapping 要求 |
|-----------|------|------------------|
| bar | 对比、排行 | dimension + metrics |
| line | 趋势分析 | dimension + metrics |
| pie | 占比、构成 | dimension + metrics（单个） |
| bar-horizontal | 横向对比 | dimension + metrics |
| gauge | 单值 KPI | metrics（单个，无 dimension） |
| funnel | 转化漏斗 | dimension + metrics |
| radar | 多维评估 | dimension + metrics（多个） |
| scatter | 相关性分析 | metrics（1-2 个） |

---

## dataMapping 配置

```json
{
  "dimension": {"field": "source", "alias": "客户来源"},
  "metrics": [
    {"field": "customer_count", "alias": "客户数量", "type": "bar", "axis": "left", "stack": "total", "smooth": true, "area": true, "formatter": "(v) => v", "styleFunction": "(v) => {}"}
  ]
}
```

| 属性 | 说明 |
|------|------|
| dimension.field | 维度字段（X 轴/分组） |
| dimension.alias | 显示名称 |
| metrics[].field | 指标字段（Y 轴/数值） |
| metrics[].alias | 显示名称 |
| metrics[].type | 混合图表：`bar` / `line` |
| metrics[].axis | 双 Y 轴：`left` / `right` |
| metrics[].stack | 堆叠分组名 |
| metrics[].smooth | 平滑曲线（line） |
| metrics[].area | 面积图（line） |
| metrics[].formatter | 格式化函数：`(value) => string` |
| metrics[].styleFunction | 条件样式：`(value) => {color: 'red'}` |

---

## SQL 规范

```sql
SELECT {维度}, {聚合函数 AS 度量}
FROM {tableName}
WHERE 1=1
  [AND ...]
[GROUP BY {维度}]
ORDER BY {排序} {ASC|DESC}
LIMIT 1000
```

**硬性要求**：
1. 字段名使用数据集字段的 `code`
2. AS 别名必须英文
3. `WHERE 1=1` 必须保留
4. `LIMIT 1000` 必须存在
5. 有聚合函数时必须 GROUP BY

```sql
-- 示例
SELECT source, COUNT(*) as customer_count
FROM customer
WHERE 1=1
GROUP BY source
ORDER BY customer_count DESC
LIMIT 1000
```

---

## 主题

| theme | 匹配关键词 |
|-------|-----------|
| default | 默认、专业、蓝色 |
| colorful | 丰富、鲜艳、多彩 |
| ocean | 清新、柔和、蓝绿 |
| royal | 高端、紫色、金色 |
| sunset | 暖色、橙色 |
| forest | 绿色、自然 |
| midnight | 深色、大屏 |
| alert | 红色、告警 |
| everyday | 朴素、简约 |
| trendy | 潮流、炫酷 |
| dreamy | 柔和、粉色 |

---

## 常见配置

### 柱状图

```json
{
  "chartType": "bar",
  "dataMapping": {
    "dimension": {"field": "source", "alias": "客户来源"},
    "metrics": [{"field": "customer_count", "alias": "数量"}]
  },
  "option": {"xAxis": {"type": "category"}, "yAxis": {"type": "value"}}
}
```

### 饼图（环形）

```json
{
  "chartType": "pie",
  "dataMapping": {"dimension": {"field": "level_name", "alias": "等级"}, "metrics": [{"field": "count", "alias": "数量"}]},
  "option": {"series": [{"radius": ["45%", "70%"]}]}
}
```

### 混合图表（柱+线）

```json
{
  "chartType": "bar",
  "dataMapping": {
    "dimension": {"field": "month", "alias": "月份"},
    "metrics": [
      {"field": "sales", "alias": "销售额", "type": "bar"},
      {"field": "rate", "alias": "增长率", "type": "line", "axis": "right"}
    ]
  }
}
```

### 仪表盘

```json
{
  "chartType": "gauge",
  "dataMapping": {"metrics": [{"field": "completion_rate", "alias": "完成率"}]},
  "option": {"series": [{"progress": {"show": true}, "axisLine": {"lineStyle": {"width": 18}}}]}
}
```

### 堆叠图表

```json
{
  "chartType": "bar",
  "dataMapping": {
    "dimension": {"field": "month", "alias": "月份"},
    "metrics": [
      {"field": "product_a", "alias": "产品A", "stack": "total"},
      {"field": "product_b", "alias": "产品B", "stack": "total"}
    ]
  }
}
```

### 格式化

```json
{"metrics": [{"field": "amount", "alias": "金额", "formatter": "(v) => `¥${v.toLocaleString()}`}]}
{"metrics": [{"field": "rate", "alias": "比率", "formatter": "(v) => `${v}%`"}]}
```

### 条件样式

```json
{"metrics": [{"field": "profit", "alias": "利润", "styleFunction": "(v) => v < 0 ? {color:'red'} : {}"}]}
```

---

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 图表不显示 | SQL 错误 | 检查 SQL 语法和字段名 |
| 数据不对 | GROUP BY 缺失 | 添加 GROUP BY |
| 字段名错误 | 使用了中文别名 | 使用英文 AS 别名 |
| 查询超时 | 无 LIMIT | 添加 LIMIT 1000 |

---

## 完整示例

```json
{
  "componentName": "LrSmartEcharts",
  "props": {
    "dataSource": {"type": "JSExpression", "value": "this.state.chart_data_<nodeId>"},
    "cardItem": {
      "title": "客户来源分布",
      "query_config": {
        "sqlContent": "SELECT source, COUNT(*) as customer_count FROM <tableName> WHERE 1=1 GROUP BY source ORDER BY customer_count DESC LIMIT 1000",
        "datasetId": "<datasetId>"
      },
      "echarts_config": {
        "chartType": "bar",
        "dataMapping": {
          "dimension": {"field": "source", "alias": "客户来源"},
          "metrics": [{"field": "customer_count", "alias": "客户数量"}]
        },
        "option": {"title": {"text": "客户来源分布"}, "xAxis": {"type": "category"}, "yAxis": {"type": "value"}},
        "theme": "default"
      }
    }
  }
}
```
