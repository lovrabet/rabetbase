# 用户组与权限（role / permit）工作流

前置知识：已完成 `rabetbase auth` 登录，当前上下文已有 `appcode`。所有写操作默认输出 `compress`，先 `--dry-run` 再 `--yes`。

## 适用范围

本工作流覆盖「谁属于哪个用户组」和「某个用户组能看什么、能操作什么」两类问题：

- 建组 / 改组 / 删组（`role create/update/delete`）
- 把员工加入或移出用户组（`role user-add/user-remove`）
- 收紧或放开某用户组的菜单可见（`permit role-menus-set`）
- 收紧或放开某用户组的数据接口访问（`permit role-apis-set`）
- 配置单个页面的菜单可见 / 数据操作 / 行级可见（`permit page-set`）

不适用于：

- 平台运营态的用户账号增删、租户成员开通（那是平台账号体系，不是应用内用户组）
- 运行态「登录后实际能读到哪些行」的验证 —— 交给 `lovrabet data filter` 在运行态确认

## 两个必须先分清的层面

权限配置常被混为一谈，动手前先判断问题落在哪一层：

| 层面 | 回答的问题 | 事实源 / 命令 |
|------|-----------|--------------|
| **成员归属** | 某个人在不在某个用户组里 | `role user-*`（`update-user-list` 全量回写） |
| **能力授权** | 某个用户组能看哪些菜单 / 访问哪些接口 / 在某页能做什么 | `permit *`（菜单矩阵 / API 矩阵 / 单页槽位） |

「小张不能看财务」通常**不是**把小张移出某个组，而是收紧他所在用户组的**能力授权**。

## 角色（用户组）类型

`ADMIN` / `DEV` / `USER` 为内置，`CUSTOM` 为用户创建。守卫规则：

- `role update` / `role delete`：**仅 `CUSTOM`**；对 `DEV` / `ADMIN` / `USER` 会被 validation 拒绝。
- `permit role-menus-set` / `role-apis-set`：**DEV / ADMIN 的权限矩阵不可改**，会被拒绝（内置角色按设计保持全量访问）。
- `role user-add` / `user-remove`：成员增删对**所有角色**可用（包括把人加进 DEV）。

要「限制某类人」时，正确姿势是新建或选用一个 `CUSTOM` 组，配它的能力，再把人放进去，而不是去改内置角色。

## 决策树：我要解决什么问题 → 用什么命令

```text
需要改「谁在组里」？
├─ 加人 / 移人 ──────────────→ role user-resolve（拿 userId）→ role user-add / user-remove
├─ 新建一个用户组 ───────────→ role create
├─ 改组名 / 备注 ────────────→ role update（仅 CUSTOM）
└─ 删除用户组 ──────────────→ role delete（仅 CUSTOM，high-risk）

需要改「某个组能看/能做什么」？
├─ 整个组看不到某些菜单 ─────→ permit role-menus  →  role-menus-set --revoke/--grant --menus
├─ 整个组访问不了某些数据接口 → permit role-apis-set --revoke --datasets
└─ 针对某一个页面精细配置 ───→ permit page-get  →  page-set
     ├─ 该页对哪些角色可见 ──→ page-set --menu-roles
     ├─ 该页的增/改/删/详情/导出 → page-set --create-roles/--update-roles/...
     └─ 行级「只看自己」 ─────→ page-set --row-roles SELF
```

### `role-menus-set` 与 `page-set --menu-roles` 的区别（易混）

两者都影响「菜单是否可见」，但入口和粒度不同，别用错：

- **`permit role-menus-set`**：以**角色**为主体，一次性设置该角色在整个 app 的菜单权限矩阵（`roleType: PAGE`），按 `resourceCode` 翻转。适合「销售组不看财务菜单」这类**按组批量**收紧。
- **`permit page-set --menu-roles`**：以**单个菜单页面**为主体，设置「这一页对哪些角色可见」，同时还能在同一页配置数据操作和行级权限。适合「就这一个页面的权限编排」。

判断口径：问题描述里主语是「某个组」→ 走 `role-menus-set`；主语是「某个页面」→ 走 `page-set`。

## 通用铁律

1. **先读后写**：改成员先 `role user-resolve` + `role list`；改能力先 `permit page-get` / `role-menus` 拿到真实 `resourceCode` / `menuId` / `datasetCode`，不要臆造标识。
2. **dry-run 必做**：所有写命令先 `--dry-run` 看 `data.before` / `data.after`，确认无误再复用同参数加 `--yes`。
3. **read-merge-write 是 CLI 内部行为**：`update-user-list` / `page/save` / `role/update` 服务端都是**全量回写**；CLI 已内部读取全量、只改目标片段、其余原样带回。你只需传要改的部分，但要理解一次写会覆盖整份对象。
4. **防漂移**：`user-add/user-remove` 可加 `--expect-user-count <n>`，当前成员数不符即中止。
5. **SELF 只用于行级**：`SELF` 仅允许出现在 `page-set --row-roles`（`dataPermit`）；打到菜单/操作槽位会被拒绝。`ALL` 表示所有角色。
6. **收紧不做自动 suggest**：`role-menus-set` / `role-apis-set` 必须显式 `--menus` / `--datasets`，避免误伤；先用 `dataset list` / `menu list` 定位目标。
7. **运行态验证要交接**：`page-set --row-roles SELF`、`role-apis-set --revoke` 只写权限配置。要验证「登录后是否只看到自己的行 / 是否读不到财务数据」，交给 `lovrabet data filter` 在运行态确认，不要把它伪装成 `rabetbase` 已验证。
8. **读内层协议，不读顶层 envelope**：写命令的稳定字段在 `data.*`（`operation` / `selector` / `before` / `after` / `dryRun` / `backend` / `warnings`），不含合成 `protocol` key。

## 常见剧本

### A. 把员工加入用户组（如：把小张加入开发者）

```text
role user-resolve --name 小张   → 拿 userId（重名报错，用 --user <id> 消歧）
role list --type DEV            → 拿目标角色 id
role user-add --role <id> --user <userId> --dry-run → --yes
```

### B. 新建一个用户组（如：创建销售组）

```text
role create --name 销售组 --dry-run
role create --name 销售组 --remark "华东销售" --yes   → 从 data.selector.roleId 取回填的新 id
```

> 服务端 `role/create` 返回创建后的完整角色对象，CLI 直接将 id 填入 `data.selector.roleId`，不再按名称回查。

### C. 让某组看不到财务（菜单 + API 双侧收紧）

```text
dataset list                                  → 定位财务域 datasetCode
menu list                                     → 定位财务菜单 resourceCode
permit role-menus-set --role <salesId> --revoke --menus <财务菜单codes> --dry-run → --yes
permit role-apis-set  --role <salesId> --revoke --datasets <财务datasetCodes> --dry-run → --yes
```

菜单不可见 + 数据接口不可访问，两侧都要收紧，否则可能菜单藏了但接口还能拉数据。

### D. 某个列表页只看自己的行（如：请假列表只看自己）

```text
menu list                                     → 定位该页 menuId
permit page-get --menu-id <id>                → 确认 dataPermit 槽位存在
permit page-set --menu-id <id> --row-roles SELF --dry-run → --yes
```

行级 `SELF` 打在 `dataPermit`，**不要** revoke 菜单本身。

## 参考

- [role list / detail / create / update / delete](../references/rabetbase-role-list.md)
- [role user-resolve](../references/rabetbase-role-user-resolve.md)
- [role user-add / user-remove](../references/rabetbase-role-user-add.md)
- [permit page-get / page-set](../references/rabetbase-permit-page-set.md)
- [permit role-menus / role-menus-set](../references/rabetbase-permit-role-menus-set.md)
- [permit role-apis-set](../references/rabetbase-permit-role-apis-set.md)
- [SKILL.md](../SKILL.md)
