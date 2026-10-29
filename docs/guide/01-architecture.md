# 架构、目录与模块开发

## 1. 规范等级与技术基线

### 1.1 规范等级

| 等级 | 含义 | 典型内容 |
| --- | --- | --- |
| 必须 | 正确性、安全性、资源生命周期和团队协作的硬性要求 | 严格配置校验、业务不变量、事务边界、密钥保护、优雅退出 |
| 建议 | 默认采用的工程组织方式，有明确理由时可调整 | 按模块分层、构造函数注入、统一响应、测试目录 |
| 按需 | 需求出现后引入，不为架构完整而增加依赖 | MongoDB、Redis、Tinypool、队列、指标和追踪 |

调整必须项前必须修改项目约定并评估兼容性、安全性和测试影响。替换某个库不改变所属层的职责。

### 1.2 默认技术组合

| 领域 | 默认选择 | 约定 |
| --- | --- | --- |
| 运行时 | Node.js Active LTS | 用 `engines`、`.nvmrc` 和 CI 固定主版本，不使用 EOL 版本 |
| 语言 | TypeScript，ESM，strict | 生产运行编译后的 JavaScript，不用 `any` 逃避契约 |
| HTTP | Hono + `@hono/node-server` | Controller 注册路由，公共策略由中间件实现 |
| 开发执行 | `tsx` | 使用 `tsx watch` 启动开发进程；生产使用 `node dist/...` |
| 验证 | Zod | 配置、环境变量、HTTP DTO、外部消息统一在边界解析 |
| 主数据库方案 A | PostgreSQL + Drizzle ORM + `pg` | 关系模型、约束、Join 和事务优先 |
| 主数据库方案 B | MongoDB + Mongoose | 文档模型、聚合和灵活演进优先 |
| 缓存抽象 | Keyv + 明确选择的 Adapter | 普通 KV 缓存与后端解耦，不与 Redis 原生能力混为一谈 |
| Redis | 官方 `redis` 客户端 | 原子计数、限流、锁、Pub/Sub、Stream 等按需直接使用 |
| 工具函数 | es-toolkit | 使用经过测试的通用函数，保持可 tree-shake 的导入 |
| CPU 并行 | Tinypool | CPU 密集任务放入 Worker Threads，不当作持久化队列 |
| 测试 | Vitest | 单元、集成、契约和 E2E 分层验证 |
| 质量 | ESLint + Prettier + TypeScript | 格式化、静态检查和类型检查分别执行 |

**PostgreSQL + Drizzle 与 MongoDB + Mongoose 是两种主数据库技术选型。** 默认一个业务模块只选择一种作为事实数据源，不要求同一模块同时写入两者。确有异构持久化需求时，必须通过 ADR 明确每个字段的事实来源、同步方式、失败恢复和对账机制。

**Keyv 与 Redis 分开决策。** Keyv 是可替换后端的 KV 抽象；Redis 是具体存储及原子操作平台。`@keyv/redis` 只是 Keyv 的一种 Adapter，不意味着所有 Redis 使用都要经过 Keyv。

依赖版本以 lockfile 和实际官方文档为准。引入或升级 Hono、Drizzle、Mongoose、Keyv、Zod 等库时，先阅读对应版本的迁移说明，不混用不同大版本的示例。

## 3. 架构与面向对象组织

### 3.1 推荐形态

采用一个应用包、按进程职责拆分入口、按业务模块组织代码、模块内按职责分层的 monolith 起点：

- `api`：长期运行的 HTTP 服务。
- `migrate`：数据库结构迁移，独立执行并退出。
- `worker`：确有持久后台任务时增加，不把 API 进程当任务队列。
- `script`：一次性维护命令，必须与正常请求共用业务约束。

业务增加不自动意味着拆微服务。先保持模块边界、数据所有权和资源生命周期清晰。

### 3.2 面向对象的含义

面向对象不是给所有函数套 `class`，而是：

- 用类封装明确职责和 collaborator，例如 `OrderController`、`OrderService`、`DrizzleOrderRepository`。
- 用构造函数显式注入依赖，业务类不自行读取环境变量、创建连接或查找全局容器。
- 用消费方定义的窄接口隔离业务与实现，例如 Service 依赖 `OrderRepository`，不依赖 Drizzle 或 Mongoose 类型。
- 领域对象保护自身不变量；不要把所有业务规则都放到 Controller 或数据库 Hook。
- 只在有状态、有生命周期或需要替换实现时建立类。无状态纯函数无需包装成类。

禁止 Service Locator 式 `container.get()` 散布在业务代码中。默认手工 composition root；大型项目确需 DI 容器时，容器只能在 bootstrap 出现。

### 3.3 依赖方向

```text
HTTP request
    -> Controller / Middleware
    -> Application Service / Domain
    -> Repository, Cache, Client 等消费方接口
    -> Drizzle / Mongoose / Keyv / Redis / HTTP Adapter

bootstrap 负责创建并连接以上对象；基础设施不反向依赖 Controller。
```

Controller 可以依赖 Service 接口；Service 不 import Hono。Repository 可以实现 Service 定义的接口，但不得把数据库行、Mongoose Document 或 Redis 返回值直接泄漏为公共业务契约。

## 4. 目录与命名

### 4.1 推荐目录

```text
.
 src/
    main.ts                       # 唯一进程启动副作用入口
    bootstrap/
       application.ts            # 创建、启动、关闭整个应用
       container.ts              # 显式装配，不提供全局 get
       modules/                  # 各模块装配函数
    config/
       loader.ts                 # TOML/.env/进程环境合并
       schema.ts                 # Zod 配置契约
       defaults.ts               # 非敏感默认值
       types.ts
    http/
       server.ts                 # Hono 与 Node server 生命周期
       router.ts                 # 公共中间件与模块路由
       middleware/
       transport/                # 响应、错误、DTO 解析辅助
    modules/
       order/
           order.controller.ts
           order.routes.ts
           order.service.ts
           order.repository.ts   # 消费方接口
           order.types.ts        # 领域/应用类型
           order.errors.ts
           order.schemas.ts      # HTTP Zod DTO
           infrastructure/
               drizzle-order.repository.ts
               mongoose-order.repository.ts # 选型之一，不要求并存
    infrastructure/
       database/
          postgres/             # Pool、Drizzle、schema、迁移适配
          mongodb/              # Mongoose connection、models、迁移适配
       cache/                    # Keyv 实例与 Cache 接口适配
       redis/                    # 独立 Redis client 与原生能力
       logging/
       workers/                  # Tinypool 池与 worker 入口
    shared/                       # 真正跨模块且稳定的通用能力
 drizzle/                          # 选 PostgreSQL 时的生成迁移
 scripts/                          # 工程脚本，不放业务快捷逻辑
 test/
    integration/
    contract/
    e2e/
 config.example.toml               # 可提交的完整非秘密示例
 config.toml                       # 本地配置，按项目策略忽略
 .env.example                      # 只有变量名和占位值
 .env                              # 本地秘密，必须忽略
 package.json
 tsconfig.json
 README.md
```

只为当前选型创建目录。选择 PostgreSQL 时不要保留空的 MongoDB 实现；选择 MongoDB 时也不要生成 Drizzle 迁移。

### 4.2 命名与文件

- 类、接口、类型：`UpperCamelCase`，如 `OrderService`、`Clock`、`OrderRepository`。
- 变量、函数、方法：`lowerCamelCase`，如 `orderId`、`createOrder()`。
- 文件：kebab-case，如 `order.service.ts`；测试为 `order.service.test.ts`。
- HTTP JSON、查询参数：`camelCase` 或公开 API 已约定的风格，全项目统一。
- 数据库表、列：PostgreSQL 用 `snake_case`；MongoDB 字段按集合契约统一。
- 环境变量：`APP_` 前缀加 `UPPER_SNAKE_CASE`。
- acronym 按一个词处理：`HttpServer`、`OrderId`、`parseUrl`，公开协议中的 `URL`、`ID` 不强制跟随代码标识符。
- 布尔值使用 `isEnabled`、`hasPermission`、`shouldRetry` 等可读谓词。

接口不以 `I` 开头。实现名称体现技术，接口名称体现业务能力，例如 `OrderRepository` 与 `DrizzleOrderRepository`。

## 5. 分层职责与依赖规则

### 5.1 Controller / Routes

负责 HTTP 协议，不负责业务计算：

- 注册路由，声明认证、权限、限流和请求体验证中间件。
- 用 Zod 解析 path、query、headers 和 body。
- 将 HTTP DTO 显式转换为 Service 输入。
- 将 Service 输出显式转换为响应 DTO，设置状态码和 `Location` 等 header。
- 不拼接 SQL、不操作 Mongoose Model、不读取 `process.env`、不决定事务。

路由注册使用类方法时保留接收者，例如 `app.post(path, (c) => controller.create(c))`，不要随意传递未绑定的方法引用。

### 5.2 Service / Domain

负责用例和业务规则：

- 权限之外的业务不变量、状态迁移、幂等语义和事务需求。
- 定义自己需要的 Repository、Clock、IdGenerator、Cache 等窄接口。
- 统一规范化输入，例如 trim、大小写和时区；但边界 DTO 验证仍不可省略。
- 返回业务类型，抛出可分类的领域错误。
- 不知道 HTTP 状态码、Hono Context、Drizzle Table、Mongoose Document 和 Redis key 的序列化细节。

只有业务流程需要跨多个 Repository 原子提交时，由 Service 请求一个明确的 Unit of Work；事务实现仍由基础设施提供。

### 5.3 Repository / Infrastructure

负责持久化和外部协议：

- 将业务输入映射为 Drizzle 查询或 Mongoose 操作。
- 将数据库唯一约束、版本冲突、重复键等翻译为稳定领域错误。
- 明确 `null`、空数组和不存在的差异，不能用 falsy 判断代替。
- 拥有查询形状、索引需求、分页游标和事务实现。
- 不返回密码哈希、内部凭据和数据库元数据，除非该业务接口明确需要。

一个 Repository 不应成为包含所有表的 `DatabaseService`。按聚合或清晰查询职责拆分，避免无边界通用 `save(entity)`。

### 5.4 DTO、领域类型与持久化类型

三者默认分离：

| 类型 | 所有者 | 变化原因 |
| --- | --- | --- |
| HTTP DTO | Controller / schema | API 兼容性和输入输出协议 |
| 领域/应用类型 | Service / Domain | 业务规则与用例 |
| Drizzle Row / Mongoose Document | Infrastructure | 查询和存储结构 |

不要用 `typeof table.$inferSelect` 作为 HTTP 响应，也不要把 Zod DTO 直接传给数据库。显式映射虽增加少量代码，却能防止内部字段泄漏和无意 API 破坏。

### 5.5 shared 的准入

`shared` 只放两个以上模块真实使用、职责稳定且不拥有业务的代码。业务错误、业务常量和领域 ID 留在所属模块。不要建立 `utils.ts` 收纳随机函数；通用数组、对象、异步辅助优先使用 es-toolkit 或标准库。

## 6. 完整模块开发流程

### 6.1 实施顺序

新增模块按以下顺序推进，每一步都明确输入、输出和失败边界：

1. 写用例、业务不变量、权限、幂等和事实数据源说明。
2. 选择 PostgreSQL + Drizzle 或 MongoDB + Mongoose；需要缓存时另选 Keyv Adapter，需要 Redis 原子能力时单独设计。
3. 定义 Service 输入输出、领域错误和消费方接口。
4. 设计表/集合、约束、索引和兼容迁移。
5. 实现 Repository，并用真实数据库做契约与集成测试。
6. 实现 Service，用内存 fake 验证业务分支，不模拟所有数据库行为。
7. 定义 Zod HTTP DTO，编写 Controller 和路由。
8. 在 bootstrap 显式装配，最后才开放路由。
9. 补齐 API 文档、正常/失败测试、日志字段、指标和回滚说明。

禁止先建立所有空目录再填实现。模块从一条可验证的完整链路开始。

### 6.2 贯穿示例：创建订单

以下只展示依赖关系，不是可直接复制的完整实现：

```ts
// order.repository.ts：接口由业务消费方定义。
export interface OrderRepository {
    findByIdempotencyKey(ownerId: string, key: string): Promise<Order | null>;
    create(input: NewOrder): Promise<Order>;
}

// order.service.ts：不知道 Hono 和数据库。
export class OrderService {
    constructor(
        private readonly orders: OrderRepository,
        private readonly ids: IdGenerator,
        private readonly clock: Clock,
    ) {}

    async create(input: CreateOrderInput): Promise<Order> {
        const existing = await this.orders.findByIdempotencyKey(
            input.ownerId,
            input.idempotencyKey,
        );
        if (existing) return existing;

        const order = Order.create({
            id: this.ids.next(),
            ownerId: input.ownerId,
            items: input.items,
            now: this.clock.now(),
        });
        return this.orders.create(order);
    }
}
```

生产实现必须处理查询后并发插入的竞争：幂等约束由数据库保证，捕获重复键后读取并返回首次结果。Service 中的预查询只能减少正常重复请求的数据库写入，不能代替唯一约束或原子操作。

```ts
// order.controller.ts：HTTP 与业务类型显式转换。
export class OrderController {
    constructor(private readonly service: OrderService) {}

    readonly create = async (c: Context) => {
        const body = createOrderBodySchema.parse(await c.req.json());
        const principal = getPrincipal(c);
        const order = await this.service.create({
            ownerId: principal.accountId,
            idempotencyKey: getRequiredIdempotencyKey(c),
            items: body.items,
        });
        c.header("Location", `/orders/${order.id}`);
        return c.json({ data: toOrderResponse(order) }, 201);
    };
}
```

重复请求返回首次创建结果时，可以统一返回 `200` 或在 API 契约中规定仍返回首次的 `201`；不要让客户端靠响应体猜测是否新建。完整实现应把 Zod 错误交给集中错误映射，避免在 Controller 各处重复 try/catch。

## 6.3 应用服务与领域服务如何区分

- 应用服务面向一个用例，例如 `CreateOrderService`，协调权限后的输入、Repository、事务和事件。
- 领域服务表达跨领域对象且不属于单个对象的规则，例如库存分配、价格计算。
- 领域对象适合保护自身状态，例如 `Order.cancel()` 检查当前状态，而不是在 Controller 任意修改 status。
- 不要把所有类都命名为 `Manager`、`Helper` 或 `BaseService`；名称应对应用例或领域能力。
- 一个 Service 可以按模块包含多个相关用例，但方法过多时按业务能力拆分，而不是机械按行数拆分。

## 6.4 跨模块协作规则

模块 A 需要模块 B 的能力时，优先依赖 B 暴露的应用接口或领域查询，不直接读取 B 的表、Model 和内部 Repository。

```text
billing application service
    -> pricing quote interface
    -> pricing module public application service

禁止：billing -> pricing infrastructure -> pricing tables
```

- 依赖必须保持无环；出现循环时重新识别上游能力或提取真正独立的领域概念。
- 跨模块同步调用只适用于一致性和延迟边界明确的场景。
- 跨模块异步协作需要事件版本、幂等消费、乱序处理和失败恢复，不能只发一个 Pub/Sub 消息。
- 同数据库跨模块事务属于强耦合决策，必须在 ADR 中说明，不能由 Repository 暗中Join或写入他模块数据。

## 6.5 事务边界决策

开发写用例时先回答：

1. 哪些写入必须同时成功或失败？
2. 哪个数据库约束保证并发下仍成立？
3. 是否需要锁定、条件更新或乐观版本？
4. 缓存、Redis、HTTP 和消息在事务前还是事务后执行？
5. 提交成功但后续动作失败时，调用方看到什么，如何补偿？

数据库事务不能覆盖 Redis、Keyv、外部 HTTP 或 Tinypool。需要可靠后续动作时考虑事务 outbox、可重试任务或对账，不在提交后假装所有系统具有同一事务。

## 6.6 常见反模式

| 反模式 | 问题 | 修正 |
| --- | --- | --- |
| Controller 直接调用 Drizzle/Mongoose | 协议、业务和查询耦合 | 提取 Service 与 Repository |
| Service 接收 Hono Context | 业务无法脱离 HTTP 测试 | 传递明确应用输入 |
| 全局 `db`、`config`、`logger` 可写变量 | 所有权和测试隔离不清 | bootstrap 创建并构造注入 |
| Repository 返回 Mongoose Document | 存储行为泄漏到业务 | 映射为领域/应用类型 |
| 通用 `BaseRepository` 承载所有业务 | 抹平查询和事务差异 | 按聚合定义窄接口 |
| 用缓存或 Redis 代替数据库唯一约束 | 并发下产生重复和超卖 | 数据库约束或原子条件更新兜底 |
| 为替换数据库设计最低能力 DAO | 失去所选数据库优势 | 只隔离业务契约，保留技术特性 |

## 6.7 模块评审清单

- [ ] 用例、权限、事实来源和业务不变量有文字说明。
- [ ] 依赖方向无环，Service 不包含 HTTP 和数据库 imports。
- [ ] 写用例有唯一性、幂等、并发和失败恢复设计。
- [ ] Repository 有真实数据库测试，不只验证 mock 调用。
- [ ] DTO 与持久化类型分离，秘密字段不会泄漏。
- [ ] bootstrap 装配、日志、文档和关闭行为同步更新。

[返回总目录](../development-guide.md) | 下一篇：[Hono HTTP 与 Zod API](02-http-api.md)
