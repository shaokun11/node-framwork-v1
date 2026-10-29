# TypeScript 与 ESM 编码规范

## 16. TypeScript 编码与文档规范

### 16.1 类型和控制流

- 开启 `strict`、`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes` 等严格选项，并处理由此暴露的真实问题。
- 不使用非空断言 `!`、双重 cast 或 `any` 跳过验证；确需 `unknown` 后逐步缩窄。
- async 函数必须 await 或显式处理 Promise；禁止漂浮 Promise 和吞掉异常的 catch。
- `catch (error)` 中 error 为 unknown，用类型守卫提取信息并保留 cause。
- 优先提前返回，避免深层嵌套；一个方法只做一层清晰职责。
- 不用可变模块级变量保存请求状态；共享只读配置冻结，缓存与连接有明确所有者。
- 日期、ID、金额、时长和大小使用一致类型，不在层间隐式转换。
- ESM 采用项目统一的 NodeNext 解析规则；源码相对导入使用 `.ts`，开启 rewriteRelativeImportExtensions，路径别名和生产输出必须在构建后验证。

### 16.2 注释与文档

- 公共类、接口和复杂契约用英文 JSDoc/TSDoc 说明用途、参数、返回、抛出和所有权；中文手册解释团队规范。
- 注释说明为什么和边界，不复述代码。TODO 必须有条件和跟踪，不留无所有者注释。
- README 记录当前真实启动方式、配置、依赖和验证命令；变化时同步更新。
- ADR 记录数据库、缓存、认证、任务和兼容发布等重要决策及替代方案。
- 文档示例与实际选型保持同一主版本，升级时检查 import、配置项和行为差异。

## 16.3 全面使用 ESM

项目 `package.json` 必须声明：

```json
{
    "type": "module"
}
```

- 源码、测试、Vitest 配置、脚本和 Worker 新代码全部采用 ESM import/export。
- 禁止新增 `require`、`module.exports`、`exports.*`、`__dirname` 和 `__filename`。
- 文件路径使用 `fileURLToPath(import.meta.url)`，模块定位使用 `new URL(..., import.meta.url)`。
- 同步类型导入使用 `import type`，避免运行时不存在的类型 export 被保留。
- CommonJS 第三方依赖按 Node.js 实际 interop 规则接入，不假定所有 named exports 都可用。
- JSON、WASM 等特殊模块按 Node 版本要求使用 import attributes，不通过 fs 伪装模块导入。

## 16.4 TypeScript 配置与相对扩展名

开启 `rewriteRelativeImportExtensions`。源码中的相对导入写真实 `.ts` 文件，构建时 TypeScript 将其改写为 `.js`：

```ts
import { OrderService } from "./order.service.ts";
import type { OrderRepository } from "./order.repository.ts";
```

推荐 `tsconfig` 基线：

```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "NodeNext",
        "moduleResolution": "NodeNext",
        "lib": ["ES2022"],
        "rootDir": "src",
        "outDir": "dist",
        "strict": true,
        "noUncheckedIndexedAccess": true,
        "exactOptionalPropertyTypes": true,
        "noImplicitOverride": true,
        "noFallthroughCasesInSwitch": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "verbatimModuleSyntax": true,
        "isolatedModules": true,
        "forceConsistentCasingInFileNames": true,
        "rewriteRelativeImportExtensions": true,
        "sourceMap": true,
        "skipLibCheck": true
    },
    "include": ["src"]
}
```

- 构建配置与编辑器/noEmit 配置可以继承共同基线，但 build 只包含生产源码，避免把测试输出到 dist。
- 相对 import/export 统一使用 `.ts`；包导入不写扩展名。不要同目录混用 `.ts`、`.js` 和无扩展三种风格。
- 该选项只改写支持的相对模块 specifier，不会改写 `new URL("./worker.ts", ...)` 等普通字符串。
- 不依赖 tsconfig paths 在 Node 生产运行时自动解析；如使用 `#imports` 等 Node 支持机制，必须验证 tsx、Vitest 和 dist 行为一致。
- 构建后检查 dist 不含 `.ts` 相对导入，并实际执行 `node dist/main.js`。

## 16.5 异步、错误和资源清理

```ts
async function run(signal: AbortSignal): Promise<void> {
    const controller = AbortSignal.any([signal, AbortSignal.timeout(5000)]);
    try {
        await dependency.execute({ signal: controller });
    } catch (error) {
        throw new DependencyError("dependency execute failed", { cause: error });
    }
}
```

- 并发任务有上限，批量处理使用明确的 pool/map 实现，不无界 `Promise.all`。
- `finally` 清理 listener、timer、session 和临时资源；清理失败保留原始错误并聚合。
- 不在异步循环中无意识地串行执行独立任务，也不为追求并行破坏数据库事务顺序。
- 自定义 Error 的 code、公开/内部边界和 cause 明确，instanceof 跨 realm 不可靠时使用稳定判别字段。

## 16.6 TypeScript 提交检查

- [ ] package.json 包含 type=module，没有新增 CommonJS 文件。
- [ ] 相对导入统一 `.ts`，构建后已改写 `.js`。
- [ ] rewriteRelativeImportExtensions、NodeNext 和 strict 配置生效。
- [ ] typecheck、ESLint 类型感知规则和生产 build 全部通过。
- [ ] dist 入口、Worker、动态 import 和资源路径经过真实启动验证。

上一篇：[性能与 Tinypool](11-performance-workers.md) | [返回总目录](../development-guide.md) | 下一篇：[Vitest 测试](13-testing.md)
