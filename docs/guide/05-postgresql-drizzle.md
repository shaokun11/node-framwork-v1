# PostgreSQL + Drizzle ORM

## 9. 主数据库方案 A：PostgreSQL + Drizzle ORM

### 9.1 适用条件

优先用于关系清晰、需要唯一/外键/检查约束、复杂查询、Join、强事务和精确报表的业务。连接由 `pg.Pool` 管理，Drizzle 使用 `node-postgres` 适配；Repository 接收调用方创建的 Drizzle 实例，不自行建立 Pool。

### 9.2 Schema 与类型

- Table 定义放在基础设施层，按聚合或清晰边界拆分，不在 Controller 中声明。
- 主键、唯一约束、外键、检查约束、索引和删除行为必须在迁移中可审计。
- 金额使用定点 decimal 对应的安全领域类型，不能用 JavaScript 浮点做财务运算。
- PostgreSQL `bigint`、numeric 和 timestamp 的 JS 映射必须显式规定，避免精度或时区隐式转换。
- 时间统一保存 UTC，API 使用带时区的 ISO 字符串。
-  nullable、undefined 和空字符串具有不同含义，更新时区分字段未提供和清空字段。

```ts
export const orders = pgTable(
    "orders",
    {
        id: text("id").primaryKey(),
        ownerId: text("owner_id").notNull(),
        status: text("status").notNull(),
        createdAt: timestamp("created_at", { withTimezone: true }).notNull(),
    },
    (table) => [index("orders_owner_created_idx").on(table.ownerId, table.createdAt)],
);
```

Table inference 类型仅在持久化层使用。数据库行映射为领域对象后才能离开 Repository。

### 9.3 查询与事务

- 使用 Drizzle 参数化构建查询；必须写原始 SQL 时同样绑定参数，不拼接用户输入。
- 列表必须限制数量并设计稳定排序；深分页优先 keyset/cursor。
- 事务通过 `db.transaction(async (tx) => ...)` 明确边界，事务内所有 Repository 操作必须使用同一个 `tx`。
- 不在事务中执行无超时外部 HTTP、长时间 Worker 或缓存等待。
- 唯一冲突按 PostgreSQL code 和 constraint 映射为领域冲突；不要只靠预先 `SELECT`。
- 需要串行化竞争时选择行锁、原子 UPDATE、适当隔离级别或乐观版本，不用 JS Promise 锁冒充数据库并发控制。

```ts
await db.transaction(async (tx) => {
    const order = await tx.insert(orders).values(newOrder).returning();
    await tx.insert(orderItems).values(toItemRows(order));
});
```

跨 Repository 事务通过 Unit of Work 或将 tx 限定在基础设施用例实现中传递，不允许 Service 获取全局 db 后随意切换连接。

### 9.4 迁移与发布

- 使用 `drizzle-kit` 根据已评审 schema 生成迁移，人工检查生成 SQL 后提交。
- `drizzle-kit push` 只可用于可丢弃的本地实验，不能作为共享环境和生产发布流程。
- 迁移由独立命令或部署步骤执行，API 启动不能自动修改生产结构。
- 迁移文件不可在应用后改写；修正通过新迁移完成。
- 滚动发布先 expand，再发布兼容代码，最后 contract。旧代码和新代码必须能在迁移窗口共同工作。
- 大表加索引、改类型、回填和删除列需要评估锁、耗时和回滚，不把小库经验直接套到生产。
- 就绪检查不能替代 schema 版本兼容性检查。

## 9.5 连接池与启动检查

Pool 由 infrastructure factory 创建，至少配置 max、connection timeout、idle timeout 和应用级 shutdown timeout。容量预算按API 实例数  每实例 max + worker/脚本连接计算，不能只看单进程运行正常。

启动时执行轻量连接验证，确认凭据、网络和数据库可用；不要把全表查询当健康检查。Pool 的 error event 必须有日志处理，避免未监听事件导致进程异常。

读查询可使用只读账号或只读连接，但事务和权限边界必须清楚。TLS、server name 和证书验证按部署环境配置，production 不为方便关闭证书校验。

## 9.6 Repository 查询实现规范

```ts
export class DrizzleOrderRepository implements OrderRepository {
    constructor(private readonly db: OrderDatabase) {}

    async findById(id: string, ownerId: string): Promise<Order | null> {
        const rows = await this.db
            .select()
            .from(orders)
            .where(and(eq(orders.id, id), eq(orders.ownerId, ownerId)))
            .limit(1);
        return rows[0] ? toDomainOrder(rows[0]) : null;
    }
}
```

- `limit(1)` 明确查询上限，不能用 `rows[0]!` 隐藏不存在。
- 资源归属放进查询条件，避免先读取再泄露其他租户数据。
- 更新同时带状态/版本条件，并检查 affected rows；0 行需要区分不存在和冲突。
- 查询返回所需列，宽表和大型 JSON 不默认 select 全部。
- 批量写入分批并限制参数数量；批量读取注意 IN 列表大小和结果顺序。

## 9.7 并发与锁选择

| 场景 | 首选机制 | 注意事项 |
| --- | --- | --- |
| 防止重复账号/幂等键 | 唯一约束 + 冲突映射 | 预查询只作优化 |
| 余额/库存扣减 | 带条件的原子 UPDATE | 条件必须包含余额/库存下限 |
| 短事务内读取后修改 | SELECT FOR UPDATE | 控制锁顺序和事务时间 |
| 可重试的离线编辑 | version 乐观锁 | 0 affected rows 返回冲突 |
| 多表强一致写入 | 数据库事务 | 所有语句使用同一 tx |
| 跨实例定时抢占 | 数据库抢占记录/建议锁等明确方案 | 要处理持锁者崩溃和任务幂等 |

事务因 serialization failure 或 deadlock 重试时，必须重试整个事务函数并设置次数/退避上限。非幂等外部副作用不能放在可重试事务里。

## 9.8 数据迁移实施步骤

1. 描述线上数据量、查询方式和锁风险。
2. 新增结构保持旧代码可运行，例如新列先 nullable 或有安全默认值。
3. 分批回填，设置批次大小、暂停、进度和幂等。
4. 发布读写兼容代码并验证新旧数据。
5. 回填后核对数量、抽样和业务不变量。
6. 在独立变更中增加 NOT NULL、外键或唯一约束。
7. 等旧版本完全退出后再删除旧列和兼容代码。

大型表加约束可评估分批/在线方式；不能直接在高峰期执行长时间排他锁。回滚优先向前修复，备份恢复演练要有真实时间和数据量记录。

## 9.9 Drizzle 评审清单

- [ ] Schema、迁移和数据库实际结构一致，生成 SQL 经过人工评审。
- [ ] 金额、bigint、timestamp 和 null 的映射有测试。
- [ ] 所有写事务使用同一 tx，失败会回滚。
- [ ] 唯一约束、条件更新和并发冲突经过真实数据库测试。
- [ ] 查询有索引和结果上限，没有 N+1 和无界批量。
- [ ] API 启动不自动执行生产迁移，回滚方案明确。


## 9.10 Drizzle 安装、连接与 Schema

安装：

```sh
npm install drizzle-orm pg
npm install --save-dev drizzle-kit @types/pg
```

定义 Schema：

```ts
import { index, pgTable, text, timestamp, uniqueIndex } from "drizzle-orm/pg-core";

export const orders = pgTable(
    "orders",
    {
        id: text("id").primaryKey(),
        ownerId: text("owner_id").notNull(),
        status: text("status", { enum: ["pending", "paid", "cancelled"] }).notNull(),
        idempotencyKey: text("idempotency_key").notNull(),
        createdAt: timestamp("created_at", { withTimezone: true, mode: "date" })
            .notNull()
            .defaultNow(),
    },
    (table) => [
        uniqueIndex("orders_owner_idempotency_uq").on(table.ownerId, table.idempotencyKey),
        index("orders_owner_created_idx").on(table.ownerId, table.createdAt),
    ],
);

export type OrderRow = typeof orders.$inferSelect;
export type NewOrderRow = typeof orders.$inferInsert;
```

`OrderRow` 和 `NewOrderRow` 仅在 infrastructure 使用。对外领域对象通过显式 mapper 创建。

创建 Pool 和 Drizzle：

```ts
import { Pool } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema.ts";

export function createOrderDatabase(config: PostgresConfig) {
    const pool = new Pool({
        connectionString: config.url,
        max: config.poolMax,
        connectionTimeoutMillis: config.connectionTimeoutMs,
        idleTimeoutMillis: config.idleTimeoutMs,
        ssl: config.ssl,
    });

    pool.on("error", (error) => {
        logger.error("postgres idle client error", { error });
    });

    return {
        db: drizzle(pool, { schema }),
        close: () => pool.end(),
    };
}
```

应用关闭时调用 `pool.end()`，Repository 不负责关闭 Pool。不要在每次请求创建 new Pool。

## 9.11 Drizzle CRUD 使用

### 查询单条与列表

```ts
import { and, desc, eq, gt } from "drizzle-orm";

const row = await db.query.orders.findFirst({
    where: and(eq(orders.id, orderId), eq(orders.ownerId, ownerId)),
});

const rows = await db
    .select()
    .from(orders)
    .where(and(eq(orders.ownerId, ownerId), gt(orders.createdAt, after)))
    .orderBy(desc(orders.createdAt), desc(orders.id))
    .limit(20);
```

`findFirst` 不存在时按当前返回类型处理 undefined，不做非空断言。列表必须有 limit 和稳定排序。

### 插入

```ts
const inserted = await db
    .insert(orders)
    .values({
        id: order.id,
        ownerId: order.ownerId,
        status: order.status,
        idempotencyKey: order.idempotencyKey,
        createdAt: order.createdAt,
    })
    .returning();

const row = inserted[0];
if (!row) throw new Error("insert returned no row");
```

批量 values 需要分批，避免参数上限和超大事务。插入冲突使用 onConflictDoNothing/onConflictDoUpdate 时，要明确返回行数和冲突语义。

### 条件更新

```ts
const updated = await db
    .update(orders)
    .set({ status: "paid" })
    .where(and(eq(orders.id, orderId), eq(orders.status, "pending")))
    .returning();

if (!updated[0]) {
    // 区分订单不存在与状态冲突；必要时再查询。
}
```

条件更新把只有 pending 才能支付放进数据库原子操作，避免先查后改的竞争。

### 删除

```ts
const deleted = await db
    .delete(orders)
    .where(and(eq(orders.id, orderId), eq(orders.ownerId, ownerId)))
    .returning({ id: orders.id });
```

业务删除需要权限、归属和状态条件。软删除则使用条件 UPDATE，并明确唯一索引和查询过滤。

## 9.12 Drizzle 事务与错误映射

```ts
const created = await db.transaction(async (tx) => {
    const orderRows = await tx.insert(orders).values(orderRow).returning();
    const order = orderRows[0];
    if (!order) throw new Error("order insert returned no row");

    await tx.insert(orderItems).values(toItemRows(order, items));
    return order;
});
```

事务 callback 抛错会回滚。所有语句必须使用 tx，混用 db 会使部分语句脱离事务。事务外再执行缓存删除、消息发送和 Winston 业务摘要。

PostgreSQL 错误通过类型守卫读取 code：

```ts
function isPostgresError(error: unknown): error is { code: string; constraint?: string } {
    return typeof error === "object" && error !== null && "code" in error;
}

try {
    await insertOrder();
} catch (error) {
    if (
        isPostgresError(error) &&
        error.code === "23505" &&
        error.constraint === "orders_owner_idempotency_uq"
    ) {
        throw new OrderIdempotencyConflict({ cause: error });
    }
    throw error;
}
```

不要只比较 error.message，数据库驱动升级和消息语言可能变化。

## 9.13 drizzle-kit 配置与迁移命令

`drizzle.config.ts`：

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
    dialect: "postgresql",
    schema: "./src/infrastructure/database/postgres/schema.ts",
    out: "./drizzle",
    dbCredentials: {
        url: process.env.APP_DATABASE_URL ?? "",
    },
    strict: true,
    verbose: true,
});
```

CLI 是配置边界的例外，可以读取进程环境；应用业务代码仍通过 ConfigLoader。不能在仓库中硬编码真实 URL。

常用命令：

```sh
# 根据 schema 生成迁移，生成后必须审查 SQL
npx drizzle-kit generate

# 对当前配置数据库执行已生成迁移
npx drizzle-kit migrate

# 本地检查 schema/生成物；具体子命令以当前 drizzle-kit 版本为准
npx drizzle-kit check
```

`generate` 不等于已应用迁移。共享开发、测试和生产环境使用同一套已提交迁移文件；不用 `drizzle-kit push` 替代正式迁移。

## 9.14 Drizzle 使用常见问题

- 查询类型正确但数据库报错：先检查迁移是否应用、连接的是否预期数据库。
- returning 数组为空：检查冲突分支和 affected rows，不要直接 `[0]!`。
- 事务没有回滚全部语句：检查是否误用了 db 而不是 tx。
- timestamp 时区偏移：检查 mode、withTimezone、Node 时区和 API 序列化。
- numeric 变成字符串：这是避免精度丢失的常见驱动行为，用领域 Decimal 处理，不直接 Number。
- 测试通过但生产重复数据：检查唯一约束是否真实存在，InMemory fake 不能证明。

上一篇：[配置加载](04-configuration.md) | [返回总目录](../development-guide.md) | 下一篇：[MongoDB + Mongoose](06-mongodb-mongoose.md)
