# gonex

gonex 是一个面向中后台系统的后端独立程序：单一可执行文件即可部署，通过 YAML
配置文件管理多用户、登录认证与权限校验，并将 HTTP 请求按配置路径转发到实际的
业务服务处理具体业务逻辑。管理界面由 React 页面在运行时动态加载，本仓库
不包含任何前端代码。

技术栈：Go + Gin（HTTP）+ sqlc/goose（数据库访问与迁移）+ PostgreSQL（数据库）+
YAML（配置，`go.yaml.in/yaml/v4`）+ Cobra（命令行）。

详见项目宪法：[.specify/memory/constitution.md](.specify/memory/constitution.md)