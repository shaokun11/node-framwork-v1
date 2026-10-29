# MongoDB + Mongoose

## 10. 主数据库方案 B：MongoDB + Mongoose

### 10.1 适用条件

优先用于自然文档聚合、读取模型围绕文档构建、结构随业务演进且不需要关系数据库约束的场景。不要仅因MongoDB 不需要 schema而选择它；Mongoose Schema、唯一索引、验证和迁移仍是必须的工程契约。

MongoDB 事务需要满足部署拓扑要求，单节点开发和生产副本集行为不同。能用一个原子文档更新表达的不变式，不要强行拆成跨集合事务。

### 10.2 Schema 与 Model

- Schema/Model 放在 infrastructure，Service 不直接 import Mongoose Model。
- 明确字段类型、required、default、enum、索引、timestamps 和 collection 名称。
- `strict` 策略默认拒绝或显式处理未知字段，避免客户端额外字段静默进入文档。
- 写入更新按需启用 `runValidators`，知道 Mongoose 验证与 MongoDB 服务端验证并不等同。
- 默认 `_id` 使用 ObjectId 时，在 API 边界验证合法格式并映射为字符串；自定义 ID 也必须保证唯一和稳定。
- 使用 `lean()` 返回普通对象时明确其没有 Document methods、getters 和完整 Mongoose 行为。
- 不用 Mongoose middleware 隐藏核心业务规则；hook 只处理持久化层面且可发现的行为。
- 生产默认 `autoIndex = false`，索引通过受控迁移/运维流程创建。

```ts
const orderSchema = new Schema(
    {
        _id: { type: String, required: true },
        ownerId: { type: String, required: true, index: true },
        status: { type: String, enum: ["pending", "paid", "cancelled"], required: true },
        items: { type: [orderItemSchema], default: [] },
    },
    { timestamps: true, strict: "throw", collection: "orders", autoIndex: false },
);
```

### 10.3 查询、并发与事务

- 查询必须有过滤条件和结果上限；防止无界数组、无界 populate 和返回大型文档。
- 唯一性由 MongoDB 唯一索引最终保证，将 duplicate key 映射为领域冲突。
- 更新采用明确条件和原子操作，避免读改写覆盖其他请求；需要时使用 `__v` 或单独版本字段实现乐观并发。
- 多文档事务显式创建 ClientSession，把同一 session 传给所有操作，并在 finally 结束 session。
- 事务内不做无超时网络调用；处理 TransientTransactionError 等重试时必须保证整个用例幂等，重试上限明确。
- Aggregation pipeline 和字段投影属于 Repository，禁止把未经校验的用户输入作为 operator 或任意 pipeline 注入。

### 10.4 文档演进

Mongoose 不会自动完成业务数据迁移。字段重命名、拆分、嵌入改引用和格式变化需要版本化脚本、回填、验证与回滚方案。读取旧文档时可做显式版本转换，但不能无限累积隐式兼容分支。

### 10.5 两种主数据库的选型记录

| 问题 | PostgreSQL + Drizzle | MongoDB + Mongoose |
| --- | --- | --- |
| 核心模型 | 表、关系、规范化与约束 | 文档、嵌入/引用与访问模式 |
| 一致性 | 强事务、约束和关系完整性优先 | 单文档原子性优先，多文档事务按需 |
| 查询 | Join、SQL、报表和关系查询 | 文档读取、聚合管道 |
| 结构演进 | 版本化 SQL 迁移 | Schema + 数据迁移 + 文档版本 |
| 默认事实源 | 当前模块选择后为 PostgreSQL | 当前模块选择后为 MongoDB |

选型 ADR 至少记录访问模式、事务范围、数据量、索引、一致性、备份恢复、团队经验和迁移成本。禁止为以后可能换数据库设计一个最低能力通用 ORM 层；Repository 隔离的是业务契约，不是抹平数据库差异。

## 10.6 文档建模：嵌入还是引用

按访问模式和一致性选择，不按字段看起来相关决定：

| 条件 | 更适合嵌入 | 更适合引用 |
| --- | --- | --- |
| 生命周期 | 子项随父文档创建和删除 | 子项独立存在 |
| 一致性 | 需要单文档原子更新 | 可接受跨文档协调 |
| 大小 | 数量和大小有明确上限 | 可能持续增长 |
| 查询 | 通常随父文档一起读取 | 经常独立查询或被多方共享 |
| 更新 | 整体更新为主 | 高频独立更新 |

订单快照可以嵌入已购商品名称和价格，避免商品后续改价影响历史订单；用户收藏等无界列表不应无限嵌入一个文档。无论哪种方式，都要评估 16MB 文档限制、工作集和写放大。

## 10.7 Mongoose Repository 实现规范

- 在 Model 上定义明确查询方法或 Repository 私有辅助，不把长链查询散布到 Service。
- `findOne` 不存在返回 null，更新不存在也必须单独处理。
- 只返回业务需要字段，使用 projection 限制大字段。
- 对 Mongoose cast error、duplicate key、version error 和 validation error 分类映射。
- `toObject()`/`lean()` 后显式移除 `__v`、密码哈希和内部字段；virtuals/depopulate 行为要一致。
- 分页按稳定排序加 `_id` tie-breaker，排序字段有对应索引。

```ts
const document = await this.model
    .findOne({ _id: id, ownerId })
    .select({ _id: 1, ownerId: 1, status: 1, items: 1, createdAt: 1 })
    .lean()
    .exec();
return document ? toDomainOrder(document) : null;
```

## 10.8 查询安全与更新契约

不要把客户端对象直接传给 `find(req.query)` 或 `updateOne(req.body)`。先由 Zod 白名单解析，再由 Repository 构造固定 operator。

- 查询键和操作符由服务端控制，防止 `$where`、`$ne` 等操作符注入改变查询语义。
- 更新区分 `$set`、`$unset`、`$inc` 和数组操作，避免替换整个文档造成字段丢失。
- `findOneAndUpdate` 明确返回更新前还是更新后文档，并检查 upsert 是否会意外创建资源。
- 多条件更新包含 ownerId/tenantId 和状态/版本条件。
- 数组字段使用有界 push/slice 等策略，避免无限增长。

## 10.9 Session 使用与重试

```ts
const session = await connection.startSession();
try {
    await session.withTransaction(async () => {
        await orders.create([newOrder], { session });
        await inventories.updateOne(reservationFilter, reservationUpdate, { session });
    });
} finally {
    await session.endSession();
}
```

`withTransaction` 可能按驱动规则重试 callback，因此 callback 内不能发送不可撤销邮件、支付请求或其他非幂等外部副作用。需要这类动作时，在事务后通过可靠任务执行。

开发环境若要测试事务，Compose 必须启动支持事务的 replica set，而不是普通 standalone MongoDB。

## 10.10 Mongoose 评审清单

- [ ] 文档大小、嵌入数组和查询访问模式有边界。
- [ ] Schema 验证、strict、索引和生产 autoIndex 配置明确。
- [ ] 查询/更新由白名单构造，无操作符注入。
- [ ] 唯一冲突、版本冲突和 null 映射有真实数据库测试。
- [ ] 多文档事务使用同一 session，拓扑和重试副作用明确。
- [ ] 文档演进有版本、回填和回滚方案。


## 10.11 Mongoose 安装、连接与 Model

安装：

```sh
npm install mongoose
```

推荐由基础设施创建独立 Connection，而不是在业务模块使用全局 mongoose 连接：

```ts
import mongoose from "mongoose";

export async function createMongoConnection(config: MongoConfig) {
    const connection = mongoose.createConnection(config.url, {
        autoIndex: false,
        serverSelectionTimeoutMS: config.connectTimeoutMs,
        maxPoolSize: config.maxPoolSize,
    });

    connection.on("error", (error) => {
        logger.error("mongodb connection error", { error });
    });

    await connection.asPromise();
    return connection;
}
```

关闭时由创建者调用 `await connection.close()`。不要在每次请求 createConnection，也不要让 Repository 关闭共享 Connection。

定义 Schema 和 Model：

```ts
import { Schema, type InferSchemaType } from "mongoose";

const orderItemSchema = new Schema(
    {
        sku: { type: String, required: true, trim: true, maxlength: 64 },
        quantity: { type: Number, required: true, min: 1, max: 1000 },
    },
    { _id: false },
);

export const orderSchema = new Schema(
    {
        _id: { type: String, required: true },
        ownerId: { type: String, required: true },
        idempotencyKey: { type: String, required: true },
        status: {
            type: String,
            enum: ["pending", "paid", "cancelled"],
            required: true,
        },
        items: { type: [orderItemSchema], default: [] },
    },
    {
        collection: "orders",
        timestamps: true,
        strict: "throw",
        autoIndex: false,
        optimisticConcurrency: true,
    },
);

orderSchema.index({ ownerId: 1, createdAt: -1 });
orderSchema.index({ ownerId: 1, idempotencyKey: 1 }, { unique: true });

export type OrderDocument = InferSchemaType<typeof orderSchema>;
```

把 Model 绑定到指定 Connection：

```ts
const OrderModel = connection.model("Order", orderSchema);
```

生产索引通过迁移创建。开发临时开启 autoIndex 也不能代替共享环境的索引发布流程。

## 10.12 Mongoose CRUD 使用

### 创建

```ts
const document = await OrderModel.create({
    _id: order.id,
    ownerId: order.ownerId,
    idempotencyKey: order.idempotencyKey,
    status: order.status,
    items: order.items,
});

return toDomainOrder(document.toObject({ versionKey: false }));
```

`create()` 会执行 Mongoose 验证。重复唯一索引要捕获 MongoServerError 风格的 code 11000，并映射为领域冲突。

### 查询

```ts
const document = await OrderModel.findOne({
    _id: orderId,
    ownerId,
})
    .select({ _id: 1, ownerId: 1, status: 1, items: 1, createdAt: 1 })
    .lean()
    .exec();

const orders = await OrderModel.find({ ownerId })
    .sort({ createdAt: -1, _id: -1 })
    .limit(20)
    .lean()
    .exec();
```

`lean()` 返回普通对象，适合只读 Repository；没有 Document save、methods 和部分 Mongoose 行为。需要修改并 save 时不要 lean。

### 更新

```ts
const updated = await OrderModel.findOneAndUpdate(
    { _id: orderId, ownerId, status: "pending" },
    { $set: { status: "paid" } },
    { new: true, runValidators: true },
).exec();

if (!updated) {
    // 区分不存在和状态冲突。
}
```

- `new: true` 返回更新后的文档。
- `runValidators: true` 对更新执行支持的 Mongoose validators。
- filter 中的 status 保证状态转换原子性。
- 不直接传 req.body，Repository 白名单选择 `$set`、`$unset`、`$inc`。

### 删除

```ts
const result = await OrderModel.deleteOne({ _id: orderId, ownerId }).exec();
if (result.deletedCount === 0) return null;
```

删除前检查业务状态。软删除使用条件 update，并在所有查询中统一过滤。

## 10.13 验证、类型与 Document 映射

```ts
orderSchema.path("items").validate((items: unknown[]) => {
    return items.length > 0 && items.length <= 100;
}, "order must contain 1-100 items");
```

Mongoose 验证是最后持久化防线之一，API 仍需 Zod 验证。Mongoose cast error、ValidationError、duplicate key 和 VersionError 是不同类别，Repository 映射时保留 cause 但对外不泄漏。

Document 到领域对象使用显式 mapper：

```ts
function toDomainOrder(document: OrderPersistenceShape): Order {
    return {
        id: document._id,
        ownerId: document.ownerId,
        status: document.status,
        items: document.items.map((item) => ({
            sku: item.sku,
            quantity: item.quantity,
        })),
        createdAt: document.createdAt,
    };
}
```

不要直接返回 Mongoose Document，也不要用 JSON.parse(JSON.stringify(document)) 代替映射。

## 10.14 Mongoose 多文档事务使用

```ts
const session = await connection.startSession();
try {
    await session.withTransaction(async () => {
        await OrderModel.create([orderRow], { session });
        await InventoryModel.updateOne(
            { sku, available: { $gte: quantity } },
            { $inc: { available: -quantity } },
            { session, runValidators: true },
        );
    });
} finally {
    await session.endSession();
}
```

- 每个参与操作都传同一个 session。
- callback 可能重试，不能包含不可撤销外部副作用。
- 检查条件更新 matchedCount，防止库存条件不满足却仍提交订单。
- 测试环境必须是 replica set 或支持事务的部署。
- 单文档可完成的原子更新不需要事务。

## 10.15 Mongoose 常见问题

- Model 编译重复：不要在热更新或请求中重复 connection.model，统一在装配阶段创建。
- 更新验证没执行：检查 runValidators 和当前 Mongoose 对 update validators 的支持范围。
- autoIndex 导致启动慢：生产关闭并用迁移管理索引。
- lean 后方法不存在：这是预期行为，只读使用 lean，写 Document 时显式选择。
- 文档无限增长：检查嵌入数组，改为有界数组或独立集合。
- 查询被注入 operator：不要直接传 req.query/req.body，使用 Zod + Repository 白名单构造。
- 事务报拓扑不支持：本地 standalone 不支持，启动 replica set 测试环境。

上一篇：[PostgreSQL + Drizzle](05-postgresql-drizzle.md) | [返回总目录](../development-guide.md) | 下一篇：[Keyv 缓存](07-keyv-cache.md)
