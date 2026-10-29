# 开发、Docker Compose、构建与交付

## 18. 开发、构建与交付

### 18.1 包管理和 Node 版本

项目选择 npm、pnpm 或 yarn 之一并提交对应 lockfile，禁止混用多个 lockfile。以下以 npm 为例。Node 主版本在 `package.json#engines`、`.nvmrc`、Dockerfile 和 CI 保持一致。

依赖分为运行时和开发时：`tsx`、TypeScript、Vitest、ESLint、Prettier、Drizzle Kit 等通常为 devDependencies；Hono、Zod、Drizzle ORM、Mongoose、Keyv、Redis 等按实际运行路径为 dependencies。

### 18.2 tsx 开发执行

开发环境统一使用 `tsx watch` 直接执行 TypeScript：

```json
{
    "scripts": {
        "dev": "tsx watch src/main.ts",
        "build": "tsc -p tsconfig.build.json",
        "start": "node dist/main.js",
        "typecheck": "tsc -p tsconfig.json --noEmit",
        "test": "vitest run",
        "test:watch": "vitest",
        "lint": "eslint .",
        "format": "prettier --write .",
        "format:check": "prettier --check ."
    }
}
```

- `tsx` 负责执行，不做完整类型检查；提交前仍必须运行 `typecheck`。
- `tsx watch` 只用于开发，生产入口是编译后的 `node dist/main.js`。
- watch 重启要验证旧进程收到信号并释放端口和连接，避免多个实例占用同一资源。
- Worker、迁移和静态资源路径必须同时验证 tsx 和编译产物，不以 import.meta.url 猜错目录。
- 传多个配置文件可使用 `npm run dev -- --config ./config.local.toml --config ./config.override.toml`；loader 按顺序覆盖。

数据库脚本按选型添加，例如 PostgreSQL 的 `db:generate`、`db:migrate`；MongoDB 提供独立版本化迁移命令。脚本名称和参数写入项目 README，不要求未选型的数据库命令存在。

### 18.3 本地开发流程

1. 安装指定 Node 和包管理器版本，使用 lockfile 执行干净安装。
2. 复制 `config.example.toml` 和 `.env.example`，只填写当前选型和需要的秘密。
3. 启动当前主数据库；只有启用相应功能时启动 Keyv Adapter 后端或独立 Redis。
4. 执行受控迁移，启动 `npm run dev`。
5. 检查 `/livez`、`/readyz` 和一条真实业务链路。
6. 修改代码时先补失败测试，再实现，再运行对应层测试。
7. 提交前执行格式、lint、类型、测试、build，并检查 diff 无秘密和无关生成物。

本地 Compose 只启动当前选型需要的服务，暴露端口尽量绑定 loopback，不使用生产秘密。开发数据库和缓存视为可丢弃，正式环境备份与恢复另行验证。

### 18.4 构建和 Docker

- 生产构建只编译必要源码和 Worker，检查 `dist` 入口、source map 策略和声明文件是否需要发布。
- Docker 使用多阶段构建，基于 lockfile 安装，生产阶段只保留运行时依赖和构建产物。
- 不以 root 运行，不写死秘密，使用只读根文件系统和最小系统权限（应用确需写入的目录除外）。
- 迁移文件、Mongoose 迁移脚本和 Worker 文件按实际运行方式复制；不能只复制 `dist/main.js`。
- 容器停止发送 SIGTERM 时验证优雅退出；`stop_grace_period` 大于应用 shutdown timeout。
- 健康检查区分进程存活和依赖就绪，避免依赖短暂故障造成重启风暴。
- 镜像固定基础镜像策略并扫描漏洞，构建结果记录版本、commit 和依赖锁。

### 18.5 分支、评审和 CI

- 一个变更聚焦一个可评审目标，迁移、生成物和格式化避免混入无关改动。
- commit 使用清晰约定，例如 Conventional Commits；不提交秘密、本地配置和临时产物。
- PR 描述行为、配置、数据库/缓存兼容性、回滚和测试证据。
- 评审重点包括业务不变量、权限、事务、并发、错误协议、资源关闭和日志泄密。
- CI 至少运行干净安装、格式检查、lint、typecheck、单元测试和 build；真实依赖集成测试按项目流水线执行。
- 发布采用 expand/contract 和可回滚应用版本。数据库回滚不能只靠向下迁移恢复已丢失数据。

## 18.6 Docker Compose 开发环境

开发环境默认采用依赖运行在 Docker Compose，API 在宿主机通过 tsx watch 运行的方式。这样保持 Node 调试和文件监听简单，同时让 PostgreSQL、MongoDB、Redis 等依赖可重复创建。

项目提供 `compose.dev.yaml`，只用于可丢弃的本地开发环境，不直接作为生产 Compose。

### 18.6.1 Compose 示例

```yaml
name: hono-api-dev

services:
    postgres:
        image: postgres:17-alpine
        profiles: ["postgres"]
        environment:
            POSTGRES_USER: app
            POSTGRES_PASSWORD: local-only-password
            POSTGRES_DB: app
        ports:
            - "127.0.0.1:5432:5432"
        volumes:
            - postgres-data:/var/lib/postgresql/data
        healthcheck:
            test: ["CMD-SHELL", "pg_isready -U app -d app"]
            interval: 2s
            timeout: 2s
            retries: 30

    mongodb:
        image: mongo:8
        profiles: ["mongodb"]
        environment:
            MONGO_INITDB_ROOT_USERNAME: app
            MONGO_INITDB_ROOT_PASSWORD: local-only-password
            MONGO_INITDB_DATABASE: app
        ports:
            - "127.0.0.1:27017:27017"
        volumes:
            - mongodb-data:/data/db
        healthcheck:
            test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
            interval: 3s
            timeout: 5s
            retries: 30

    redis:
        image: redis:7-alpine
        profiles: ["redis"]
        command: ["redis-server", "--appendonly", "yes"]
        ports:
            - "127.0.0.1:6379:6379"
        volumes:
            - redis-data:/data
        healthcheck:
            test: ["CMD", "redis-cli", "ping"]
            interval: 2s
            timeout: 2s
            retries: 30

volumes:
    postgres-data:
    mongodb-data:
    redis-data:
```

镜像主版本只是示例，项目应固定经过验证的版本并同步 CI。开发密码不能用于任何共享或生产环境。端口只绑定 loopback，避免局域网访问无保护的开发服务。

MongoDB 示例是 standalone，只用于普通 CRUD。Mongoose 多文档事务的集成/E2E 测试必须提供单独 replica set Compose 配置和初始化健康检查，不能声称该示例支持事务测试。

### 18.6.2 按选型启动服务

PostgreSQL + Drizzle：

```sh
docker compose -f compose.dev.yaml --profile postgres up -d --wait
```

MongoDB + Mongoose：

```sh
docker compose -f compose.dev.yaml --profile mongodb up -d --wait
```

需要独立 Redis 或使用 `@keyv/redis` 时：

```sh
docker compose -f compose.dev.yaml --profile redis up -d --wait
```

可以组合 profiles，例如 `--profile postgres --profile redis`。不要求默认同时启动 PostgreSQL 和 MongoDB；Keyv 选择其他 Adapter 时启动对应后端即可。

常用检查：

```sh
docker compose -f compose.dev.yaml ps
docker compose -f compose.dev.yaml logs --tail 100 postgres
docker compose -f compose.dev.yaml down
```

`down -v` 会删除命名卷中的全部开发数据，只能在明确不再需要这些数据时使用。不要对包含非开发资源的 Compose project 执行该命令。

### 18.6.3 配置、迁移与启动顺序

1. 启动当前选型的依赖并等待 healthcheck。
2. 配置 `.env` 中宿主机可访问的 `127.0.0.1` URL。
3. 执行数据库迁移或 Mongoose 初始化/迁移。
4. 执行 `npm run dev`，由 tsx watch 启动 API。
5. 运行 live/ready 和 smoke 请求。

容器内 API 与宿主机 API 使用不同主机名：宿主机连接 `127.0.0.1`；同一 Compose network 内的 API 连接服务名，如 `postgres:5432`、`mongodb:27017`、`redis:6379`。不要在两种环境硬编码同一 URL。

如果项目选择在容器中运行 API，应单独提供 development target、源码挂载、容器内依赖安装和 watch 配置；不能直接把生产镜像当开发镜像，也不能用宿主机 node_modules 无意覆盖容器依赖。

### 18.6.4 测试环境隔离

集成、E2E 和 Smoke 测试使用独立 Compose project、database、Redis keyspace 和卷：

```sh
docker compose -p hono-api-test -f compose.test.yaml up -d --wait
npm run test:integration
npm run test:e2e
npm run test:smoke
docker compose -p hono-api-test -f compose.test.yaml down -v
```

测试清理必须校验 project name 和环境，不能删除开发者正在使用的 compose volumes。CI 每次使用全新 project；本地测试可使用 Testcontainers 自动分配端口和清理资源。

## 18.7 ESM 与构建产物检查

- package.json 声明 `type: module`，Docker 生产阶段也保留该文件。
- tsconfig 开启 `rewriteRelativeImportExtensions`，相对 imports 在源码中使用 `.ts`。
- build 后检查 dist 的相对 imports 为 `.js`，没有依赖 tsx 才能解析的入口。
- 复制 package.json、dist、必要 migrations 和 Worker 产物，生产 command 为 `node dist/main.js`。
- Smoke 测试必须从接近生产的构建产物启动，不能只运行 tsx 源码。

## 18.8 标准交付流水线

```text
干净安装
  -> format:check + lint + typecheck
  -> Vitest unit
  -> build
  -> Compose 隔离依赖 + migrations
  -> Vitest integration + e2e
  -> 构建产物 smoke
  -> Docker image build
  -> 镜像内 smoke
  -> 兼容发布与部署后 smoke
```

任何阶段失败都停止后续发布。部署后 smoke 与本地 smoke 分开记录结果，不能用 CI 通过替代线上验证。


## 18.9 tsx 安装与实际使用

安装为开发依赖：

```sh
npm install --save-dev tsx
```

直接执行：

```sh
npx tsx src/main.ts
```

watch 开发：

```sh
npx tsx watch src/main.ts
# 或统一通过 package script：
npm run dev
```

传应用参数：

```sh
npm run dev -- --config ./config.local.toml
```

tsx 使用 Node.js 运行 TypeScript，适合本地快速反馈，但仍需注意：

- 不代替 `npm run typecheck`。
- 不把 tsx 当生产进程管理器。
- watch 重启时验证旧进程释放数据库、Redis、Winston 和端口。
- ESM import 使用项目统一的 `.ts` 相对扩展名。
- 动态 Worker/new URL 路径单独验证，tsx 不会修复所有构建路径问题。
- 生产必须运行 `npm run build` 后的 `node dist/main.js`，并执行 Smoke。

## 18.10 Prettier 安装与格式化

安装：

```sh
npm install --save-dev prettier
```

`.prettierrc.json` 示例：

```json
{
    "printWidth": 100,
    "tabWidth": 4,
    "useTabs": false,
    "semi": true,
    "singleQuote": false,
    "trailingComma": "all",
    "bracketSpacing": true,
    "endOfLine": "lf"
}
```

命令：

```sh
# 格式化本任务文件
npx prettier --write src/main.ts docs/api.md

# 全仓检查，不写文件
npm run format:check

# 全仓格式化后检查 diff
npm run format
```

Prettier 只负责格式，不验证 import 是否有效、配置是否正确或测试是否通过。生成代码和迁移按生成器规则处理，不用手工格式化掩盖未重新生成的问题。

## 18.11 ESLint 安装与类型感知规则

安装：

```sh
npm install --save-dev eslint @eslint/js typescript-eslint eslint-config-prettier
```

`eslint.config.js` 使用 ESM flat config：

```js
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import prettier from "eslint-config-prettier";
import { fileURLToPath } from "node:url";

export default tseslint.config(
    { ignores: ["dist", "coverage", "node_modules"] },
    js.configs.recommended,
    ...tseslint.configs.recommendedTypeChecked,
    {
        languageOptions: {
            parserOptions: {
                projectService: true,
                tsconfigRootDir: fileURLToPath(new URL(".", import.meta.url)),
            },
        },
        rules: {
            "@typescript-eslint/no-floating-promises": "error",
            "@typescript-eslint/no-misused-promises": "error",
            "@typescript-eslint/no-explicit-any": "error",
            "@typescript-eslint/consistent-type-imports": "error",
        },
    },
    prettier,
);
```

运行：

```sh
npx eslint src test
npx eslint . --fix
```

`--fix` 只自动修复机械问题，执行后仍需 review、typecheck 和测试。类型感知规则需要正确 tsconfig；生成文件、配置文件和测试文件不能被错误排除后又假装已检查。

## 18.12 依赖安装与 lockfile

```sh
# 新项目按 package.json 安装并更新 lockfile
npm install

# CI/干净环境严格按 lockfile 安装
npm ci
```

- 新依赖明确安装为 dependencies 或 devDependencies。
- 升级一个技术栈时阅读官方迁移说明，并同步本手册对应示例。
- 不混用 npm/pnpm/yarn lockfile。
- 不提交 node_modules、dist、coverage、.env 和 runtime logs。
- 生产 Docker 使用 npm ci，构建后裁剪开发依赖并运行 Smoke。

上一篇：[Vitest 测试](13-testing.md) | [返回总目录](../development-guide.md) | 下一篇：[裁剪与验收](15-adoption-acceptance.md)
