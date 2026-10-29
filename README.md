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
- [Winston、生命周期与可观测性](docs/guide/10-security-observability.md)
- [TypeScript 全 ESM 规范](docs/guide/12-typescript-conventions.md)
- [Vitest 单元、集成、E2E 与 Smoke](docs/guide/13-testing.md)
- [tsx、Docker Compose 与交付](docs/guide/14-development-delivery.md)

技术约定包括 `"type": "module"`、`rewriteRelativeImportExtensions`、面向对象分层、构造函数注入、Zod 边界验证、es-toolkit 和 Tinypool。完整导航及阅读顺序见手册总目录。
