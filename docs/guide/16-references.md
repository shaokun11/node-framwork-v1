# 官方资料与版本参考

## 附录 B：官方资料

采用和升级时查阅对应版本官方文档：

- [Hono：Node.js 入门](https://hono.dev/docs/getting-started/nodejs)
- [Drizzle ORM 文档](https://orm.drizzle.team/docs/overview)
- [Mongoose Schema/Guide](https://mongoosejs.com/docs/guide.html)
- [Keyv Getting Started](https://keyv.org/docs/#1-make-a-project-directory)
- [Keyv Redis Adapter](https://keyv.org/docs/storage-adapters/redis/)
- [Redis 文档](https://redis.io/docs/latest/)
- [Zod 文档](https://zod.dev/)
- [tsx 文档](https://tsx.is/)
- [es-toolkit 中文介绍](https://es-toolkit.dev/zh_hans/intro.html)
- [Tinypool](https://github.com/tinylibs/tinypool)
- [smol-toml](https://github.com/squirrelchat/smol-toml)
- [Node.js 文档](https://nodejs.org/docs/latest/api/)
- [Vitest 文档](https://vitest.dev/guide/)

本文规定工程职责与验收方式，不构成任何依赖或部署方案的生产可用性保证。实际项目以当前源码、lockfile、配置、测试和运行环境共同证明其行为。

## 参考资料使用规则

1. 官方文档是库 API 和版本行为的主要依据，本手册规定工程职责与团队约定。
2. 搜索文档时选择 lockfile 对应主版本，不能直接复制 latest 的选项到旧版本。
3. 升级前阅读 migration/changelog，并通过单元、集成、E2E 和 Smoke 验证。
4. 第三方博客只用于补充理解，不覆盖官方安全、配置和生命周期说明。

补充工程资料：

- [TypeScript：rewriteRelativeImportExtensions](https://www.typescriptlang.org/tsconfig/rewriteRelativeImportExtensions.html)
- [TypeScript：NodeNext 模块](https://www.typescriptlang.org/tsconfig/module.html)
- [Node.js：ES Modules](https://nodejs.org/api/esm.html)
- [Winston 日志](https://github.com/winstonjs/winston)
- [Docker Compose](https://docs.docker.com/compose/)
- [Docker Compose profiles](https://docs.docker.com/compose/profiles/)
- [dotenv](https://github.com/motdotla/dotenv)
- [node-postgres](https://node-postgres.com/)
- [Zod Hono Validator](https://github.com/honojs/middleware/tree/main/packages/zod-validator)


## 其他工具官方文档

- [ESLint](https://eslint.org/docs/latest/)
- [typescript-eslint](https://typescript-eslint.io/getting-started)
- [Prettier](https://prettier.io/docs/)
- [npm](https://docs.npmjs.com/)
- [Docker](https://docs.docker.com/)

上一篇：[裁剪与验收](15-adoption-acceptance.md) | [返回总目录](../development-guide.md)
