# 接口约定：认证（登录/登出）

**关联规格**：[spec.md](../spec.md)（用户故事 1）

## POST /api/gonex/auth/login

用户登录，成功后签发身份凭证（会话令牌）。

**认证要求**：无（公开接口）。

**请求体**：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| login_id | string | 是 | 用户名或邮箱 |
| password | string | 是 | 明文密码（仅通过 HTTPS 传输） |

**成功响应（200）**：

| 字段 | 类型 | 说明 |
|---|---|---|
| token | string | 会话令牌，后续请求通过 `Authorization: Bearer <token>` 携带 |
| expires_at | string（ISO 8601） | 令牌过期时间 |

**失败响应**：

| 状态码 | 场景 |
|---|---|
| 401 | 用户名或密码错误（不区分"用户不存在"与"密码错误"，见 spec.md 验收场景 2） |
| 403 | 账号被禁用，或账号处于锁定期内（见 spec.md 验收场景 3、4） |
| 429 | 触发登录失败限流（可选，视实现细节，与 403 锁定语义可能合并） |

**副作用**：
- 成功登录 MUST 写入一条 `login_success` 审计记录，并重置
  `failed_login_attempts`。
- 失败登录 MUST 写入一条 `login_failed` 审计记录，并递增
  `failed_login_attempts`；达到阈值时设置 `locked_until`。

## POST /api/gonex/auth/logout

使当前请求所携带的会话令牌失效。

**认证要求**：需要有效的 `Authorization: Bearer <token>`。

**成功响应（204）**：无响应体；对应会话的 `revoked_at` 被置为当前时间，同一
账号的其他并发会话不受影响（FR-003 澄清项）。

**副作用**：MUST 写入一条 `logout` 审计记录。
