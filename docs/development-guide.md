# Node.js + Hono 面向对象 API 工程开发手册

本手册用于建立和维护 Node.js + TypeScript + Hono HTTP API 工程。核心方式是按业务模块组织代码，使用 Controller / Service / Repository 分层、构造函数依赖注入和统一资源生命周期。

本仓库只提供开发规范，不包含可直接运行的脚手架。各专题独立维护，项目实际采用的技术、配置和测试以源码与 ADR 为准。

## 1. 技术决策总览

| 领域 | 默认规范 |
| --- | --- |
| 运行时与语言 | Node.js Active LTS、TypeScript strict、全部 ESM |
| 模块模式 | `package.json` 使用 `type=module`，开启 `rewriteRelativeImportExtensions` |
| HTTP | Hono + `@hono/node-server` |
| 开发执行 | `tsx watch`；生产运行 `tsc` 构建后的 JavaScript |
| 配置 | smol-toml + TOML + dotenv + 进程环境变量 + Zod 最终校验 |
| 主数据库 | PostgreSQL + Drizzle ORM 或 MongoDB + Mongoose，明确选型 |
| 缓存抽象 | Keyv，显式选择一个 Adapter |
| Redis | 需要原子操作、限流、锁、Pub/Sub、Stream 时独立使用官方客户端 |
| 日志 | Winston，统一结构化 format、脱敏、transport 和关闭 |
| 通用工具 | 标准库与 es-toolkit |
| CPU 并行 | 有明确 CPU 密集任务时使用 Tinypool |
| 测试与质量 | Vitest 单元/集成/E2E/Smoke，ESLint、Prettier、TypeScript |
| 开发依赖环境 | Docker Compose profiles，按数据库和 Redis 选型启动 |

两个边界必须先理解：

1. **PostgreSQL + Drizzle 和 MongoDB + Mongoose 是主数据库备选方案，不要求同时使用。** 混合持久化必须另行记录事实来源、失败恢复和对账方式。
2. **Keyv 和 Redis 不是同一层。** Keyv 是可替换后端的 KV 抽象；Redis 客户端提供具体数据结构和原子能力。`@keyv/redis` 只是 Keyv 的一种 Adapter。

## 2. 文档目录

| 顺序 | 文档 | 主要内容 |
| --- | --- | --- |
| 01 | [架构、目录与模块开发](guide/01-architecture.md) | 面向对象边界、依赖方向、目录、Controller/Service/Repository、模块流程 |
| 02 | [Hono HTTP 与 Zod API](guide/02-http-api.md) | 路由、中间件、DTO、`{error,message}`、认证、幂等、分页 |
| 03 | [api.md 编写规范](guide/03-api-documentation.md) | API 文档结构、字段、示例、错误码、兼容变更、文档 Smoke |
| 04 | [TOML 与环境变量配置](guide/04-configuration.md) | 加载优先级、重复 `--config`、dotenv、Zod、秘密、配置测试 |
| 05 | [PostgreSQL + Drizzle ORM](guide/05-postgresql-drizzle.md) | Schema、Pool、Repository、事务、并发、迁移、滚动发布 |
| 06 | [MongoDB + Mongoose](guide/06-mongodb-mongoose.md) | Schema/Model、文档设计、验证、Session、索引、文档迁移 |
| 07 | [Keyv 缓存与 Adapter](guide/07-keyv-cache.md) | Adapter 选择、TTL、namespace、序列化、Cache-Aside、失效和契约 |
| 08 | [Redis 原生能力](guide/08-redis.md) | 独立客户端、原子限流、锁、Pub/Sub、Stream、可靠性和容量 |
| 09 | [依赖装配与生命周期](guide/09-application-lifecycle.md) | Composition Root、启动失败清理、请求取消、优雅退出 |
| 10 | [安全、Winston 日志与可观测性](guide/10-security-observability.md) | 安全基线、日志脱敏、健康检查、指标和追踪 |
| 11 | [多实例、性能与 Tinypool](guide/11-performance-workers.md) | 水平扩展、Worker Threads、es-toolkit、性能测量 |
| 12 | [TypeScript 与 ESM 规范](guide/12-typescript-conventions.md) | `type=module`、扩展名改写、严格类型、异步和模块解析 |
| 13 | [Vitest 测试策略](guide/13-testing.md) | 单元、集成、E2E、Smoke、真实依赖、并发与生命周期测试 |
| 14 | [开发、Compose、构建与交付](guide/14-development-delivery.md) | tsx、Docker Compose、npm scripts、Docker、CI 和发布 |
| 15 | [新项目裁剪与验收](guide/15-adoption-acceptance.md) | 采用步骤、完整验收清单、ADR 和交付证据 |
| 16 | [官方资料](guide/16-references.md) | 全部技术的官方文档入口和版本使用规则 |

## 3. 推荐阅读路径

### 3.1 从零建立项目

按 `01  02  03  04  05 或 06  09  13  14  15` 阅读。先跑通一个真实业务链路，再增加缓存、Redis、Worker 等可选能力。

### 3.2 开发普通业务模块

阅读 `01  02  当前数据库专题  13`。修改 API 契约时同步维护 `03` 所要求的 api.md；涉及缓存时增加 `07`，涉及 Redis 原子操作时增加 `08`。

### 3.3 修改配置或部署

阅读 `04  09  10  12  14  15`。任何新配置都必须同步示例、schema、优先级测试和部署文档。

### 3.4 排查性能或稳定性

阅读 `09  10  11  13`。先通过 Winston 日志、指标和 profiling 定位，再决定缓存、连接池、Worker 或数据库优化。

## 4. 公共协议与规范等级

成功响应默认使用 `{data,meta}`；错误响应体固定为：

```json
{
    "error": "ORDER_NOT_FOUND",
    "message": "订单不存在"
}
```

`error` 是稳定机器可读 code，`message` 是安全的人类可读信息。请求 ID 通过 `X-Request-ID` 响应头和 Winston 日志关联，不在错误 body 中增加嵌套对象。

| 等级 | 含义 |
| --- | --- |
| 必须 | 正确性、安全性、资源所有权、配置和团队交付的硬性要求 |
| 建议 | 默认采用；有具体理由时可以调整，并记录影响和验证方式 |
| 按需 | 需求出现后引入，不为目录或技术清单完整而添加 |

替换库不能改变所属层职责。例如 Mongoose 替代 Drizzle 不会让 Mongoose Model 进入 Service；Keyv 更换 Adapter 不会把缓存变成事实数据源。

## 5. 文档维护规则

- 修改技术行为时，同步更新对应专题、配置示例、api.md、测试要求和验收清单。
- 示例与项目实际依赖主版本保持一致，升级时查阅迁移文档。
- 跨文档链接使用相对路径；移动文件时检查所有引用。
- 不在多个专题复制同一套规则。总目录只做导航，具体规则由所属专题维护。
- 未实现的示例明确标记为示例；未执行的验收不能写成已经通过。
