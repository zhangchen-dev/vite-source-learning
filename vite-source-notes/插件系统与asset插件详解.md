# Vite 源码学习：插件系统与 asset 插件详解

> 基于 vite 官方仓库源码 `packages/vite/src/node/plugins/asset.ts`
> 日期：2026-11-22

---

## 一、插件钩子全景时间线

一个 Rolldown/Rollup 插件的生命周期钩子，按**执行先后顺序**排列：

```
┌─────────────────────────────────────────────────────────────────┐
│ ① 初始化阶段（整个构建只执行一次）                                  │
│                                                                   │
│   options            → 拿到/修改构建配置                            │
│   buildStart         → 初始化内部状态（清空缓存等）                   │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ② 模块处理阶段（每个 import 的模块都会走一遍这三步）                  │
│                                                                   │
│   resolveId          → 模块定位：import 字符串 → 绝对路径             │
│       ↓                                                           │
│   load               → 模块加载：绝对路径 → 源码字符串/code           │
│       ↓                                                           │
│   transform          → 模块转译：源码字符串 → 转译后的 JS             │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ③ 产物生成阶段（build 专属）                                       │
│                                                                   │
│   renderChunk        → 每个 chunk 生成后，改产物代码                  │
│   generateBundle     → 所有 chunk/asset 生成后，增删改 bundle         │
│   writeBundle        → 写到磁盘前                                    │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ④ 结束阶段                                                         │
│                                                                   │
│   buildEnd           → 构建结束                                     │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ⑤ Dev 专属钩子                                                     │
│                                                                   │
│   watchChange        → 监听到文件变化时                               │
│   handleHotUpdate    → HMR 热更新时                                  │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│ ⑥ Vite / Rolldown 扩展钩子（原生 Rollup 没有）                      │
│                                                                   │
│   resolveFileUrl     → import.meta.ROLLDOWN_FILE_URL_* 的生成       │
│   applyToEnvironment → 多环境（client/ssr/webworker）下是否启用      │
│   perEnvironmentStartEndDuringDev → dev 时按环境分别触发生命周期      │
└─────────────────────────────────────────────────────────────────┘
```

### 不同插件的钩子实现对比

| 钩子 | asset 插件 | define 插件 | 为什么不同 |
|------|-----------|------------|-----------|
| resolveId | ✅ | ❌ | asset 需要接住特定 id；define 不管模块在哪 |
| load | ✅ | ❌ | asset 需要读文件生成源码；define 改的是别人 load 好的代码 |
| transform | ❌ | ✅ | asset 输出已经是 JS；define 专门做代码替换 |
| renderChunk | ✅ | ❌ | asset 的占位符在产物阶段替换；define 在源码阶段就替换完了 |

### 核心模式：不是每个插件都要实现所有钩子

好的插件只实现它职责范围内的钩子：

```
这个插件关心什么类型的模块？
  → 决定要不要实现 resolveId（拦截特定 id）

这些模块的源码从哪来？
  → 决定要不要实现 load（自己读文件）

这些模块的源码要怎么改？
  → 决定要不要实现 transform（转译/AST 变换）

产物里的占位符/标记要怎么处理？
  → 决定要不要实现 renderChunk / generateBundle
```

---

## 二、filter 与 handler：Rolldown 性能优化设计

### Rollup 旧格式 vs Rolldown 新格式

```typescript
// Rollup：每个 import 都要进 handler，自己在里面判断要不要处理
resolveId(id, importer) {
  if (!urlRE.test(id)) return   // ← JS 层判断
  // 真正的处理逻辑...
}

// Rolldown：filter 先在 Rust 层做预筛选，匹配上了才进 JS 的 handler
resolveId: {
  filter: {                           // ← Rust 层快速判断
    id: [urlRE, DEFAULT_ASSETS_RE]
  },
  handler(id) {                       // ← 只有匹配上才进这里
    // 真正的处理逻辑...
  },
},
```

### 为什么要分两层？—— 性能

假设一个项目有 **10000 个 import**，其中只有 **5 个**是 asset 资源：

| 格式 | 发生了什么 |
|------|-----------|
| Rollup 旧格式 | 10000 次全部进入 JS handler，每次 `if (!urlRE.test(id)) return`，9995 次白跑 |
| Rolldown 新格式 | Rust 层用正则快速扫一遍 10000 个 id → 只有 5 个匹配 → 进入 JS handler |

**正则匹配在 Rust 里比 JS 快 10-100 倍**，而且 Rolldown 可以并行处理多个 filter。

### filter 和 handler 的职责边界

| | filter | handler |
|--|--------|---------|
| 在哪执行 | Rolldown Rust 内核 | Node.js JS 层 |
| 作用 | 快速预筛选，减少 JS 调用次数 | 实际处理 resolve/load/transform 逻辑 |
| 能做什么判断 | 正则匹配 id/code/moduleType | 任意 JS 逻辑（读配置、查文件系统等） |
| Rollup 支持吗 | ❌ 不支持，这是 Rolldown 独有 | ✅ 支持（旧格式直接就是 handler） |

### Rolldown filter 能匹配什么？

```typescript
// resolveId / load 的 filter
{ id: RegExp | RegExp[] }

// transform 的 filter 更丰富
{
  id?: RegExp | RegExp[]
  code?: RegExp | RegExp[]
  moduleType?: string | string[]
}
```

### 两层守卫的关系（以 asset 插件为例）

```
import './logo.svg?url'
    ↓
filter.id 正则数组（Rust 层）
    → DEFAULT_ASSETS_RE 匹配 .svg ✅ → 放行
    ↓
handler(id)（JS 层）
    → config.assetsInclude() 再次确认
    → checkPublicFile() 检查是不是 public 目录的文件
    → 确认是，返回 id
```

---

## 三、asset 插件 load 钩子详解

### 整体结构：先分叉，再汇合

```
handler(id) 进来
  │
  ├── 分叉1：?raw 请求？ → 读文件原文 → 直接返回字符串 export
  │
  └── 分叉2：?url 或普通资源 → 走 dev/build 分支 → 返回 URL 或 reference
```

### 分叉1：?raw 请求（asset.ts 第242-253行）

```typescript
if (rawRE.test(id)) {
  const file = checkPublicFile(id, config) || cleanUrl(id)
  this.addWatchFile(file)
  return {
    code: `export default ${JSON.stringify(await fsp.readFile(file, 'utf-8'))}`,
    map: { mappings: '' },
    moduleType: 'js',
  }
}
```

例子：`poem.txt` 内容是 `"床前明月光"`，load 返回的就是 `export default "床前明月光"`。

**不涉及 inline/emit**，就是原样读文件 → 包一层 `export default`。

### 分叉2：普通资源或 ?url 请求（核心！）

整体逻辑流程图：

```
id 进来（比如 D:/project/src/logo.svg?url）
  │
  ├─ removeUrlQuery() → 去掉 ?url，得到 D:/project/src/logo.svg
  │
  ├─ 判断环境 isBundled？
  │   │
  │   ├─ NO（dev 服务器）→ fileToDevUrl()
  │   │     → 返回实际可以被浏览器请求的 URL 字符串
  │   │     → resolved = { type: 'string', value: '/src/logo.svg' }
  │   │
  │   └─ YES（build 打包）→ resolveBuiltAsset()
  │         → 读文件内容
  │         → 判断 inline 还是 emit
  │         → inline  → { type: 'string', value: 'data:image/svg+xml;base64,...' }
  │         → emit    → { type: 'reference', referenceId: 'abc123', postfix: '' }
  │
  ├─（dev 专属）继承 HMR 时间戳
  │   如果这个 asset 的 module 之前被 HMR 失效过，URL 后面加 ?t=1234567890
  │
  └─ 最终用 formatBuiltAsset() 包一层 export default
```

### 关键函数：resolveBuiltAsset（asset.ts 第630-703行）

```
resolveBuiltAsset() {
  │
  ├─ 检查是不是 /public 目录的文件？
  │   └─ 是 → 返回 { type: 'string', value: '__VITE_PUBLIC_ASSET__hash__' }
  │        （public 目录的文件永远不打包，build 时原样复制）
  │
  ├─ 查 assetCache 有没有缓存？
  │   └─ 有 → 直接返回（同一个文件被多个 import 引用，只处理一次）
  │
  ├─ fsp.readFile(file) 读文件内容
  │
  ├─ shouldInline() 判断要不要 inline
  │   ├─ 强制 inline（?inline 后缀）→ YES
  │   ├─ 强制 emit（?no-inline 后缀）→ NO
  │   ├─ 文件大小 < assetsInlineLimit（默认 4KB）→ YES
  │   └─ 否则 → NO
  │
  ├─ YES → { type: 'string', value: 'data:image/svg+xml;base64,...' }
  │
  └─ NO → emitFile() 交给 Rolldown 去写入产物目录
         返回 { type: 'reference', referenceId: 'abc123', postfix: '' }
}
```

### 最终输出：formatBuiltAsset

不管 dev 还是 build，不管 inline 还是 emit，最后都统一包成 JS 模块：

| resolved 类型 | formatBuiltAsset 输出 | 最终 code |
|---|---|---|
| `{ type: 'string', value: '/src/logo.svg' }` | `'/src/logo.svg'` | `export default '/src/logo.svg'` |
| `{ type: 'string', value: 'data:image/svg+xml;base64,...' }` | `'data:image/svg+xml;base64,...'` | `export default 'data:image/svg+xml;base64,...'` |
| `{ type: 'reference', referenceId: 'abc123' }` | `import.meta.ROLLDOWN_FILE_URL_abc123` | `export default import.meta.ROLLDOWN_FILE_URL_abc123` |

第三种就是**占位符**——它不是 URL，而是留给后续 resolveFileUrl 钩子解析的标记。

### 一句话总结 load 钩子

> load 钩子本质上是**把一个"非 JS 文件"转换成"能被 import 的 JS 模块"**的过程。
> - `?raw` → 原样读，输出 `export default "文件内容"`
> - 普通资源 → dev 给可访问 URL，build 决定 inline (data URL) 还是 emit (reference ID)，最后统一输出 `export default ...`

---

## 四、resolveFileUrl：Vite 自定义钩子

### 为什么找不到调用点？

`resolveFileUrl` 不是 JS 代码里显式调用的，而是 **Rolldown Rust 内核隐式触发的**。

Rolldown 在处理 chunk 生成时，会扫描产物代码里有没有这样的语法：

```js
import.meta.ROLLDOWN_FILE_URL_<referenceId>
```

一旦发现，Rolldown 内部就知道：这是一个 emitFile 产出的 asset。然后它会：

1. 找到这个 referenceId 对应的 asset，算出它最终的产物路径 `fileName`
2. 遍历所有插件，找到定义了 `resolveFileUrl` 钩子的那个
3. 调用它，传入 `{ fileName, chunkId, format, urlId }`
4. 用插件返回的字符串/runtime 表达式替换掉原来的占位符

### 调用链全流程（build 专属）

```
步骤1：asset 插件 load 钩子（build 时）
    ↓ emitFile({ type: 'asset', source: content })
    ↓ 拿到 referenceId = 'abc123'
    ↓ 返回 export default import.meta.ROLLDOWN_FILE_URL_abc123
    ↓
步骤2：Rolldown 内核处理这个 import.meta 语法（隐式）
    ↓ 检测到 ROLLDOWN_FILE_URL_ 前缀
    ↓ 找到对应的 emit 产物，算出 fileName = 'assets/logo-8f3a2b.svg'
    ↓ 触发插件的 resolveFileUrl 钩子
    ↓ 传入：{ fileName, chunkId, format, urlId }
    ↓
步骤3：asset 插件 resolveFileUrl 钩子
    ↓ toOutputFilePathInJS() 生成最终 URL
    ↓ 返回 '/assets/logo-8f3a2b.svg' 或 runtime 表达式
    ↓
步骤4：Rolldown 用返回值替换占位符
    ↓ 最终产物 JS：export default "/assets/logo-8f3a2b.svg"
```

### 为什么只有 build 有？

**dev 时 `isBundled` 为 false**，load 钩子走 `fileToDevUrl` 分支，直接返回浏览器能访问的真实 URL（如 `'/src/logo.svg'`），根本不会生成 `import.meta.ROLLDOWN_FILE_URL_*` 占位符。

### resolveFileUrl 做了什么？（asset.ts 第305-343行）

```typescript
resolveFileUrl({ fileName, chunkId, format, urlId }) {
  // ① 收集当前 chunk 引用了哪些 asset
  let files = importedByChunk.get(chunkId)
  files.add(cleanUrl(fileName))

  // ② 创建相对路径运行时工具函数
  const toRelativeRuntime = createToImportMetaURLBasedRelativeRuntime(format, isWorker)

  // ③ 取元数据（urlId → fileUrlMetadata → asFileUrl / postfix）
  const metadata = urlId ? fileUrlMetadata.get(environment)?.get(urlId) : undefined

  // ③-A 特殊情况：需要 file:// 协议 URL（比如 Worker）
  if (metadata?.asFileUrl) {
    return toRelativeRuntime(fileName, chunkId).runtime
  }

  // ③-B 常规情况：toOutputFilePathInJS 生成最终 URL
  const replacement = toOutputFilePathInJS(
    environment,
    fileName + (metadata?.postfix ?? ''),
    'asset', chunkId, 'js', toRelativeRuntime,
  )

  // ④ 返回值：字符串 or runtime 表达式
  return typeof replacement === 'string'
    ? JSON.stringify(encodeURIPath(replacement))
    : replacement.runtime
}
```

### 参数解释

| 参数 | 来自哪里 | 举例 |
|------|----------|------|
| `fileName` | Rolldown 内核 emitFile 后算出的最终产物名 | `'assets/logo-8f3a2b.svg'`（带 contenthash） |
| `chunkId` | 当前正在处理的 chunk 标识 | `'D:/project/src/main.js'` |
| `format` | 当前 chunk 的模块格式 | `'esm'` / `'cjs'` |
| `urlId` | load 钩子里存元数据时生成的随机 key（可选） | 用来取 fileUrlMetadata |

### 返回值两种类型

```typescript
// 类型1：字符串（硬编码绝对路径）
return JSON.stringify('/assets/logo-8f3a2b.svg')
// Rolldown 替换后：export default "/assets/logo-8f3a2b.svg"

// 类型2：runtime 表达式（动态计算相对路径）
return "new URL('../assets/logo-8f3a2b.svg', import.meta.url).href"
// Rolldown 替换后：export default new URL('...', import.meta.url).href
// 浏览器运行时再算出真实 URL
```

### 为什么需要 resolveFileUrl？

如果 asset 插件不定义 resolveFileUrl，Rolldown 会默认直接硬编码 fileName：

```js
// Rolldown 默认（❌ 没有前导 /，相对路径容易找不到）
export default "assets/logo-8f3a2b.svg"

// Vite resolveFileUrl 处理后（✅ 正确的绝对或相对路径）
export default "/assets/logo-8f3a2b.svg"
```

resolveFileUrl 通过 `toOutputFilePathInJS` 处理了：

| 场景 | 输出 |
|------|------|
| `base: '/'`（默认） | `"/assets/logo-8f3a2b.svg"` |
| `base: './'`（相对路径） | `new URL('../assets/logo-8f3a2b.svg', import.meta.url).href` |
| `base: '/my-app/'` | `"/my-app/assets/logo-8f3a2b.svg"` |
| SSR 特殊场景 | runtime 表达式 |

---

## 五、referenceId 的完整传递链

### 一个具体例子

项目结构：
```
src/
  main.js          → import logoUrl from './logo.svg?url'
  logo.svg         → 8KB，超过 inline 限制
```

### 步骤1：asset 插件 resolveId

```
import './logo.svg?url'
    ↓ resolveId 钩子
    ↓ DEFAULT_ASSETS_RE 匹配 .svg ✅
    ↓ checkPublicFile 没命中（不是 public 目录）
    ↓ Rolldown resolve 成绝对路径：D:/project/src/logo.svg
```

### 步骤2：asset 插件 load（build 时）

```
resolveBuiltAsset() {
    ↓ shouldInline() → 8KB > 4KB → NO
    ↓ emitFile({ type: 'asset', name: 'logo.svg', source: 二进制内容 })
    ↓ Rolldown 返回 referenceId = 'abc123XYZ789$abcdefghij'
    ↓ result = { type: 'reference', referenceId: 'abc123XYZ789$abcdefghij', postfix: '' }
}
    ↓
formatBuiltAsset(resolved, 'js') {
    ↓ `import.meta.ROLLDOWN_FILE_URL_${resolved.referenceId}`
}
    ↓
load 返回给 Rolldown 的 JS 代码：
export default import.meta.ROLLDOWN_FILE_URL_abc123XYZ789$abcdefghij
```

### 步骤3：Rolldown 内核处理 emit + 触发 resolveFileUrl

```
Rolldown 扫描 main.js 产物代码
    ↓ 发现 import.meta.ROLLDOWN_FILE_URL_ 前缀
    ↓ 提取 referenceId = 'abc123XYZ789$abcdefghij'
    ↓ 查 emit 清单 → contenthash = '8f3a2b' → fileName = 'assets/logo-8f3a2b.svg'
    ↓ 触发 resolveFileUrl({ fileName, chunkId, format })
    ↓ toOutputFilePathInJS() → '/assets/logo-8f3a2b.svg'
    ↓ 替换占位符
```

### 步骤4：最终产物

```
dist/
  assets/logo-8f3a2b.svg   ← emitFile 发出的
  main.js                   ← export default "/assets/logo-8f3a2b.svg"
```

### 极简传递链

```
emitFile()           → 返回 referenceId = "abc123"     （Rolldown 内核生成）
   ↓
resolveBuiltAsset    → result = { type: 'reference', referenceId: "abc123" }
   ↓
load 钩子            → formatBuiltAsset → "import.meta.ROLLDOWN_FILE_URL_abc123"
   ↓                                                           ↑
                                    ← 字符串拼接，referenceId 塞进 JS 源码
   ↓
Rolldown 内核扫 AST  → 发现前缀 → 提取 referenceId → 查 emit 清单 → 算出 fileName
   ↓
resolveFileUrl({ fileName: "assets/logo-8f3a2b.svg", ... })
   ↓
toOutputFilePathInJS → 返回 "/assets/logo-8f3a2b.svg"
   ↓
最终产物 JS           → export default "/assets/logo-8f3a2b.svg"
```

### 另一条入口：CSS / HTML 插件调用

```
CSS 插件处理 url('./logo.svg')
  ↓
fileToBuiltUrl(pluginContext, id, 'string')    // asset.ts 第541行
  ├─ resolveBuiltAsset(pluginContext, id)        // 同一个函数
  │     └─ emitFile() → { type: 'reference', referenceId: 'abc123' }
  ├─ addFileUrlMetadataForAsset(...)             // 生成 urlId
  └─ formatBuiltAsset(resolved, 'string', urlId) // 注意是 'string' 格式！
        → 返回 "__VITE_ASSET__abc123__"          // CSS 插件用的占位符
```

这个 `__VITE_ASSET__` 占位符**不是 resolveFileUrl 处理的**，而是 renderChunk 里的 `renderAssetUrlInJS` 处理的——因为 CSS 不是 JS，不能用 `import.meta` 语法。

---

## 六、两个占位符，两种处理方式

| 占位符 | 生成位置 | 处理钩子 | 场景 |
|--------|----------|----------|------|
| `import.meta.ROLLDOWN_FILE_URL_*` | load 钩子（asset 的 JS 模块） | **resolveFileUrl** | JS 代码里的 `import img from './logo.svg?url'` |
| `__VITE_ASSET__xxx__` | CSS 插件（CSS 转译到 JS 时的资源引用） | **renderChunk** → `renderAssetUrlInJS` | CSS 里的 `background: url('./logo.svg')` 被打包成 JS 字符串时 |

CSS 被转译成 JS 字符串时：
```css
/* src/main.css */
.logo { background: url('./logo.svg') }
```
build 时 CSS 插件会把它变成：
```js
var css = ".logo{ background: url(__VITE_ASSET__abc123__) }"
```
然后 renderChunk 里的 `renderAssetUrlInJS` 替换掉 `__VITE_ASSET__abc123__`。

---

## 七、Vite 插件的职责域划分

从 `plugins/index.ts` 的注册顺序看，29 个插件分成 5 个职责域：

```
① 预处理器域（改源码 AST）
   preAlias → resolve → oxc → esbuild → define → css → cssPost

② 资源/文件域
   asset → json → wasm → worker → html → assetImportMetaUrl → workerImportMetaUrl

③ 依赖优化域（dev 专属）
   optimizedDeps → watchPackageData

④ 构建增强域（build 专属）
   importAnalysisBuild → dynamicImportVars → importMetaGlob → license → manifest → terser → prepareOutDir

⑤ Server/Dev 域（dev 专属）
   clientInjections → cssAnalysis → importAnalysis → forwardConsole
```

### 推荐阅读顺序（按认知增量排序）

1. **resolve 插件**（30分钟）—— 核心模块定位逻辑
2. **importAnalysis（dev）/ importAnalysisBuild（build）**（20分钟）—— dev vs build 最大差异点
3. **css + html 插件**（按需）—— asset 占位符的 CSS 场景
4. **define + asset 联动**（回头扫）—— resolve 之后重看会发现之前漏掉的细节

---

## 八、文件内容的完整处理链

> 之前一直在讲 referenceId 和 URL 的流转，现在补上文件内容本身是怎么处理的。

### 内容分叉点

所有内容处理的起点是 `resolveBuiltAsset` 里这两行代码：

```typescript
const content = await fsp.readFile(file)   // Buffer，所有内容流转的源头

if (shouldInline(...)) {
  assetToDataURL(environment, file, content)   // 路径 A：inline
} else {
  emitFile({ source: content })                  // 路径 B：emit
}
```

**文件内容 `content`（Buffer）就在这里分叉了。**

```
fsp.readFile(file) → content: Buffer (原始二进制)
    │
    ├── 路径 A: shouldInline() = true  → assetToDataURL()
    │       content 被 base64 编码或 URL 编码 → 变成 JS 字符串的一部分
    │       → 内容直接嵌在 JS 代码里
    │
    └── 路径 B: shouldInline() = false → emitFile({ source: content })
            content 原样传给 Rolldown
            → Rolldown 算 contenthash + 写入 dist/assets/
            → 内容和代码分离，独立文件
```

### shouldInline 的判断逻辑（asset.ts 第754-784行）

```typescript
function shouldInline(environment, file, id, content, pluginContext, forceInline) {
  if (noInlineRE.test(id)) return false    // ?no-inline 后缀 → 强制 emit
  if (inlineRE.test(id)) return true       // ?inline 后缀 → 强制 inline

  if (buildPluginContext) {
    if (environment.config.build.lib) return true   // lib 模式 → 全 inline
    if (pluginContext.getModuleInfo(id)?.isEntry) return false  // 入口文件 → 不 inline
  }

  if (file.endsWith('.html')) return false              // HTML 不 inline
  if (file.endsWith('.svg') && id.includes('#')) return false  // SVG 带 fragment 不 inline

  // 核心判断：文件大小
  return content.length < assetsInlineLimit  // 默认 4096 字节
}
```

---

### 路径 A：inline —— 内容嵌进 JS 代码

#### 普通二进制文件（PNG、字体等）

```typescript
// asset.ts 第800-802行
const mimeType = mrmime.lookup(file)  // 'image/png' 或 'font/woff2'
return `data:${mimeType};base64,${content.toString('base64')}`
```

数据流：
```
content: Buffer (8320 bytes，PNG 原始二进制)
    ↓
content.toString('base64')
    ↓
'iVBORw0KGgoAAAANSUhEUgAA...'  (base64 字符串，变长约 11KB)
    ↓
`data:image/png;base64,iVBORw0KGgo...`
    ↓
formatBuiltAsset → 包进 JS export default
    ↓
最终 JS：export default "data:image/png;base64,iVBORw0KGgo..."
```

**内容被 base64 编码后变成 JS 字符串字面量，完全嵌在产物 JS 里。** Rolldown 在这里不再碰文件内容。

#### SVG —— 特殊处理，两种编码策略

SVG 是纯文本格式，有专门的 `svgToDataURL` 函数（第806-835行）：

```typescript
function svgToDataURL(content: Buffer): string {
  const stringContent = content.toString()

  // 策略1：如果 SVG 有 <text>、<foreignObject>、嵌套引号 → 老老实实 base64
  if (stringContent.includes('<text') || nestedQuotesRE.test(stringContent)) {
    return `data:image/svg+xml;base64,${content.toString('base64')}`
  }

  // 策略2：否则用 URL 编码（更短、更好压缩）
  return 'data:image/svg+xml,' +
    stringContent.trim()
      .replaceAll(/>\s+</g, '><')       // 去掉标签间空白
      .replaceAll('"', "'")            // 双引号改单引号
      .replaceAll('%', '%25')          // URL 编码特殊字符
      .replaceAll('#', '%23')
      .replaceAll('<', '%3c')
      .replaceAll('>', '%3e')
      .replaceAll(/\s+/g, '%20')       // 空格编码
}
```

**为什么 SVG 要单独处理？** SVG 是纯文本，URL 编码比 base64 **体积小 15-20%**，而且压缩率更高。PNG/字体等二进制文件没法 URL 编码，只能 base64。

#### inline 路径总结

```
content: Buffer
    │
    ├─ SVG 文本 → svgToDataURL() → URL 编码或 base64
    │
    └─ 二进制 → base64 编码
    │
    ↓ 结果统一是 JS 字符串
"data:image/svg+xml,%3Csvg%3E..."  或  "data:image/png;base64,iVBOR..."
    ↓
formatBuiltAsset 包进 export default
    ↓
最终 JS 里直接带着完整的 data URL 字符串
```

**inline 路径 Rolldown 不碰内容，全是 Vite 插件自己在 JS 层完成编码。**

---

### 路径 B：emit —— 内容交给 Rolldown，独立写盘

```typescript
// asset.ts 第675-681行
const referenceId = pluginContext.emitFile({
  type: 'asset',
  name: path.basename(file),   // 'logo.svg'
  originalFileName,             // 'src/logo.svg'
  source: content,              // ← 原始 Buffer，完整二进制数据！
})
```

**注意 `source: content`——原始 Buffer 直接传给了 Rolldown，没有任何编码！**

内容在 Rolldown 内部的流转：

```
emitFile({ source: content: Buffer })
    │
    ↓ Rolldown Rust 内核内部
    │
    ├─ 存进内部 asset 清单：{ referenceId, source: Buffer, name, originalFileName }
    │
    ├─ 等所有 emitFile 都注册完毕后，统一处理产物
    │   ├─ 用 source Buffer 算 contenthash
    │   │   → content.toString() 的 hash → '8f3a2b'
    │   ├─ 拼出产物名：'assets/logo-8f3a2b.svg'
    │   │
    │   ├─ 触发 resolveFileUrl（这步只处理 URL 字符串，不碰内容）
    │   │
    │   └─ 最终写盘：把原始 Buffer 原样写入 dist/assets/logo-8f3a2b.svg
    │
    ↓
dist/assets/logo-8f3a2b.svg  ← 内容和源文件一模一样，没有 base64、没有编码
```

#### 关键：emit 路径下内容全程不变

| 阶段 | 内容状态 |
|------|----------|
| `fsp.readFile` 读入 | `Buffer`（原始二进制） |
| `emitFile({ source: content })` | `Buffer` 原样传给 Rolldown |
| Rolldown 内部算 hash | 用 `Buffer` 算，不改内容 |
| Rolldown 写盘 | `Buffer` 原样写入 dist |

**emit 路径没有任何编码转换！** Rolldown 只管重命名（加 contenthash）和写入磁盘，文件内容完全保留原样。

---

### 补充：?raw 路径的内容处理

`?raw` 是第三种情况，适用于文本文件：

```typescript
// asset.ts 第247-249行
code: `export default ${JSON.stringify(
  await fsp.readFile(file, 'utf-8'),   // ← 注意：utf-8 读取！返回 string
)}`
```

```
fsp.readFile(file, 'utf-8')
    ↓
"床前明月光\n疑是地上霜"  (JS string)
    ↓
JSON.stringify → "\"床前明月光\\n疑是地上霜\""
    ↓
export default "\"床前明月光\\n疑是地上霜\""
    ↓
浏览器 import 拿到原始文本字符串
```

**?raw 只适用于文本文件**（`.txt`、`.html` 等），二进制文件不能用。不是 base64 也不是 URL 编码，就是原样读文本 → JSON.stringify 包一下。

---

### 三条路径对比

```
                        inline 路径                      emit 路径                      ?raw 路径
                        ──────────                      ─────────                      ─────────
读入                    Buffer (8KB PNG)                Buffer (8KB PNG)                String (utf-8)
                        ↓                                ↓                              ↓
内容处理                base64 编码 → 11KB 字符串        无处理，原样保留                 JSON.stringify 包一层
                        ↓                                ↓                              ↓
存放在哪                JS 代码里的字符串字面量            Rolldown 内部 asset 清单        JS 代码里的字符串字面量
                        ↓                                ↓                              ↓
最终产物                dist/main.js 里有                  dist/assets/logo-8f3a2b.svg    dist/main.js 里有
                        export default "data:image/..."   (独立二进制文件)                export default "原始文本..."
                        ↓                                ↓                              ↓
浏览器行为               解析 JS 字符串中的 data URL       通过 HTTP fetch 独立文件        import 拿到字符串
                        → 直接渲染                         → 下载后渲染                    → 直接使用
```

---

### 一句话总结内容处理

> 文件内容的处理，asset 插件只负责**两件事**：
> 1. **判断走哪条路** —— `shouldInline()` 用文件大小 + 后缀决定
> 2. **inline 时自己编码** —— `assetToDataURL()` 做 base64 或 URL 编码，嵌进 JS 字符串
>
> **emit 时内容就不管了**，直接把原始 Buffer 丢给 Rolldown，Rolldown 算个 hash 重命名一下，原样写盘。

---

## 九、术语表

| 术语 | 含义 |
|------|------|
| **resolveId** | 模块定位钩子，把 `import 'lodash'` 解析成绝对路径 |
| **load** | 模块加载钩子，返回模块的源码字符串 |
| **transform** | 模块转译钩子，修改源码（TypeScript→JS、define替换等） |
| **emitFile** | Rolldown 内核方法，告诉打包器"把这个文件加到产物里"，返回 referenceId |
| **referenceId** | emitFile 返回的唯一标识，让 Rolldown 能在后续阶段找到对应的 asset |
| **resolveFileUrl** | Vite 自定义钩子，把 `import.meta.ROLLDOWN_FILE_URL_*` 转成最终 URL |
| **Rolldown** | Vite 8 引入的 Rust 打包器内核，替代 Rollup（dev 仍用 Rollup） |
| **filter** | Rolldown 插件钩子的 Rust 层预筛选器，减少 JS 调用开销 |
| **handler** | Rolldown 插件钩子的 JS 层实际处理函数 |
| **inline** | 文件内容直接 base64 编码进 JS（data URL），适合小文件 |
| **emit** | 文件作为独立资源输出到 dist/assets/ |
| **isBundled** | 是否走 Rolldown 打包路径，dev=false，build=true |
