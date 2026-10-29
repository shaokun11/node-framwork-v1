# 多实例、Croner、性能与 Tinypool

## 16. 多实例、定时任务与性能

### 16.1 API 多实例要求

API 从开发阶段就必须支持水平扩展：任一健康实例处理同类请求时，身份、权限、幂等和业务数据一致性一致。

- 不在进程内保存必须跨请求共享的业务事实、session、限流配额或幂等结果。
- 本地内存 Keyv 只用于允许实例私有且可重建的缓存，不能宣称共享状态。
- 文件上传不能默认依赖某实例本地磁盘；使用共享存储或明确的上传架构。
- 全局定时任务需要单领导者、数据库条件抢占或分布式协调，并保证任务幂等。
- 数据库唯一约束、原子更新和消息去重是最终并发保障，Node.js 实例内锁只保护当前进程。

### 16.2 兼容发布与迁移

滚动发布期间旧代码、新代码和迁移可能同时存在。先新增可空列/新表/新字段并双读兼容，再回填和切换，最后确认旧版本不再运行后清理。接口字段新增也要检查严格客户端；删除、重命名和语义改变需要版本或兼容窗口。

### 16.3 Tinypool：CPU 密集任务

Tinypool 用于把 CPU 密集 JavaScript 工作放入 Worker Threads，避免阻塞 API event loop，例如可并行计算、图片/数据转换和受控算法任务。它不是数据库任务队列，不提供持久化、跨进程重试和崩溃恢复。

- 只有 profiling 证明 CPU 工作或明确需求时引入；普通数据库/HTTP I/O 不需要 Worker。
- 由 bootstrap 创建单一池，设置 `minThreads`、`maxThreads`、任务超时和关闭策略。
- Worker 输入输出使用可结构化克隆的数据或明确 Transferable，不传 Hono Context、class 实例、数据库连接和 logger。
- Worker 文件必须是运行时真实存在的 `.js` 路径；`tsx` 开发与 `tsc` 生产构建都要验证路径。
- 每个任务有大小、时间和返回体限制；池满时采用有界排队、拒绝或降级，不能无限等待。
- 主线程只等待必要结果，超时后处理孤立结果和后续任务污染。
- Worker 中发生未捕获异常、退出和内存增长要有日志、指标及重建策略。
- 需要持久任务时，由 [BullMQ](09-bullmq.md) 等可靠队列交付任务，Tinypool 只承担其中的 CPU 执行阶段。

```ts
import Tinypool from "tinypool";

const pool = new Tinypool({
    filename: new URL("./workers/order-calculator.worker.js", import.meta.url).href,
    minThreads: 0,
    maxThreads: config.workers.maxThreads,
});
// shutdown 时 await pool.destroy()，并为关闭设置上限。
```

### 16.4 es-toolkit 工具函数

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

### 16.5 性能原则

先测量再优化。记录 event loop 延迟、CPU、内存、请求延迟和数据库/Redis 指标，使用生产近似数据验证。避免 N+1 查询、无界 Promise 并发、大对象序列化和重复深度克隆；批量任务必须有并发上限和背压。

缓存、连接池和 Worker 都不是默认越快越好。池大小应结合数据库容量、实例数和下游限额做全局预算。

## 16.6 多实例会话与任务策略

| 状态类型 | 可行方案 | 不可接受方式 |
| --- | --- | --- |
| 无状态 Bearer token | 验签 + 必要的数据库状态检查 | 依赖签发实例内存 |
| 可撤销 session | 共享 session store 或数据库 | 本地 Map 作为唯一会话 |
| 严格全局限流 | Redis 等共享原子实现 | 每实例独立 bucket 声称全局 |
| 幂等结果 | 数据库唯一约束与共享记录 | 单进程 Promise/Map |
| 定时任务 | 抢占记录、领导者协调或单 worker | 每个 API 副本无条件执行 |
| 上传文件 | 共享对象存储/明确存储服务 | 容器本地临时目录当永久存储 |

无状态 token 仍要考虑撤销、权限变更和密钥轮换。共享 Redis 也不是天然可靠 session 数据库，持久性与安全要求必须匹配。

## 16.7 Tinypool 任务协议

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

## 16.8 Worker 路径与 ESM

项目全部采用 ESM。Worker 入口通过 `new URL(..., import.meta.url)` 定位，不能使用 `__dirname`。TypeScript 的 `rewriteRelativeImportExtensions` 会改写相对 import/export specifier，但不会自动改写任意字符串或 new URL 参数。

因此必须分别验证：

- tsx 开发时 Worker 实际入口和模块格式。
- tsc 构建后 dist 中 Worker 文件位置。
- new URL 指向的开发/生产文件是否存在。
- Worker 内部相对 imports 在构建后是否已改写为 `.js`。

可以为开发/生产提供明确的 Worker URL 解析辅助，但不能靠复制临时文件掩盖构建缺失。

## 16.9 性能测试与容量记录

每次重要优化记录 baseline、工作负载、环境、指标和结果：

- 吞吐量、p50/p95/p99、错误率和超时率。
- CPU、event loop delay、heap、GC 和外部内存。
- 数据库慢查询、连接池等待、Redis 延迟和缓存命中率。
- Worker 队列长度、任务耗时和主线程复制成本。

负载测试必须使用有界并发和生产近似数据，不把本机 loopback 结果直接当生产容量。优化后保留正确性和并发测试，防止只提升 benchmark。


## 16.10 es-toolkit 安装与常用函数

安装：

```sh
npm install es-toolkit
```

使用具名 ESM 导入：

```ts
import { chunk, groupBy, keyBy, uniqBy } from "es-toolkit";

const batches = chunk(orderIds, 500);
for (const batch of batches) {
    await orderRepository.deleteBatch(batch);
}

const ordersByStatus = groupBy(orders, (order) => order.status);
const ordersById = keyBy(orders, (order) => order.id);
const uniqueSkus = uniqBy(items, (item) => item.sku);
```

- `chunk`：限制数据库批量和外部请求批量大小。
- `groupBy`：按键分组，返回普通对象，注意键会转为对象属性。
- `keyBy`：构建索引；重复 key 的覆盖规则要确认，不能当唯一性验证。
- `uniqBy`：按规范化键去重；先明确大小写和 trim 规则。

防抖与节流：

```ts
import { debounce, throttle } from "es-toolkit";

const refresh = debounce(() => {
    void refreshCache();
}, 300);

const report = throttle(() => {
    void reportMetrics();
}, 1000);

// 组件/进程停止时按当前 API 取消或 flush，避免遗留 timer。
refresh.cancel();
report.cancel();
```

debounce/throttle 只控制调用频率，不自动提供异步串行、取消正在运行的 Promise 或持久任务语义。异步回调需显式处理 rejection 和 shutdown。

深拷贝与合并：

- `cloneDeep` 用于确实需要隔离嵌套数据的场景，不作为性能默认选择。
- 配置加载不直接拿通用 merge 处理不可信来源，仍需防原型污染、数组替换和 Zod strict。
- 领域对象构造、金额计算和权限逻辑不放入通用 es-toolkit 调用中隐藏。

## 16.11 Tinypool 安装与完整 Worker 示例

安装：

```sh
npm install tinypool
```

Worker 文件 `order-calculator.worker.ts`：

```ts
export interface CalculateOrderTask {
    version: 1;
    orderId: string;
    items: readonly {
        sku: string;
        quantity: number;
        unitPriceMinor: number;
    }[];
}

export interface CalculateOrderResult {
    orderId: string;
    totalMinor: number;
}

export default function calculateOrder(
    task: CalculateOrderTask,
): CalculateOrderResult {
    if (task.version !== 1) throw new Error("unsupported task version");

    let totalMinor = 0;
    for (const item of task.items) {
        totalMinor += item.quantity * item.unitPriceMinor;
    }
    return { orderId: task.orderId, totalMinor };
}
```

Worker 必须同步或异步返回可结构化克隆的结果，不 import Hono Context、Winston transports 或主线程数据库连接。示例只做计算，真实价格仍需以服务端事实数据为准。

主线程：

```ts
import { fileURLToPath } from "node:url";
import Tinypool from "tinypool";

const pool = new Tinypool({
    filename: fileURLToPath(
        new URL("./order-calculator.worker.js", import.meta.url),
    ),
    minThreads: 0,
    maxThreads: config.workers.maxThreads,
    idleTimeout: 30_000,
});

const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);
try {
    const result = await pool.run(
        { version: 1, orderId, items },
        { signal: controller.signal },
    );
    return result;
} finally {
    clearTimeout(timeout);
}
```

关闭：

```ts
await pool.destroy();
```

`pool.destroy()` 前停止提交新任务，并决定正在运行任务等待还是取消。销毁也纳入总 shutdown timeout。

## 16.12 Tinypool 开发/生产路径与并发控制

前面的 `.js` URL 指向构建产物。tsx 开发环境需要项目自己的 Worker URL 解析策略，使源码 `.ts` 与 dist `.js` 各指向真实文件；不要用 `rewriteRelativeImportExtensions` 假设 new URL 字符串也会被改写。

```ts
interface WorkerPaths {
    orderCalculator: string;
}

// 由 infrastructure 在开发和生产装配时分别提供真实绝对路径。
const pool = new Tinypool({
    filename: workerPaths.orderCalculator,
    maxThreads: config.workers.maxThreads,
});
```

提交任务前设置业务并发上限：

- maxThreads 控制 Worker 数量，不等于等待队列有上限。
- API 中间件或应用服务限制同时等待任务数，池满返回 429/503 或执行有界排队。
- 每个任务限制输入数组、字节、计算时间和输出大小。
- 超时只停止主线程等待不一定意味着所有 CPU 副作用立即消失，按 Tinypool 当前取消语义验证。
- Worker 异常、任务超时和池重建记录 Winston 和指标。

## 16.13 什么时候不要使用 Tinypool

- 只是等待 PostgreSQL、MongoDB、Redis 或 HTTP：这是 I/O，不需要 Worker。
- 任务很小且频率低：序列化和线程调度可能比直接计算更贵。
- 需要崩溃后重新执行：使用持久队列，Tinypool 不提供。
- 需要跨多台机器水平扩展：使用 [BullMQ](09-bullmq.md) 等分布式 Worker/Queue。
- 只是为了利用所有 CPU：先 profiling，Worker 过多会争用 CPU 和内存。

## 16.14 Croner 进程内定时任务

[Croner](https://github.com/hexagon/croner) 用于 cron 表达式解析、下次执行时间计算和当前进程内的轻量调度。它支持 TypeScript、异步函数、时区和 overrun protection，但不提供分布式领导者选举、持久任务记录或进程宕机后的补跑。

| 需求 | 选择 |
| --- | --- |
| 单进程缓存清理、短期轮询、本地定时触发 | Croner |
| 计算 cron 下次执行时间 | Croner |
| 多副本只能有一个调度器 | 单 scheduler 部署、数据库租约或其他协调机制 |
| 延迟任务需要持久化、重试和崩溃恢复 | [BullMQ](09-bullmq.md) |
| CPU 密集执行 | Croner/BullMQ 只负责触发，Tinypool 负责计算 |

安装与使用：

```sh
npm install croner
```

```ts
import { Cron } from "croner";

let currentRun: Promise<void> = Promise.resolve();

const nightlyReportJob = new Cron(
    // Croner 六字段：秒 分 时 日 月 星期；每天 02:00:00。
    "0 0 2 * * *",
    {
        timezone: "Asia/Shanghai",
        // 前一次异步任务未结束时，不触发下一次。
        protect: true,
    },
    () => {
        currentRun = reportService.generateDaily().catch((error: unknown) => {
            logger.error("nightly report failed", {
                errorCode: "NIGHTLY_REPORT_FAILED",
                error,
            });
        });
        return currentRun;
    },
);

const nextRun = nightlyReportJob.nextRun();
logger.info("nightly report scheduled", {
    nextRunAt: nextRun?.toISOString() ?? null,
});

export async function closeScheduler(): Promise<void> {
    // 阻止后续触发；stop() 不会取消已经运行的 Promise。
    nightlyReportJob.stop();
    await currentRun;
}
```

调度规则：

- 表达式、`timezone` 和业务用途写入配置或代码评审，不能依赖部署主机时区。需要固定 UTC offset 时才用 `utcOffset`；有时区/DST 要求时使用 IANA timezone。
- 默认可选秒、年字段与传统五字段 cron 不完全相同。示例明确使用六字段，迁移表达式时用当前 Croner 版本验证。
- 所有全局任务默认可重入风险都要处理。`protect: true` 只保护当前 Croner 实例，不能阻止另一台机器同时执行。
- 回调异常必须记录并进入指标/告警；不要设置 `catch: true` 静默吞错。任务超时时使用 `AbortSignal` 或下游 timeout，仅停止等待不等于副作用消失。
- `stop()` 后等待当前任务，并纳入应用总 shutdown timeout。`pause()` 可恢复，`stop()` 是永久停止。
- 多实例部署使用单 scheduler、数据库条件抢占/租约或明确协调方案。锁过期、实例宕机和任务重试仍需业务幂等。
- Croner 不保存执行记录。进程停止期间错过的任务不会自动补跑；需要补跑时保存 schedule/checkpoint，并在启动时定义恢复策略。
- DST 春季不存在时间和秋季重复时间按当前 Croner 语义编写测试；财务、账单等关键时间不能只凭表达式推断。

日期时间值在调度边界是 JavaScript `Date`，需要格式化或时区展示时按 [Day.js 规范](13-typescript-conventions.md#179-dayjs-日期与时间)转换，不用字符串截取计算时间。

上一篇：[安全与可观测性](11-security-observability.md) | [返回总目录](../development-guide.md) | 下一篇：[TypeScript 规范](13-typescript-conventions.md)
