# 多实例、性能与 Tinypool

## 15. 多实例、任务与性能

### 15.1 API 多实例要求

API 从开发阶段就必须支持水平扩展：任一健康实例处理同类请求时，身份、权限、幂等和业务数据一致性一致。

- 不在进程内保存必须跨请求共享的业务事实、session、限流配额或幂等结果。
- 本地内存 Keyv 只用于允许实例私有且可重建的缓存，不能宣称共享状态。
- 文件上传不能默认依赖某实例本地磁盘；使用共享存储或明确的上传架构。
- 全局定时任务需要单领导者、数据库条件抢占或分布式协调，并保证任务幂等。
- 数据库唯一约束、原子更新和消息去重是最终并发保障，Node.js 实例内锁只保护当前进程。

### 15.2 兼容发布与迁移

滚动发布期间旧代码、新代码和迁移可能同时存在。先新增可空列/新表/新字段并双读兼容，再回填和切换，最后确认旧版本不再运行后清理。接口字段新增也要检查严格客户端；删除、重命名和语义改变需要版本或兼容窗口。

### 15.3 Tinypool：CPU 密集任务

Tinypool 用于把 CPU 密集 JavaScript 工作放入 Worker Threads，避免阻塞 API event loop，例如可并行计算、图片/数据转换和受控算法任务。它不是数据库任务队列，不提供持久化、跨进程重试和崩溃恢复。

- 只有 profiling 证明 CPU 工作或明确需求时引入；普通数据库/HTTP I/O 不需要 Worker。
- 由 bootstrap 创建单一池，设置 `minThreads`、`maxThreads`、任务超时和关闭策略。
- Worker 输入输出使用可结构化克隆的数据或明确 Transferable，不传 Hono Context、class 实例、数据库连接和 logger。
- Worker 文件必须是运行时真实存在的 `.js` 路径；`tsx` 开发与 `tsc` 生产构建都要验证路径。
- 每个任务有大小、时间和返回体限制；池满时采用有界排队、拒绝或降级，不能无限等待。
- 主线程只等待必要结果，超时后处理孤立结果和后续任务污染。
- Worker 中发生未捕获异常、退出和内存增长要有日志、指标及重建策略。
- 需要持久任务时，由可靠队列交付任务，Tinypool 只承担其中的 CPU 执行阶段。

```ts
import Tinypool from "tinypool";

const pool = new Tinypool({
    filename: new URL("./workers/order-calculator.worker.js", import.meta.url).href,
    minThreads: 0,
    maxThreads: config.workers.maxThreads,
});
// shutdown 时 await pool.destroy()，并为关闭设置上限。
```

### 15.4 es-toolkit 工具函数

通用数组、对象、数学和异步辅助优先使用标准库或 es-toolkit，不重复维护 `chunk`、`groupBy`、`debounce` 等工具。

```ts
import { chunk, groupBy, keyBy } from "es-toolkit";
```

- 使用具名导入并保持生产 bundle 可 tree-shake，不引入全量动态属性访问。
- 使用前阅读当前版本对输入、空值、原地修改和返回副本的契约。
- `cloneDeep`、merge 等操作不能代替领域对象构造、安全深合并或配置防原型污染规则。
- debounce/throttle 只处理调用时机，不自动解决取消、并发和持久幂等。
- 简单标准库已足够时不必为一次 `map` 引入辅助函数；不并存 Lodash、Ramda 等多套相似工具。
- 不把数据库查询构造、权限、金额计算等核心业务藏入通用工具函数。

### 15.5 性能原则

先测量再优化。记录 event loop 延迟、CPU、内存、请求延迟和数据库/Redis 指标，使用生产近似数据验证。避免 N+1 查询、无界 Promise 并发、大对象序列化和重复深度克隆；批量任务必须有并发上限和背压。

缓存、连接池和 Worker 都不是默认越快越好。池大小应结合数据库容量、实例数和下游限额做全局预算。

## 15.6 多实例会话与任务策略

| 状态类型 | 可行方案 | 不可接受方式 |
| --- | --- | --- |
| 无状态 Bearer token | 验签 + 必要的数据库状态检查 | 依赖签发实例内存 |
| 可撤销 session | 共享 session store 或数据库 | 本地 Map 作为唯一会话 |
| 严格全局限流 | Redis 等共享原子实现 | 每实例独立 bucket 声称全局 |
| 幂等结果 | 数据库唯一约束与共享记录 | 单进程 Promise/Map |
| 定时任务 | 抢占记录、领导者协调或单 worker | 每个 API 副本无条件执行 |
| 上传文件 | 共享对象存储/明确存储服务 | 容器本地临时目录当永久存储 |

无状态 token 仍要考虑撤销、权限变更和密钥轮换。共享 Redis 也不是天然可靠 session 数据库，持久性与安全要求必须匹配。

## 15.7 Tinypool 任务协议

每个 Worker 任务定义版本化输入输出：

```ts
interface CalculateOrderTask {
    version: 1;
    orderId: string;
    items: readonly { sku: string; quantity: number }[];
}

interface CalculateOrderResult {
    orderId: string;
    totalMinor: number;
}
```

- 使用 Zod 在 Worker 边界验证不可信或跨版本 payload。
- 错误返回稳定 task error，不直接序列化包含秘密的 Error 对象。
- 大数据优先 Transferable 或有界共享内存，避免重复复制。
- Worker 不读取 Hono Context，不依赖主线程模块全局状态。
- 若 Worker 直接访问数据库，必须独立管理连接与关闭；默认优先把 I/O 留在主线程。

## 15.8 Worker 路径与 ESM

项目全部采用 ESM。Worker 入口通过 `new URL(..., import.meta.url)` 定位，不能使用 `__dirname`。TypeScript 的 `rewriteRelativeImportExtensions` 会改写相对 import/export specifier，但不会自动改写任意字符串或 new URL 参数。

因此必须分别验证：

- tsx 开发时 Worker 实际入口和模块格式。
- tsc 构建后 dist 中 Worker 文件位置。
- new URL 指向的开发/生产文件是否存在。
- Worker 内部相对 imports 在构建后是否已改写为 `.js`。

可以为开发/生产提供明确的 Worker URL 解析辅助，但不能靠复制临时文件掩盖构建缺失。

## 15.9 性能测试与容量记录

每次重要优化记录 baseline、工作负载、环境、指标和结果：

- 吞吐量、p50/p95/p99、错误率和超时率。
- CPU、event loop delay、heap、GC 和外部内存。
- 数据库慢查询、连接池等待、Redis 延迟和缓存命中率。
- Worker 队列长度、任务耗时和主线程复制成本。

负载测试必须使用有界并发和生产近似数据，不把本机 loopback 结果直接当生产容量。优化后保留正确性和并发测试，防止只提升 benchmark。

上一篇：[安全与可观测性](10-security-observability.md) | [返回总目录](../development-guide.md) | 下一篇：[TypeScript 规范](12-typescript-conventions.md)
