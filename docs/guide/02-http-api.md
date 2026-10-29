# Hono HTTP 与 Zod API 规范

## 7. Hono HTTP 规范

### 7.1 应用与路由注册

- 使用 `@hono/node-server` 的 `serve({ fetch: app.fetch, port, hostname })` 启动 Node.js 服务。
- `Hono` 实例由 `HttpApplication` 或 `Router` 创建，不在各业务模块创建互相孤立的全局 app。
- 每个模块实现统一的 `RouteModule.register(app)` 契约；bootstrap 提供模块列表。
- 路由顺序从具体到宽泛，避免 `/:id` 吞掉固定路径。
- 公共中间件按明确顺序注册；认证前执行的日志、请求 ID、限流策略必须记录其行为。

推荐公共顺序：请求 ID  安全响应头  可信代理处理  请求日志开始  请求体限制  CORS  路由级认证/权限  DTO 验证  Controller  集中错误处理。实际顺序需结合是否对未认证请求限流、CORS preflight 是否免认证进行评审。

### 7.2 Zod 边界验证

Zod 是默认的运行时验证与类型推导工具，至少覆盖四个边界：

1. 启动配置和敏感环境变量。
2. HTTP path、query、header、JSON body、multipart 元数据。
3. 外部 HTTP、Webhook、队列 payload 和缓存反序列化。
4. 测试 fixture 或 CLI 输入等不可信数据。

```ts
export const createOrderBodySchema = z
    .object({
        items: z
            .array(
                z.object({
                    sku: z.string().trim().min(1).max(64),
                    quantity: z.number().int().positive().max(1000),
                }),
            )
            .min(1)
            .max(100),
        note: z.string().trim().max(500).optional(),
    })
    .strict();

export type CreateOrderBody = z.infer<typeof createOrderBodySchema>;
```

- 请求对象默认 `.strict()` 或显式剥离未知键，安全敏感字段必须拒绝未知键。
- query 和 env 中的文本不能直接当 number/boolean；使用明确的预处理，拒绝空串和含糊布尔值。
- Zod 验证不能代替数据库约束、对象归属检查或业务不变量。
- 对外错误响应只返回稳定的 error code 和安全、可读的 message；详细 Zod issue、堆栈和原始输入不直接暴露给客户端。
- 不用 `as SomeType` 把未验证值伪装为可信值；优先 `schema.parse()`，需要统一失败分支时使用 `safeParse()`。

### 7.3 成功与错误响应

建议成功响应：

```json
{
    "data": { "id": "ord_123", "status": "pending" },
    "meta": { "requestId": "req_123" }
}
```

建议错误响应：

```json
{
    "error": "ORDER_ITEMS_INVALID",
    "message": "订单商品不符合要求"
}
```

全项目选择一种 envelope 后保持一致。错误响应体固定为 `{error,message}`，其中 error 是稳定机器可读 code，message 是安全的人类可读信息。请求 ID 通过 `X-Request-ID` 响应头和日志关联。列表分页的 `meta` 使用 `nextCursor`、`hasMore` 等字段；游标必须签名或严格解析，限制 page size。删除成功可使用 `204` 且无响应体。

建立统一 `AppError` 分类，例如 `VALIDATION_ERROR`、`UNAUTHORIZED`、`FORBIDDEN`、`NOT_FOUND`、`CONFLICT`、`RATE_LIMITED`、`DEPENDENCY_UNAVAILABLE`。集中错误处理器完成：

- 领域错误到 HTTP 状态和公开 code 的映射。
- JSON 解析、Zod、Hono `HTTPException` 的映射。
- 5xx 记录内部原因，对客户端只返回通用消息。
- HEAD、已提交响应和客户端断开等特殊情况。
- 未知错误兜底，不能泄漏 SQL、连接 URL、堆栈和秘密。

不要用 `Error.message` 充当稳定机器可读 code；不要把所有失败都转成 200。

### 7.4 中间件和常见策略

- 请求 ID：接受格式受限的外部 ID 或自行生成，写入日志和响应头。
- 请求体：默认 JSON，设置字节上限；只有声明的路由接受 multipart。
- 超时：返回超时响应不够，数据库、Redis、HTTP 和 Worker 调用必须能取消或停止等待。
- CORS：只允许可信 origin，凭证模式不使用通配符。
- 认证：验证 token/session 后生成 principal；授权同时检查动作和资源归属。
- CSRF：Cookie 认证需要 SameSite、Origin 校验或 token；Bearer API 记录自身跨站边界。
- 限流：多实例部署用共享且原子的实现，进程内 token bucket 只适用于单实例或非严格保护。
- 幂等：写接口明确 key 的作用域、请求指纹、有效期、并发处理和响应重放规则。

### 7.5 API 文档

每个路由至少记录 method、path、认证、权限、请求 schema、成功/失败响应、幂等、限流和示例。可以使用 Zod OpenAPI 生态生成文档，但生成前必须验证 Node/Hono/Zod 版本兼容，生成结果也要经过契约测试和人工审查。

## 7.6 请求解析实施细则

对 JSON 请求先检查 `Content-Type`，再读取 body 并交给 Zod。 malformed JSON、超出 body limit 和 schema invalid 是不同错误类别，公开 code 应可区分。

Path/query 示例：

```ts
const orderParamsSchema = z.object({ orderId: z.string().regex(/^ord_[a-zA-Z0-9]+$/) });
const orderQuerySchema = z
    .object({
        cursor: z.string().max(512).optional(),
        limit: z.coerce.number().int().min(1).max(100).default(20),
    })
    .strict();
```

`z.coerce` 只用于协议明确的字符串转换。不要对任意配置或布尔值使用隐式 coercion；例如 `Boolean("false")` 结果为 true，不能作为布尔解析。

文件上传还需验证实际字节数、允许 media type、文件名处理、临时文件清理和存储归属。扩展名和客户端 Content-Type 都不能单独证明内容安全。

## 7.7 认证、权限与多租户

建议将过程拆为：

1. AuthenticationMiddleware 验证凭证并建立 principal。
2. AuthorizationPolicy 判断 principal 能否执行动作。
3. Service/Repository 用 ownerId/tenantId 限定资源查询。

```text
错误：先按 id 读取资源，再只检查 principal 已登录
正确：按 id + tenantId/ownerId 查询，并执行动作级权限检查
```

为防止跨租户 ID 枚举，可按 API 安全策略统一返回 `NOT_FOUND`。管理员工具和普通用户使用不同权限策略，不复用一个可布尔绕过的 `isAdmin` 参数。

多租户应用的 tenantId 必须来自已验证 principal 或受信路由上下文，不能信任请求 body 自报的 tenantId。数据库查询、缓存 key、Redis key、日志和指标都要检查租户边界；高基数 tenantId 不直接作为指标 label。

## 7.8 幂等协议详细约定

写接口使用 Idempotency-Key 时至少记录：

- key 的作用域：通常是 principal + operation + key。
- 请求指纹：规范化 method、path 和 body，防止同 key 被用于不同请求。
- 状态：processing、succeeded、failed，以及哪些失败允许重试。
- 保存的响应：状态码和响应体，重放时保持一致。
- TTL：与业务重试窗口一致，过期后语义明确。
- 并发：第一个请求未完成时返回 409/425 或等待，按契约选择。

唯一约束是最终防线。仅把 key 存入 Keyv 或进程 Map 不能保证多实例和崩溃恢复下的幂等。

## 7.9 版本、兼容和错误码

- URI、header 或 media type 版本策略只选一种，避免混用。
- 新增可选请求字段通常兼容，新增必填字段不兼容。
- 响应新增字段对严格客户端也可能不兼容，需要确认消费方。
- 错误 code 是稳定契约，不因内部异常文案改变；message 可以改进。
- 不建议把数据库 SQLSTATE、Mongoose error name 或第三方 code 直接暴露为公共 code。
- 429 可返回标准 Retry-After；503 的客户端重试必须与幂等策略配合。

## 7.10 HTTP 评审清单

- [ ] 所有不可信输入在边界解析，未知字段处理明确。
- [ ] 路由中间件顺序、认证豁免和 CORS preflight 已验证。
- [ ] 响应 envelope、错误 code、分页和状态码一致。
- [ ] 权限包含资源归属与租户约束。
- [ ] 写接口的幂等、限流、请求体和超时已定义。
- [ ] API 文档、契约测试和真实路由保持一致。

上一篇：[架构、目录与模块开发](01-architecture.md) | [返回总目录](../development-guide.md) | 下一篇：[配置加载](04-configuration.md)
