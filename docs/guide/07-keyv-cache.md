# Keyv 缓存与 Adapter

## 11. Keyv 缓存抽象

### 11.1 独立职责

Keyv 用于统一的 Promise KV API、TTL、namespace 和 Adapter 替换。它适合读取缓存、短期计算结果、可重建状态和明确不需要 Redis 特有命令的 KV 数据。

Keyv 不是主数据库 ORM，也不是分布式锁、可靠消息队列或强一致事务协议。默认缓存丢失后可以重建；若数据不可丢失，必须回到主数据库或明确设计的持久化方案。

### 11.2 Adapter 选择

Keyv 默认内存存储，也可按需求安装官方或受信第三方 Adapter。常见选择包括：

| Adapter | 适用场景 | 必查边界 |
| --- | --- | --- |
| 默认内存 | 单进程开发、测试、可丢失本地缓存 | 多实例不共享，重启丢失，受进程内存限制 |
| `@keyv/redis` | 多实例共享缓存、原生 TTL | Redis 可用性、序列化、namespace、Cluster/TLS |
| `@keyv/postgres` | 已有 PostgreSQL 且希望减少基础设施 | 不假定有 Redis 原生 TTL/性能，清理与索引 |
| `@keyv/mongo` | 已有 MongoDB 且希望复用连接平台 | TTL 索引、文档大小、主业务负载隔离 |
| `@keyv/sqlite` | 单实例本地持久缓存 | 文件位置、单写者、容器持久卷与多实例限制 |
| 其他官方/第三方 Adapter | 需求明确且团队愿意维护 | 维护状态、版本兼容、TTL/namespace/clear 语义 |

Adapter 是可替换实现，不代表各后端性能、TTL、原子性、持久性和错误行为完全一致。更换 Adapter 必须重跑缓存契约测试，不能只改连接字符串就宣称兼容。

### 11.3 创建与配置

```ts
import Keyv from "keyv";
import KeyvRedis from "@keyv/redis";

const store = new KeyvRedis(config.cache.url, {
    namespace: config.cache.namespace,
});
const cache = new Keyv<OrderSnapshot>({
    store,
    ttl: config.cache.defaultTtlMs,
    throwOnErrors: true,
});
cache.on("error", (error) => logger.error({ error }, "cache operation failed"));
```

示例只说明装配。非 Redis Adapter 在各自工厂创建；业务 Service 不 import Adapter。根据当前 Keyv 大版本核对 namespace 由核心还是 Adapter 负责，避免双重 prefix；不要同时叠加 Redis client `keyPrefix` 和 Adapter namespace 而不检查实际 key。

- 为不同用途建立不同 Keyv 实例和 namespace，避免 `clear()` 影响其他模块。
- key 包含环境、模块、实体和版本，例如 `prod:order:v2:<id>`；不包含未经规范化的秘密或敏感原文。
- TTL 统一毫秒配置，每个用途有明确上限；重要缓存增加抖动，避免同时过期。
- 明确 `throwOnErrors` 和 error event 策略，不能出现有错误监听器就当缓存成功。
- 自定义 serializer 前验证 Date、BigInt、undefined、Buffer 等类型及跨版本兼容，不随意更换生产序列化格式。

### 11.4 Cache-Aside 与失效

```text
读取：get -> hit 后解析/校验 -> miss 时读取主数据库 -> set TTL -> 返回
写入：主数据库事务成功 -> 删除或更新对应缓存 -> 返回
```

先提交事实数据源，再处理缓存。缓存失败后的响应策略按业务决定：普通读取可回源并记录降级；依赖缓存完成的写入必须明确返回失败还是接受短暂不一致。

- 更新优先 delete，避免并发写覆盖为新值后又写入旧缓存。
- 允许短暂旧值时记录最大 staleness；不能接受旧值的数据不要只靠 TTL。
- 防空值 TTL 应较短，避免掩盖刚创建的数据。
- 热点重建可使用 singleflight/请求合并；跨实例重建风暴另评分布式协调。
- 缓存值解析失败按 miss 处理并删除损坏项，记录指标，不直接信任 `as T`。

## 11.5 Cache 接口与 Adapter 工厂

业务层依赖自己需要的缓存能力，不直接依赖完整 Keyv：

```ts
interface OrderSnapshotCache {
    get(orderId: string): Promise<OrderSnapshot | null>;
    set(orderId: string, value: OrderSnapshot, ttlMs: number): Promise<void>;
    delete(orderId: string): Promise<void>;
}
```

`KeyvOrderSnapshotCache` 负责 key 拼接、TTL、序列化后的 Zod 校验和错误映射。`CacheAdapterFactory` 根据已校验配置创建一个 Adapter；不要根据字符串在业务 Service 中 new Adapter。

工厂只允许安装过的 adapter 分支。配置写了未安装 adapter 时启动失败，不能静默退回内存缓存。

## 11.6 Adapter 配置不能强行统一

不同 Adapter 对连接池、TTL、表/集合、namespace 和持久化有不同配置。公共配置只包含 enabled、adapter、defaultTtl 等真正公共项，专属项放在对应子配置：

```toml
[cache]
adapter = "redis"
defaultTtl = "5m"

[cache.redis]
namespace = "order-api:cache"

[cache.postgres]
table = "keyv_cache"

[cache.sqlite]
path = "runtime/cache.sqlite"
```

秘密 URL 仍按配置手册通过环境来源注入。Adapter 的 TTL 是否原生实现、clear 是否扫描、iterator 是否可用和错误是否抛出，必须通过当前版本文档与契约测试确认。

## 11.7 失效竞争与一致性示例

Cache-Aside 的典型竞争：请求 A 读取旧数据库值，请求 B 更新数据库并删除缓存，随后 A 把旧值写回缓存。可选处理包括：

- 接受有界旧值并缩短 TTL。
- 更新后再次延迟删除，但仍需理解不是绝对一致。
- 使用版本化 value/key，只有新版本可写。
- 对强一致数据不使用 Cache-Aside 结果作为最终判断。

任何方案都不能把 Keyv 的 get/set 当成跨实例事务。强一致业务读应回主数据库或使用数据库条件验证。

## 11.8 缓存故障策略表

| 操作 | 故障策略 | 必须记录 |
| --- | --- | --- |
| 普通读取 get | 按 miss 回源，或返回明确依赖错误 | error、fallback、hit/miss |
| 读取后 set | 可降级时继续返回主数据 | set failure，不伪称缓存成功 |
| 写后 delete | 失败会造成旧值 | 告警、重试/补偿、最大旧值窗口 |
| 安全状态读取 | 通常 fail-closed | 拒绝原因与可用性指标 |
| clear | 限制 namespace 和工作量 | 影响范围、耗时和错误 |

是否 fail-open 由业务风险决定。登录状态、权限撤销和配额等安全数据不能沿用普通商品缓存的降级策略。

## 11.9 Keyv 契约测试矩阵

每个实际启用的 Adapter 至少测试：

- set/get/delete、覆盖、不存在和 delete 返回值语义。
- TTL 到期、不过期值、时间边界和不同 TTL 单位。
- namespace 隔离、clear 影响范围、特殊字符 key。
- undefined、null、Date、大对象和项目实际 DTO 的序列化。
- 连接失败、超时、重连和 throwOnErrors/error event。
- 多实例共享（如果设计声称支持）以及重启后的持久性预期。
- 应用关闭时连接释放，无未处理错误事件。

上一篇：[MongoDB + Mongoose](06-mongodb-mongoose.md) | [返回总目录](../development-guide.md) | 下一篇：[Redis 原生能力](08-redis.md)
