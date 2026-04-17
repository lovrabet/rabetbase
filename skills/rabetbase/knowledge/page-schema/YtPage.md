---
name: YtPage
description: YtPage 页面布局组件完整配置
tags: [page, layout, page-schema]
---

# YtPage

页面容器组件，支持磁贴布局和流式布局两种模式。

---

## 布局类型

| 类型 | 判断条件 | 特点 | 场景 |
|------|----------|------|------|
| 磁贴布局 | 包含 `YtRglContainer` | 网格定位、支持并排 | 仪表盘、数据看板 |
| 流式布局 | 无 `YtRglContainer` | 上下排列 | 表单页、详情页 |

---

## 磁贴布局（YtRglContainer）

### 结构

```json
{
  "componentName": "YtPage",
  "children": [{
    "componentName": "YtRglContainer",
    "props": {
      "RGLlayout": [
        {"i": "node_filter", "w": 12, "h": 10, "x": 0, "y": 0, "autoHeight": true},
        {"i": "node_table", "w": 12, "h": 30, "x": 0, "y": 10, "autoHeight": false}
      ]
    },
    "children": [...]
  }]
}
```

### RGLlayout 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| i | string | 组件 ID（对应 children 中的 id） |
| w | number | 宽度（总列数 12） |
| h | number | 高度（单位行数） |
| x | number | X 起始位置（0-11） |
| y | number | Y 起始位置 |
| autoHeight | boolean | 自动高度（true 时 h 无效） |

### 布局规则

| 规则 | 说明 |
|------|------|
| 每行总宽度 | 必须为 12 |
| Filter/Table | 必须独占一行（w: 12） |
| 图表并排 | 两个图表各占 w: 6 |

---

## 流式布局

### 结构

```json
{
  "componentName": "YtPage",
  "children": [
    {"componentName": "LrSmartFilter"},
    {"componentName": "LrSmartTable"}
  ]
}
```

### 布局规则

- Filter 必须在 Table 前
- 组件按 children 数组顺序排列

---

## 常见布局场景

### 并排图表

```json
{
  "RGLlayout": [
    {"i": "chart1", "w": 6, "h": 20, "x": 0, "y": 0},
    {"i": "chart2", "w": 6, "h": 20, "x": 6, "y": 0}
  ]
}
```

### 统计卡 + 图表

```json
{
  "RGLlayout": [
    {"i": "stat1", "w": 3, "h": 10, "x": 0, "y": 0},
    {"i": "stat2", "w": 3, "h": 10, "x": 3, "y": 0},
    {"i": "stat3", "w": 3, "h": 10, "x": 6, "y": 0},
    {"i": "stat4", "w": 3, "h": 10, "x": 9, "y": 0},
    {"i": "chart1", "w": 12, "h": 20, "x": 0, "y": 10}
  ]
}
```

### Filter + Table

```json
{
  "RGLlayout": [
    {"i": "filter", "w": 12, "h": 10, "x": 0, "y": 0, "autoHeight": true},
    {"i": "table", "w": 12, "h": 30, "x": 0, "y": 10}
  ]
}
```

---

## 禁止事项

| 禁止 | 说明 |
|------|------|
| 添加/删除组件 | 只能调整布局 |
| 更改组件类型 | 只能调整位置 |
| 每行宽度不为 12 | 磁贴布局必须填满 12 列 |

---

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 布局空白 | 宽度不为 12 | 确保每行 w 总和为 12 |
| 高度不一致 | autoHeight 设置错误 | 统一设置 autoHeight |
| 组件重叠 | x/y 坐标冲突 | 检查坐标是否正确 |
| Filter 不生效 | 在 Table 后 | 将 Filter 移到 Table 前 |

---

## 完整示例

```json
{
  "componentName": "YtPage",
  "children": [{
    "componentName": "YtRglContainer",
    "props": {
      "RGLlayout": [
        {"i": "node_stat1", "w": 3, "h": 8, "x": 0, "y": 0},
        {"i": "node_stat2", "w": 3, "h": 8, "x": 3, "y": 0},
        {"i": "node_stat3", "w": 3, "h": 8, "x": 6, "y": 0},
        {"i": "node_stat4", "w": 3, "h": 8, "x": 9, "y": 0},
        {"i": "node_filter", "w": 12, "h": 10, "x": 0, "y": 8, "autoHeight": true},
        {"i": "node_table", "w": 12, "h": 30, "x": 0, "y": 18}
      ]
    },
    "children": [
      {"id": "node_stat1", "componentName": "LrSmartStatistic"},
      {"id": "node_stat2", "componentName": "LrSmartStatistic"},
      {"id": "node_stat3", "componentName": "LrSmartStatistic"},
      {"id": "node_stat4", "componentName": "LrSmartStatistic"},
      {"id": "node_filter", "componentName": "LrSmartFilter"},
      {"id": "node_table", "componentName": "LrSmartTable"}
    ]
  }]
}
```
