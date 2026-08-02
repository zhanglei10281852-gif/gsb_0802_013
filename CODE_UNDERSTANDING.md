# Webpack 构建基础设施：第一版理解记录

> 面向新维护团队的交接文档。版本 `5.99.9`（见 [package.json](file:///e:/newGsb/questions/GSB-013/Thor/package.json) 第 3 行），工具链沿用 Yarn 1（`yarn@1.22.22`）、Jest、TypeScript（`declarations.d.ts` / `types.d.ts` 由 `tsc` 从 JSDoc 生成）。
>
> 本文只描述当前仓库源码可确认的事实。所有 `文件:行号` 引用均已核对；无法从当前版本确认的内容显式标注 **未证实**。本轮及后续均只新增/修改本文件，不动生产代码、测试、schema、配置或 lockfile。

## 0. 三个执行域（贯穿全文的命名约定）

后文一律使用下面三个名字区分“什么时候、谁在跑”：

- **build-time compiler（构建期 compiler）**：`webpack()` → `Compiler` → `Compiler.compile` → `Compilation` 的一次前台编译。产物是内存里的 `Compilation`（模块图、chunk 图、`compilation.assets`）。此域内的对象大多是**每次编译新建、编译结束即可丢弃**。
- **output/runtime（写入产物的 runtime）**：`Compilation.seal` 结束后，`Compiler.emitAssets` 把 `compilation.assets` 写到 `outputFileSystem`。这里也包含被打进产物、在浏览器/Node 里执行的 webpack runtime 代码（`RuntimeModule` / `RuntimeGlobals`，由 `RuntimePlugin` 注入）。**注意区分**：seal 阶段“生成 runtime 代码”属于 build-time；把文件真正落盘属于 output。
- **watch/cache（watch 与缓存域）**：跨多次编译存活的长生命周期对象——`Compiler` 本身、`Cache`、`Watching`、`ResolverFactory`。它们**持有并复用**在 build-time 之间；watch 模式下反复触发 build-time，cache 在编译空档进入 idle。

一句话记忆：**`Compiler` / `Cache` / `Watching` 长命（watch/cache 域），`Compilation` 短命（build-time 域），落盘是 output 域**。

---

## 1. 对外入口 `webpack()`：从调用到一个可用的 `Compiler`

入口文件 [lib/index.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/index.js#L128-L132)。`module.exports` 是一个 lazy 函数，首次调用时 `require("./webpack")`，并把 `Compiler`、`Compilation`、`ModuleGraph`、`ChunkGraph` 等作为 getter 挂在同一个导出对象上（[lib/index.js:179-190](file:///e:/newGsb/questions/GSB-013/Thor/lib/index.js#L179-L190)）。真正的实现是 [lib/webpack.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js)。

`webpack(options, callback)` 定义在 [lib/webpack.js:121-193](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L121-L193)。流程：

1. **校验**：内部闭包 `create()` 先用预编译的 `webpackOptionsSchemaCheck` 快速校验；不通过才回退到完整 `validateSchema`（[lib/webpack.js:128-136](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L128-L136)）。
2. **单/多编译分流**：数组 → `createMultiCompiler`（[lib/webpack.js:44-58](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L44-L58)，内部对每个子配置调 `createCompiler`），否则 → `createCompiler`（[lib/webpack.js:151-157](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L151-L157)）。
3. **驱动**：若传了 `callback`，`watch` 为真调 `compiler.watch(...)`，否则 `compiler.run(...)`（run 完成后自动 `compiler.close`）（[lib/webpack.js:160-176](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L160-L176)）。若没有 `callback`，只返回 `compiler`，不触发编译。

### 1.1 配置规范化 → base 默认值 → 建 Compiler → 插件 → 完整默认值

核心是 `createCompiler(rawOptions, compilerIndex)`（[lib/webpack.js:65-97](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L65-L97)）。**顺序至关重要**，因为默认值只填 `undefined`，用户插件必须能在“完整默认值”之前介入：

1. `getNormalizedWebpackOptions(rawOptions)`（[lib/webpack.js:66](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L66)）——见 §1.2。
2. `applyWebpackOptionsBaseDefaults(options)`（[lib/webpack.js:67](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L67)）——只填 `context`（默认 `process.cwd()`）和 `infrastructureLogging`（[lib/config/defaults.js:162-165](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/defaults.js#L162-L165)）。这两项要早于插件，因为下一步就要建 `Compiler` 并可能打日志。
3. **`new Compiler(options.context, options)`**（[lib/webpack.js:68-71](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L68-L71)）——见 §1.3。
4. `new NodeEnvironmentPlugin(...).apply(compiler)`（[lib/webpack.js:72-74](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L72-L74)）——装上 Node 的输入/输出/watch 文件系统与 infra 日志（属 watch/cache 与 output 域的底座）。
5. **应用用户插件**（[lib/webpack.js:75-84](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L75-L84)）：函数插件以 `plugin.call(compiler, compiler)` 调用，对象插件走 `plugin.apply(compiler)`。此时用户可读写 `options` 并 tap 任意 `compiler.hooks`。
6. `applyWebpackOptionsDefaults(options, compilerIndex)`（[lib/webpack.js:85-88](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L85-L88)）——填 target/mode/output/optimization/cache/resolve 等全部剩余默认值，**必须在用户插件之后**。返回值含 `platform`，随即写入 `compiler.platform`（[lib/webpack.js:89-91](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L89-L91)）。
7. `compiler.hooks.environment.call()` 与 `afterEnvironment.call()`（[lib/webpack.js:92-93](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L92-L93)）——两个同步 `SyncHook`。
8. **`new WebpackOptionsApply().process(options, compiler)`**（[lib/webpack.js:94](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L94)）——把规范化后的选项翻译成内建插件栈，见 §2。
9. `compiler.hooks.initialize.call()`（[lib/webpack.js:95](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L95)）后返回 compiler。

以上 1–9 **全部同步执行**，`createCompiler` 返回时 compiler 已完全装配好，尚未开始任何编译。

### 1.2 `getNormalizedWebpackOptions`（配置规范化）

定义 [lib/config/normalization.js:127](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/normalization.js#L127)，导出于同文件 565 行。它是**纯同步**变换：接收用户原始 `WebpackOptions`，返回一个形状固定的 `WebpackOptionsNormalized`（只做归一化/克隆，不填默认值）。要点：

- `entry`（[normalization.js:181-189](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/normalization.js#L181-L189)）：`undefined` → `{ main: {} }`；函数 entry 被包成一层，异步结果再过 `getNormalizedEntryStatic`；其余走 `getNormalizedEntryStatic`（[:480-536](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/normalization.js#L480-L536)），把字符串/数组/对象统一成 `{ name: { import: [...], ... } }`。
- `output`（[:298-401](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/normalization.js#L298-L401)）：把旧的 `library*`/`libraryTarget` 等合并进单一 `library` 对象。
- `cache`（[:130-173](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/normalization.js#L130-L173)）：`true` → `{type:"memory"}`。
- `optimization`（[:274-297](file:///e:/newGsb/questions/GSB-013/Thor/lib/config/normalization.js#L274-L297)）：归一化 `runtimeChunk`/`splitChunks`，并把废弃的 `noEmitOnErrors` 迁到 `emitOnErrors`。

唯一对外导出符号是 `getNormalizedWebpackOptions`。

### 1.3 `Compiler` 的创建、hooks 与它持有什么

`Compiler` 定义在 [lib/Compiler.js:137](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L137)。构造函数一次性 `Object.freeze` 出全部 hooks（[Compiler.js:143-217](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L143-L217)）。构造时**立即持有**（长生命周期，属 watch/cache 域）：

- `this.resolverFactory = new ResolverFactory()`（[Compiler.js:266](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L266)）——解析器工厂，`WebpackOptionsApply` 之后由 `resolveOptions` hook 注入配置；跨编译复用。
- `this.cache = new Cache()`（[Compiler.js:287](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L287)）——watch/cache 域核心，见 §5。
- `this.requestShortener`、`this.platform`（先占位，后由 §1.1 步骤 6 覆盖）、以及一批 `this._lastCompilation` / `this._lastNormalModuleFactory` 等**跨编译清理**用的引用（[Compiler.js:305-308](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L305-L308)）。
- 文件系统槽位 `inputFileSystem` / `outputFileSystem` / `watchFileSystem` / `intermediateFileSystem` 初始为 `null`（[Compiler.js:232-239](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L232-L239)），由 `NodeEnvironmentPlugin` 填入。

关键 hooks 分域速查（均在构造函数内）：

| hook | 类型 | 域 | 行 |
| --- | --- | --- | --- |
| `environment` / `afterEnvironment` / `initialize` | Sync | 装配期 | [208-216](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L208-L216), [145](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L145) |
| `beforeRun` / `run` | AsyncSeries | build-time 起点 | [156-158](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L156-L158) |
| `thisCompilation` / `compilation` | Sync | build-time（新建 Compilation 时） | [166-169](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L166-L169) |
| `make` | **AsyncParallelHook** | build-time（模块构建入口） | [180](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L180) |
| `finishMake` / `afterCompile` | AsyncSeries | build-time | [182-184](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L182-L184) |
| `shouldEmit` | **SyncBailHook** | build↔output 边界（可短路跳过写盘） | [148](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L148) |
| `emit` / `assetEmitted` / `afterEmit` | AsyncSeries | output | [160-164](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L160-L164) |
| `watchRun` / `invalid` / `watchClose` | Async/Sync | watch/cache | [192-198](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L192-L198) |

> **注意 `make` 是 `AsyncParallelHook`**（[Compiler.js:180](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L180)），因此多个 `EntryPlugin`（每个 entry import 一个）tap 的 `addEntry` 会并行启动——这是多入口并行构建的根源。

---

## 2. `WebpackOptionsApply.process`：把选项翻译成内建插件栈

定义在 [lib/WebpackOptionsApply.js:78](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L78)，同步执行，返回 `options`（第 793 行）。它按固定顺序 `.apply()` 一整套内建插件，最后触发同步 hooks。对本文主链路最关键的几处：

1. 先把 `outputPath` / `recordsInputPath` / `recordsOutputPath` / `name` 写到 compiler（[WebpackOptionsApply.js:79-82](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L79-L82)）。
2. 依次装 externals、chunkFormat、chunkLoading、library、devtool、模块类型（`JavascriptModulesPlugin` / `JsonModulesPlugin` / `AssetModulesPlugin`，[:310-312](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L310-L312)）、experiments 等插件。
3. **入口装配块**：`new EntryOptionPlugin().apply(compiler)`（[:391](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L391)），紧接着 `compiler.hooks.entryOption.call(options.context, options.entry)`（[:392-396](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L392-L396)）——这一步**同步**地把用户 entry 变成 `EntryPlugin`/`DynamicEntryPlugin` 实例（见 §2.1）。
4. 紧随其后 `new RuntimePlugin().apply(compiler)`（[:398](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L398)）——output/runtime 域代码注入的入口，刻意排在 entry 之后。
5. 之后装依赖/parser 插件、stats 插件、optimization 插件（sideEffects、providedExports、usedExports、concatenateModules、`SplitChunksPlugin`、`runtimeChunk` 等）、`moduleIds`/`chunkIds` 策略、minimizer、`TemplatedPathPlugin`、`RecordIdsPlugin`。
6. **cache 插件**（[:654-752](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L654-L752)）：按 `options.cache.type` 装 `MemoryCachePlugin` 或文件系统缓存插件，外加 `ResolverCachePlugin`——watch/cache 域的实际实现从这里挂上 `compiler.cache.hooks`。
7. 收尾：`compiler.hooks.afterPlugins.call(compiler)`（[:760](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L760)）→ 在 `resolverFactory.hooks.resolveOptions` 上 tap 注入 resolve 配置（[:764-791](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L764-L791)）→ `compiler.hooks.afterResolvers.call(compiler)`（[:792](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L792)）。

### 2.1 entry 是怎么接到 `make` hook 上的

- **EntryOptionPlugin**（[lib/EntryOptionPlugin.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryOptionPlugin.js)）：`apply` 里 tap `compiler.hooks.entryOption`（SyncBailHook），回调中调 `EntryOptionPlugin.applyEntryOption(...)` 并 `return true` 表示已处理（[:20-25](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryOptionPlugin.js#L20-L25)）。函数 entry → `DynamicEntryPlugin`，静态 entry → 对每个 `import` 各 `new EntryPlugin(context, entry, options).apply(compiler)`（[:33-54](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryOptionPlugin.js#L33-L54)）。
- **EntryPlugin**（[lib/EntryPlugin.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryPlugin.js)）`apply`：
  - tap `compiler.hooks.compilation`，在 `compilation.dependencyFactories` 里登记 `EntryDependency → normalModuleFactory`（[:34-42](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryPlugin.js#L34-L42)）。这是每次新建 Compilation 时都要重登记的，因为 `normalModuleFactory` 每次编译新建。
  - `const dep = EntryPlugin.createDependency(entry, options)`（[:45](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryPlugin.js#L45)），然后 **tap `compiler.hooks.make`（tapAsync）** 在其中调 `compilation.addEntry(context, dep, options, err => callback(err))`（[:47-51](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryPlugin.js#L47-L51)）。

**结论**：entry 的“注册”是在装配期同步完成的（`entryOption` → 生成插件 → tap `make`/`compilation`），但真正“开始构建 entry 模块”发生在 build-time 的 `make` hook 触发时（§3.2）。`make` 是异步并行边界。

---

## 3. 一次非 watch 构建的主链路

调用方无 watch 时走 `compiler.run(callback)`。

### 3.1 `Compiler.run`

定义 [Compiler.js:474-609](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L474-L609)。

- 若 `this.running` 已为真，直接回 `ConcurrentCompilationError`（[:475-477](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L475-L477)）——同一 compiler 不能并发编译。
- **cache 空档处理**：若 `this.idle`，先 `this.cache.endIdle(...)` 退出 idle，再跑；否则直接跑（[:599-608](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L599-L608)）。这是 build-time 与 watch/cache 域的握手点。
- 内层 `run()`（[:583-597](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L583-L597)）：`beforeRun` → `run` → `readRecords` → **`this.compile(onCompiled)`**（[:593](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L593)）。均为 async 串行边界。
- `onCompiled`（[:509-581](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L509-L581)）：
  - `this.hooks.shouldEmit.call(compilation)` 返回 `false` 时**短路**：不写盘，直接建 `Stats` 并走 `done` hook（[:514-523](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L514-L523)）。
  - 否则 `process.nextTick` 后 `this.emitAssets(compilation, ...)`（output 域，§4）（[:525-528](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L525-L528)）。
  - `compilation.hooks.needAdditionalPass.call()` 为真时，置 `needAdditionalPass` 并在 `additionalPass` hook 后**再次 `this.compile(onCompiled)`**（[:533-552](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L533-L552)）——这是“多次编译”的一种来源（另一种是 seal 内部的 `needAdditionalSeal`，§3.5）。
  - 然后 `emitRecords` → `done` hook → `this.cache.storeBuildDependencies(...)`（[:556-576](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L556-L576)）。
- `finalCallback`（[:486-498](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L486-L498)）：完成后 `this.cache.beginIdle()`、`this.running = false`、必要时触发 `failed`，再回调 `callback` 并 `afterDone.call`。

### 3.2 `Compiler.compile`：一次编译的骨架

定义 [Compiler.js:1310-1355](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1310-L1355)：

1. `const params = this.newCompilationParams()`（[:1311](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1311)）——建 `NormalModuleFactory` 与 `ContextModuleFactory`（[:1298-1304](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1298-L1304)）。`NormalModuleFactory` 每次编译新建，旧的先 `_cleanupLastNormalModuleFactory`（[:1277-1290](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1277-L1290)）。
2. `beforeCompile` hook → `compile` hook（[:1312-1315](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1312-L1315)）。
3. **`const compilation = this.newCompilation(params)`**（[:1317](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1317)）：`createCompilation` 先 `_cleanupLastCompilation` 再 `new Compilation(this, params)` 存入 `_lastCompilation`（[:1259-1262](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1259-L1262)），随后同步触发 `thisCompilation` 与 `compilation` 两个 hook（[:1268-1275](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1268-L1275)）。**`EntryPlugin` 登记 `EntryDependency` 工厂正是在这一刻发生**（§2.1）。
4. **`make` hook（AsyncParallelHook）**（[:1322](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1322)）→ `finishMake`（[:1327](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1327)）→ `process.nextTick` 后 `compilation.finish(...)`（[:1333](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1333)）→ `compilation.seal(...)`（[:1338](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1338)）→ `afterCompile` hook（[:1343](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1343)）→ `callback(null, compilation)`。

每一步都是回调式异步边界，任一 `err` 立即冒泡到 `callback`。

### 3.3 `Compilation` 的创建、持有与 `make`（图构建）

`Compilation` 构造函数（[lib/Compilation.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js)）建立 build-time 域的核心状态：

- **`this.moduleGraph = new ModuleGraph()`**（[Compilation.js:1057](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1057)）——**在构造时即创建**，随 `make` 逐步填充。
- **`this.chunkGraph = undefined`**（[Compilation.js:1059](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1059)）——构造时**不存在**，直到 seal 才创建（§3.5）。
- 四个 `AsyncQueue`（模块构建流水线的引擎，parent 链式：build → factorize → addModule → processDependencies）：`processDependenciesQueue`（[:1063](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1063)）、`addModuleQueue`（[:1069](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1069)）、`factorizeQueue`（[:1076](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1076)）、`buildQueue`（[:1082](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1082)）。
- 集合：`this.entries`（Map，[:1103](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1103)）、`this.globalEntry`（[:1105](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1105)）、`this.modules`（Set，[:1125](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1125)）、`this.chunks`（Set，[:1117](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1117)）。
- 构建相关 hooks：`buildModule`（[:709](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L709)）、`succeedModule`（[:715](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L715)）、`finishModules`（AsyncSeries，[:739](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L739)）、`seal`（[:745](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L745)）、`processAssets`（[:937](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L937)）。

**`make` 触发后的模块构建流水线**（由上面四个队列驱动，方法名都是 `queue.add` 的薄封装，真逻辑在 `_` 前缀的处理器里）：

1. `addEntry(context, entry, optionsOrName, callback)`（[:2320](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2320)）→ `_addEntryItem`（[:2355](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2355)）：把依赖记进 `this.entries`，触发 `hooks.addEntry`（[:2403](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2403)），调 `addModuleTree`（[:2405](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2405)）。
2. `addModuleTree`（[:2269](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2269)）用 `dependencyFactories.get(Dep)` 找到工厂（即 §2.1 登记的 `normalModuleFactory`），调 `handleModuleCreation(...)`（[:2289](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2289)）。
3. `handleModuleCreation`（[:1935](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1935)）：`factorizeModule`（[:1952](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1952)）→ `addModule`（[:1999](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1999)，其中 `moduleGraph.setResolvedModule` / `setIssuerIfUnset` 写入 ModuleGraph）→ `_handleModuleBuildAndDependencies`（[:2062](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2062)）。
4. `_buildModule`（[:1492](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1492)）：触发 `hooks.buildModule`（[:1517](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1517)），调 `module.build(...)`（[:1519](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1519)），成功后 `hooks.succeedModule`（[:1547](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1547)）。
5. `_processModuleDependencies`（[:1593](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1593)）：对模块的依赖 `moduleGraph.setParents`（[:1672](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1672)），再对每个依赖递归 `handleModuleCreation`（[:1637](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1637)）——这就是沿依赖树的递归展开。

**异步边界**：每个阶段跳转（handleModuleCreation → factorize → addModule → build → processDependencies → 再回 handleModuleCreation）都跨一个 `AsyncQueue`；并行度由 `options.parallelism`（默认 100）控制。cycle 防护靠 `creatingModuleDuringBuild` WeakMap 与各队列的 `isProcessing` 判断。

### 3.4 `Compilation.finish`

`finish(callback)`（[Compilation.js:2781](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2781)）：清 `factorizeQueue`，`_computeAffectedModules`（[:2980](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2980)），触发 `hooks.finishModules`（AsyncSeries，[:2984](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2984)，如 `FlagDependencyExportsPlugin` 在此做跨模块导出分析），随后 `moduleGraph.freeze` → 收集依赖错误/警告 → `moduleGraph.unfreeze`（[:2989-3024](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2989-L3024)）。

**`ModuleGraph` 的可用时机**：对象自构造起就在（[:1057](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1057)），随 `make` 逐步填充；`finish` 完成后即完整稳定。此时 `ChunkGraph` 仍是 `undefined`——“make/finish 之后、seal 之前”正是 ModuleGraph 完整而 ChunkGraph 尚不存在的窗口。

### 3.5 `Compilation.seal`：从模块图到 chunk 图、代码生成、资源

`seal(callback)`（[Compilation.js:3050](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3050)）。`buildChunkGraph` 于文件 57 行 import，`createHash` 于 76 行 import。关键顺序（含 `ChunkGraph`/`assets` 的诞生点）：

1. **创建 ChunkGraph**：`const chunkGraph = new ChunkGraph(this.moduleGraph, this.outputOptions.hashFunction)`（[:3063](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3063)），`this.chunkGraph = chunkGraph`（[:3067](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3067)）。**这是 `ChunkGraph` 第一次成为实对象**——由 `Compilation.seal` 创建、`Compilation` 持有，构造函数持有的 `moduleGraph` 作为入参。
2. `hooks.seal.call()`（[:3075](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3075)）；`optimizeDependencies` while 循环（[:3078](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3078)）。
3. **从 entries 造 chunk/chunkGroup**：`moduleGraph.freeze("seal")`（[:3086](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3086)），循环 `this.entries`：`addChunk(name)`（[:3091](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3091)）、`new Entrypoint(options)`（[:3095](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3095)）、`chunkGraph.connectChunkAndEntryModule`（[:3116](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3116)），最后 **`buildChunkGraph(this, chunkGraphInit)`**（[:3227](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3227)）真正把 module→chunk 关系铺开，`afterChunks` hook（[:3228](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3228)）。**此后 ChunkGraph 才完整可用**。
4. **优化 hook 序列**：`optimize`（[:3232](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3232)）→ `optimizeModules` while（[:3234](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3234)）→ `optimizeChunks` while（[:3239](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3239)）→ **异步** `optimizeTree.callAsync`（[:3244](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3244)）→ **异步** `optimizeChunkModules.callAsync`（[:3253](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3253)）。
5. **ID 分配**：`beforeModuleIds`/`moduleIds`/`optimizeModuleIds`（[:3272-3275](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3272-L3275)），`beforeChunkIds`/`chunkIds`/`optimizeChunkIds`（[:3282-3285](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3282-L3285)），`assignRuntimeIds`（[:3287](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3287)）。
6. `optimizeCodeGeneration`（[:3308](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3308)）→ `createModuleHashes`（[:3313](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3313)）。
7. **代码生成**：`beforeCodeGeneration`（[:3318](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3318)）→ **异步** `this.codeGeneration(...)`（[:3319](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3319)）。`codeGeneration`（[:3468](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3468)）先建 `this.codeGenerationResults = new CodeGenerationResults(...)`（[:3470](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3470)），把每个模块×runtime 排成 jobs，交 `_runCodeGenerationJobs` 用 `asyncLib.eachLimit(jobs, options.parallelism, ...)`（[:3525](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3525)）并行执行，每个 job 缓存未命中时调 **`module.codeGeneration({ chunkGraph, moduleGraph, runtimeTemplate, runtime, codeGenerationResults, ... })`**（[:3651](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3651)），结果 `results.add(...)`（[:3673](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3673)）。
8. `processRuntimeRequirements`（[:3328](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3328)）——output/runtime 域：决定每个 chunk 需要哪些 `RuntimeGlobals` 并注入 `RuntimeModule`。
9. **哈希**：`const codeGenerationJobs = this.createHash()`（[:3334](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3334)，方法在 4321 行）→ 之后 `_runCodeGenerationJobs`（[:3338](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3338)）补跑依赖 fullHash 的模块。
10. **资源产出**：`clearAssets()`（[:3353](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3353)）→ `createModuleAssets()`（[:3356](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3356)）→ **异步** `createChunkAssets(...)`（[:3412](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3412)，内部 `asyncLib.forEachLimit(this.chunks, 15, ...)`，每 chunk 走 `getRenderManifest` 渲染源码并 `emitAsset` 进 `this.assets`）。
11. **`processAssets`（异步）**（[:3361](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3361)）→ `afterProcessAssets`（[:3367](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3367)），随后 `this.assets` 被冻结/包成弃用代理（[:3369-3382](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3369-L3382)）。
12. `needAdditionalSeal` 为真时 `unseal()` 后重新 `seal(callback)`（[:3393-3396](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3393-L3396)）——seal 内部触发多次编译的来源。否则 `afterSeal.callAsync`（[:3397](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3397)）→ `finalCallback`。

**`this.assets` 的可用时机**：`clearAssets` 后由 `createModuleAssets`/`createChunkAssets` 填充，`processAssets` 再加工，最后冻结；冻结后不应再变。此时 `Compilation` 携带完整的 `moduleGraph` / `chunkGraph` / `codeGenerationResults` / `assets`，交回 `Compiler.compile` 的回调。

---

## 4. output 域：`Compiler.emitAssets` 写盘

`onCompiled` 在 `shouldEmit` 不短路时调 `emitAssets(compilation, callback)`（[Compiler.js:676-1021](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L676-L1021)）：

1. 先触发 `hooks.emit`（AsyncSeries，[:1012](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1012)）——**插件修改最终 assets 的最后机会**；计算 `outputPath` 并 `mkdirp`（[:1014-1019](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1014-L1019)）。
2. `emitFiles` 用 `asyncLib.forEachLimit(assets, 15, ...)`（[:693](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L693)）逐个写：处理 query string 与 immutable、大小写冲突检测（`caseInsensitiveMap`）、`compareBeforeEmit`（存在且内容相同则跳过写盘以保 mtime）、`writeFile` 后把 Source 换成 `SizeOnlySource` 以释放内存，并触发 `hooks.assetEmitted`（[:832](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L832)）。
3. 全部完成后 `hooks.afterEmit.callAsync`（[:1003](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1003)）。

写入目标是 `this.outputFileSystem`（Node 下由 `NodeEnvironmentPlugin` 提供）。

---

## 5. watch/cache 域

### 5.1 `Watching`：一个 compiler 上的多次编译

`compiler.watch(watchOptions, handler)`（[Compiler.js:459-468](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L459-L468)）置 `running`/`watchMode` 并 `new Watching(this, watchOptions, handler)`。`Watching`（[lib/Watching.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js)）：

- 构造末尾 `process.nextTick` 触发首次 `_invalidate`（[:74-76](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L74-L76)）——首编译由此启动，而非显式调用。
- `_go`（[:109-239](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L109-L239)）是每次 rebuild 的驱动：暂停 watcher、把变更集写到 **`compiler.modifiedFiles`**（[:155](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L155)）/ **`compiler.removedFiles`**（[:157](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L157)）；若 idle 先 `compiler.cache.endIdle`（[:161-167](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L161-L167)）；触发 `compiler.hooks.watchRun`（[:178](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L178)），其中 **`this.compiler.compile(onCompiled)`**（[:234](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L234)）进入 build-time 域。
- `_done`（[:255-346](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L255-L346)）：完成后 `compiler.cache.beginIdle()` 并重新 arm watcher；若期间又 `invalid` 则 `storeBuildDependencies` 后再次 `_go`（[:281-298](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L281-L298)）。
- 文件变更经 `compiler.watchFileSystem.watch(...)`（[:356](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L356)）回调 → `_invalidate`；`invalid` 回调触发 `compiler.hooks.invalid.call(...)`（[:390](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L390)）。`_invalidate`（[:420-442](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L420-L442)）在 `running` 时只置 `invalid=true` 延后，否则立即 `_go`。

因为每次 rebuild 都走 `compiler.compile` → `newCompilation` → `new Compilation`，**每次编译都是全新的 `Compilation`**；`Compiler`/`Cache`/`Watching`/`ResolverFactory` 跨编译存活。

### 5.2 `Cache`（钩子驱动）与 `CacheFacade`

`Cache`（[lib/Cache.js:53](file:///e:/newGsb/questions/GSB-013/Thor/lib/Cache.js#L53)）本身**不存数据**，只暴露 hooks（[:55-68](file:///e:/newGsb/questions/GSB-013/Thor/lib/Cache.js#L55-L68)）：`get`（AsyncSeriesBailHook）、`store` / `storeBuildDependencies` / `endIdle` / `shutdown`（AsyncParallelHook）、`beginIdle`（SyncHook）。方法 `get`/`store`/`storeBuildDependencies`/`beginIdle`/`endIdle`/`shutdown`（[:78-159](file:///e:/newGsb/questions/GSB-013/Thor/lib/Cache.js#L78-L159)）只是转发到对应 hook。阶段常量 `STAGE_MEMORY=-10` / `STAGE_DEFAULT=0` / `STAGE_DISK=10` / `STAGE_NETWORK=20`（[:162-165](file:///e:/newGsb/questions/GSB-013/Thor/lib/Cache.js#L162-L165)）决定各缓存插件的先后。

- `MemoryCachePlugin`（[lib/cache/MemoryCachePlugin.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/MemoryCachePlugin.js)）在 `STAGE_MEMORY` tap `cache.hooks.store`/`get`/`shutdown`，用一个 `Map` 做内存缓存（[:22-56](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/MemoryCachePlugin.js#L22-L56)）。
- `compiler.getCache(name)`（[Compiler.js:331-337](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L331-L337)）返回 `CacheFacade`（[lib/CacheFacade.js:196](file:///e:/newGsb/questions/GSB-013/Thor/lib/CacheFacade.js#L196)），为每个 identifier 加 `${compilerPath}${name}|` 前缀并带上 `output.hashFunction`，再委托给根 `Cache`。

### 5.3 run 与 cache idle 的握手

`Compiler.run` 起点检查 `this.idle` 并 `cache.endIdle`（[Compiler.js:599-608](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L599-L608)），`finalCallback` 结束时 `cache.beginIdle`（[:486-498](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L486-L498)）；`onCompiled` 收尾处 `cache.storeBuildDependencies`（[:570-576](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L570-L576)）。`compiler.close`（[:1361-1378](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1361-L1378)）触发 `shutdown` hook 并 `cache.shutdown`。

---

## 6. 对象生命周期与归属速查

| 对象 | 由谁创建 | 由谁持有 | 何时可用 | 域 |
| --- | --- | --- | --- | --- |
| `Compiler` | `createCompiler`（[webpack.js:68](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L68)） | 调用方 | `createCompiler` 返回后 | watch/cache（长命） |
| `Cache` | `Compiler` 构造（[Compiler.js:287](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L287)） | `Compiler.cache` | 构造后，跨编译存活 | watch/cache |
| `Watching` | `compiler.watch`（[Compiler.js:466](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L466)） | `Compiler.watching` | watch 模式下 | watch/cache |
| `NormalModuleFactory` | `compile`→`newCompilationParams`（[Compiler.js:1300](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1300)） | `CompilationParams` / `Compilation` | 每次编译新建 | build-time |
| `Compilation` | `newCompilation`（[Compiler.js:1261](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1261)） | `Compiler._lastCompilation` + 传给回调 | `compile` 内，每次编译新建 | build-time（短命） |
| `ModuleGraph` | `Compilation` 构造（[Compilation.js:1057](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1057)） | `Compilation.moduleGraph` | 构造即在，`make`/`finish` 后完整 | build-time |
| `ChunkGraph` | `Compilation.seal`（[Compilation.js:3063](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3063)） | `Compilation.chunkGraph` | seal 中 `buildChunkGraph` 后完整；此前为 `undefined` | build-time |
| `CodeGenerationResults` | `Compilation.codeGeneration`（[Compilation.js:3470](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3470)） | `Compilation.codeGenerationResults` | seal 代码生成阶段后 | build-time→output |
| `compilation.assets` | `createModuleAssets`/`createChunkAssets` | `Compilation.assets` | seal 资源阶段后、冻结 | build→output |

---

## 7. 待核实 / 未证实清单（后续深入方向）

以下为本轮**未直接读源确认**的点，标为 **未证实**，避免误导：

- **未证实**：`buildChunkGraph`（[lib/buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js)）内部把 module 分配到 chunk 的具体算法（可用/available modules 传播、`SplitChunksPlugin` 在 `optimizeChunks` 中的介入细节）尚未逐行读。
- **未证实**：`RuntimePlugin` 如何依据 `RuntimeGlobals` 决定注入哪些 `RuntimeModule`，以及 output/runtime 域运行时的 chunk 加载协议（jsonp / import / require）细节。
- **未证实**：文件系统缓存（`cache.type === "filesystem"`）在 `WebpackOptionsApply` 中所装插件（`IdleFileCachePlugin` / `PackFileCacheStrategy` 等）的序列化与失效判定流程。
- **未证实**：`MultiCompiler`（[lib/MultiCompiler.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/MultiCompiler.js)）如何按 `setDependencies` 调度子 compiler 的先后与并行。
- **未证实**：`module.build` / `module.codeGeneration` 在 `NormalModule` 中的 loader 执行与 parser（`JavascriptParser`）AST 遍历细节。

> 需要时可用仓库现有 Jest（`yarn jest <file>`）与 TypeScript（`yarn type-check` 相关脚本，见 [package.json](file:///e:/newGsb/questions/GSB-013/Thor/package.json)）在不改生产代码的前提下核对上述行为。
