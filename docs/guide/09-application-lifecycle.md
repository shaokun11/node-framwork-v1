# 依赖装配与资源生命周期

## 13. 依赖装配与资源生命周期

### 13.1 Composition Root

只有 `main.ts` 负责触发进程启动：

```ts
const application = await Application.create();
await application.start();
```

`Application.create()` 的顺序建议为：

1. 加载并校验配置。
2. 创建 logger，记录脱敏启动信息。
3. 创建并连接当前选型的主数据库。
4. 按需创建 Keyv Adapter/Keyv 和独立 Redis client。
5. 按需创建 Tinypool、HTTP client 等基础设施。
6. 创建 Repository、Service、Controller。
7. 创建 Hono app 并注册公共中间件、健康检查和业务路由。
8. 所有启动条件满足后才监听端口。

类构造函数保持廉价且不同步做隐藏 I/O。需要异步连接的资源由工厂或显式 `connect()` 完成。启动中途失败必须关闭已经创建的资源，不能留下半初始化进程。

### 13.2 所有权与关闭

谁创建资源，谁负责关闭。Repository 不关闭注入的 Pool、Mongoose Connection、Keyv 或 Redis client。

收到 `SIGINT`/`SIGTERM` 后：

1. 标记 not ready，停止接受新连接。
2. 在 shutdown timeout 内排空进行中 HTTP 请求。
3. 停止接收新任务并等待或取消 Worker。
4. 按依赖反序关闭业务资源、Tinypool、Redis/Keyv、数据库。
5. 刷新日志并以明确退出码结束；超时强制退出并记录未关闭资源。

信号处理器只注册一次，第二次信号可强制退出。所有 close 操作应幂等并有超时。Hono Node adapter 返回的 server 由应用显式 close；不要在每个模块监听 process 信号。

### 13.3 请求上下文与取消

- requestId、principal 等请求数据通过 Hono Context 或明确上下文对象传递，不使用模块全局变量。
- Node.js 单进程可能并发处理请求，禁止把当前请求写入 Service 实例字段。
- 需要取消的下游方法接收 `AbortSignal`；超时、客户端断开和 shutdown 的关系要有统一策略。
- `Promise.all` 不会自动取消其他任务；失败时需要 AbortController 或可接受的等待策略。
- 所有 setInterval、setTimeout、事件订阅和流都要有所有者及清理路径。

## 13.4 应用状态机

建议显式区分 `created`、`starting`、`running`、`stopping`、`stopped`。只有 running 接收业务流量；stopping 的 readiness 必须先失败。

- start 和 stop 应具备明确并发语义，重复调用不能重复连接或注册信号。
- 启动失败后状态进入 stopping/stopped，并执行已创建资源清理。
- stop 后不接受新任务；正在进行任务根据策略等待或取消。
- 未处理异常和 unhandled rejection 的进程策略统一记录，不能在随机模块调用 process.exit。

## 13.5 启动失败回滚

装配过程维护资源栈：

```text
配置成功
  -> logger 创建成功
  -> 数据库连接成功（登记 close）
  -> Redis 连接成功（登记 close）
  -> HTTP listen 失败
       -> 关闭 Redis
       -> 关闭数据库
       -> 关闭 logger/刷新输出
       -> 启动失败退出
```

清理按创建反序执行。某个 close 失败仍继续关闭剩余资源，最后聚合错误。启动测试应逐个注入连接和 listen 失败，验证没有端口、句柄和定时器遗留。

## 13.6 HTTP 排空与强制退出

Node server close、连接保持和进行中的请求行为要按当前 Node/Hono adapter 版本验证。仅调用 `server.close()` 不一定立即终止所有 keep-alive 或升级连接；必要时使用对应 API 关闭 idle 连接，并为活跃请求设置最后期限。

WebSocket、SSE、流式响应和长轮询必须单独定义 shutdown 行为。普通 HTTP 请求排空策略不能直接套用到永久连接。

## 13.7 AbortSignal 传递规范

- Controller 从请求上下文取得 signal，并作为应用调用的取消信号。
- Service 将 signal 传给支持取消的 Repository、HTTP client 和 Worker。
- 不支持取消的数据库驱动操作仍要有 command/socket timeout，并记录无法立即中止的边界。
- 组合超时使用明确 parent/child 关系，不在每层创建互相矛盾的超时。
- 请求结束后清理 AbortSignal listener、timer 和流，避免监听器泄漏。

## 13.8 生命周期测试清单

- [ ] 配置错误在任何外部连接前失败。
- [ ] 每个启动阶段失败都会逆序清理已创建资源。
- [ ] SIGTERM/SIGINT 先 readiness 失败，再停止接收流量。
- [ ] 正常请求可完成，超时请求被取消或强制终止。
- [ ] Tinypool、Redis、Keyv、数据库、timer 和订阅全部关闭。
- [ ] 重复 stop、二次信号和关闭超时不会悬挂进程。

上一篇：[Redis 原生能力](08-redis.md) | [返回总目录](../development-guide.md) | 下一篇：[安全与可观测性](10-security-observability.md)
