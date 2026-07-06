# 接口约定：角色与权限管理

**关联规格**：[spec.md](../spec.md)（用户故事 2）

所有接口均需要具备相应管理权限的已登录用户身份。

## GET /api/gonex/roles

查询角色列表，含每个角色关联的权限项摘要。

## POST /api/gonex/roles

创建角色（FR-006）。

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| name | string | 是 | 角色名称，系统内唯一 |
| permission_ids | array\<id\> | 否 | 初始授予的权限项 |

**副作用**：MUST 写入 `role_create` 审计记录。

## PATCH /api/gonex/roles/{id}

更新角色名称或权限集合（FR-006）。

**约束**：`is_builtin_super_admin = true` 的角色 MUST NOT 被修改（见
data-model.md）。

**副作用**：MUST 写入 `role_permission_update` 审计记录，记录权限变更前后差异
摘要（对应 spec.md 用户故事 4 验收场景 2）。

**生效时机**：按 FR-016，已登录用户的当前会话在其凭证有效期内继续使用变更前的
权限，新权限在用户下次登录（重新签发凭证）时生效；接口本身无需实现权限推送。

## DELETE /api/gonex/roles/{id}

删除角色（FR-014）。

**约束**：`is_builtin_super_admin = true` 的角色 MUST NOT 被删除。

**副作用**：MUST 级联解除所有用户与该角色的绑定关系，并写入 `role_delete`
审计记录（记录受影响的用户列表摘要）。

## GET /api/gonex/permissions

查询系统中已定义的权限项列表（用于角色配置界面选择）。

## POST /api/gonex/permissions

创建权限项。

**请求体**：`path_pattern`（string，必填）、`method`（string，必填）、
`description`（string，可选）。
