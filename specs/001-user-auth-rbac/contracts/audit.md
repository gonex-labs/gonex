# 接口约定：审计日志查询

**关联规格**：[spec.md](../spec.md)（用户故事 4）

## GET /api/gonex/audit-logs

分页查询审计日志（FR-011），需要具备相应管理权限的已登录用户身份。

**查询参数（均可选）**：`actor_user_id`、`action`、`from`（起始时间）、
`to`（结束时间）。

**响应字段（每项）**：`id`、`actor_user_id`、`action`、`target`、`detail`、
`occurred_at`。

**约束**：本接口 MUST 为只读；系统 MUST NOT 提供修改或删除审计记录的接口
（data-model.md 校验规则）。
