# BullMQ 持久任务队列

## 13. BullMQ 持久任务队列

BullMQ 用于需要持久保存、异步执行、延迟调度、失败重试和多 Worker 分配的任务。例如发送邮件、生成报表、处理 webhook、执行订单后续步骤和定时同步。

官方 Quick Start：<https://docs.bullmq.io/quick-start>。本章同时覆盖官方支持的 Redis 与 PostgreSQL 后端；项目实际采用前仍应核对 lockfile 中 BullMQ 版本的 API。

### 13.1 什么时候使用 BullMQ

| 需求 | 推荐方式 | 原因 |
| --- | --- | --- |
| 请求内立即完成的小计算 | 普通 Service | 无需增加队列和异步状态 |
| CPU 密集、结果立即返回 | Tinypool | Worker Threads 解决 event loop 阻塞 |
| 可丢失的在线通知 | Pub/Sub 等轻量机制 | 不要求持久重试 |
| 单进程轻量 cron 触发，不要求宕机补跑 | [Croner](12-performance-workers.md#1614-croner-进程内定时任务) | 表达式和进程内调度足够 |
| 崩溃后仍需继续执行 | BullMQ | 任务保存在后端，由 Worker 再消费 |
| 延迟、重试、跨进程消费 | BullMQ | Queue/Worker 提供任务生命周期 |
| 必须严格只执行一次的业务写入 | 数据库约束、幂等键和事务 | 队列通常提供至少一次语义，不能代替业务幂等 |

Controller 不直接 import BullMQ。由应用服务依赖一个窄的 `WelcomeEmailQueue`/`ReportJobQueue` 接口，infrastructure 实现负责 `queue.add()`。这样业务层不暴露 Redis、PostgreSQL 或 BullMQ 连接细节。

API 入队成功后应返回任务 ID 或可查询的业务操作 ID。不能把“已入队”描述成“业务已全部完成”；需要结果时提供状态查询、通知或 webhook。

## 13.2 Redis 与 PostgreSQL 后端怎么选

BullMQ 默认使用 Redis，也提供可选 PostgreSQL 后端。两种后端使用同一套 `Queue`、`Worker`、`QueueEvents`、`FlowProducer` API，但部署、迁移和容量特征不同。

| 维度 | Redis 后端 | PostgreSQL 后端 |
| --- | --- | --- |
| 版本要求 | Redis 6.2+ 或兼容服务 | PostgreSQL 13+，官方建议 14+ |
| Node 依赖 | `bullmq`，通常配合 `ioredis` | `bullmq` + `pg` |
| 默认定位 | 默认且最成熟的后端 | 可选后端，复用关系数据库 |
| 适合场景 | 高吞吐队列、已有 Redis 运维能力 | 不希望单独运维 Redis，任务与关系数据放在一起 |
| 初始化 | 准备 Redis 和连接配置 | 必须先显式执行 BullMQ schema migrations |
| 容量重点 | 内存、持久化、复制、`maxmemory` 策略 | 连接池、`max_connections`、WAL、表和索引维护 |
| 性能 | 通常任务处理吞吐更高 | 单次延迟可接近，吞吐取决于实际负载和数据库配置 |

选型建议：

- 已有高吞吐队列或 Redis 基础设施时，默认优先 Redis。
- 已有 PostgreSQL，队列规模可控，希望减少一种基础设施时，可以选择 PostgreSQL。
- 不用“和业务表在同一个数据库”推导“入队与业务写入自动属于同一事务”。需要原子业务一致性时，另行设计事务、outbox 或对账恢复。
- 不以官网示例吞吐作为项目容量结论，必须使用真实 payload、Worker 并发、网络和后端配置压测。
- 同名队列不会因切换后端而自动迁移。环境必须统一后端，生产者、Worker 和事件监听端使用一致配置；切换需要独立的排空/迁移方案。

## 13.3 安装与本地依赖

Redis 后端：

```sh
npm install bullmq ioredis
docker compose -f compose.dev.yaml --profile redis up -d --wait
```

PostgreSQL 后端：

```sh
npm install bullmq pg
npm install --save-dev @types/pg
docker compose -f compose.dev.yaml --profile postgres up -d --wait
```

配置通过既有 Loader 和 Zod 校验，连接 URL 从秘密环境变量注入，不提交真实账号密码：

```dotenv
APP_QUEUE_BACKEND=redis
APP_REDIS_URL=redis://127.0.0.1:6379
# 选择 postgres 时改为：
# APP_QUEUE_BACKEND=postgres
# APP_POSTGRES_URL=postgres://app:local-only-password@127.0.0.1:5432/app
```

```ts
const queueConfigSchema = z
    .object({
        backend: z.enum(["redis", "postgres"]),
        redisUrl: z.string().url().optional(),
        postgresUrl: z.string().url().optional(),
        concurrency: z.number().int().min(1).max(100),
    })
    .strict()
    .superRefine((value, context) => {
        const required = value.backend === "redis" ? value.redisUrl : value.postgresUrl;
        if (!required) {
            context.addIssue({
                code: "custom",
                path: [value.backend === "redis" ? "redisUrl" : "postgresUrl"],
                message: "queue backend requires its connection URL",
            });
        }
    });
```

不要把两种 URL 都设成无条件必填。未启用队列时可以不装配 Queue/Worker；启用后才校验当前后端所需配置。

## 13.4 Redis Quick Start：生产者与 Worker

以下示例展示完整关系：定义 payload、创建 Queue、创建 Worker、添加工厂和关闭资源。真实项目把这些对象放入 Composition Root，不在模块 import 时隐式启动。

```ts
import { Queue, Worker } from "bullmq";
import IORedis from "ioredis";

export interface WelcomeEmailJobData {
    version: 1;
    userId: string;
}

const producerConnection = new IORedis(config.redisUrl);
const workerConnection = new IORedis(config.redisUrl, {
    // Worker 的阻塞连接必须关闭单请求最大重试限制。
    maxRetriesPerRequest: null,
});

export const emailQueue = new Queue<WelcomeEmailJobData>("email", {
    connection: producerConnection,
    defaultJobOptions: {
        attempts: 3,
        backoff: { type: "exponential", delay: 1000 },
        timeout: 30_000,
        removeOnComplete: { age: 3600, count: 1000 },
        removeOnFail: { age: 7 * 24 * 3600 },
    },
});

export const emailWorker = new Worker<WelcomeEmailJobData>(
    "email",
    async (job) => {
        if (job.data.version !== 1) {
            throw new Error("unsupported welcome email job version");
        }
        await emailService.sendWelcome(job.data.userId);
    },
    { connection: workerConnection, concurrency: config.queue.concurrency },
);

export async function enqueueWelcomeEmail(userId: string): Promise<string> {
    const job = await emailQueue.add("welcome", { version: 1, userId });
    if (!job.id) throw new Error("BullMQ returned a job without id");
    return job.id;
}

export async function closeEmailQueue(): Promise<void> {
    // 先停止 Worker 接收新任务并等待当前任务，再关闭生产端。
    await emailWorker.close();
    await emailQueue.close();
    // 外部创建并传入的 ioredis 连接由应用自己关闭。
    await Promise.all([workerConnection.quit(), producerConnection.quit()]);
}
```

关键说明：

- Queue 与 Worker 的队列名必须完全一致，payload 类型和 `version` 由生产者、消费者共同维护。
- 不传连接时 BullMQ 默认连接 `localhost:6379`；项目应显式注入配置，避免容器或生产环境误连本机。
- Worker 所需 `maxRetriesPerRequest: null` 不能机械复制到所有 Redis 用途。普通 Redis client 和生产者的重试上限按各自可靠性策略配置。
- 示例分别持有生产与 Worker 连接，便于说明所有权。若按官方连接复用能力共享 client，也必须验证阻塞连接和各角色选项，关闭责任仍由创建者承担。
- `concurrency` 是单 Worker 的并发，不等于全系统吞吐承诺。总并发要受邮件商、数据库和下游限流约束。

## 13.5 PostgreSQL Quick Start：迁移、生产者与 Worker

PostgreSQL 后端不会在普通连接时自动执行迁移。启动 Queue/Worker 前，在一个受控部署步骤执行：

```ts
import { Pool } from "pg";
import { runMigrations } from "bullmq";

const pool = new Pool({ connectionString: config.postgresUrl });
const client = await pool.connect();
try {
    await runMigrations(client);
} finally {
    client.release();
    await pool.end();
}
```

`runMigrations` 幂等并使用 advisory lock 防止并发初始化冲突，但仍建议由迁移 job 统一执行，不让每个 API 副本自行决定升级。BullMQ schema 默认位于 `bullmq` schema；官方不支持 schema downgrade，升级和回滚必须按所用版本制定兼容方案。

迁移成功后，用 `withBackend(createPostgresBackend)` 保留 job 泛型并自动选择 PostgreSQL：

```ts
import { createPostgresBackend, withBackend } from "bullmq";

const { Queue, Worker } = withBackend(createPostgresBackend);
const opts = {
    connection: {
        connectionString: config.postgresUrl,
        max: 10,
    },
};

interface WelcomeEmailJobData {
    version: 1;
    userId: string;
}

const emailQueue = new Queue<WelcomeEmailJobData>("email", {
    ...opts,
    defaultJobOptions: {
        attempts: 3,
        backoff: { type: "exponential", delay: 1000 },
        removeOnComplete: { age: 3600, count: 1000 },
        removeOnFail: { age: 7 * 24 * 3600 },
    },
});

const emailWorker = new Worker<WelcomeEmailJobData>(
    "email",
    async (job) => emailService.sendWelcome(job.data.userId),
    { ...opts, concurrency: config.queue.concurrency },
);

export async function closeEmailQueue(): Promise<void> {
    // 与 Redis 后端相同，shutdown 时先 close Worker，再 close Queue。
    await emailWorker.close();
    await emailQueue.close();
}
```

也可以不用 `withBackend`，把 `createPostgresBackend` 作为最后一个构造参数传入。对带 job 泛型的 TypeScript 代码，优先使用上面的 `withBackend`，避免手动填写冗长的 backend 类型参数。

连接与生命周期注意事项：

- `connection` 可以是连接字符串、`pg.PoolConfig` 或已有 `pg.Pool`。
- BullMQ 根据字符串/config 自己创建 Pool 时，会随 `close()` 关闭；注入已有 Pool 时，Pool 由应用关闭。
- 默认 schema 为 `bullmq`。自定义 schema 时，优先让 BullMQ 根据 config 建 Pool；若注入已有 Pool，必须自行正确配置 `search_path`。
- 每个 Worker 除普通 Pool 查询外还需要专用 `LISTEN` 连接。容量预算必须包含所有 Queue、Worker、QueueEvents、API/迁移连接以及专用 LISTEN 连接。
- PostgreSQL 后端不使用 ioredis，因此不要给它设置 `maxRetriesPerRequest`。

## 13.6 重试、幂等与失败处理

`attempts` 只表示失败后的重试策略，不代表一定成功，也不代表只执行一次。Worker 崩溃、超时或结果确认丢失都可能造成再次执行。

每个 job processor 必须：

1. 用 `job.id`、业务操作 ID 或独立幂等键建立消费记录；关键唯一性最终由数据库约束保证。
2. 将“检查是否完成”和“完成业务写入”放在可行的同一数据库事务中，不能只依赖先查后写。
3. 对外部 API 使用对方支持的 idempotency key；没有该能力时设计补偿、对账或人工恢复。
4. 对明确不可重试的数据/权限错误使用当前 BullMQ 版本提供的不可重试机制，不让配置错误无意义重试。
5. 对临时网络、限流和不可用错误抛出异常交给 BullMQ；不要 catch 后吞掉，否则队列会认为成功。
6. 最终失败进入可观测和恢复流程，包括告警、人工重放、补偿或丢弃审批。`removeOnFail` 的保留时间要覆盖排查窗口。

日志只记录 jobId、队列名、attempt、耗时和稳定错误码，不默认记录完整 payload、邮件地址、token 或个人数据。`failed` 事件可能在每次失败尝试时触发，不能把它一律当成最终失败。

## 13.7 应用装配与优雅关闭

建议将 API producer 与 Worker 拆成可独立扩缩容的进程；小型项目可以同进程，但必须共享同一套生命周期纪律。

启动顺序：

1. 校验配置并创建 logger。
2. PostgreSQL 后端先执行受控迁移；Redis 后端确认连接和部署策略。
3. 创建业务 Repository/Service，再创建 Queue 和 Worker。
4. readiness 只有在当前进程必需资源可用后才成功。

收到 SIGTERM/SIGINT：

1. readiness 失败，停止新的 HTTP 流量和新任务提交。
2. `await worker.close()`，停止拉取新任务并按当前 BullMQ 语义等待进行中任务。
3. 关闭 Queue、可选 QueueEvents 和其他 producer。
4. 关闭应用拥有的 ioredis/pg Pool，再关闭数据库和 logger。
5. 全部关闭纳入总 shutdown timeout；超时有日志和明确退出策略。

不要在每个 Queue/Worker 模块注册 process 信号。关闭函数应幂等，单个资源关闭失败仍继续释放其余资源，并最终聚合错误。

## 13.8 测试与生产验收

单元测试：

- 应用服务通过 fake queue 验证入队时机、job name、payload version 和业务 ID，不断言 Redis/SQL 行为。
- processor 覆盖成功、可重试失败、不可重试失败、重复投递和 shutdown 中的分支。
- 配置覆盖 backend 切换、缺失 URL、非法并发和秘密脱敏。

真实后端集成测试：

- Redis 与 PostgreSQL 按实际启用范围分别测试，不用 mock 替代队列语义。
- 每个套件使用独立 database/schema/前缀和临时资源，禁止清空开发队列。
- 验证入队到完成、延迟/重试、重复消费幂等、最终失败、保留策略和优雅关闭。
- PostgreSQL 额外验证 `runMigrations`、schema/search_path、Pool 所有权和 LISTEN 连接释放。
- 发布前验证 producer/consumer 版本兼容、滚动发布、Worker 重启和任务恢复。

- [ ] 后端选型及原因写入 ADR，所有实例配置一致。
- [ ] 任务 payload 有类型、版本和大小限制，无秘密和不必要个人数据。
- [ ] 重试有上限和退避，业务副作用幂等，最终失败可恢复。
- [ ] 完成/失败任务有保留上限，不无限占用 Redis 内存或 PostgreSQL 存储。
- [ ] 队列深度、最老等待时间、执行耗时、失败/重试和最终失败有指标与告警。
- [ ] Worker 并发、后端连接和下游限额经过容量验证。
- [ ] 关闭、崩溃恢复和真实后端集成测试通过，未执行项明确记录。

上一篇：[Redis 原生能力](08-redis.md) | [返回总目录](../development-guide.md) | 下一篇：[应用生命周期](10-application-lifecycle.md)
