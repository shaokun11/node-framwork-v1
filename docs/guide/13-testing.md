# Vitest 测试策略与工程检查

## 17. 测试与工程检查

### 17.1 测试分层

| 层级 | 主要目标 | 依赖方式 |
| --- | --- | --- |
| 单元测试 | 领域规则、配置合并、DTO、错误映射、纯函数 | fake 实现，快速且确定 |
| Repository 契约 | CRUD、唯一约束、事务、并发、映射 | 当前选型的真实 PostgreSQL 或 MongoDB |
| Adapter 契约 | Keyv TTL、namespace、clear、序列化、错误 | 每个实际启用的 Keyv Adapter |
| Redis 集成 | Lua 原子性、TTL、锁所有权、Stream ACK | 真实 Redis，必要时 Cluster |
| HTTP 集成 | 中间件、Zod、权限、状态码、响应 envelope | `app.request()` + 可控业务依赖 |
| E2E | 真实进程、配置、迁移、健康检查、完整链路 | 编译产物和真实依赖 |
| 生命周期 | 启动失败清理、信号、请求排空、关闭超时 | 独立进程与信号测试 |

单元测试通过不等于数据库行为正确。不要用 mock 验证 SQL、Mongoose 查询或 Redis 原子性，也不要让所有测试共享一个无隔离数据库。

### 17.2 必测行为

每个业务写接口至少覆盖：

- 正常创建/更新、非法输入、未认证、无权限和非资源所有者。
- 不存在、重复、状态冲突和并发更新。
- 幂等重复、相同 key 不同请求体、并发首次请求。
- 事务中途失败后的回滚和可重试性。
- 下游超时、缓存不可用、Redis 不可用时的明确降级策略。
- 多实例共享身份、配额和事实数据，不依赖本地进程状态。

配置加载必须独立覆盖第 8.5 节列出的优先级和非法输入。数据库测试验证真实约束、事务、索引假设和迁移升级；MongoDB 事务测试使用支持事务的拓扑。

### 17.3 隔离与确定性

- 使用 Testcontainers 或受控 Compose 启动真实依赖，测试后清理。
- 每个测试套件使用独立 database/keyspace/collection/namespace；禁止随机清理开发者数据库。
- 测试配置只能指向临时资源，启动前检查环境标识和地址安全边界。
- 时间通过 Clock fake，ID 通过 IdGenerator fake；不把 sleep 当同步机制。
- 并发测试使用 barrier/latch 制造真实交错，重复运行验证稳定性。
- 网络测试限制总耗时，失败时保留必要日志但不泄密。
- snapshot 适合稳定协议，不代替关键字段和失败行为断言。

### 17.4 工程检查

```sh
npm run format:check
npm run lint
npm run typecheck
npm run test
npm run build
```

- Prettier 负责格式，ESLint 负责代码问题，TypeScript 负责类型，三者不可相互替代。
- ESLint 至少启用 floating promises、unsafe any、misused promises 等类型感知规则。
- `git diff --check` 检查空白错误；生成代码和迁移必须与工作源一致。
- 测试覆盖率用于发现未测分支，不设一个数字代替关键行为验收。
- PR 必须说明验证范围；未运行的集成/E2E 测试明确列出，不能写成全部通过。

## 17.5 Vitest 项目组织

所有测试统一使用 Vitest，不混用 Jest、node:test 或另一套断言/mock 体系。按依赖和运行成本拆分配置：

```text
src/**/*.test.ts                  # 单元测试，靠近源码
test/integration/**/*.test.ts     # 真实数据库/Redis/Keyv 集成
test/e2e/**/*.e2e.test.ts         # 完整业务链路
test/smoke/**/*.smoke.test.ts     # 最小发布探活
```

建议脚本：

```json
{
    "scripts": {
        "test": "npm run test:unit",
        "test:unit": "vitest run --config vitest.unit.config.ts",
        "test:integration": "vitest run --config vitest.integration.config.ts",
        "test:e2e": "vitest run --config vitest.e2e.config.ts",
        "test:smoke": "vitest run --config vitest.smoke.config.ts",
        "test:watch": "vitest --config vitest.unit.config.ts",
        "test:coverage": "vitest run --config vitest.unit.config.ts --coverage"
    }
}
```

`npm test` 默认快速单元测试；CI 和交付流程显式运行 integration、e2e 和 smoke。不要让 `npm test` 暗中依赖开发者本机数据库。

## 17.6 Vitest 单元测试

- 使用 `describe`/`it`/`expect` 表达行为和边界，不逐行镜像实现。
- 使用 `vi.fn()`、`vi.spyOn()` 验证必要的 collaborator 交互，但优先断言输出和状态。
- 使用 `vi.useFakeTimers()` 测试 timeout/debounce 时，在 afterEach 恢复真实 timer。
- 时间优先通过注入 Clock fake；必须 mock 系统时间时保持测试隔离。
- 使用 `vi.resetModules()` 等模块级隔离功能要说明原因，避免掩盖全局状态设计问题。
- 每个测试独立构造 Service，不共享可变 Repository fake。

```ts
describe("OrderService", () => {
    it("returns the original order for an existing idempotency key", async () => {
        const repository = new InMemoryOrderRepository([existingOrder]);
        const service = new OrderService(repository, new FixedIds(), new FixedClock());

        await expect(service.create(existingInput)).resolves.toEqual(existingOrder);
        expect(repository.createCallCount).toBe(0);
    });
});
```

并发数据库冲突不能靠 InMemoryRepository 单线程行为证明，必须放到真实集成测试。

## 17.7 Vitest HTTP 集成测试

路由/中间件测试通过 `app.request()` 调用 Hono，无需监听端口：

```ts
const response = await app.request("/v1/orders", {
    method: "POST",
    headers: { "content-type": "application/json", "idempotency-key": "key-1" },
    body: JSON.stringify(validBody),
});

expect(response.status).toBe(201);
await expect(response.json()).resolves.toMatchObject({ data: { status: "pending" } });
```

测试覆盖 Zod 错误 envelope、CORS、认证、权限、body limit 和集中错误映射。Repository 集成测试则使用真实数据库，两者不要混成难以定位的大测试。

## 17.8 Vitest E2E 测试

E2E 验证真实应用进程、真实配置和当前选型基础设施：

1. 使用 Docker Compose/Testcontainers 启动隔离依赖并执行迁移。
2. 使用随机可用端口和专用测试秘密启动 `node dist/main.js`。
3. 有超时地轮询 ready，失败时保留脱敏进程日志。
4. 通过真实 HTTP 完成注册/登录/创建/查询等关键链路。
5. 验证权限、幂等和持久化，不只检查 200。
6. afterAll 发送 SIGTERM，确认进程退出并清理依赖。

E2E 可以用 Vitest globalSetup 管理环境，也可以由脚本先启动 Compose；无论哪种方式都要避免并行套件删除彼此数据。

## 17.9 Vitest Smoke 测试

Smoke 是最小、快速、面向发布产物的能否运行验证，不重复完整 E2E：

- 使用生产配置形态启动编译后的进程。
- `/livez` 和 `/readyz` 符合预期。
- 一个最关键的公开路由可用。
- 一个需要主数据库的核心读写路径可用（若项目有数据库）。
- 启动日志无 fatal/error，SIGTERM 能在规定时间退出。

```ts
describe("production smoke", () => {
    it("starts, becomes ready, and stops cleanly", async () => {
        const app = await startBuiltApplication();
        try {
            const ready = await waitForReady(app.baseUrl);
            expect(ready.status).toBe(200);
        } finally {
            await app.stop();
        }
    }, 30_000);
});
```

Smoke 测试数据独立且可清理。部署后对远端环境执行 smoke 时，只使用专用测试账号/资源，不能随意创建生产业务数据。

## 17.10 Vitest 验收清单

- [ ] test:unit 无需外部服务且可重复运行。
- [ ] test:integration 使用真实当前数据库、Keyv Adapter 和 Redis。
- [ ] test:e2e 覆盖完整关键业务链路和权限/幂等。
- [ ] test:smoke 验证 dist 启动、健康检查、核心路由和优雅退出。
- [ ] Vitest 配置、setup、teardown 和超时隔离清晰。
- [ ] 测试失败输出不包含秘密，清理逻辑不会删除非测试资源。

上一篇：[TypeScript 规范](12-typescript-conventions.md) | [返回总目录](../development-guide.md) | 下一篇：[开发与交付](14-development-delivery.md)
