# Redis 原生能力

## 12. Redis 原生能力

### 12.1 与 Keyv 分开使用

需要 Redis 数据结构和命令语义时，直接使用官方 `redis` 客户端，封装在专门的基础设施类中：

| 需求 | 使用方式 | 不应如何使用 |
| --- | --- | --- |
| 精确计数、固定窗口限流 | INCR + EXPIRE 或 Lua 原子脚本 | Keyv get/set 的非原子读改写 |
| 滑动窗口限流 | Sorted Set + Lua 等明确算法 | 每实例内存计数宣称全局限流 |
| 分布式锁 | SET NX PX + 唯一 token + 安全释放 | 简单 set/delete，可能释放他人锁 |
| 短期共享状态 | Hash、String、TTL，明确一致性 | 把缓存误当永久事实源 |
| Pub/Sub | 专用连接、订阅生命周期 | 用于必须持久送达的任务 |
| Stream | 消费组、ACK、重试与 pending 处理 | 当作无需 ACK 的普通队列 |
| 排行榜 | Sorted Set | 用 Keyv 数组排序模拟高并发更新 |

`@keyv/redis` 是否满足普通缓存与是否启用独立 Redis client 是两个决策。两者可以共存，但应明确连接、key 前缀、权限和关闭顺序，禁止对同一个 key 同时用不同 serializer 读写。

### 12.2 连接与可靠性

- Redis client 在 bootstrap 创建并连接，Repository/Service 通过窄接口使用。
- 配置 connect/command timeout、重连退避和最大重试；不能无限重连阻塞退出。
- 多实例共享 key 使用环境、应用、模块和用途前缀。
- 每个命令失败都有降级或失败策略；限流存储不可用时是 fail-open 还是 fail-closed 必须由安全需求决定。
- Lua 脚本保持短小、原子、可测试；Cluster 下所有 key 满足 hash slot 约束。
- 大 key、无界集合、无 TTL 数据和 KEYS 全库扫描禁止进入生产；遍历使用受控 SCAN 并限制工作量。
- Pub/Sub、阻塞命令和订阅连接按客户端要求使用独立连接，不影响普通命令。
- Redis 持久化、复制和高可用不是应用代码自动获得的，部署文档必须说明。

### 12.3 锁与后台任务

分布式锁必须设置 TTL、唯一 owner token、原子释放和续约策略；锁过期后任务仍执行可能产生并发，因此关键写入还需数据库条件更新、幂等键或 fencing token。Tinypool 和 Redis 都不自动提供可靠任务语义；需要持久重试、调度和至少一次消费时，选择明确队列方案并设计幂等。

## 12.4 原子限流设计

固定窗口可用一个短 Lua 脚本完成 INCR 和首次设置 EXPIRE，避免两个独立命令之间崩溃造成永不过期 key。滑动窗口可用 Sorted Set 清理过期成员、统计并写入当前成员，整个判断必须原子执行。

设计时明确：

- 配额维度：IP、principal、tenant、route 或组合。
- 可信客户端 IP：只信任配置过的代理，不能直接信任任意 X-Forwarded-For。
- cost：普通请求、登录、上传和昂贵查询是否不同。
- 响应：剩余配额、重置时间和 Retry-After 的协议。
- fail-open/fail-closed：安全敏感登录通常与普通读取不同。
- key TTL 和内存：每个唯一用户/IP 是否可能造成无限 key。

Redis 故障期间如果 fail-open，应记录显著指标和告警；不能悄悄失去全部安全限流。

## 12.5 分布式锁实施步骤

1. 生成每个持有者唯一且不可预测的 token。
2. 使用 `SET key token NX PX ttl` 获取锁。
3. 只在任务明显可能超过 TTL 时实现续约，并监控续约失败。
4. 用 Lua 比较 token 后 DEL，禁止直接 DEL。
5. 关键写入使用 fencing token、数据库条件或幂等约束，防止旧持锁者继续写。
6. 获取失败、锁过期、续约失败和释放失败分别记录指标。

锁不是提升数据库正确性的通用工具。能用唯一约束或原子 UPDATE 解决的，不额外引入锁。

## 12.6 Pub/Sub 与 Stream 的选择

| 需求 | Pub/Sub | Stream |
| --- | --- | --- |
| 在线通知、可丢失提示 | 适合 | 通常不必要 |
| 离线后补收 | 不提供 | 可按消费组语义处理 |
| 消费确认 | 无 | XACK 与 pending 管理 |
| 失败重试 | 应用另行实现 | 可设计 claim/retry/dead-letter |
| 多消费者负载分配 | 所有订阅者都收到 | 消费组内分配 |

Stream 消费者必须处理至少一次投递：业务幂等、处理成功后 ACK、pending 超时认领、重试上限和死信。不要在收到消息后立即 ACK 再开始业务处理。

## 12.7 Redis 安全与容量验收

- [ ] 不暴露无认证 Redis 到公网，使用 TLS/私有网络与最小权限。
- [ ] key 有前缀和 TTL/容量策略，无用户输入导致的无限命名空间。
- [ ] 不使用 KEYS、FLUSHALL、无控制 DEBUG 等危险生产操作。
- [ ] Lua 原子性、锁 token 和消费重试经过真实 Redis 测试。
- [ ] Cluster hash slot、重连和命令兼容性按部署形态验证。
- [ ] 应用关闭时订阅、阻塞命令和普通 client 都能退出。

上一篇：[Keyv 缓存](07-keyv-cache.md) | [返回总目录](../development-guide.md) | 下一篇：[应用生命周期](09-application-lifecycle.md)
