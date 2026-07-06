# 接口约定：用户管理

**关联规格**：[spec.md](../spec.md)（用户故事 2）

所有接口均需要具备相应管理权限的已登录用户身份（`Authorization: Bearer <token>`
+ 权限校验中间件放行）。

## GET /api/gonex/users

分页查询用户列表。

**响应字段（每项）**：`id`、`login_id`、`is_enabled`、`is_super_admin`、
`locked_until`、`roles`（角色名称数组）、`created_at`。

## POST /api/gonex/users

创建新用户（FR-005、FR-015：无自助注册，仅管理员创建）。

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| login_id | string | 是 | 全局唯一，创建后不可修改（FR-018） |
| password | string | 是 | 初始密码，服务端 MUST 立即 bcrypt 哈希后存储 |
| role_ids | array\<id\> | 否 | 创建时可直接分配角色 |

**失败响应**：409（`login_id` 已存在）。

**副作用**：MUST 写入 `user_create` 审计记录。

## PATCH /api/gonex/users/{id}

编辑用户（启用/禁用状态、密码重置、角色分配的整体更新等）。

**请求体（均为可选字段，按需传递）**：

| 字段 | 类型 | 说明 |
|---|---|---|
| is_enabled | bool | 启用/禁用（FR-005） |
| password | string | 管理员重置密码（FR-017：不提供用户自助找回） |
| role_ids | array\<id\> | 覆盖式更新用户的角色集合 |

**约束**：MUST NOT 允许修改 `login_id`（FR-018）；MUST NOT 允许将系统中唯一的
`is_super_admin = true` 用户设置为禁用（FR-012）。

**失败响应**：403（试图禁用唯一超级管理员）、404（用户不存在）。

**副作用**：MUST 写入对应的 `user_update` / `user_role_assign` 审计记录，记录
变更前后差异摘要。

## DELETE /api/gonex/users/{id}

删除用户。

**约束**：MUST NOT 允许删除唯一的超级管理员账号（FR-012）。

**副作用**：MUST 写入 `user_delete` 审计记录；该用户所有有效会话 MUST 被吊销。
