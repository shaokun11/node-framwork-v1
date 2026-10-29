# Node.js Hono API 工程开发规范

本仓库只维护工程开发文档，不包含可直接运行的脚手架。

## 入口

完整手册：[Node.js + Hono 面向对象 API 工程开发手册](docs/development-guide.md)。

## 专题

- [架构、目录与模块开发](docs/guide/01-architecture.md)
- [Hono HTTP 与 Zod](docs/guide/02-http-api.md)
- [api.md 编写规范](docs/guide/03-api-documentation.md)
- [TOML 与环境变量配置](docs/guide/04-configuration.md)
- [PostgreSQL + Drizzle](docs/guide/05-postgresql-drizzle.md) / [MongoDB + Mongoose](docs/guide/06-mongodb-mongoose.md)
- [Keyv 缓存](docs/guide/07-keyv-cache.md) / [Redis 原生能力](docs/guide/08-redis.md)
- [BullMQ 持久任务队列（Redis / PostgreSQL）](docs/guide/09-bullmq.md)
- [应用生命周期](docs/guide/10-application-lifecycle.md)
- [Hono JWT、Casbin、Winston 与可观测性](docs/guide/11-security-observability.md)
- [Croner、性能与 Tinypool](docs/guide/12-performance-workers.md)
- [TypeScript、Day.js 与全 ESM 规范](docs/guide/13-typescript-conventions.md)
- [Vitest 单元、集成、E2E 与 Smoke](docs/guide/14-testing.md)
- [tsx、Docker Compose 与交付](docs/guide/15-development-delivery.md)

技术约定包括 `"type": "module"`、`rewriteRelativeImportExtensions`、面向对象分层、构造函数注入、Zod 边界验证、BullMQ、Croner、Day.js、Hono JWT/Casbin、es-toolkit 和 Tinypool。完整导航及阅读顺序见手册总目录。
