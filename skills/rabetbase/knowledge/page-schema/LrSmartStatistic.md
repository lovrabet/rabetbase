---
name: LrSmartStatistic
description: LrSmartStatistic 统计卡片组件完整配置
tags: [statistic, page-schema]
---

# LrSmartStatistic

统计卡片组件，用于展示单值 KPI。

---

## 数据源配置

统计数据需要在 `page.dataSource.list` 中定义，组件通过 JSExpression 绑定：

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
      "sqlContent": "SELECT COUNT(*) AS value FROM <tableName> WHERE 1=1 LIMIT 1000",
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
    "title": "客户总数",
    "display_type": "statistic",
    "query_config": {"sqlContent": "...", "datasetId": "<datasetId>"},
    "statistic_config": {
      "value": "value",
      "prefix": "¥",
      "suffix": "个",
      "precision": 2,
      "groupSeparator": ",",
      "theme": "default",
      "formatter": "(v) => v"
    }
  }
}
```

---

## 禁止事项

| 禁止 | 说明 |
|------|------|
| 修改 `id` | 会丢失数据接口绑定 |
| 修改 `statistic_config.value` | 必须为 "value" |
| `meta.handleProps` | 使用纯 JSON |
| SQL 返回多行多列 | 必须单行单列 |
| SQL 不使用 `AS value` | 列名必须为 value |

---

## SQL 规范

**必须满足**：
1. 返回单行单列，列名为 `value`
2. 使用 `WHERE 1=1`
3. 保留 `LIMIT 1000`
4. 字段名使用数据集字段的 `code`

```sql
SELECT {聚合函数} AS value FROM <tableName> WHERE 1=1 [AND ...] LIMIT 1000
```

**常用聚合**：`COUNT(*)` / `SUM(field)` / `AVG(field)` / `MAX(field)` / `MIN(field)`

**示例**：
```sql
SELECT COUNT(*) AS value FROM <tableName> WHERE 1=1 LIMIT 1000
SELECT SUM(amount) AS value FROM <tableName> WHERE 1=1 AND status=2 LIMIT 1000
SELECT AVG(score) AS value FROM <tableName> WHERE 1=1 LIMIT 1000
```

---

## statistic_config 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| value | string | 固定 `"value"` |
| prefix | string | 前缀（如 `¥`） |
| suffix | string | 后缀（如 `个`） |
| precision | number | 小数位：计数类=0，金额类=2 |
| groupSeparator | string | 千分位分隔符：`,` |
| theme | string | 主题（见下表） |
| formatter | string | 自定义格式化：`(value) => string` |
| valueStyle | object | 自定义样式（仅 theme 不满足时使用） |

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

## formatter 自定义格式化

用于大数字转换（万/亿）、百分比等场景：

```json
// 大数字转万
{"formatter": "(value) => value >= 10000 ? `${(value/10000).toFixed(1)}万` : `${value}`"}

// 自动转换单位
{"formatter": "(value) => { if (value >= 100000000) return `${(value/100000000).toFixed(2)}亿`; if (value >= 10000) return `${(value/10000).toFixed(1)}万`; return `${value}`; }"}

// 百分比
{"formatter": "(value) => `${(value * 100).toFixed(1)}%`"}
```

**注意**：使用 formatter 时，`prefix`/`suffix`/`precision` 可能不生效

---

## 常见场景

### 计数类

```json
{
  "title": "客户总数",
  "query_config": {"sqlContent": "SELECT COUNT(*) AS value FROM <tableName> WHERE 1=1 LIMIT 1000", "datasetId": "<datasetId>"},
  "statistic_config": {"value": "value", "suffix": "个", "precision": 0, "groupSeparator": ","}
}
```

### 金额类

```json
{
  "title": "销售总额",
  "query_config": {"sqlContent": "SELECT SUM(amount) AS value FROM <tableName> WHERE 1=1 LIMIT 1000", "datasetId": "<datasetId>"},
  "statistic_config": {"value": "value", "prefix": "¥", "precision": 2, "groupSeparator": ","}
}
```

### 时间范围

```json
// 本月
{"sqlContent": "SELECT SUM(amount) AS value FROM <tableName> WHERE 1=1 AND YEAR(create_time)=YEAR(CURDATE()) AND MONTH(create_time)=MONTH(CURDATE()) LIMIT 1000"}

// 最近30天
{"sqlContent": "SELECT COUNT(*) AS value FROM <tableName> WHERE 1=1 AND create_time >= DATE_SUB(CURDATE(), INTERVAL 30 DAY) LIMIT 1000"}
```

---

## 完整示例

```json
{
  "componentName": "LrSmartStatistic",
  "props": {
    "dataSource": {"type": "JSExpression", "value": "this.state.chart_data_<nodeId>"},
    "cardItem": {
      "title": "本月销售总额",
      "display_type": "statistic",
      "query_config": {
        "sqlContent": "SELECT SUM(amount) AS value FROM <tableName> WHERE 1=1 AND YEAR(create_time)=YEAR(CURDATE()) AND MONTH(create_time)=MONTH(CURDATE()) LIMIT 1000",
        "datasetId": "<datasetId>"
      },
      "statistic_config": {
        "value": "value",
        "theme": "sunset",
        "prefix": "¥",
        "precision": 2,
        "groupSeparator": ",",
        "formatter": "(value) => value >= 10000 ? `${(value/10000).toFixed(1)}万` : `${value}`"
      }
    }
  }
}
```
