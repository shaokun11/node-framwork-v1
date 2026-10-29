# 安全、Hono JWT、Casbin、Winston 日志与可观测性

## 15. 认证授权、日志、安全与可观测性

### 15.1 结构化日志

统一采用 Winston Logger，由 bootstrap 创建并注入。每条请求日志至少包含 requestId、method、route template、status、duration 和 outcome；错误日志包含稳定 error code 和内部 cause。

不要记录原始 URL query、Authorization、Cookie、密码、连接 URL、完整请求体、token、个人隐私和秘密配置。SQL debug 默认关闭；即使开启也必须考虑参数敏感信息，不能在生产无差别打印完整参数。

日志应能区分客户端输入失败、权限拒绝、业务冲突、依赖超时、依赖不可用和程序错误。请求失败只在最终边界记录一次，避免每层重复 error 日志。

### 15.2 安全基线

- 密码使用 Argon2id 或 bcrypt 等合适算法和参数，不自行设计密码散列。
- Token/session 的签名密钥、加密密钥和哈希用途分离；密钥可轮换。
- 登录失败对未知账号和错误密码保持一致的公开响应，并考虑时间侧信道。
- 所有权限检查包含资源归属，不能只验证已登录。
- 输出编码、安全 header、CORS、CSRF、请求体限制、上传类型/大小和 SSRF 防护按路由落实。
- 数据库和 Redis 使用最小权限账号，开发管理员凭据不能复用于生产。
- 依赖安装、脚本执行和 CI token 按供应链安全要求控制；定期检查漏洞但不能只靠扫描。
- 错误消息不泄露 SQL、schema、内部主机、堆栈和第三方秘密。

### 15.2.1 简单场景：Hono JWT

只需要验证 Bearer JWT、读取 `sub`/`tenantId`，以及检查少量固定角色时，使用 Hono 官方 JWT 中间件即可，不必为了技术完整引入 Casbin。

| 需求 | 默认选择 |
| --- | --- |
| 验证 access token、获取 principal | Hono JWT |
| 少量稳定角色，例如 admin/user | Hono JWT + 简单角色中间件 |
| 每资源 owner/tenant 限制 | Service/Repository 条件查询，不能仅靠角色 |
| 动态角色权限、RBAC、ABAC、委托和策略管理 | Casbin，按需引入 |

安装与最小接入：

```sh
npm install hono zod
```

```ts
import { Hono } from "hono";
import { jwt } from "hono/jwt";
import type { Context, Next } from "hono";
import { z } from "zod";

const principalSchema = z.object({
    sub: z.string().min(1),
    tenantId: z.string().min(1),
    roles: z.array(z.enum(["admin", "user"])).default([]),
});

type Principal = z.infer<typeof principalSchema>;
type AppEnv = { Variables: { principal: Principal } };

const app = new Hono<AppEnv>();
app.use("/api/*", jwt({ secret: config.jwt.secret, alg: "HS256" }));
app.use("/api/*", async (c, next) => {
    const parsed = principalSchema.safeParse(c.get("jwtPayload"));
    if (!parsed.success) {
        return c.json({ error: "UNAUTHORIZED", message: "无效凭证" }, 401);
    }
    c.set("principal", parsed.data);
    await next();
});

function requireRole(role: Principal["roles"][number]) {
    return async (c: Context<AppEnv>, next: Next) => {
        if (!c.get("principal").roles.includes(role)) {
            return c.json({ error: "FORBIDDEN", message: "没有操作权限" }, 403);
        }
        await next();
    };
}

app.get("/api/admin/reports", requireRole("admin"), (c) => {
    return c.json({ data: [], meta: {} });
});
```

这里的 JWT middleware 负责验签和过期等标准校验，Zod 再把 payload 转成应用 principal。Hono JWT 不负责登录、用户管理、权限矩阵或资源归属：

- 签发 token 时使用当前 `hono/jwt` 的 `sign` 能力或统一身份服务，校验 `iss`、`aud`、`exp` 等项目要求的 claims；密钥来自配置，不写进源码。
- 显式允许算法，不接受 `none`，不根据不可信 token 随意选择密钥。密钥轮换、`kid` 和撤销策略按身份架构实现。
- 未认证返回 401，已认证但无权限返回 403；隐藏资源是否存在时可按 API 策略统一返回 404。
- JWT 中角色可能陈旧。高敏操作需要 fresh token、服务端状态或 Casbin/数据库中的当前权限。
- `roles.includes()` 只表示路由级粗粒度角色。订单是否属于当前 tenant/user 仍必须由业务查询约束。

### 15.2.2 复杂场景：Casbin

当权限规则需要独立维护，出现多角色继承、资源/动作矩阵、ABAC、动态授权或管理后台调整策略时，可以引入 [Casbin](https://casbin.apache.org/docs/get-started/)。Casbin 回答“某个 subject 能否对 object 执行 action”，不代替 JWT 验签，也不自动完成数据库行级隔离。

```sh
npm install casbin
```

先用官方风格的 model 和 policy 跑通 RBAC。`model.conf`：

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && keyMatch2(r.obj, p.obj) && r.act == p.act
```

`policy.csv`：

```csv
p, admin, /api/orders/*, GET
p, admin, /api/orders/*, POST
p, user, /api/orders/*, GET
g, user-123, user
g, user-456, admin
```

在 Composition Root 创建 enforcer，并包装为 Hono 授权中间件：

```ts
import { newEnforcer } from "casbin";
import type { Enforcer } from "casbin";
import type { Context, Next } from "hono";

const enforcer = await newEnforcer("model.conf", "policy.csv");

function authorize(enforcer: Enforcer) {
    return async (c: Context<AppEnv>, next: Next) => {
        const principal = c.get("principal");
        const allowed = await enforcer.enforce(
            principal.sub,
            c.req.path,
            c.req.method,
        );
        if (!allowed) {
            return c.json({ error: "FORBIDDEN", message: "没有操作权限" }, 403);
        }
        await next();
    };
}

app.use("/api/*", authorize(enforcer));
```

这个示例假设前面的 JWT/principal 中间件已经运行。`enforce()` 抛错是授权基础设施异常，应走集中错误处理并默认拒绝，不能当作 allow。开发/生产的 model/policy 文件必须进入构建或部署产物，使用明确绝对路径，不能依赖偶然 cwd。

文件 policy 适合最小示例和静态规则。需要动态策略时选择与当前数据库匹配的 Casbin Adapter，并明确：

- 策略事实来源、创建/更新/删除 API、输入验证、审计和备份。
- 多实例策略加载与刷新机制；不要在每个请求无控制地 `loadPolicy()`。
- 角色/权限更新与 JWT 有效期之间的撤销窗口。
- model、policy、Adapter schema 的版本兼容、迁移和回滚。
- enforcer 与业务数据库之间没有自动事务；授权变化和资源状态变化需要一致性设计。

Casbin 通过不等于可以读取任意订单。比如 `user` 能 `GET /api/orders/*` 只表示允许该动作，Repository 仍必须使用 `orderId + tenantId`（以及必要的 ownerId）查询。管理员跨租户能力也要显式建模和审计，不能在 Controller 用任意布尔参数绕过。

复杂授权至少测试：无 token、过期/非法 claims、无角色、允许/拒绝、通配符边界、角色继承、跨用户/跨租户、策略故障 fail-closed、策略刷新以及多实例一致性。

### 15.3 健康检查与指标

- `/livez` 只表示进程未进入不可恢复状态，不查询所有依赖。
- `/readyz` 检查当前服务真正必需的依赖和启动状态；缓存是否必需由降级策略决定。
- 健康检查有超时、缓存合理结果并避免对数据库造成放大流量。
- 指标使用 route template，不把原始 path、订单 ID、用户 ID 作为 label。
- 至少观察请求速率/错误/延迟、数据库 Pool、关键依赖错误、缓存 hit/miss 和 Worker 队列/耗时。
- 需要分布式追踪时统一 trace/span 语义并传播上下文，不能只安装 SDK 就视为完成。

## 15.4 Winston 日志实现规范

项目统一使用 Winston，由 infrastructure logging factory 创建 Logger 并注入。业务类不调用全局 `console.log`，也不直接选择 transport。

- development 可使用带颜色的可读 format，production 默认 JSON。
- level 来自强类型配置，transport 的 level 与 logger level 关系明确。
- 使用 Winston format 统一 timestamp、error stack、request context 和 redaction。
- `Error` 对象必须保留 name、message、stack 和 cause，不能序列化成空对象。
- Console、File、HTTP 等 transport 按环境显式配置，文件 rotation 和磁盘容量有上限。
- 每个异步 transport 的错误事件必须处理；日志故障不能形成无限错误循环。
- shutdown 在超时内结束 transports/flush，不能把 logger.close 当作无限等待。

```ts
logger.info("request completed", {
    requestId,
    method,
    route: routeTemplate,
    status,
    durationMs,
    outcome,
});
```

字段采用一致 lowerCamelCase。不要在 message 拼接可变 ID 后又重复输出无结构文本；message 描述事件，metadata 携带上下文。

## 15.5 日志脱敏与错误边界

Winston 的 redaction format 在日志写入前执行，至少覆盖：

- headers 中的 authorization、cookie、set-cookie、x-api-key。
- body/config 中的 password、secret、token、databaseUrl、mongodbUrl、redisUrl。
- 连接 URL 的 userinfo、查询中的 token 和签名参数。

redaction 采用递归键白名单/黑名单策略，并限制深度和循环引用。不能依赖某一层已经脱敏；第三方 error 也可能包含 URL 或请求配置。

开发 debug SQL/请求体日志仍需经过同一 redaction。生产临时开启 debug 必须有期限和审计。

## 15.6 安全事件与审计

登录成功/失败、权限拒绝、账号锁定、秘密轮换、管理员操作和数据导出等安全事件应有结构化审计日志。审计记录 actor、action、target type/id、requestId、result 和时间，但不记录密码、完整 token 或不必要的个人数据。

审计日志的业务保留期、防篡改和访问权限由合规需求决定，普通 console 日志不自动满足审计要求。

## 15.7 指标与告警实施

| 指标 | 建议维度 | 主要用途 |
| --- | --- | --- |
| http_requests_total | method、route、status class | 流量和错误率 |
| http_request_duration_ms | method、route | 延迟分布与 SLO |
| db_pool_clients | state | 池耗尽和泄漏 |
| cache_operations_total | operation、result | hit/miss/error |
| redis_operations_total | operation、result | 依赖失败 |
| worker_tasks_total | task、result | Worker 饱和与失败 |

使用 histogram 观察延迟，不只记录平均值。告警结合错误率、延迟分位数、饱和度和业务影响；单次 cache miss 不应触发页面告警，持续写后缓存删除失败可能需要。

## 15.8 安全与可观测性验收

- [ ] Winston 输出结构稳定，Error/cause 和 requestId 可查询。
- [ ] 秘密、Cookie、Authorization 和连接 URL 的脱敏有单元测试。
- [ ] live/ready 不会放大依赖流量，多实例指标 label 无高基数。
- [ ] 认证、授权、租户隔离、上传和 SSRF 有对抗性测试。
- [ ] 安全审计事件和日志保留满足项目要求。
- [ ] shutdown 能结束 Winston transports，不因日志 I/O 无限悬挂。


## 15.9 Winston 安装与开发/生产配置

安装：

```sh
npm install winston
```

基础 Logger：

```ts
import winston from "winston";

const { combine, timestamp, errors, json } = winston.format;

export function createHttpLogger(config: LoggingConfig): winston.Logger {
    return winston.createLogger({
        level: config.level,
        defaultMeta: { service: "order-api" },
        format: combine(
            errors({ stack: true }),
            timestamp(),
            createRedactionFormat(),
            json(),
        ),
        transports: [new winston.transports.Console()],
    });
}
```

使用：

```ts
logger.info("application started", { port: config.server.port });
logger.warn("cache fallback", { operation: "order.read", requestId });
logger.error("request failed", {
    requestId,
    error: serializeError(error),
});
```

常用级别从详细到严重通常为 debug、info、warn、error。应用配置只允许项目声明的级别，不用动态字符串直接控制 transport。

开发可读输出：

```ts
const developmentFormat = combine(
    winston.format.colorize(),
    timestamp(),
    createRedactionFormat(),
    winston.format.printf((info) => {
        const { timestamp, level, message, ...metadata } = info;
        return `${timestamp} ${level} ${message} ${JSON.stringify(metadata)}`;
    }),
);
```

printf 必须在 redaction 之后，否则可能把原始秘密 stringify。Error 的 stack、cause 和 metadata 也要经过序列化与脱敏。

## 15.10 请求日志与 child logger

为请求创建带默认 metadata 的 child：

```ts
app.use("*", async (c, next) => {
    const requestLogger = httpLogger.child({
        requestId: c.get("requestId"),
        method: c.req.method,
        route: c.req.routePath,
    });
    c.set("logger", requestLogger);
    await next();
});
```

Controller 不直接输出日志；需要记录业务事件时注入 logger 或使用请求级辅助。避免每个中间件重复打印同一请求完成事件。

推荐事件：

```ts
requestLogger.info("order created", {
    orderId: order.id,
    ownerId: order.ownerId,
    outcome: "created",
});
```

orderId/ownerId 是否需要记录由隐私和日志保留策略决定；它们可以进日志，但不能作为无上限指标 label。

## 15.11 正确记录 Error 和 cause

Winston 不会自动把所有嵌套 Error 字段变成可读堆栈，使用显式序列化：

```ts
function serializeError(error: unknown): Record<string, unknown> {
    if (!(error instanceof Error)) {
        return { name: "NonErrorThrow", message: String(error) };
    }

    return {
        name: error.name,
        message: error.message,
        stack: error.stack,
        cause: error.cause instanceof Error
            ? {
                  name: error.cause.name,
                  message: error.cause.message,
                  stack: error.cause.stack,
              }
            : error.cause,
    };
}
```

实现时限制 cause 深度并处理循环引用。对客户端仍只返回 `{error,message}` 的通用 5xx，完整 stack 只进入受控日志。

## 15.12 文件 transport 与日志轮转

```ts
const fileTransport = new winston.transports.File({
    filename: "runtime/logs/api.log",
    maxsize: 100 * 1024 * 1024,
    maxFiles: 7,
    tailable: true,
});

fileTransport.on("error", (error) => {
    // 使用受控 stderr fallback，避免 logger.error 形成循环。
    console.error("winston file transport failed", error);
});
```

需要按日期轮转时可引入 `winston-daily-rotate-file`，但要锁定版本并配置文件名、日期、maxFiles、压缩和权限。每个进程/副本使用独立文件路径，容器优先 stdout 交给平台收集。

日志目录由部署创建并限制权限；不提交 runtime logs，不把日志卷当无限容量。

## 15.13 Winston 关闭与测试

正常退出时停止接收新业务日志，再按 transport 语义 flush/end。不要仅 `process.exit(0)`，否则异步写入可能丢失。

```ts
async function closeLogger(logger: winston.Logger): Promise<void> {
    await new Promise<void>((resolve, reject) => {
        const timer = setTimeout(() => reject(new Error("logger close timeout")), 5000);
        logger.once("finish", () => {
            clearTimeout(timer);
            resolve();
        });
        logger.end();
    });
}
```

实际项目需按所用 transports 验证 finish/close 行为，并避免 close timeout 后留下未处理 rejection。

Winston 单元测试使用 MemoryTransport 或自定义测试 transport：

- 验证 level 过滤和必要字段。
- 验证 authorization、cookie、token、URL userinfo 被脱敏。
- 验证 Error stack/cause 可序列化。
- 验证 Hono 请求日志只记录一次且包含 route template。
- 验证文件 transport error 和 shutdown 不会无限悬挂。

上一篇：[应用生命周期](10-application-lifecycle.md) | [返回总目录](../development-guide.md) | 下一篇：[性能与 Tinypool](12-performance-workers.md)
