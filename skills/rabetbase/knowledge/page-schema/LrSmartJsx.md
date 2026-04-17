---
name: LrSmartJsx
description: LrSmartJsx 自定义 JSX 组件完整配置
tags: [jsx, custom-component, page-schema]
---

# LrSmartJsx

自定义 JSX 组件。只在标准组件无法满足时使用。

---

## 禁止事项

| 禁止 | 替代方案/说明 |
|------|---------------|
| 直接修改 `_context.state` | 使用 `_context.setState()` 方法 |
| 在标准组件能满足需求时使用 | 优先使用 LrSmartTable、LrSmartFilter 等标准组件 |

---

## 机制

`render` 只写 `source`，不写编译后的 `value`。`source` 会被编译成 React 组件，组件收到的入参是 `props`，页面上下文在 `props.env`。

推荐固定写法：

```jsx
function render(props) {
  const _context = props.env;
  // 业务逻辑
  return <div />;
}
```

实际写入页面 Schema JSON 配置时，`source` 仍然是字符串，使用 `\n` 表示换行。

---

## 参数

### 配置参数

| 参数 | 必须 | 说明 |
|------|------|------|
| `props.render.type` | 是 | 固定为 `JSFunction` |
| `props.render.source` | 是 | JSX 源码 |
| `props._context` | 是 | 固定为 `{"type": "JSExpression", "value": "this"}` |
| `props._uniqueId` | 否 | 组件唯一标识 |

### 运行时参数

| 参数 | 说明 |
|------|------|
| `props.env.state` | 页面状态对象（只读） |
| `props.env.setState(state)` | 更新页面状态（浅合并） |
| `props.env.dataSourceMap` | 数据源映射表 |
| `props.env.dataSourceMap[id].load()` | 手动刷新数据源 |
| `props.env.reloadDataSource()` | 重新加载所有数据源 |
| `props.env.appHelper.yt.smartPageHistory` | 页面跳转 |
| `props.sdkClient` | SDK 客户端实例 |

### 可用 import

| import 来源 | 说明 |
|------------|------|
| `'react'` | React |
| `'antd'` | Ant Design |
| `'@ant-design/icons'` | 图标 |
| `'dayjs'` | 日期处理 |
| `'lodash'` | 工具函数 |

---

## 示例

机制：

1. 通过 `props.env` 读取页面上下文
2. 通过 `setState` 修改绑定在数据源参数上的 state
3. 需要立即刷新时调用 `dataSourceMap[id].load()`

```jsx
import React from 'react';
import { Button } from 'antd';

function render(props) {
  const _context = props.env;
  const stateKey = '<stateKey>';
  const dataSourceId = '<dataSourceId>';

  const handleClick = () => {
    _context.setState({ [stateKey]: <value> });
    setTimeout(() => {
      _context.dataSourceMap?.[dataSourceId]?.load();
    }, 0);
  };

  return <Button onClick={handleClick}>按钮文本</Button>;
}
```

替换点：

| 占位 | 含义 |
|------|------|
| `stateKey` | 要写入的页面 state key |
| `dataSourceId` | 需要刷新的数据源 id |
| `value` | 写入 state 的值 |

落盘说明：真正写入配置时，把这段源码放进 `render.source`，并保持 `source` 字符串 + `\n` 换行转义。

---

## 注意事项

| 要点 | 说明 |
|------|------|
| `_context` 必须配置 | 否则 `props.env` 缺失 |
| 只需提供 `source` | `value` 由系统自动生成 |
| source 中访问上下文 | 使用 `props.env` |
| `setState` 触发刷新的前提 | 数据源 params 必须绑定 `this.state.xxx` |
