# TOML 与环境变量配置加载

## 8. TOML 与环境变量配置加载

### 8.1 配置来源与优先级

配置只由 `ConfigLoader` 读取。业务类通过构造函数接收冻结后的强类型配置或必要子配置，不得直接读取 `process.env`、`.env` 或 TOML。

普通配置与敏感配置采用不同优先级，参考 Go 项目的加载方式：

| 配置类型 | 从低到高的优先级 |
| --- | --- |
| 普通设置 | 内置默认值 < 工作目录 `config.toml` < 第一个 `--config` < 后续 `--config` |
| 敏感设置 | TOML/默认值 < `.env` < 进程环境变量 |

重复 `--config` 必须按命令行出现顺序合并，后加载文件覆盖先加载文件。`.env` 只补充本地开发秘密，绝不覆盖已经存在的进程环境变量。生产环境推荐通过密钥管理器注入进程环境，不发布 `.env`。

普通监听地址、超时、池大小、日志级别等默认不通过环境变量覆盖，避免同一配置同时存在多套不明确覆盖规则。项目确需全部配置环境化时，应单独修改规范，明确嵌套键映射和完整优先级。

### 8.2 TOML 解析与合并

使用 `smol-toml` 解析 TOML，不使用 YAML，也不手写 TOML 解析器。.env 文件使用 dotenv 解析器，加载时不覆盖进程环境，也不手写解析器。加载步骤固定为：

1. 在创建 logger 和连接前解析最小 CLI 参数，收集重复的 `--config <path>`。
2. 读取内置默认值。
3. 若工作目录 `config.toml` 存在则读取；显式指定的文件不存在必须报错。
4. 按顺序读取显式文件并深度合并。
5. 单独读取 `.env`，再叠加进程环境中的秘密变量。
6. 将所有来源合并为一个普通对象，用 Zod 做最终严格校验和类型转换。
7. 返回深冻结配置；启动日志只输出脱敏后的有效配置摘要。

合并规则：普通 table 递归合并；字符串、数字、布尔值、日期和数组整体替换。数组不做按下标拼接。未知键默认拒绝，防止 `poolSize` 写成 `poolsize` 后静默使用默认值。TOML 原生日期只能用于确实需要日期语义的配置，网络传输仍转换为明确格式。

不得递归合并不可信对象到全局原型；加载器只接受 TOML 解析结果，并使用无原型或安全合并方式。

### 8.3 配置示例

```toml
# config.example.toml：只放非秘密示例。
[app]
name = "order-api"
environment = "development" # development | test | production

[server]
host = "127.0.0.1"
port = 3000
bodyLimit = "1MiB"
requestTimeout = "5s"
shutdownTimeout = "15s"

[database]
driver = "postgres" # postgres | mongodb

[database.postgres]
poolMax = 10
connectionTimeout = "5s"
idleTimeout = "30s"

[database.mongodb]
connectTimeout = "5s"
autoIndex = false

[cache]
enabled = true
adapter = "redis" # memory | redis | postgres | mongo | sqlite | 其他明确安装的 adapter
namespace = "order-api:cache"
defaultTtl = "5m"
throwOnErrors = true

[redis]
enabled = false
keyPrefix = "order-api:"
connectTimeout = "5s"

[logging]
level = "info" # debug | info | warn | error
format = "json"
redactPaths = ["req.headers.authorization", "req.headers.cookie"]
```

```dotenv
# .env.example：只提供占位，不填写真实秘密。
APP_DATABASE_URL=postgresql://user:replace-me@127.0.0.1:5432/app
APP_MONGODB_URL=mongodb://user:replace-me@127.0.0.1:27017/app
APP_REDIS_URL=redis://:replace-me@127.0.0.1:6379/0
APP_CACHE_URL=redis://:replace-me@127.0.0.1:6379/0
APP_SESSION_SECRET=replace-with-a-long-random-value
```

只要求当前选型需要的 URL：`driver = "postgres"` 时校验 `APP_DATABASE_URL`，`driver = "mongodb"` 时校验 `APP_MONGODB_URL`；`cache.adapter = "redis"` 时校验缓存 Adapter 所需 URL；`redis.enabled = true` 时独立校验 `APP_REDIS_URL`。Keyv 缓存连接和 Redis 原生客户端可以使用不同实例、数据库和权限，不能默认共享配置。

### 8.4 Zod 配置契约

```ts
const configSchema = z
    .object({
        app: z.object({
            name: z.string().min(1),
            environment: z.enum(["development", "test", "production"]),
        }),
        server: z.object({
            host: z.string().min(1),
            port: z.number().int().min(1).max(65535),
            requestTimeoutMs: z.number().int().positive(),
            shutdownTimeoutMs: z.number().int().positive(),
        }),
        database: z.object({ driver: z.enum(["postgres", "mongodb"]) }),
        // 其余子配置按相同方式严格声明。
    })
    .strict();

export type AppConfig = z.infer<typeof configSchema>;
```

实际 schema 必须包含全部配置，并通过 `superRefine` 校验跨字段规则，例如当前 driver 的连接 URL 存在、生产秘密不是示例值、超时范围合理、pool max 为正数。

时长在 TOML 中用 `"5s"`、`"500ms"` 等受限字符串，加载后统一转成毫秒整数；大小限制统一转成字节。禁止到处调用解析函数。布尔值只接受 TOML 原生 boolean 或 env 中明确约定的 `true`/`false`，不接受 `yes`、`on` 等隐式转换。

### 8.5 秘密与配置测试

- `.env`、本地 `config.toml`、真实证书和密钥必须进入 `.gitignore`；提交 `.env.example` 和 `config.example.toml`。
- 日志、异常、诊断路由和测试快照不得输出连接 URL、密码、Cookie、Authorization 和完整配置对象。
- 配置缺失或非法时启动失败，不能在 production 自动退回弱默认值。
- 启动时解析一次并冻结，不监听文件自动热改数据库和认证配置；需要动态配置的功能必须单独设计。
- 测试覆盖默认值、多个 TOML 覆盖、数组替换、`.env` 与进程环境优先级、未知键、空值、非法时长、缺失当前选型 URL 和秘密脱敏。
- 测试不得修改开发者真实环境；通过 loader 参数传入文件系统、dotenv 值和 env 对象，并隔离临时目录。

## 8.6 Loader 输入与输出契约

建议让 Loader 显式接收外部输入，而不是在内部直接绑定全局进程：

```ts
interface LoadConfigOptions {
    cwd: string;
    configPaths: readonly string[];
    dotenvValues: Readonly<Record<string, string>>;
    processEnv: Readonly<Record<string, string | undefined>>;
}

interface LoadedConfig {
    value: AppConfig;
    sources: readonly ConfigSource[];
}
```

`ConfigSource` 只记录文件路径、来源类型和非敏感键的覆盖来源，用于诊断这个值从哪里来。秘密键可以记录来源名称，但绝不记录原值。

Loader 输出与 `process.env` 分离后，单元测试可以并行测试不同环境，不会互相污染。`main.ts` 只负责把真实 argv、文件系统和环境接入 Loader。

## 8.7 CLI 与文件加载细则

- 只解析 `--config` 和必要的早期参数，业务命令参数由后续 CLI 层处理。
- `--config` 缺少路径、路径为目录、读取失败和 TOML parse error 都是启动错误。
- 默认 `config.toml` 不存在可以忽略；显式文件不存在不能忽略。
- 相对路径统一相对启动工作目录或明确的项目根目录，不能因 import 文件位置不同而变化。
- 同一个文件重复传入按重复来源处理或明确去重，行为需测试。
- 配置加载发生在 logger 初始化前时，使用简洁 stderr 启动错误；logger 创建后补充脱敏上下文。

## 8.8 环境变量映射与秘密轮换

秘密变量采用白名单映射，不把所有 `APP_*` 自动展开为任意配置路径：

| 环境变量 | 配置用途 | 何时必需 |
| --- | --- | --- |
| APP_DATABASE_URL | PostgreSQL 连接 | driver=postgres |
| APP_MONGODB_URL | MongoDB 连接 | driver=mongodb |
| APP_CACHE_URL | Keyv Adapter 连接 | 当前 Adapter 需要远程连接时 |
| APP_REDIS_URL | 独立 Redis client | redis.enabled=true |
| APP_SESSION_SECRET | session 签名/加密 | 启用对应认证方式时 |

需要秘密轮换时，配置结构应支持 current/previous key 或独立密钥提供器，并明确旧 token 的宽限期。不能只修改一个字符串后假定所有已签发凭证仍然有效。

URL 密码包含特殊字符时按 URL 编码规则处理。启动摘要不打印完整 URL；即使只打印 host，也要先安全解析并移除 userinfo。

## 8.9 配置变更实施步骤

新增一个配置项时必须同时完成：

1. 在 defaults 或示例 TOML 声明含义与默认值。
2. 在 Zod schema 声明类型、范围、未知键策略和跨字段规则。
3. 若为秘密，加入环境变量白名单和 `.env.example` 占位。
4. 在对应 bootstrap 工厂消费，不让业务类重新读取来源。
5. 增加默认值、覆盖、非法值和来源优先级测试。
6. 更新 README、Compose、部署变量和脱敏启动摘要。
7. 检查删除旧配置时的启动兼容和错误提示。

## 8.10 配置故障示例

- `driver=mongodb` 但只提供 `APP_DATABASE_URL`：启动失败并指出缺少 `APP_MONGODB_URL`，不能悄悄连接 PostgreSQL。
- `cache.adapter=memory` 且 API 多实例需要共享限流：配置本身可以合法，但架构评审必须拒绝把该缓存当共享状态。
- `redis.enabled=false` 但配置原生 Redis 限流：跨字段校验失败。
- production 使用 `APP_SESSION_SECRET=replace-me`：启动失败。
- TOML 写错 `shutdownTimeOut`：因未知键失败，不回退默认 15 秒。

上一篇：[API 文档规范](03-api-documentation.md) | [返回总目录](../development-guide.md) | 下一篇：[PostgreSQL + Drizzle](05-postgresql-drizzle.md)
