# Resolve 插件深度解析

> 源码位置：`packages/vite/src/node/plugins/resolve.ts`
> 前置知识：已读完 define + asset 插件，理解了模块处理三步曲（resolveId → load → transform）
> 关联学习：`optimizer/index.ts`（依赖预构建）、`importAnalysis.ts`（transform 阶段改写）
> 日期：2026-11-23

---

## 一、Resolve 插件的定位与核心职责

### 一句话定义

> **Resolve 插件是 Vite 的"模块定位官"**——把 import 语句里的字符串（裸模块、相对路径、别名）翻译成**浏览器或 Rolldown 能找到的真实文件路径**。

### 它在插件链中的位置

从 `plugins/index.ts` 的注册顺序看：

```
注册顺序    插件                    职责域
────────    ────                    ────────
③          preAliasPlugin           模块定位域（前奏，处理 @fs/ /@id/ 前缀）
④          aliasPlugin (@rollup)    模块定位域（用户配置的 resolve.alias）
⑦          ┌─ oxcResolvePlugin      ★★★ 模块定位域核心 ★★★
           │     （返回两个插件，见下文）
```

### 为什么它是核心中的核心

1. **所有 import 都要先经过它**——没有 resolve，load 和 transform 根本不知道读哪个文件
2. **它连接了 Vite 和 Rolldown**——Vite 自己写了一部分逻辑，Rolldown 内置节点解析引擎写了另一部分
3. **依赖预构建的关键注入点**——`finalizeBareSpecifier` 是预构建闭环的枢纽

---

## 二、两个子插件：vite:resolve-dev + vite:resolve-builtin

`oxcResolvePlugin` 返回**两个**插件，按顺序执行：

```typescript
// resolve.ts 第 221-229 行
export function oxcResolvePlugin(...): Plugin[] {
  return [
    // ① 第一个：vite:resolve-dev（条件注入）
    ...(resolveOptions.optimizeDeps && !resolveOptions.isBuild
      ? [optimizerResolvePlugin(resolveOptions)]
      : []),

    // ② 第二个：vite:resolve-builtin（始终有，来自 Rolldown）
    ...perEnvironmentOrWorkerPlugin(
      'vite:resolve-builtin',
      ...
    ),
  ]
}
```

### 条件注入的判断逻辑

```typescript
resolveOptions.optimizeDeps && !resolveOptions.isBuild
//  ↑ 启用了预构建          ↑ 不是 build 模式
//  dev 模式下两个条件都满足 → 注入 vite:resolve-dev
//  build 模式下 isBuild=true → 不注入，只剩 vite:resolve-builtin
```

### 执行接力模型

```
import 'vue'  →  resolveId 钩子
                    │
                    ▼
            ① vite:resolve-dev（Vite 自己写的）
                    │
                    ├─ 裸模块？→ tryOptimizedResolve → 查 .vite/deps/
                    │   ├─ 命中 → return 预构建路径 → 流程结束
                    │   └─ 未命中 → return undefined → 交给 ②
                    │
                    ├─ 相对路径？→ 算真实路径
                    │   ├─ 命中预构建？→ 加 browserHash → return
                    │   └─ 否则 return undefined → 交给 ②
                    │
                    └─ 返回 undefined → 交给 ②
                    │
                    ▼
            ② vite:resolve-builtin（Rolldown Rust 层）
                    │
                    ├─ Node.js 标准模块解析算法
                    │   package.json exports → main → browser → module
                    │   extensions fallback / symlinks / pnpm
                    │   tsconfig paths
                    │
                    ├─ 裸模块 → finalizeBareSpecifier（Vite 注入的回调）
                    │   决定走预构建 or 原样保留
                    │
                    ├─ 非裸模块但在 node_modules → finalizeOtherSpecifiers
                    │   补 ?v=browserHash
                    │
                    ├─ #xxx 私有导入 → resolveSubpathImports（Vite 接管）
                    │
                    └─ 返回最终 resolvedId
```

---

## 三、裸模块（Bare Specifier）的判断与处理

### 什么是裸模块

**不带任何路径前缀**的模块 specifier——Node.js 认为这是"从 node_modules 找"的信号：

```javascript
// ✅ 裸模块（bare specifier）
import vue from 'vue'                    // 无 scope
import commonjsDep from '@vitejs/test-commonjs-dep'  // 有 scope @scope/pkg
import slicedToArray from '@babel/runtime/helpers/esm/slicedToArray'  // 裸模块 + 子路径

// ❌ 不是裸模块
import data from './data.json'           // 相对路径
import foo from '../foo.js'              // 相对路径
import bar from '/@vite/client'          // 绝对路径 / Vite 内部虚拟路径
import baz from 'data:xxx'               // data URL
```

Rolldown Rust 层的判断逻辑：**不以 `.`、`/`、`\`、协议名开头**的 specifier 就是裸模块。

### Vite 侧的判断：bareImportRE

resolve.ts 第 480 行用 Rolldown 导出的正则：

```typescript
if (bareImportRE.test(id)) {
  // 裸模块分支处理
}
```

---

## 四、vite:resolve-dev 源码逐行解读

源码位置：resolve.ts 第 418-498 行

### 分支 1：相对路径处理（第 450-476 行）

```typescript
if (id[0] === '.') {
  // 1. 算出真实文件系统路径
  fsPath = path.resolve(path.dirname(importer), id)

  // 2. 判断是不是预构建产物（.vite/deps/ 下的）
  if (depsOptimizer.isOptimizedDepFile(normalizedFsPath)) {
    // 是 → 注入 browserHash 版本号 → return
  }

  // 3. 检查是否存在 → 不存在给友好错误
}
```

### 分支 2：裸模块处理（第 480-498 行）

```typescript
if (bareImportRE.test(id)) {
  // 1. 只在 dev + 非 scan 阶段才走预构建查询
  if (asSrc && !options.scan && (res = await tryOptimizedResolve(...))) {
    return res  // 命中预构建 → 直接返回预构建路径
  }
  // 没命中 → 隐式 return undefined → 交给 vite:resolve-builtin
}
```

### tryOptimizedResolve 的核心逻辑（第 896-920 行）

```typescript
async function tryOptimizedResolve(depsOptimizer, id, ...) {
  await depsOptimizer.scanProcessing  // 等 scan 阶段完成
  const metadata = depsOptimizer.metadata
  const optimized = optimizedDepInfoFromId(metadata, id)
  if (optimized) {
    return depsOptimizer.getOptimizedDepId(optimized)
    // 返回 .vite/deps/xxx.js?v=browserHash
  }
  return undefined
}
```

### isOptimizedDepFile 的实现（第 983-988 行）

```typescript
export function createIsOptimizedDepFile(environment) {
  const depsCacheDirPrefix = withTrailingSlash(getDepsCacheDir(environment))
  return (id) => id.startsWith(depsCacheDirPrefix)
}
// 就是个简单的路径前缀匹配：是不是以 .vite/deps/ 开头
```

---

## 五、vite:resolve-builtin 的三个 Vite 回调

源码位置：resolve.ts 第 298-366 行，由 Rolldown Rust 层在 resolve 完成后调用。

### Rolldown 暴露的接口签名

```typescript
// binding-BHW9n9O-.d.mts 第 1984-2002 行
interface BindingViteResolvePluginConfig {
  resolveOptions: {
    mainFields: string[];    // package.json 入口字段优先级
    conditions: string[];   // package.json exports 的 conditions
    extensions: string[];   // 缺后缀时试哪些
    preserveSymlinks: boolean;
    tryPrefix?: string;
    tsconfigPaths: boolean;
    // ...
  };

  // —— Vite 干预回调（3 个）——
  finalizeBareSpecifier?: (resolvedId, rawId, importer) => string | void;
  finalizeOtherSpecifiers?: (resolvedId, rawId) => string | void;
  resolveSubpathImports: (id, importer, isRequire, scan) => string | void;
}
```

### 回调 1：finalizeBareSpecifier（第 298-348 行）—— ★ 核心

**职责：裸模块 resolve 完后，决定走预构建 or 原样保留。**

```typescript
finalizeBareSpecifier(resolvedId, rawId, importer) {
  // 跳过预构建的条件（全满足则 skip）
  skipOptimization =
    depsOptimizer.options.noDiscovery ||
    !isJsType ||                                    // 不是 JS 类型
    (importer && isInNodeModules(importer)) ||     // importer 在 node_modules 里
    exclude?.includes(pkgId) ||                    // 在 exclude 列表里
    exclude?.includes(rawId) ||
    SPECIAL_QUERY_RE.test(resolvedId)

  if (skipOptimization) {
    // 跳过预构建 → 只注入 browserHash
    newId = injectQuery(resolvedId, `v=${browserHash}`)
  } else {
    // 没发现过的新依赖 → 注册到预构建队列
    const optimizedInfo = depsOptimizer.registerMissingImport(rawId, newId)
    newId = depsOptimizer.getOptimizedDepId(optimizedInfo)
  }
  return newId
}
```

**触发条件**：vite:resolve-dev return 了 undefined（没命中预构建）→ Rolldown 接手 resolve → resolve 完后调此回调。

**什么时候会被调**：scan 没发现的新依赖（动态 import、运行时 import 等）。scan 发现的依赖会被 vite:resolve-dev 在前面拦截。

### 回调 2：finalizeOtherSpecifiers（第 349-360 行）

**职责：非裸模块但路径在 node_modules 里 → 补 ?v=browserHash。**

```typescript
finalizeOtherSpecifiers(resolvedId, rawId) {
  const newResolvedId = ensureVersionQuery(resolvedId, rawId, options, depsOptimizer)
  return newResolvedId === resolvedId ? undefined : newResolvedId
}

// ensureVersionQuery 逻辑（第 574-600 行）：
if (!isBuild && !scan && depsOptimizer) {
  const isNodeModule = isInNodeModules(id) || isInNodeModules(resolved)
  if (isNodeModule && !DEP_VERSION_RE.test(resolved)) {
    resolved = injectQuery(resolved, `v=${browserHash}`)
  }
}
```

**为什么存在**：裸模块有 finalizeBareSpecifier 处理，但如果代码里直接写了 `import vue from '../node_modules/vue/dist/vue.esm-browser.js'` 这种 node_modules 相对路径，Rolldown 不会当裸模块处理，也不会走预构建。这个回调就是给"漏网之鱼"补版本号。

**什么时候会被调**：预构建产物内部互相 import、node_modules 里的嵌套依赖等。dev 模式下很少触发，主要活跃在 build 模式。

### 回调 3：resolveSubpathImports（第 361-366 行）

**职责：完全接管 package.json 的 `imports` 私有导入（`#xxx` 开头）解析。**

```typescript
// Rolldown 回调入口（第 361 行）
resolveSubpathImports(id, importer, isRequire) {
  return resolveSubpathImports(id, importer, {
    ...options,
    isRequire: resolveOptions.isRequire ?? isRequire,
  })
}

// Vite 实现逻辑（第 510-572 行）—— 只处理 #xxx 开头
export function resolveSubpathImports(id, importer, options) {
  if (!importer || !id.startsWith('#')) return          // ① 过滤非私有导入
  const pkgData = findNearestPackageData(basedir)        // ② 找 importer 的 package.json
  const importsPath = resolveExportsOrImports(..., 'imports')  // ③ 匹配 imports key
  importsPath = path.relative(basedir, path.join(pkgData.dir, importsPath))  // ④ 转相对路径
  return importsPath + postfix
}
```

**与另外两个回调的本质区别**：

| | resolveSubpathImports | finalizeBareSpecifier | finalizeOtherSpecifiers |
|---|---|---|---|
| 时机 | Rolldown 解析**之前/之中**完全接管 | Rolldown 解析**之后**事后干预 | Rolldown 解析**之后**事后干预 |
| 谁主导 | Vite 自己算路径 | Rolldown 算完 Vite 改 | Rolldown 算完 Vite 改 |
| 处理对象 | 只有 `#xxx` 私有导入 | 裸模块 | 非裸模块的 node_modules 路径 |

---

## 六、Dev vs Build 模式下的 Resolve 差异

### Dev 模式：浏览器驱动的请求-响应模型

```
浏览器请求 index.html
  ↓
Vite 返回处理后的 HTML（inline script 被拆成独立 .js）
  ↓
浏览器解析 HTML，发现 script[src] → 发 HTTP 请求
  ↓
Vite transformRequest → resolveId → importAnalysis 改写
  ↓
浏览器拿到改写后的 JS，解析 import → 发新请求
  ↓
循环直到所有依赖加载完
```

**特点**：
- 每个浏览器请求 = 一次独立的 resolveId 调用
- resolveId 可能被内部多个环节重复调（见下一节）
- 裸模块在 transform 阶段被改写为真实路径（浏览器不认识裸模块）

### Build 模式：Rolldown 驱动的递归构建模型

```
pnpm run build
  ↓
Rolldown 在 Node.js 里从入口文件开始递归
  ↓
resolveId → 读文件 → 解析 AST → 发现 import → 再 resolveId
  ↓
构建完整模块依赖图
  ↓
scope hoisting → code splitting → minify → 生成 bundle
```

**特点**：
- resolveId 被循环调用几百到几千次（每个 import 一次）
- 全在 Node.js 里完成，浏览器不参与
- 最终打包成少数几个 bundle 文件

### 核心对比

| | Dev 模式 | Build 模式 |
|---|---|---|
| 谁驱动 | 浏览器（原生 ESM） | Rolldown（构建管线） |
| resolveId 来源 | 浏览器每个请求 → Vite 内部多个环节各自调 | Rolldown 内部递归循环 |
| vite:resolve-dev？ | ✅ 注入 | ❌ 不注入 |
| finalizeBareSpecifier 触发 | scan 没发现的新依赖 | 所有没被预构建的裸模块 |
| finalizeOtherSpecifiers 触发 | 极少 | 多 |
| 浏览器请求数 | 多（每个模块一个） | 少（只有 bundle） |

---

## 七、为什么同一个 import 打印多次？——Dev 模式下 resolveId 的 6 个调用方

### 调用方总览

| # | 调用方 | 位置 | 触发时机 | importer |
|---|---|---|---|---|
| ① | scan 阶段 | optimizer/scan.ts:225 | dev server 启动时 | index.html |
| ② | transformRequest | server/transformRequest.ts:171 | 每个请求进来先 resolve | undefined（请求 URL） |
| ③ | importAnalysis 插件 | plugins/importAnalysis.ts | transform 阶段改写 import | html-proxy 拆出来的 JS |
| ④ | moduleGraph | server/environment.ts:144 | 模块图初始化 | URL |
| ⑤ | resolveEntries | server/environment.ts:249 | 环境启动时 resolve 入口 | undefined |
| ⑥ | Rolldown 内部递归 | Rolldown Rust 层 | vite:resolve-dev 没 return 时 | 同上 |

### importer 不同导致 skipCalls 去重失效

pluginContainer 的去重逻辑（pluginContainer.ts:837）：**只有 id 和 importer 都相同才去重**。

同一个裸模块可能有多个 importer：

```
scan 阶段：
  resolveId('@vitejs/test-commonjs-dep', importer='D:/.../index.html')

transform 阶段（importAnalysis）：
  resolveId('@vitejs/test-commonjs-dep', importer='D:/.../index.html?html-proxy&index=0.js')
  ↑ 和 scan 的 importer 不一样！skipCalls 认为是新的请求 → 再调一次

Rolldown 内部（如果 vite:resolve-dev 没 return）：
  resolveId('@vitejs/test-commonjs-dep', importer=同上)
  ↑ 可能 Rolldown Rust 层又调了一次
```

### 完整时序图（playground/define 启动时的 resolveId 调用）

```
pnpm run dev → dev server 启动
  │
  ├─ 1. resolveEntries（entry 初始化）
  │     resolveId('index.html', undefined)
  │     resolveId('/@vite/client', undefined)
  │     ← 你的插件打印可能打几次
  │
  ├─ 2. 自动 scan（optimizer.scan.ts）
  │     扫 index.html 里的 import
  │     resolveId('@vitejs/test-commonjs-dep', 'D:/.../index.html', { scan: true })
  │     resolveId('./data.json', 'D:/.../index.html', { scan: true })
  │     ← ★ 第一次批量打印
  │
  ├─ 3. scan 完 → 预构建开始（不调 resolveId）
  │
  ├─ 4. 浏览器请求 GET /
  │     transformRequest → resolveId('/index.html', undefined)
  │     Vite 返回处理后的 HTML
  │
  ├─ 5. 浏览器请求 GET /index.html?html-proxy&index=0.js
  │     importAnalysis 解析 AST 里的 import：
  │       resolveId('@vitejs/test-commonjs-dep', 'D:/.../index.html?html-proxy&index=0.js')
  │       resolveId('./data.json', 'D:/.../index.html?html-proxy&index=0.js')
  │     ← ★★ 最多的几次打印！importAnalysis 对每个 import 都调
  │
  ├─ 6. 浏览器请求 GET /node_modules/.vite/deps/xxx.js
  │     transformRequest → resolveId('完整绝对路径')
  │     ← 可能又打
  │
  └─ 7. 后续 import 循环
```

---

## 八、预构建闭环的完整链路

### 启动时预构建（scan 发现）

```
dev server 启动
  ↓
optimizer.scan.ts 自动扫描 index.html
  ↓
发现 import '@vitejs/test-commonjs-dep'
  ↓
metadata.optimized 记录这个包
  ↓
预构建引擎（Rolldown/esbuild）把它打成 .vite/deps/@vitejs_test-commonjs-dep.js
  ↓
浏览器请求时：
  vite:resolve-dev → tryOptimizedResolve → metadata 里有！
  → return 预构建路径 → 流程结束
```

### 运行时预构建（finalizeBareSpecifier 发现新依赖）

```
scan 没发现的新依赖（如动态 import、运行时 import）
  ↓
浏览器请求时 vite:resolve-dev return undefined
  ↓
vite:resolve-builtin（Rolldown）接手 resolve
  ↓
resolve 完裸模块 → finalizeBareSpecifier 回调
  ↓
不在 exclude 里 + 是 JS + 不是 node_modules 内部 import
  → registerMissingImport(rawId, resolvedId) 注册到预构建队列
  → getOptimizedDepId() 返回（可能预构建还没完成）
  ↓
后台触发预构建 → 完成后返回给浏览器
```

### 预构建被跳过的场景

| 场景 | 条件 | 处理 |
|---|---|---|
| exclude 列表 | 配置了 `optimizeDeps.exclude: ['xxx']` | skipOptimization = true → 注入 browserHash |
| 不是 JS 类型 | `.css`、`.png` 等 | skipOptimization = true |
| importer 在 node_modules | 依赖内部互相 import | skipOptimization = true |
| noDiscovery | 关掉自动发现 | skipOptimization = true |
| build 模式 | isBuild = true | vite:resolve-dev 不注入，预构建整体禁用 |

---

## 九、调试实战笔记（基于 playground/define）

### 快速定位问题的命令

```powershell
# 删掉预构建缓存看完整的新依赖链路
Remove-Item -Recurse -Force playground\define\node_modules\.vite

# 关掉自动扫描让依赖走 finalizeBareSpecifier 路径
# 在 vite.config.js 里加：
# optimizeDeps: { exclude: ['@vitejs/test-commonjs-dep'] }

# 注意：noDiscovery: true + include 空 = optimizer 整体禁用！
# 不要这么配！
```

### resolve.ts 里的 console.log 位置建议

```typescript
// 第 438 行：importer 是不是 .html（html-proxy 的判断）
console.log('[ importer?.endsWith(".html") ❀]-438', true/false)

// 第 482 行：bareImportRE 是否命中
console.log('[ bareImportRE.test(id) ❀]-482', true/false)

// 第 494 行：tryOptimizedResolve 是否命中预构建
console.log('[ tryOptimizedResolve 命中 ❀]-494')

// finalizeBareSpecifier 里：
console.log('[ skipOptimization ❀]-318', skipOptimization)
console.log('[ newId ❀]-346', rawId, '→', newId)

// finalizeOtherSpecifiers 里：
console.log('[ ensureVersionQuery ❀]-363', resolvedId, '→', newResolvedId)
```

### 预期观察

| 观察 | 说明 |
|---|---|
| scan 阶段（第 3-14 行日志） | 以原始 index.html 为 importer 调 resolve |
| transform 阶段（第 64-76 行日志） | 以 html-proxy 拆出来的 JS 为 importer 调 resolve |
| 裸模块 494 行打印 | 预构建命中 → vite:resolve-dev return → Rolldown 不执行 |
| 裸模块没命中预构建 + 没 exclude | 会看到 finalizeBareSpecifier 的打印 |
| finalizeOtherSpecifiers 打印 | dev 模式很少见，build 模式看 |

---

## 十、一句话总结每个核心点

| 问题 | 答案 |
|---|---|
| Resolve 插件做什么？ | 把 import 字符串翻译成真实文件路径 |
| 为什么两个子插件？ | vite:resolve-dev 管预构建捷径，vite:resolve-builtin 管 Rolldown 标准 node resolve |
| 浏览器认识裸模块吗？ | 不认识——Vite 在 transform 阶段改写 import 语句 |
| resolveId 调几次？ | dev 下 1 个浏览器请求可能触发 3-6 次内部 resolveId 调用 |
| 为什么打印多次？ | 不同 importer + 多个内部调用方导致 skipCalls 去重失效 |
| finalizeBareSpecifier 什么时候调？ | vite:resolve-dev 没命中预构建时，Rolldown resolve 完后调 |
| finalizeOtherSpecifiers 什么时候调？ | 非裸模块但路径在 node_modules 里，dev 很少、build 多 |
| resolveSubpathImports 处理什么？ | 只有 `#xxx` 私有导入，exports 子路径 Rolldown 自己搞 |
| 预构建什么时候触发？ | scan 阶段自动发现 + finalizeBareSpecifier 运行时发现新依赖 |

---

## 十一、后续学习建议

读完 resolve 插件后，强烈建议继续读：

```
importAnalysis 插件（plugins/importAnalysis.ts）
  └─ 读完能完整回答：裸模块是怎么在 transform 阶段被改写的？
     resolveId 结果怎么被用在 AST 改写里？

preAlias 插件（plugins/preAlias.ts）
  └─ resolve 之前的路径前缀处理（@fs/、/@id/）

html.ts 里的 htmlInlineProxyPlugin
  └─ 为什么 inline script 会被拆成 index.html?html-proxy&index=0.js？
     这个拆分对 importAnalysis 的 importer 有什么影响？
```
