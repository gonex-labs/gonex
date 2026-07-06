# 实施计划：多用户登录与角色权限管理

**分支**：`001-user-auth-rbac` | **日期**：2026-07-06 | **规格**：[spec.md](./spec.md)

**输入**：来自 `/specs/001-user-auth-rbac/spec.md` 的功能规格说明

**说明**：本文件由 `/speckit-plan` 命令生成。执行流程参见
`.specify/templates/plan-template.md`。

## 概述

gonex 需要为中后台系统提供内置的多用户登录、基于角色的权限管理（RBAC），以及在
请求转发到下游业务服务之前的权限前置校验能力。技术方案：使用 `gin` 构建 HTTP
层与中间件链（认证 → 载入用户角色/权限 → 权限校验 → 转发）；密码使用 `bcrypt`
加盐哈希；身份凭证采用数据库持久化的不透明会话令牌（而非纯无状态 JWT），以满足
"服务重启后凭证仍然有效""支持登出使会话失效""支持并发多会话"等已澄清需求；
角色-权限-用户关系通过 `sqlc` 生成的类型安全查询访问，`goose` 管理迁移，且必须为
MySQL、PostgreSQL、SQLite 三种数据库分别维护迁移与查询实现；命令行（初始化超级
管理员、启动服务等）基于 `cobra`；配置文件使用 YAML（`go.yaml.in/yaml/v4`）。

## 技术上下文

**语言/版本**：Go 1.26（与仓库 `go.mod` 声明一致）

**主要依赖**：
- `github.com/gin-gonic/gin`（HTTP 路由与中间件）
- `github.com/sqlc-dev/sqlc`（开发期代码生成，非运行时依赖）
- `github.com/pressly/goose/v3`（数据库迁移）
- `go.yaml.in/yaml/v4`（YAML 配置解析）
- `github.com/spf13/cobra`（命令行接口）
- `golang.org/x/crypto/bcrypt`（密码加盐哈希）
- 数据库驱动：`github.com/go-sql-driver/mysql`、`github.com/jackc/pgx/v5/stdlib`、
  `modernc.org/sqlite`（纯 Go 实现，不依赖 CGO，见 research.md 决策 3）

**存储**：MySQL、PostgreSQL、SQLite（三者 MUST 同时支持，行为一致，见宪法原则 V）

**测试**：标准库 `testing` + `net/http/httptest`（HTTP handler 测试）；`go test`
矩阵化运行以覆盖三种数据库方言的集成测试

**目标平台**：Linux / macOS / Windows 服务器环境，编译为单一静态可执行文件部署

**项目类型**：单体后端服务（无前端代码，符合宪法原则 III）

**性能目标**：登录请求 3 秒内完成（SC-001）；100 个并发登录会话下无明显响应延迟
（SC-004）

**约束**：
- 密码 MUST 加盐哈希存储（bcrypt），不得明文或可逆存储
- 身份凭证的有效性 MUST NOT 依赖服务进程持续运行（见 spec.md 澄清项）
- 权限校验 MUST 在请求转发到下游业务服务之前完成
- 数据库相关代码 MUST 为三种数据库分别维护且行为一致

**规模/范围**：初期支持至少 1000 个用户账号、100 个并发会话；权限粒度为
"请求路径前缀 + HTTP 方法"

## 宪法检查

*门禁：必须在 Phase 0 研究前通过；Phase 1 设计后重新检查。*

| 原则 | 检查结果 | 说明 |
|---|---|---|
| I. Go 语言与社区规范优先 | 通过 | 采用标准 Go 项目布局（`cmd/`、`internal/`），CI 中运行 `gofmt`/`go vet`/`golangci-lint` |
| II. 中文文档与交互 | 通过 | 本计划及 research/data-model/quickstart 等文档均使用中文；代码标识符使用英文 |
| III. 单一可执行文件与前端外部化 | 通过 | 本功能不涉及任何前端代码；SQLite 驱动选用纯 Go 实现以保持单一静态二进制、免 CGO 交叉编译能力 |
| IV. 中后台基础能力与业务转发边界 | 通过 | 功能范围严格限定在登录、角色权限、转发前置校验，不实现任何具体业务逻辑 |
| V. 固定技术栈 | 通过 | 使用 gin、sqlc+goose、MySQL/PostgreSQL/SQLite 三库、`go.yaml.in/yaml/v4`、cobra，未引入替代技术栈 |
| VI. 多用户权限设计 | 通过 | bcrypt 加盐哈希、RBAC 多对多模型、转发前权限校验、登录与敏感操作审计留痕 |

未发现违反宪法原则的设计，**Complexity Tracking 章节无需填写**。

## 项目结构

### 文档（本功能）

```text
specs/001-user-auth-rbac/
├── plan.md              # 本文件（/speckit-plan 命令输出）
├── research.md          # Phase 0 输出（/speckit-plan 命令）
├── data-model.md        # Phase 1 输出（/speckit-plan 命令）
├── quickstart.md        # Phase 1 输出（/speckit-plan 命令）
├── contracts/           # Phase 1 输出（/speckit-plan 命令）
└── tasks.md             # Phase 2 输出（/speckit-tasks 命令，本命令不生成）
```

### 源代码（仓库根目录）

```text
cmd/
└── gonex/
    └── main.go                 # 程序入口，装配 cobra 根命令

internal/
├── config/                     # YAML 配置结构体与加载（go.yaml.in/yaml/v4）
├── httpserver/
│   ├── router.go               # gin 路由装配
│   ├── middleware/
│   │   ├── auth.go             # 身份凭证校验中间件
│   │   ├── rbac.go             # 权限前置校验中间件
│   │   └── audit.go            # 审计日志记录中间件
│   └── handler/
│       ├── auth_handler.go     # 登录/登出接口
│       ├── user_handler.go     # 用户管理接口
│       ├── role_handler.go     # 角色/权限管理接口
│       ├── audit_handler.go    # 审计日志查询接口
│       └── gateway_handler.go  # 转发规则匹配与反向代理
├── auth/
│   ├── password.go             # bcrypt 密码哈希与校验
│   ├── session.go              # 会话令牌签发/校验/吊销
│   └── lockout.go              # 登录失败计数与自动解锁
├── rbac/
│   ├── service.go               # 角色/权限领域逻辑、权限并集合并
│   └── model.go
├── forward/
│   └── proxy.go                 # 基于转发规则的反向代理实现
├── audit/
│   └── service.go                # 审计日志写入与查询
├── store/                        # sqlc 生成代码的仓储封装（按方言隔离）
│   ├── postgres/
│   ├── mysql/
│   └── sqlite/
└── cli/
    ├── root.go                   # cobra 根命令（serve 等）
    └── admin_init.go             # 初始化超级管理员账号的命令

db/
├── migrations/
│   ├── postgres/                 # goose 迁移脚本（PostgreSQL 方言）
│   ├── mysql/                    # goose 迁移脚本（MySQL 方言）
│   └── sqlite/                   # goose 迁移脚本（SQLite 方言）
└── queries/
    ├── postgres/                  # sqlc 查询定义（PostgreSQL 方言）
    ├── mysql/                     # sqlc 查询定义（MySQL 方言）
    └── sqlite/                    # sqlc 查询定义（SQLite 方言）

tests/
├── contract/                      # 针对 contracts/ 中接口约定的契约测试
├── integration/                   # 端到端集成测试（跨三种数据库矩阵运行）
└── unit/                          # 单元测试（auth、rbac、forward 等领域逻辑）
```

**结构决策**：采用单体后端项目结构（模板 Option 1 的变体），因为本仓库不含前端
代码（宪法原则 III）。按领域拆分 `internal/` 子包（`auth`、`rbac`、`forward`、
`audit`），数据库访问通过 `store/` 下按方言隔离的 `sqlc` 生成代码封装，`db/` 下
迁移与查询按 `postgres`/`mysql`/`sqlite` 三个方言目录分别维护，以满足宪法原则 V
"三种数据库行为必须一致"的要求，同时避免不同方言的 SQL 语法混杂在同一文件中。

## 复杂度追踪

> 本功能未发现需要论证的宪法违规项，故本表为空。

| 违规项 | 为何需要 | 被拒绝的更简单替代方案 |
|---|---|---|
| （无） | — | — |
