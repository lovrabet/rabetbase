# 菜单异常审计与人工修复

用于重复 `path`、根级菜单重名、疑似菜单同步重复创建等场景。推荐方式是只读审计并生成平台人工处理清单。

## 执行原则

- 菜单事实只从 `rabetbase menu list --verbose` 获取。
- 不绕过 `rabetbase menu list --verbose` 读取菜单配置。
- Agent 只生成异常清单、平台链接和复查命令，不代替用户删除。
- 用户删除前必须在平台页面核对 `id / label / path / parentId`，不能只凭名称或链接判断。

## 平台入口

先按当前 `rabetbase` 环境计算 `<appBaseUrl>`：

```text
production https://app.lovrabet.com/app/<appCode>
daily      https://daily.lovrabet.com/web-app/app/<appCode>
```

总入口：

```text
<appBaseUrl>/pages
```

精细入口：

```text
<appBaseUrl>/pages/<menuCode>
```

当菜单 `path` 可作为页面标识使用时，可生成：

```text
<appBaseUrl>/pages/<encodeURIComponent(menu.path)>
```

重复 `path` 场景下，多条异常菜单的精细入口可能相同；这只能帮助用户快速进入页面配置，不能替代按 `id / parentId / label` 做删除确认。

## 审计步骤

1. 读取菜单事实：

```bash
rabetbase menu list --appcode <appCode> --verbose --format json
```

`menu list` 返回扁平数组；`childrenCount` 需要用 `menus.filter(item => item.parentId === menu.id).length` 反推。

2. 按规则分类异常：

必须修复：
- 非空 `path` 重复。
- `parentId` 指向不存在的菜单。
- 菜单树出现循环引用。
- 同一父级下重复关键业务菜单。

需要人工确认：
- 根级 `label` 重复。
- `visible=false` 但仍有 `path` 或 `resources`。
- 多个菜单共享同一 `pageId`。
- `procode` 菜单缺少 resources，但同组重复菜单存在 resources。

不默认视为异常：
- 空 `path` 的 folder。
- 不同父级下的同名 `label`。
- folder 没有 resources。

3. 输出人工处理清单：

```text
平台总入口: <appBaseUrl>/pages
精细入口: <appBaseUrl>/pages/<menuCode>

异常类型: duplicate_path
path: <path>
建议保留:
- id: <id>, label: <label>, parentId: <parentId>, resources: <count>
建议删除:
- id: <id>, label: <label>, parentId: <parentId>, childrenCount: <count>, resources: <count>
停止条件:
- 平台无法确认该 id
- children 不为空
- pageId 被保留菜单共享
- 平台显示与清单不一致
```

4. 用户完成平台手动删除后，再读取菜单事实并复查：

```bash
rabetbase menu list --appcode <appCode> --verbose --format json
```

复查至少确认重复 `path` 清零、目标保留菜单仍存在、resources 符合预期。
