---

description: "多用户登录与角色权限管理 功能实现任务清单"
---

# 任务清单：多用户登录与角色权限管理

**输入**：来自 `/specs/001-user-auth-rbac/` 的设计文档

**前置条件**：plan.md（必需）、spec.md（必需，含用户故事优先级）、
research.md、data-model.md、contracts/、quickstart.md（均已生成）

**测试**：本功能包含测试任务。依据宪法"开发工作流与质量门禁"章节的强制要求——
"涉及权限、认证、请求转发核心逻辑的变更 MUST 附带单元测试或集成测试覆盖关键
路径"，用户故事 1/2/3（登录、用户角色管理、权限转发）为安全关键路径，测试任务
为**必须**而非可选；用户故事 4（审计）测试任务同样保留以验证审计留痕的完整性。

**组织方式**：任务按用户故事分组，以支持每个故事的独立实现与独立测试。

## 格式：`[ID] [P?] [Story] 描述`

- **[P]**：可并行执行（不同文件、无依赖关系）
- **[Story]**：任务所属的用户故事（US1/US2/US3/US4）
- 描述中包含准确的文件路径

## 路径约定

单体后端项目（无前端），路径以仓库根目录为基准：`cmd/`、`internal/`、`db/`、
`tests/`，具体结构见 [plan.md](./plan.md) 的"项目结构"章节。

---

## Phase 1：Setup（共享基础设施）

**目的**：项目初始化与基础结构搭建

- [ ] T001 按 plan.md 的项目结构创建目录骨架：`cmd/gonex/`、`internal/{config,httpserver,auth,rbac,forward,audit,store,cli}/`、`db/{migrations,queries}/{postgres,mysql,sqlite}/`、`tests/{contract,integration,unit}/`
- [ ] T002 在 `go.mod` 中引入核心依赖：`github.com/gin-gonic/gin`、`github.com/pressly/goose/v3`、`go.yaml.in/yaml/v4`、`github.com/spf13/cobra`、`golang.org/x/crypto/bcrypt`、`github.com/go-sql-driver/mysql`、`github.com/jackc/pgx/v5`、`modernc.org/sqlite`
- [ ] T003 [P] 添加 `.golangci.yml` 配置及 `Makefile`（或等价脚本）用于运行 `gofmt`/`go vet`/`golangci-lint`（宪法原则 I）
- [ ] T004 [P] 在 `internal/config/config.go` 中定义 YAML 配置结构体（数据库连接、监听地址、转发规则、锁定阈值/时长、凭证有效期等）及加载/校验函数
- [ ] T005 [P] 编写 `sqlc.yaml`，为 postgres/mysql/sqlite 三个方言分别配置 `queries` 与 `schema` 路径，生成目标分别指向 `internal/store/{postgres,mysql,sqlite}`
- [ ] T006 [P] 为三种方言在 `db/migrations/{postgres,mysql,sqlite}/` 下各创建 goose 迁移目录与首个占位迁移文件

**检查点**：项目骨架、依赖、代码生成配置就绪

---

## Phase 2：Foundational（阻塞性前置任务）

**目的**：所有用户故事开始前必须完成的核心基础设施

**⚠️ 关键**：本阶段完成前不得开始任何用户故事的开发

- [ ] T007 [P] 编写 PostgreSQL 方言的初始建表迁移（users/roles/permissions/user_roles/role_permissions/sessions/forwarding_rules/audit_logs，字段见 [data-model.md](./data-model.md)）于 `db/migrations/postgres/0001_init.sql`
- [ ] T008 [P] 编写 MySQL 方言的等价初始建表迁移于 `db/migrations/mysql/0001_init.sql`
- [ ] T009 [P] 编写 SQLite 方言的等价初始建表迁移于 `db/migrations/sqlite/0001_init.sql`
- [ ] T010 [P] 编写 PostgreSQL 方言的核心 CRUD 查询（用户、角色、权限、会话、审计日志的增删改查）于 `db/queries/postgres/*.sql`
- [ ] T011 [P] 编写 MySQL 方言的等价查询于 `db/queries/mysql/*.sql`
- [ ] T012 [P] 编写 SQLite 方言的等价查询于 `db/queries/sqlite/*.sql`
- [ ] T013 运行 `sqlc generate` 生成三个方言的类型安全查询代码，确认输出到 `internal/store/{postgres,mysql,sqlite}`（依赖 T005、T007-T012）
- [ ] T014 在 `internal/store/store.go` 中定义与方言无关的仓储接口（UserStore/RoleStore/PermissionStore/SessionStore/AuditStore），屏蔽三种数据库差异（依赖 T013）
- [ ] T015 在 `internal/store/store.go` 中实现按配置的数据库类型选择具体方言实现的工厂函数（依赖 T014）
- [ ] T016 [P] 在 `internal/httpserver/router.go` 中搭建 gin 路由骨架与中间件注册顺序（认证 → 权限校验 → 审计 → 业务处理器）
- [ ] T017 [P] 在 `internal/cli/root.go` 中实现 cobra 根命令与 `serve` 子命令，装配配置加载、数据库连接、路由启动
- [ ] T018 在 `internal/audit/service.go` 中实现审计日志写入服务（供后续所有故事复用），依赖 T014
- [ ] T019 [P] 在 `tests/integration/testutil/` 下搭建跨三种数据库运行集成测试的测试夹具（如临时 SQLite 文件 + MySQL/PostgreSQL 测试容器或已有测试实例的连接参数化）

**检查点**：数据库、仓储层、路由骨架、CLI 骨架、审计基础设施就绪，可开始用户故事开发

---

## Phase 3：用户故事 1 - 用户登录（优先级：P1）🎯 MVP

**目标**：用户可通过用户名/邮箱与密码登录，获得身份凭证；错误密码、账号禁用、
连续失败锁定均有正确响应（对应 [spec.md](./spec.md) 用户故事 1）

**独立测试标准**：在不依赖角色权限、转发功能的情况下，仅通过登录接口即可验证
正确/错误凭证、禁用账号、锁定与自动解锁四类场景均按预期响应

### 用户故事 1 的测试

> **注意：先编写以下测试，确认测试失败后再开始实现**

- [ ] T020 [P] [US1] 依据 [contracts/auth.md](./contracts/auth.md) 编写登录/登出接口的契约测试于 `tests/contract/auth_test.go`
- [ ] T021 [P] [US1] 编写登录集成测试（正确凭证、错误密码、禁用账号、连续失败触发锁定、锁定到期自动解锁）于 `tests/integration/auth_login_test.go`（对应 quickstart.md 场景 1）

### 用户故事 1 的实现

- [ ] T022 [P] [US1] 在 `internal/auth/password.go` 中实现 bcrypt 密码哈希与校验函数
- [ ] T023 [P] [US1] 在 `internal/auth/session.go` 中实现会话令牌签发、校验、吊销（数据库持久化，见 research.md 决策 1）
- [ ] T024 [US1] 在 `internal/auth/lockout.go` 中实现登录失败计数与到期自动解锁逻辑（依赖 T014 仓储接口）
- [ ] T025 [US1] 在 `internal/httpserver/handler/auth_handler.go` 中实现 `POST /api/gonex/auth/login` 处理器（依赖 T022-T024）
- [ ] T026 [US1] 在 `internal/httpserver/handler/auth_handler.go` 中实现 `POST /api/gonex/auth/logout` 处理器（依赖 T023）
- [ ] T027 [US1] 在 `internal/httpserver/middleware/auth.go` 中实现认证中间件（解析 Bearer 令牌、校验会话、注入当前用户到上下文）
- [ ] T028 [US1] 在 `internal/httpserver/router.go` 中注册登录/登出路由及认证中间件（依赖 T025-T027）
- [ ] T029 [US1] 在登录成功/失败/登出流程中调用审计服务写入 `login_success`/`login_failed`/`logout` 记录（依赖 T018、T025、T026）
- [ ] T030 [US1] 在 `internal/cli/admin_init.go` 中实现初始化超级管理员账号的 CLI 命令（依赖 T022）

**检查点**：用户故事 1 完整可用，可独立测试（MVP 交付点）

---

## Phase 4：用户故事 2 - 用户与角色管理（优先级：P2）

**目标**：管理员可创建/编辑/启用禁用用户，创建角色并配置权限，将角色分配给
用户（对应 [spec.md](./spec.md) 用户故事 2）

**独立测试标准**：使用管理员账号创建用户、创建角色并分配权限、将角色赋给该
用户，确认列表与权限配置正确保存，且被禁用用户无法登录（复用用户故事 1 能力）

### 用户故事 2 的测试

- [ ] T031 [P] [US2] 依据 [contracts/users.md](./contracts/users.md) 编写用户管理接口契约测试于 `tests/contract/users_test.go`
- [ ] T032 [P] [US2] 依据 [contracts/roles.md](./contracts/roles.md) 编写角色/权限管理接口契约测试于 `tests/contract/roles_test.go`
- [ ] T033 [P] [US2] 编写集成测试：创建用户+角色+权限分配、禁用用户后无法登录、无法删除/禁用唯一超级管理员于 `tests/integration/user_role_test.go`（对应 quickstart.md 场景 2）

### 用户故事 2 的实现

- [ ] T034 [P] [US2] 在 `internal/rbac/service.go` 中实现角色/权限领域逻辑（角色 CRUD、权限项 CRUD、用户-角色分配、权限并集合并规则）
- [ ] T035 [US2] 在 `internal/httpserver/handler/user_handler.go` 中实现用户管理接口（列表/创建/编辑/删除，含超级管理员保护规则）（依赖 T034）
- [ ] T036 [US2] 在 `internal/httpserver/handler/role_handler.go` 中实现角色与权限项管理接口（含内置超级管理员角色保护规则、角色删除级联解绑）（依赖 T034）
- [ ] T037 [US2] 在 `internal/httpserver/router.go` 中注册用户/角色/权限管理路由，并接入管理员权限校验（依赖 T035、T036）
- [ ] T038 [US2] 为用户创建/编辑/删除、角色创建/权限变更/删除等操作接入审计服务写入对应记录（依赖 T018、T035、T036）

**检查点**：用户故事 1、2 均可独立正常工作

---

## Phase 5：用户故事 3 - 基于权限的请求转发控制（优先级：P3）

**目标**：已登录用户访问业务路径时，系统按权限配置决定是否转发请求到下游
业务服务（对应 [spec.md](./spec.md) 用户故事 3）

**独立测试标准**：为测试角色仅授予部分路径权限，分别用有/无权限的用户凭证
请求对应路径，验证有权限的请求被转发、无权限的请求被拦截且不转发

### 用户故事 3 的测试

- [ ] T039 [P] [US3] 依据 [contracts/gateway.md](./contracts/gateway.md) 编写网关契约测试于 `tests/contract/gateway_test.go`
- [ ] T040 [P] [US3] 编写集成测试：有权限请求被转发、无权限请求返回 403 且未转发、未认证返回 401、下游超时返回 502/504 于 `tests/integration/gateway_test.go`（对应 quickstart.md 场景 3）

### 用户故事 3 的实现

- [ ] T041 [P] [US3] 在 `internal/forward/proxy.go` 中实现转发规则匹配逻辑（按路径前缀+方法匹配 `Forwarding Rule`）
- [ ] T042 [US3] 在 `internal/httpserver/middleware/rbac.go` 中实现权限校验中间件（依赖匹配到的转发规则所需权限，校验当前用户角色权限并集）（依赖 T027、T034、T041）
- [ ] T043 [US3] 在 `internal/httpserver/handler/gateway_handler.go` 中基于 `net/http/httputil.ReverseProxy` 实现转发处理器，并处理下游不可用/超时返回 502/504（依赖 T041）
- [ ] T044 [US3] 在 `internal/httpserver/router.go` 中注册网关兜底路由，确保中间件链顺序为 认证 → 权限校验 → 转发（依赖 T027、T042、T043）
- [ ] T045 [US3] 在 `internal/config/config.go` 与 `internal/forward/proxy.go` 中实现从 YAML 配置加载转发规则列表（依赖 T004、T041）

**检查点**：用户故事 1、2、3 均可独立正常工作，权限体系产生实际转发控制效果

---

## Phase 6：用户故事 4 - 安全操作审计留痕（优先级：P4）

**目标**：管理员可查询登录、登出与权限敏感操作的审计记录（对应
[spec.md](./spec.md) 用户故事 4）

**独立测试标准**：执行若干登录与权限变更操作后，通过审计查询接口验证记录中
准确包含操作人、操作时间、操作内容

### 用户故事 4 的测试

- [ ] T046 [P] [US4] 依据 [contracts/audit.md](./contracts/audit.md) 编写审计查询接口契约测试于 `tests/contract/audit_test.go`
- [ ] T047 [P] [US4] 编写集成测试：登录事件与角色权限变更均可在审计查询中检索到，且接口为只读于 `tests/integration/audit_test.go`（对应 quickstart.md 场景 4）

### 用户故事 4 的实现

- [ ] T048 [US4] 在 `internal/httpserver/handler/audit_handler.go` 中实现 `GET /api/gonex/audit-logs` 查询接口（支持按操作人/类型/时间范围过滤与分页）（依赖 T018）
- [ ] T049 [US4] 在 `internal/httpserver/router.go` 中注册审计查询路由并接入管理员权限校验（依赖 T048）
- [ ] T050 [US4] 核查用户故事 1、2 中所有敏感操作（登录、登出、用户/角色/权限变更）均已正确调用审计写入，补齐遗漏点（依赖 T029、T038、T048）

**检查点**：所有用户故事均可独立正常工作，审计链路完整闭环

---

## Phase 7：Polish & 跨领域事项

**目的**：影响多个用户故事的收尾工作

- [ ] T051 [P] 运行 `gofmt`/`go vet`/`golangci-lint` 并修复全部告警（宪法原则 I 门禁）
- [ ] T052 [P] 编写权限并集合并规则的单元测试（多角色重叠/冲突场景）于 `tests/unit/rbac_test.go`
- [ ] T053 [P] 编写登录锁定自动解锁边界条件的单元测试于 `tests/unit/lockout_test.go`
- [ ] T054 对 MySQL、PostgreSQL、SQLite 三种数据库分别运行完整集成测试矩阵，确认行为一致（宪法原则 V），修复发现的不一致
- [ ] T055 按 [quickstart.md](./quickstart.md) 全部场景手动或脚本化执行端到端验证，包含服务重启后会话仍有效的验证
- [ ] T056 [P] 编写面向部署者的中文使用说明（YAML 配置示例、`admin init` 命令用法）于 `README.md` 或 `docs/`

---

## 依赖关系与执行顺序

### 阶段依赖

- **Setup（Phase 1）**：无依赖，可立即开始
- **Foundational（Phase 2）**：依赖 Setup 完成——阻塞所有用户故事
- **用户故事（Phase 3+）**：均依赖 Foundational 完成
  - 各用户故事之间可并行开发，也可按优先级顺序（P1 → P2 → P3 → P4）串行推进
- **Polish（最终阶段）**：依赖所有计划交付的用户故事完成

### 用户故事依赖关系

- **用户故事 1（P1）**：Foundational 完成后即可开始，不依赖其他故事
- **用户故事 2（P2）**：Foundational 完成后即可开始；其"禁用用户无法登录"验收场景复用用户故事 1 的登录能力，但功能实现本身独立
- **用户故事 3（P3）**：依赖用户故事 1（认证中间件 T027）与用户故事 2（RBAC 服务 T034）已完成的组件，但作为独立中间件/处理器实现，可独立测试
- **用户故事 4（P4）**：依赖 Foundational 的审计服务（T018）及用户故事 1、2 中已埋点的审计调用，但查询接口本身可独立开发与测试

### 每个用户故事内部

- 测试先行，确认失败后再实现（宪法要求安全关键路径必须有测试覆盖）
- 领域逻辑（auth/rbac/forward）先于 HTTP 处理器
- 处理器先于路由集成
- 核心实现先于审计埋点集成

### 并行机会

- Phase 1 中标记 [P] 的任务可并行执行
- Phase 2 中标记 [P] 的任务（尤其是 T007-T012 三方言迁移/查询）可并行执行
- Foundational 完成后，US1/US2 可由不同开发者并行推进；US3 需等待 US1 的
  T027、US2 的 T034 完成后再开始其中间件实现（T042），但契约测试（T039）与
  转发规则匹配（T041）可提前并行开发
- 同一用户故事内标记 [P] 的测试与领域模型任务可并行执行

---

## 并行执行示例：用户故事 1

```bash
# 并行编写用户故事 1 的测试：
Task: "依据 contracts/auth.md 编写登录/登出契约测试于 tests/contract/auth_test.go"
Task: "编写登录集成测试于 tests/integration/auth_login_test.go"

# 并行编写用户故事 1 的领域逻辑：
Task: "在 internal/auth/password.go 中实现 bcrypt 密码哈希与校验"
Task: "在 internal/auth/session.go 中实现会话令牌签发/校验/吊销"
```

---

## 实施策略

### 先交付 MVP（仅用户故事 1）

1. 完成 Phase 1：Setup
2. 完成 Phase 2：Foundational（关键——阻塞所有用户故事）
3. 完成 Phase 3：用户故事 1（登录）
4. **停下并验证**：独立测试用户故事 1（quickstart.md 场景 1 + 会话持久性验证）
5. 具备部署/演示条件

### 增量交付

1. 完成 Setup + Foundational → 基础设施就绪
2. 交付用户故事 1 → 独立测试 → 部署/演示（MVP！）
3. 交付用户故事 2 → 独立测试 → 部署/演示
4. 交付用户故事 3 → 独立测试 → 部署/演示
5. 交付用户故事 4 → 独立测试 → 部署/演示
6. 每个故事在不破坏此前故事的前提下持续增加价值

---

## 备注

- [P] 任务 = 不同文件、无依赖关系
- [Story] 标签用于将任务追溯到具体用户故事
- 每个用户故事都应可独立完成与独立测试
- 实现前先确认测试失败（红-绿流程）
- 建议每完成一个任务或一组逻辑相关任务后提交一次
- 可在任意检查点停下，独立验证该用户故事
- 避免：模糊任务描述、同文件冲突、破坏用户故事独立性的跨故事依赖
