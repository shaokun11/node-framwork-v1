# 安全、Winston 日志与可观测性

## 14. 日志、安全与可观测性

### 14.1 结构化日志

统一采用 Winston Logger，由 bootstrap 创建并注入。每条请求日志至少包含 requestId、method、route template、status、duration 和 outcome；错误日志包含稳定 error code 和内部 cause。

不要记录原始 URL query、Authorization、Cookie、密码、连接 URL、完整请求体、token、个人隐私和秘密配置。SQL debug 默认关闭；即使开启也必须考虑参数敏感信息，不能在生产无差别打印完整参数。

日志应能区分客户端输入失败、权限拒绝、业务冲突、依赖超时、依赖不可用和程序错误。请求失败只在最终边界记录一次，避免每层重复 error 日志。

### 14.2 安全基线

- 密码使用 Argon2id 或 bcrypt 等合适算法和参数，不自行设计密码散列。
- Token/session 的签名密钥、加密密钥和哈希用途分离；密钥可轮换。
- 登录失败对未知账号和错误密码保持一致的公开响应，并考虑时间侧信道。
- 所有权限检查包含资源归属，不能只验证已登录。
- 输出编码、安全 header、CORS、CSRF、请求体限制、上传类型/大小和 SSRF 防护按路由落实。
- 数据库和 Redis 使用最小权限账号，开发管理员凭据不能复用于生产。
- 依赖安装、脚本执行和 CI token 按供应链安全要求控制；定期检查漏洞但不能只靠扫描。
- 错误消息不泄露 SQL、schema、内部主机、堆栈和第三方秘密。

### 14.3 健康检查与指标

- `/livez` 只表示进程未进入不可恢复状态，不查询所有依赖。
- `/readyz` 检查当前服务真正必需的依赖和启动状态；缓存是否必需由降级策略决定。
- 健康检查有超时、缓存合理结果并避免对数据库造成放大流量。
- 指标使用 route template，不把原始 path、订单 ID、用户 ID 作为 label。
- 至少观察请求速率/错误/延迟、数据库 Pool、关键依赖错误、缓存 hit/miss 和 Worker 队列/耗时。
- 需要分布式追踪时统一 trace/span 语义并传播上下文，不能只安装 SDK 就视为完成。

## 14.4 Winston 日志实现规范

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

## 14.5 日志脱敏与错误边界

Winston 的 redaction format 在日志写入前执行，至少覆盖：

- headers 中的 authorization、cookie、set-cookie、x-api-key。
- body/config 中的 password、secret、token、databaseUrl、mongodbUrl、redisUrl。
- 连接 URL 的 userinfo、查询中的 token 和签名参数。

redaction 采用递归键白名单/黑名单策略，并限制深度和循环引用。不能依赖某一层已经脱敏；第三方 error 也可能包含 URL 或请求配置。

开发 debug SQL/请求体日志仍需经过同一 redaction。生产临时开启 debug 必须有期限和审计。

## 14.6 安全事件与审计

登录成功/失败、权限拒绝、账号锁定、秘密轮换、管理员操作和数据导出等安全事件应有结构化审计日志。审计记录 actor、action、target type/id、requestId、result 和时间，但不记录密码、完整 token 或不必要的个人数据。

审计日志的业务保留期、防篡改和访问权限由合规需求决定，普通 console 日志不自动满足审计要求。

## 14.7 指标与告警实施

| 指标 | 建议维度 | 主要用途 |
| --- | --- | --- |
| http_requests_total | method、route、status class | 流量和错误率 |
| http_request_duration_ms | method、route | 延迟分布与 SLO |
| db_pool_clients | state | 池耗尽和泄漏 |
| cache_operations_total | operation、result | hit/miss/error |
| redis_operations_total | operation、result | 依赖失败 |
| worker_tasks_total | task、result | Worker 饱和与失败 |

使用 histogram 观察延迟，不只记录平均值。告警结合错误率、延迟分位数、饱和度和业务影响；单次 cache miss 不应触发页面告警，持续写后缓存删除失败可能需要。

## 14.8 安全与可观测性验收

- [ ] Winston 输出结构稳定，Error/cause 和 requestId 可查询。
- [ ] 秘密、Cookie、Authorization 和连接 URL 的脱敏有单元测试。
- [ ] live/ready 不会放大依赖流量，多实例指标 label 无高基数。
- [ ] 认证、授权、租户隔离、上传和 SSRF 有对抗性测试。
- [ ] 安全审计事件和日志保留满足项目要求。
- [ ] shutdown 能结束 Winston transports，不因日志 I/O 无限悬挂。

上一篇：[应用生命周期](09-application-lifecycle.md) | [返回总目录](../development-guide.md) | 下一篇：[性能与 Tinypool](11-performance-workers.md)
