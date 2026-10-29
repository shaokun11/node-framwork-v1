# API 文档（api.md）编写规范

项目的 `docs/api.md` 是 HTTP 消费方可以阅读和验证的接口契约，不是源码路由清单的复制。它必须描述当前实际发布的 API，并与 Hono 路由、Zod schema、错误映射和契约测试同步。

## 1. 文档定位与单一事实来源

- 对外 API 以 `docs/api.md` 或生成的 OpenAPI 文档为消费方入口，二者字段和示例必须一致。
- Zod schema 负责运行时验证，TypeScript 类型负责编译期约束，api.md 负责人类可读契约；三者不能互相替代。
- 如果使用 OpenAPI 生成，必须确定生成方向和评审流程，避免手工同时修改生成物和源文档造成漂移。
- 内部调试、私有指标和管理接口分别标注可见范围，不混入公开客户端文档。
- 文档只记录已经实现或明确标记为草案的能力，不能把计划接口写成已上线。

## 2. 推荐文件结构

小型项目维护一个 `docs/api.md`：

```text
# API 文档
1. 基本信息与 Base URL
2. 通用约定
3. 认证与权限
4. 成功、错误与分页
5. 业务接口（按模块）
6. Webhook / 异步回调（如有）
7. 错误码总表
8. 变更记录
```

大型项目可以按模块拆分，但 `docs/api.md` 保留统一入口、公共协议和模块链接。Postman/Bruno collection、OpenAPI JSON 和示例请求文件必须与当前版本同步。

## 3. 文档头部模板

```md
# 项目 API 文档

- API 版本：v1
- 文档状态：当前发布版 / 草案
- 最近更新：YYYY-MM-DD
- Base URL：`https://api.example.com`
- 本地地址：`http://127.0.0.1:3000`
- 内容类型：`application/json; charset=utf-8`
- 时间格式：UTC ISO 8601，例如 `2026-01-02T03:04:05.000Z`
- ID 格式：字符串，由服务端生成，客户端不得解析其内部结构

## 通用请求头

| Header | 必填 | 说明 |
| --- | --- | --- |
| Authorization | 受保护接口必填 | `Bearer <token>`，不得写入日志和示例真实值 |
| X-Request-ID | 否 | 客户端请求关联 ID；格式和长度限制见下文 |
| Idempotency-Key | 指定写接口必填 | 幂等键，作用域和有效期见各接口 |
```

Base URL 不包含需要客户端猜测的尾随斜杠规则。本地示例使用占位秘密，不复制 `.env` 的真实连接或 token。

## 4. 公共协议必须一次写清

### 4.1 成功响应

定义 envelope、HTTP 状态码使用方式和空响应。例如：

```json
{
    "data": { "id": "ord_123", "status": "pending" },
    "meta": { "requestId": "req_123" }
}
```

说明 `201` 是否带 `Location`、`204` 是否无 body、异步任务返回 `202` 后如何查询结果。不要只给 200 示例而遗漏创建、删除和异步语义。

### 4.2 错误响应

```json
{
    "error": "ORDER_NOT_FOUND",
    "message": "订单不存在"
}
```

错误响应体固定为 `{error,message}`：error 是稳定机器可读 code，message 用于人类阅读。请求 ID 通过 `X-Request-ID` 响应头返回并用于日志排查。文档不得公开 SQL、堆栈、详细内部 issue 和依赖错误。

### 4.3 分页、过滤与排序

| 项目 | 必须说明的内容 |
| --- | --- |
| limit | 默认值、最小值、最大值 |
| cursor | 获取方式、 opaque 属性、有效期、是否绑定过滤条件 |
| 排序 | 默认字段、方向、同值时 tie-breaker |
| hasMore/nextCursor | 空页和最后一页的表现 |
| 过滤 | 可过滤字段、格式、组合关系和非法值行为 |

不要让客户端自行构造游标 JSON 或依赖数据库自增 ID 泄漏业务规模。

## 5. 每个接口的标准条目

每个 operation 按相同顺序编写：

1. 方法和路径。
2. 一句话用途及业务前置条件。
3. 认证、权限、租户和资源归属。
4. 请求头、path、query、body schema。
5. 成功状态码、响应字段和示例。
6. 业务错误、限流和幂等行为。
7. 副作用、异步语义和兼容注意事项。

### 5.1 完整示例

````md
## 创建订单

`POST /v1/orders`

创建一个待支付订单。相同 principal 下重复提交相同 Idempotency-Key 和请求体时返回首次结果。

- 认证：Bearer session
- 权限：`order:create`
- 幂等：必需；key 有效期 24 小时；相同 key 不同 body 返回 `IDEMPOTENCY_KEY_REUSED`
- 限流：每个账号 10 次/分钟，超过返回 429 和 Retry-After

### 请求头

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| Idempotency-Key | string | 是 | 8128 个可打印 ASCII 字符 |

### 请求体

| 字段 | 类型 | 必填 | 约束 | 说明 |
| --- | --- | --- | --- | --- |
| items | array | 是 | 1100 项 | 订单商品 |
| items[].sku | string | 是 | 164 字符 | 商品 SKU |
| items[].quantity | integer | 是 | 11000 | 购买数量 |
| note | string | 否 | 最多 500 字符 | 用户备注 |

```json
{
    "items": [{ "sku": "sku_demo", "quantity": 2 }],
    "note": "示例订单"
}
```

### 201 Created

响应头：`Location: /v1/orders/ord_123`

```json
{
    "data": {
        "id": "ord_123",
        "status": "pending",
        "createdAt": "2026-01-02T03:04:05.000Z"
    },
    "meta": { "requestId": "req_123" }
}
```

### 错误

| HTTP | code | 触发条件 | 客户端处理 |
| --- | --- | --- | --- |
| 400 | VALIDATION_ERROR | 请求字段不符合 schema | 修正后重试 |
| 401 | UNAUTHORIZED | 凭证缺失或失效 | 重新认证 |
| 403 | FORBIDDEN | 无创建权限 | 不应盲目重试 |
| 409 | IDEMPOTENCY_KEY_REUSED | 同 key 对应不同请求体 | 使用新 key 并检查请求 |
| 422 | ORDER_ITEMS_UNAVAILABLE | 商品当前不可购买 | 刷新商品信息 |
| 429 | RATE_LIMITED | 超过账号配额 | 按 Retry-After 重试 |
````

示例只展示协议必要字段。完整字段表决定契约，不能靠示例反推哪些字段必填。

## 6. 字段书写规则

- 字段名、类型、必填、nullable、默认值、最小/最大长度、枚举和格式逐项列出。
- `optional` 与 `nullable` 分开：字段缺失和字段为 null 的行为不同。
- 数值明确是否整数、精度和单位；金额使用字符串定点数或明确的最小货币单位整数。
- 时间明确时区和精度；时长明确毫秒/秒；字节大小明确单位。
- 数组写明元素类型、长度、是否允许空数组、是否保序和是否允许重复。
- 枚举列出全部公开值，并说明新增枚举值时客户端应如何处理。
- 只写字段在 API 中的业务含义，不暴露数据库列、内部类和 Repository 名称。

## 7. cURL 示例与可执行验证

```sh
curl --fail-with-body --request POST "http://127.0.0.1:3000/v1/orders" \
    --header "Authorization: Bearer ${API_TOKEN}" \
    --header "Content-Type: application/json" \
    --header "Idempotency-Key: demo-order-001" \
    --data '{"items":[{"sku":"sku_demo","quantity":2}]}'
```

- 示例使用环境变量或占位符，不写真实 token。
- 同时提供成功和至少一个关键失败示例。
- 文档中的 curl、HTTP 文件或 collection 应纳入 smoke 验证，避免路径和字段长期漂移。
- Windows 用户的命令换行差异可以另附 PowerShell 示例，不让 shell 引号成为接口歧义。

## 8. 错误码总表

按 code、HTTP、含义、是否可重试和出现模块建立总表。某接口有额外触发条件时在接口条目补充。

| code | HTTP | 含义 | 可重试 |
| --- | --- | --- | --- |
| VALIDATION_ERROR | 400 | 输入未通过协议验证 | 修正请求后 |
| UNAUTHORIZED | 401 | 凭证无效或过期 | 重新认证后 |
| FORBIDDEN | 403 | 已认证但无权限 | 否 |
| NOT_FOUND | 404 | 资源不存在或不可见 | 否 |
| CONFLICT | 409 | 当前状态与操作冲突 | 状态改变后 |
| RATE_LIMITED | 429 | 超过请求配额 | 按 Retry-After |
| DEPENDENCY_UNAVAILABLE | 503 | 必需依赖暂不可用 | 幂等且有退避时 |

不要为每个内部异常创建公开 code。是否可重试还取决于操作幂等性，不能只看 5xx。

## 9. 变更记录与评审

每次 API 变更记录日期、版本、模块、变更内容和兼容影响。字段删除、重命名、类型改变、必填变化、权限收紧和错误语义改变必须醒目标记。

评审清单：

- [ ] 文档中的路由、方法、字段、枚举和状态码与实现一致。
- [ ] 认证、权限、租户、限流、幂等和分页没有遗漏。
- [ ] 请求/响应示例通过 Zod schema 或契约测试验证。
- [ ] 错误码与集中错误映射一致，不包含内部实现细节。
- [ ] 新变更包含兼容说明、迁移窗口和客户端升级要求。
- [ ] 文档中的 smoke 请求在干净开发环境实际执行。

上一篇：[Hono HTTP 与 Zod API](02-http-api.md) | [返回总目录](../development-guide.md) | 下一篇：[配置加载](04-configuration.md)
