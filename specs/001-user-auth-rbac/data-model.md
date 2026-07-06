# Phase 1 数据模型：多用户登录与角色权限管理

**关联计划**：[plan.md](./plan.md) | **关联规格**：[spec.md](./spec.md)

本文件描述的字段与约束与具体数据库方言无关；三种数据库（MySQL/PostgreSQL/
SQLite）的建表 DDL 由 `db/migrations/{postgres,mysql,sqlite}` 分别实现，字段
语义与约束 MUST 保持一致（见 research.md 决策 4）。

## 用户（User）

系统的登录主体。

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| id | 整数/UUID（方言原生自增或 UUID 主键） | 主键 | 用户唯一标识 |
| login_id | 字符串 | 全局唯一、非空、创建后不可修改 | 用户名或邮箱（FR-018） |
| password_hash | 字符串 | 非空 | bcrypt 哈希值，MUST NOT 存储明文/可逆密文（FR-002） |
| is_enabled | 布尔 | 默认 true | 启用/禁用状态（FR-005） |
| failed_login_attempts | 整数 | 默认 0 | 连续登录失败计数（FR-004） |
| locked_until | 可空时间戳 | — | 锁定到期时间；为空或早于当前时间表示未锁定（FR-004） |
| is_super_admin | 布尔 | 默认 false，且系统 MUST 保证至少一条记录为 true 且不可被禁用/删除 | 内置超级管理员标记（FR-012） |
| created_at / updated_at | 时间戳 | 非空 | 审计追溯用 |

**校验规则**：
- `login_id` 创建前 MUST 校验全局唯一（FR-018）。
- 禁止删除或禁用系统中唯一的 `is_super_admin = true` 记录（FR-012）。

**状态转换**：
- `is_enabled`：`启用 ⇄ 禁用`，由管理员操作触发，记录审计日志。
- 锁定状态：`未锁定 → 锁定`（`failed_login_attempts` 达到阈值时，写入
  `locked_until`）；`锁定 → 未锁定`（当前时间超过 `locked_until` 时，登录逻辑
  MUST 视为未锁定并允许重新计数，见 FR-004 与澄清记录）。

## 角色（Role）

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| id | 整数/UUID | 主键 | 角色唯一标识 |
| name | 字符串 | 同一系统内唯一、非空 | 角色名称 |
| is_builtin_super_admin | 布尔 | 默认 false，仅内置角色为 true | 标记不可删除的超级管理员角色（FR-012） |
| created_at / updated_at | 时间戳 | 非空 | — |

**校验规则**：`is_builtin_super_admin = true` 的角色 MUST NOT 被删除或修改其
权限集合为空。

## 权限项（Permission）

描述"对某路径/资源可执行某操作"的最小授权单元。

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| id | 整数/UUID | 主键 | 权限项唯一标识 |
| path_pattern | 字符串 | 非空 | 请求路径前缀（如 `/api/orders`），支持前缀匹配 |
| method | 字符串 | 非空，取值如 `GET`/`POST`/`*` | HTTP 方法，`*` 表示任意方法 |
| description | 字符串 | 可空 | 权限用途说明，便于管理员理解 |

**校验规则**：同一 `(path_pattern, method)` 组合 SHOULD 唯一，避免重复定义造成
管理混乱（非强制唯一约束，但界面/接口 SHOULD 提示重复）。

## 角色-权限关联（RolePermission）

角色与权限项的多对多关系表。

| 字段 | 类型 | 约束 |
|---|---|---|
| role_id | 外键 → Role.id | 与 permission_id 联合唯一 |
| permission_id | 外键 → Permission.id | 与 role_id 联合唯一 |

## 用户-角色关联（UserRole）

用户与角色的多对多关系表（FR-007）。

| 字段 | 类型 | 约束 |
|---|---|---|
| user_id | 外键 → User.id | 与 role_id 联合唯一 |
| role_id | 外键 → Role.id | 与 user_id 联合唯一 |

**校验规则**：角色被删除时，MUST 级联删除相关 `UserRole` 记录，并写入审计日志
（FR-014）。

**权限合并规则**：用户的最终生效权限 = 其所有关联角色的权限集合的并集（任一
角色允许即允许，FR-013）。

## 身份凭证 / 会话（Session）

用户登录成功后获得的、具有有效期的凭证记录（见 research.md 决策 1）。

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| id | 整数/UUID | 主键 | 会话唯一标识 |
| user_id | 外键 → User.id | 非空 | 所属用户 |
| token_hash | 字符串 | 非空、唯一 | 客户端持有令牌的哈希值（服务端不存明文令牌） |
| issued_at | 时间戳 | 非空 | 签发时间 |
| expires_at | 时间戳 | 非空 | 过期时间（FR-003） |
| revoked_at | 可空时间戳 | — | 登出时写入，非空表示该会话已失效 |
| client_info | 字符串 | 可空 | 记录客户端标识（如 User-Agent/IP），便于审计与用户自查在线设备 |

**校验规则**：同一用户 MUST 允许存在多条同时有效（`revoked_at` 为空且未过期）
的会话记录（并发会话，FR-003 澄清项）。会话有效性校验 MUST NOT 依赖服务进程内存
状态，只依据数据库记录判断（重启后仍有效）。

**状态转换**：`有效 → 已吊销`（登出，或管理员强制下线）；`有效 → 已过期`
（`expires_at` 到期，只读判断，无需后台任务主动清理，允许惰性判断 + 定期
清理旧记录）。

## 转发规则（Forwarding Rule）

描述某个请求路径应转发到哪个下游业务服务，以及该路径所需的权限项。

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| id | 整数/UUID | 主键 | — |
| path_pattern | 字符串 | 非空 | 匹配的请求路径前缀 |
| method | 字符串 | 非空，可为 `*` | HTTP 方法 |
| upstream_url | 字符串 | 非空 | 下游业务服务地址 |
| required_permission_id | 外键 → Permission.id，可空 | — | 为空表示该路径任意已登录用户均可访问 |

**说明**：转发规则的具体加载方式（YAML 配置 vs 数据库管理界面）由实现阶段结合
宪法原则 III（YAML 配置为主）决定；本规格仅约束其数据结构与权限校验语义。

## 审计记录（Audit Log）

记录操作人、操作时间、操作内容的不可篡改日志条目（FR-010）。

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| id | 整数/UUID | 主键 | — |
| actor_user_id | 外键 → User.id，可空 | — | 操作人；登录失败等场景可能为空（未知用户） |
| action | 字符串 | 非空 | 操作类型，如 `login_success`、`login_failed`、`user_create`、`role_permission_update` 等（枚举值在实现阶段定义） |
| target | 字符串 | 可空 | 操作对象标识（如被修改的用户 ID、角色 ID） |
| detail | 字符串（建议 JSON 文本） | 可空 | 操作内容摘要，如权限变更前后差异 |
| occurred_at | 时间戳 | 非空 | 操作发生时间 |

**校验规则**：审计记录一经写入 MUST NOT 被修改或删除（应用层不提供更新/删除
接口）。

## 实体关系概览

```text
User ──< UserRole >── Role ──< RolePermission >── Permission
User ──< Session
Permission ──< ForwardingRule（required_permission_id，可空）
User ──< AuditLog（actor_user_id，可空）
```
