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

## 7. 端到端场景：两入口共享一个模块 + 一次动态 `import()`

维护团队最常问的问题是：“**源码里写的一条 `import` 到底怎么变成编译图、又怎么变成浏览器里跑的装载代码？**” 本节用一个具体场景把 §1–§5 的机制串成一条线，全程复用三个执行域（build-time / output-runtime / watch-cache）与对象所有权。

### 7.0 场景设定

```js
// entryA.js
import { s } from "./shared";      // 静态：两入口都依赖
import("./lazy").then(m => m.run); // 动态：只有 A 有

// entryB.js
import { s } from "./shared";      // 静态：与 A 指向同一个 shared 模块
```

配置里 `entry: { entryA: "./entryA.js", entryB: "./entryB.js" }`，**不启用** `SplitChunksPlugin` 的抽取（用默认 `optimization.splitChunks` 但 `shared` 不满足默认抽取条件，或显式关闭），以便先看清“不做任何优化时”的裸行为。

预告结论：`shared` 会**同时复制进 entryA 与 entryB 两个 chunk**；`lazy` 会拿到**自己的 async chunk**；`import()` 表达式会被翻译成 `__webpack_require__.e(chunkId).then(...)`，而 `.e` / jsonp 装载器由 `RuntimeModule` 生成、打进 runtime chunk。

### 7.1 第一步（build-time）：parser 把三条 import 变成 Dependency / Block

模块 build 阶段（§3.3 的 `_buildModule` → `module.build` → loader/parser）里，`JavascriptParser` 遍历 AST，不同 import 形态触发不同 hook，产出**只存在于构建期**的 `Dependency`/`AsyncDependenciesBlock`，挂在 `Module` 上。存储模型：`Module extends DependenciesBlock`（[Module.js:189](file:///e:/newGsb/questions/GSB-013/Thor/lib/Module.js#L189)），`this.dependencies` / `this.blocks` 数组与 `addDependency` / `addBlock` 都来自基类（[DependenciesBlock.js:32-64](file:///e:/newGsb/questions/GSB-013/Thor/lib/DependenciesBlock.js#L32-L64)）。

**静态 `import { s } from "./shared"`**（[HarmonyImportDependencyParserPlugin.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/HarmonyImportDependencyParserPlugin.js)）：

- `parser.hooks.import` 每条 import 语句触发一次：创建 `new HarmonyImportSideEffectDependency(source, order, attributes)` 并 `parser.state.module.addDependency(sideEffectDep)`（[:124-130](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L124-L130)）——代表“模块求值”这条边。
- `parser.hooks.importSpecifier` 只给变量打 `harmonySpecifierTag`（[:134-147](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L134-L147)），**此时不建依赖**。
- 真正的 `HarmonyImportSpecifierDependency` 在被标记变量**被使用时**由 `parser.hooks.expression.for(harmonySpecifierTag)` 惰性创建并 `module.addDependency(dep)`（[:196-216](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L196-L216)）。

**关键**：静态 import **不建 block**，依赖直接挂在 Module 上——它不是代码分割点。

**动态 `import("./lazy")`**（[ImportParserPlugin.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportParserPlugin.js)）：

- `parser.hooks.importCall` 触发（[:47](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportParserPlugin.js#L47)）。默认 lazy 模式下：
  - `const depBlock = new AsyncDependenciesBlock({ ...groupOptions, name: chunkName }, expr.loc, param.string)`（[:282-289](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportParserPlugin.js#L282-L289)）；
  - `const dep = new ImportDependency(param.string, expr.range, exports, attributes)`（[:290-295](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportParserPlugin.js#L290-L295)）；
  - **`depBlock.addDependency(dep)`**（[:298](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportParserPlugin.js#L298)）把依赖装进 block；
  - **`parser.state.current.addBlock(depBlock)`**（[:299](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportParserPlugin.js#L299)）把 block 挂到 entryA 模块的 `this.blocks`。

**`AsyncDependenciesBlock`**（[AsyncDependenciesBlock.js:24-48](file:///e:/newGsb/questions/GSB-013/Thor/lib/AsyncDependenciesBlock.js#L24-L48)）`extends DependenciesBlock`，构造签名 `(groupOptions, loc, request)`；`groupOptions.name` 即 `chunkName`（可来自 `/* webpackChunkName */`）。**它就是“分割点”**：一个 block 后续对应一个 ChunkGroup。

> 域归属：`Dependency`、`AsyncDependenciesBlock` 全部是 **build-time-only**，挂在 `Module` 上，随 `Compilation` 生灭。它们经 `makeSerializable` 可进**持久化缓存**（watch/cache 域），但**绝不进 emitted runtime**——最终产物里只有它们的 `.Template` 生成的源码字符串（§7.4）。

### 7.2 第二步（build-time）：ModuleGraph 记录“谁依赖谁”

`make` 递归（§3.3 的 `handleModuleCreation` → `_processModuleDependencies`）对每条依赖调 `moduleGraph.setResolvedModule` / `setParents`，把上面的 Dependency 解析成指向具体 `Module` 的边。结果：

- `shared` 是**同一个 `Module` 实例**（按 identifier 在 `_modules` 去重，[Compilation.js:1446](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1446)），被 entryA、entryB 各自的 `HarmonyImportSideEffectDependency` 指向——图里是**一个节点、两条入边**。
- `lazy` 模块由 `ImportDependency` 指向，但那条依赖被包在 `AsyncDependenciesBlock` 里；`ImportDependency.Template` 之后要靠 `moduleGraph.getParentBlock(dep)` 找回这个 block（[ImportDependency.js:122-124](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportDependency.js#L122-L124)）。

此刻（make/finish 之后、seal 之前，见 §3.4）`ModuleGraph` 完整；`ChunkGraph` 仍是 `undefined`。**ModuleGraph 只关心模块与依赖，不关心 chunk**。

### 7.3 第三步（build-time）：`buildChunkGraph` 把模块铺进 chunk

seal 阶段先按 entries 造出 entryA-chunk、entryB-chunk 两个入口 chunk + `Entrypoint`，塞进 `chunkGraphInit`（§3.5），再调 `buildChunkGraph(this, chunkGraphInit)`（[Compilation.js:3227](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3227)）。算法（[buildChunkGraph.js:1301](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L1301)）三阶段：`visitModules`（[:247](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L247)）→ `connectChunkGroups`（[:1216](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L1216)）→ `cleanupUnconnectedGroups`（[:1280](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L1280)）。

**`shared` 为何进两个 chunk**：entryA、entryB 是两个独立 root entrypoint，各自以 `minAvailableModules = ZERO_BIGINT`（空）入队（[:418-432](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L418-L432)）。`processBlock`（[:675](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L675)）遍历各自静态依赖时，`shared` 既不在本 chunk、也不在 `minAvailableModules` 里，于是走 `ADD_AND_ENTER_MODULE`，调 **`chunkGraph.connectChunkAndModule(chunk, module)`**（[:829](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L829)）。两个 entry 之间**无父子关系**，谁都不能把 `shared` 当作“已可用”，所以 `shared` 被分别连进 **两个** entry chunk。这正是 `SplitChunksPlugin` 存在的理由——裸算法不去重。

**`lazy` 为何独立成 async chunk**：`processBlock` 结尾遍历 `module.blocks`，对 entryA 的 `AsyncDependenciesBlock` 调 `iteratorBlock`（[:488](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L488)），其中 `compilation.addChunkInGroup(...)` **新建一个 ChunkGroup + Chunk**（[:582-587](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L582-L587)），并记进 `blockConnections`（[:643-647](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L643-L647)），稍后 `connectChunkGroups` 把它连成 entryA 的子 group。

**“available modules”去重规则**：async chunk 的 `minAvailableModules` 由父（entryA-chunk）的 `resultingAvailableModules`（= 父 minAvailable ∪ 父 chunk 内所有模块，[:894-910](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L894-L910)）经**交集**合并得到（[:947-997](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L947-L997)）。因此如果 `lazy` 也 import 了 `shared`，`processBlock` 里 `isOrdinalSetInMask(minAvailableModules, refOrdinal)` 命中（[:707-711](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L707-L711)），`shared` **不会**再复制进 lazy chunk——因为它已从 entryA 父 chunk 可用。这是“共享模块/异步块何时复用 chunk”的核心判据。

**`ChunkGraph` 允许一个 Module 属于多个 Chunk**：每模块记录 `ChunkGraphModule.chunks = new SortableSet()`（[ChunkGraph.js:201-202](file:///e:/newGsb/questions/GSB-013/Thor/lib/ChunkGraph.js#L201-L202)），`connectChunkAndModule` 同时往 `cgm.chunks` 和 `cgc.modules` 各加一次（[:343-348](file:///e:/newGsb/questions/GSB-013/Thor/lib/ChunkGraph.js#L343-L348)）。这就是 `shared` 能同时在两个 chunk 的数据结构基础。`ChunkGraph` 用 `WeakMap` 挂在 live 对象上（[:255-260](file:///e:/newGsb/questions/GSB-013/Thor/lib/ChunkGraph.js#L255-L260)），**build-time-only，不序列化进产物**。

### 7.4 第四步（build-time → 翻译进 runtime）：code generation

seal 的代码生成阶段（§3.5 第 7 步）对每个模块调 `module.codeGeneration(...)`，Dependency 的 `.Template` 负责把源码里的 import 表达式替换成 `__webpack_require__` 调用，**同时**往该模块的 `runtimeRequirements`（一个 `Set<string>`）里登记需要的 `RuntimeGlobals`：

- **静态 import** → `HarmonyImportDependency.Template`（[HarmonyImportDependency.js:279](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/HarmonyImportDependency.js#L279)）经 `RuntimeTemplate.importStatement` 生成 `__webpack_require__(moduleId)`，并 `runtimeRequirements.add(RuntimeGlobals.require)`（[RuntimeTemplate.js:842-843](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimeTemplate.js#L842-L843)）。**只要 require，不涉及 chunk 装载**。
- **动态 import** → `ImportDependency.Template.apply`（[ImportDependency.js:116](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportDependency.js#L116)）经 `moduleNamespacePromise` → `blockPromise` 生成 `__webpack_require__.e(chunkId).then(__webpack_require__.bind(__webpack_require__, moduleId))`，并 `add(RuntimeGlobals.ensureChunk)`（[RuntimeTemplate.js:1009](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimeTemplate.js#L1009)）+ `add(RuntimeGlobals.require)`（[:690-691](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimeTemplate.js#L690-L691)）。`chunkId` 正是 §7.3 里为 `lazy` 新建的那个 async chunk 的 id（Template 靠 `getParentBlock(dep)` → chunkGroup 找到它）。

`RuntimeGlobals` 只是字符串常量：`require = "__webpack_require__"`、`ensureChunk = "__webpack_require__.e"`、`ensureChunkHandlers = "__webpack_require__.f"`（[RuntimeGlobals.js:76-81](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimeGlobals.js#L76-L81)）。**runtimeRequirements 是构建期元数据，不是代码**——它记录“这段产物需要哪些运行时能力”，下一步才被翻译成真实运行时代码。

### 7.5 第五步（build-time 决策 → output-runtime 落地）：runtime requirements 展开成 `RuntimeModule`

`processRuntimeRequirements`（[Compilation.js:3708](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3708)，seal 中 [:3328](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3328) 调用）把各模块的 `Set<string>` 沿 module→chunk→tree 汇聚，然后对每个 requirement 通过 `runtimeRequirementInTree.for(r).call(...)`（[:3824-3828](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3824-L3828)）扇出给插件，插件据此**追加 requirement 并注入 `RuntimeModule`**：

1. `RuntimePlugin` tap `for(RuntimeGlobals.ensureChunk)`（[RuntimePlugin.js:371-380](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimePlugin.js#L371-L380)）：若 chunk 有 async 子 chunk 就 `set.add(ensureChunkHandlers)`（[:376](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimePlugin.js#L376)），并 `compilation.addRuntimeModule(chunk, new EnsureChunkRuntimeModule(set))`（[:378-380](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimePlugin.js#L378-L380)）。
2. 上一步把 `ensureChunkHandlers`（`.f`）推入 requirement 集，触发 `JsonpChunkLoadingPlugin` tap `for(RuntimeGlobals.ensureChunkHandlers)`（[JsonpChunkLoadingPlugin.js:54](file:///e:/newGsb/questions/GSB-013/Thor/lib/web/JsonpChunkLoadingPlugin.js#L54)），它 `addRuntimeModule(chunk, new JsonpChunkLoadingRuntimeModule(set))`（[:48-51](file:///e:/newGsb/questions/GSB-013/Thor/lib/web/JsonpChunkLoadingPlugin.js#L48-L51)），并追加 `publicPath` / `loadScript` 等传递性 requirement。

`RuntimeModule extends Module`（[RuntimeModule.js:32-38](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimeModule.js#L32-L38)），是真正的“模块”，有 `generate()` 产出源码。`addRuntimeModule`（[Compilation.js:3843](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3843)）把它加入 `this.modules` 并 `connectChunkAndModule` 连进目标 chunk（通常是 runtime chunk）。它们的 `generate()` 就是**最终打进产物、在浏览器里执行的 runtime**：

- `EnsureChunkRuntimeModule.generate`（[EnsureChunkRuntimeModule.js:26](file:///e:/newGsb/questions/GSB-013/Thor/lib/runtime/EnsureChunkRuntimeModule.js#L26)）：定义 `__webpack_require__.f = {}` 与 `__webpack_require__.e = chunkId => Promise.all(Object.keys(f).reduce(...))`——即 `.e` 遍历所有已注册的 `.f` 处理器。
- `JsonpChunkLoadingRuntimeModule.generate`（[JsonpChunkLoadingRuntimeModule.js:75](file:///e:/newGsb/questions/GSB-013/Thor/lib/web/JsonpChunkLoadingRuntimeModule.js#L75)）：把 `__webpack_require__.f.j = (chunkId, promises) => {...}` 注册进 `.f`（[:147](file:///e:/newGsb/questions/GSB-013/Thor/lib/web/JsonpChunkLoadingRuntimeModule.js#L147)），内部用 `loadScript` 插入 `<script>` 拉取 `lazy` chunk 文件，并挂 `webpackJsonpCallback` 在 chunk 到达时 resolve 那个 Promise。

### 7.6 一图流：从一条 import 到浏览器装载

| 源码 | build-time 产物 | 图/所有权 | 翻译进 runtime 的部分 | 域 |
| --- | --- | --- | --- | --- |
| `import {s} from "./shared"` | `HarmonyImportSideEffectDependency` + `...SpecifierDependency`（挂 entryA/entryB 的 `module.dependencies`） | ModuleGraph 一节点两入边；ChunkGraph 把 `shared` 连进两 chunk（`connectChunkAndModule`） | `__webpack_require__(id)` + `RuntimeGlobals.require` | build→output |
| `import("./lazy")` | `AsyncDependenciesBlock` + 内含 `ImportDependency`（挂 entryA 的 `module.blocks`） | block → 新 ChunkGroup+Chunk（`iteratorBlock`/`addChunkInGroup`）；lazy 独立 async chunk | `__webpack_require__.e(chunkId).then(require.bind(...))` + `ensureChunk` | build→output |
| （由 `ensureChunk` 派生） | requirement 集 `{ensureChunk, ensureChunkHandlers, ...}` | 无图节点，是 chunk 的 `runtimeRequirements` 元数据 | `EnsureChunkRuntimeModule` + `JsonpChunkLoadingRuntimeModule` 的 `generate()` 源码，打进 runtime chunk | output-runtime |

**只存在于构建期**（编译结束即弃、不进产物）：`Dependency`、`AsyncDependenciesBlock`、`ModuleGraph`、`ChunkGraph`、`CodeGenerationResults`、`runtimeRequirements`（`Set<string>`）。
**被翻译进 emitted runtime**：Dependency `.Template` 生成的 `__webpack_require__(...)` / `.e(...).then(...)` 表达式，以及 `RuntimeModule.generate()` 产出的 `.e` / `.f.j` 装载器代码。
**跨编译存活（watch/cache 域）**：可序列化的 `Dependency`/`Block`（进持久化缓存以复用 build 结果），以及 §5 描述的 `Compiler`/`Cache`。

---

## 8. watch 模式失效与缓存复用：排查“改了没生效 / 没改却重做”

本节沿用三域（build-time / output-runtime / watch-cache）与对象所有权，专门支撑两类线上问题的排查：
- **A. 文件改了却复用旧结果**（stale reuse）——本该重建/重解析的模块被判为“可复用”。
- **B. 什么都没改却整轮重做**（over-rebuild）——本该整体复用的却重新构建。

两类问题的判定权分散在三个层次：`Watching`（要不要开新一轮、这轮认为哪些文件变了）→ `NormalModule.needBuild` + `FileSystemInfo` snapshot（单模块要不要重建）→ `PackFileCacheStrategy` + build/resolve snapshot（持久化缓存整体是否作废）。**弄错在哪一层，就会误判 A 还是 B。**

### 8.1 `Watching` 收到 invalidation：汇总变更 → 暂停 watcher → 开新 compile

文件监听回调进入 [Watching.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js)：

1. **汇总 changed/removed**：`watch(...)` 注册的 change 回调调 `_invalidate(fileTimeInfoEntries, contextTimeInfoEntries, changedFiles, removedFiles)`（[Watching.js:379-384](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L379-L384)）。`_invalidate`（[:420-442](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L420-L442)）把本次变更并入 `_collectedChangedFiles` / `_collectedRemovedFiles`，合并逻辑在 `_mergeWithCollected`（[:83-100](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L83-L100)）——**“后事件覆盖”**：changed 里出现的从 removed 里删掉，反之亦然。
2. **运行中再次 invalid 的合并（关联问题 B/A）**：若 `this.running`，`_invalidate` **只**把变更并入 collected 并置 `this.invalid = true`（[:431-433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L431-L433)），**不**立刻开新构建；被推迟到下一轮。若 `suspended`/`blocked`，只合并不置位（[:426-429](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L426-L429)）。
3. **`_go` 开一轮**（[:109-239](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L109-L239)）：
   - **暂停 watcher**：`pausedWatcher = watcher`、`watcher.pause()`、`watcher = null`、记 `lastWatcherStartTime`（[:113-120](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L113-L120)）。
   - **设 `compiler.fsStartTime = Date.now()`**（[:121](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L121)）——这是本轮的“读盘起点”，后面 snapshot 校验的关键时间基准（§8.3）。
   - **取时间信息**：优先用回调 params，其次 `pausedWatcher.getInfo()`，再退化到 `getAggregatedChanges()`（[:122-153](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L122-L153)）。
   - **把变更交给 compiler 并清空 collected**：`compiler.modifiedFiles = _collectedChangedFiles`（[:155](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L155)）/ `compiler.removedFiles = _collectedRemovedFiles`（[:157](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L157)），随即置 `undefined`（[:156-158](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L156-L158)）。**consume-and-clear**：此后到达的变更只能靠步骤 2 的 running 合并进入下一轮。
   - **进新 compile**：若 idle 先 `cache.endIdle`（[:161-167](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L161-L167)）→ `readRecords`（[:168-175](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L168-L175)）→ 复位 `invalid=false`（[:176](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L176)）→ `compiler.hooks.watchRun.callAsync`（[:178](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L178)）→ **`compiler.compile(onCompiled)`**（[:234](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L234)）。**每轮都是全新的 `Compilation`**（§3.2），旧 `Compilation` 的图/资产不跨轮携带——跨轮复用只能来自 `Cache`（§8.4）。

### 8.2 `_done`：收尾、按 compilation 依赖重挂 watcher、再失效分支

`_done`（[Watching.js:255-346](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L255-L346)）：

- **再失效分支（不进 idle 直接开下一轮）**：若 `this.invalid`（构建期间有新变更）且未 suspended/blocked，`storeBuildDependencies` 后直接 `_go()`（[:281-298](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L281-L298)）——这里消费上一轮 running 合并进 collected 的变更。
- **正常收尾**：建 `Stats`（[:303-307](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L303-L307)）→ `done` hook（[:314](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L314)）→ `cache.storeBuildDependencies(compilation.buildDependencies, ...)`（[:318-321](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L318-L321)）→ **`cache.beginIdle()` + `compiler.idle = true`**（[:326-327](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L326-L327)）。
- **重挂 watcher**：`process.nextTick` 里 `this.watch(compilation.fileDependencies, compilation.contextDependencies, compilation.missingDependencies)`（[:329-340](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L329-L340)）。**下一轮监听哪些路径，完全由上一轮 `Compilation` 汇总出的三组依赖决定**（§8.3 第 4 点）。若某依赖没被登记，改它就不会触发 invalidation → 表现为问题 A。

失效发生的三条边界（本节主题）：
- **编译图边界**：`compiler.modifiedFiles/removedFiles`（[:155-157](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L155-L157)）影响 `FileSystemInfo` 的“已知变更”，进而决定 `needBuild` 重建哪些模块 → 决定 ModuleGraph 哪些节点重解析。
- **代码生成结果边界**：模块即便不重建，其 code generation 结果是否复用由 `_codeGenerationCache` 的 etag 决定（§8.4）。
- **写出资产边界**：即便重新生成了 assets，`Compiler.emitAssets` 仍可能因 `compareBeforeEmit`/immutable 跳过实际写盘（见 §4）——“重新生成”≠“重新写盘”。

### 8.3 `FileSystemInfo` snapshot 与四类依赖：单模块复用的判据

**snapshot 保存什么事实**（[FileSystemInfo.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js) `class Snapshot` [:271-306](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L271-L306)）：文件/目录的 timestamp、hash 或两者（`snapshot.<type>` 的 mode，[:2203](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2203)）、**missing 文件的“当时不存在”事实**（`missingExistence`，[:295](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L295)）、managed/immutable 路径信息（[:297-303](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L297-L303)）。

1. **创建**：`createSnapshot(startTime, files, directories, missing, options, callback)`（[:2170](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2170)），`startTime` 由构建期传入。`checkManaged`（[:2265-2306](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2265-L2306)）把 managed/immutable 路径归入“假定不变”集合，正常校验时跳过——**若把频繁变化的路径误判进 managedPaths，会造成问题 A**。
2. **校验**：`checkSnapshotValid(snapshot, callback)`（[:2729](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2729)）先查 `_snapshotCache`（同一 `FileSystemInfo` 内 memoize，[:2730-2739](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2730-L2739)），否则走 `_checkSnapshotValidNoCache`（[:2750](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2750)）。核心时间规则在 `checkFile`：**`if (typeof startTime === "number" && c.safeTime > startTime) return false;`**（[:2829](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2829)，目录版 [:2871](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2871)）——文件 `safeTime` 晚于 `startTime` 就判失效。`missingExistence` 的新建/删除通过 `checkExistence`（[:2803](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2803)）捕获。
3. **startTime = `compiler.fsStartTime`**：`NormalModule.build` 用 `const startTime = compilation.compiler.fsStartTime || Date.now()`（[NormalModule.js:1198](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1198)）给本模块 snapshot 打时间戳；而 `fsStartTime` 正是 §8.1 `_go` 在读盘前设的（[Watching.js:121](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L121)）。**这条“读盘起点”是防止“构建途中被改的文件被当作已消化”的护栏**——构建期间（`safeTime > startTime`）改动的文件一律判失效，强制下一轮重建。
4. **`needBuild` 决策阶梯**（[NormalModule.js:1540-1591](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1540-L1591)）：`_forceBuild`（[:1543](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1543)，由 `invalidateBuild()` [:1531-1533](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1531-L1533) 置位）→ `this.error` 重试 → `!cacheable`（[:1552](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1552)）→ `!snapshot`（[:1555](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1555)）→ `valueDependencies` 与 `valueCacheVersions` 比对（DefinePlugin 值变化，[:1558-1573](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1558-L1573)）→ **`fileSystemInfo.checkSnapshotValid(snapshot, ...)`**（[:1576](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1576)），失效即 `callback(null, true)` 重建。`build()` 末尾 `createSnapshot(...)` 存入 `buildInfo.snapshot`（[:1315-1329](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1315-L1329)）。`Compilation._buildModule` 正是在 build 前调 `module.needBuild(...)`（[Compilation.js:1500](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1500)），返回 false 则跳过、触发 `stillValidModule`。
5. **loader 额外登记的依赖**：loader 通过 `loaderContext.addDependency/addContextDependency/addMissingDependency`（由 loader-runner 实现，回填到 `result.fileDependencies` 等），在 `NormalModule.build` 里 `fileDependencies.addAll(result.fileDependencies)`（[NormalModule.js:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)）并进入该模块 snapshot；loader 本身路径与 `addBuildDependency`（[:810-817](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L810-L817)）进 `buildInfo.buildDependencies`。**loader 忘记 `addDependency` 是问题 A 的经典根因**：读了某文件却没登记，snapshot 不含它，改它不触发重建。
6. **四类依赖聚合与去向**：`Compilation` 构造四个 `LazySet`：`fileDependencies`/`contextDependencies`/`missingDependencies`/`buildDependencies`（[Compilation.js:1183-1189](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1183-L1189)）。`summarizeDependencies()`（[:4203](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L4203)，seal 中 [:3384](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3384) 调用）合并子编译并对每模块 `addCacheDependencies`。前三者 → §8.2 重挂 watcher；`buildDependencies` → §8.4 持久化缓存整体校验。

### 8.4 `CacheFacade` identifier/etag 与 `PackFileCacheStrategy`：持久化缓存整体复用

**CacheFacade 的 identifier/etag 模型**（[CacheFacade.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/CacheFacade.js)）：identifier 是带 name 前缀的字符串（`getItemCache` 拼 `${this._name}|${identifier}`，[:225-231](file:///e:/newGsb/questions/GSB-013/Thor/lib/CacheFacade.js#L225-L231)）；etag 由 `getLazyHashedEtag(obj)`（[:237-239](file:///e:/newGsb/questions/GSB-013/Thor/lib/CacheFacade.js#L237-L239)，其 `toString()` 惰性算 `obj.updateHash` 的 hash）或 `mergeEtags`（[:246-248](file:///e:/newGsb/questions/GSB-013/Thor/lib/CacheFacade.js#L246-L248)，`"a|b"`）产生；`provide/providePromise`（[:318-345](file:///e:/newGsb/questions/GSB-013/Thor/lib/CacheFacade.js#L318-L345)）= get-else-compute-then-store。

具体用法在 `Compilation`：模块缓存 `_modulesCache = getCache("Compilation/modules")`（[Compilation.js:1203](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1203)）用 **etag=null**，靠 snapshot 判有效（[:1433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1433)/[:1536](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1536)）；**代码生成缓存**用真 etag：`_codeGenerationModule` 以 identifier `${module.identifier()}|${runtimeKey}`、etag `${hash}|${dependencyTemplates.getHash()}`（[:3639-3640](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3639-L3640)）。**结论：watch 下复用一个模块的产物，需要同时满足 (a) snapshot 仍有效（不重建）与 (b) codegen etag 不变（不重生成）**。

**IdleFileCachePlugin**（[IdleFileCachePlugin.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/IdleFileCachePlugin.js)）在 `STAGE_DISK` tap `cache.hooks.store`（不立刻写，入 `pendingIdleTasks` 队列，[:58-65](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/IdleFileCachePlugin.js#L58-L65)）、`get`（先跑挂起的同 id store 再 `strategy.restore`，[:67-92](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/IdleFileCachePlugin.js#L67-L92)）、`beginIdle`（起定时器，[:178-218](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/IdleFileCachePlugin.js#L178-L218)）、`processIdleTasks`（空闲时把 pack 落盘，`strategy.afterAllStored()`，[:137-175](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/IdleFileCachePlugin.js#L137-L175)）。**因此 `cache.beginIdle()`（§8.2 收尾）才是持久化真正落盘的触发点**——进程若在 idle 前退出，缓存可能未写全。

**PackFileCacheStrategy**（[PackFileCacheStrategy.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js)）：
- 构造持有 `fileSystemInfo`、`version`（salt，[:1109](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1109)）、`buildDependencies`/`newBuildDependencies`、`buildSnapshot`/`resolveBuildDependenciesSnapshot`/`resolveResults`（[:1080-1136](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1080-L1136)）。
- `store`/`restore` 把 etag `toString()` 后交给内存 Pack；`Pack.get` **etag 不匹配返回 `null`（stale）**（[:167](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L167)）。
- **整体作废条件**（`_openPack` 恢复时，[:1151-1309](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1151-L1309)）：① `packContainer.version !== version` 直接丢弃（[:1197-1202](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1197-L1202)）；② `checkSnapshotValid(buildSnapshot)` 失败（build 依赖变了，[:1206-1225](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1206-L1225)）；③ resolve 快照失败且 `checkResolveResultsValid` 也不过（[:1228-1268](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1228-L1268)）。只有 build 与 resolve 都有效才加载旧 pack，否则返回空 Pack。
- `afterAllStored`（[:1353-1534](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1353-L1534)）用 `fileSystemInfo.resolveBuildDependencies` + `createSnapshot` 生成 build/resolve 快照并写回 `PackContainer(pack, version, buildSnapshot, ...)`。

**build dependencies vs file/context/missing dependencies 的区别**：前者描述“**如何构建**”（loader 模块、config、webpack 自身），由 `resolveBuildDependencies` 单独解析、用独立快照校验，**一旦变化作废整个持久化缓存**（问题 B 的合理来源）；后者描述“**构建的输入**”，进单模块 snapshot，只影响该模块是否重建。`version` salt 变化（webpack 版本/配置指纹，构造于本文件上游，**未证实**其精确组成）同样整体作废——这是升级 webpack 或改配置后“整轮重做”的正常表现。

### 8.5 判定表：可安全复用 / 必须重建或重解析 / 只需重新生成或写出

> 说明：以下按“单模块”视角给出主判据与源码依据。“重解析”指重跑 loader+parser 从而可能改变 ModuleGraph 的出边；“重生成”指模块不重建但 code generation 结果因 etag 变化而重算；“重写盘”指 assets 变化后 `emitAssets` 实际写出。

| 变化类型 | Watching 层 | needBuild / snapshot 层 | 持久化缓存层 | 判定结果 | 主要依据 |
| --- | --- | --- | --- | --- | --- |
| **普通源码文件改动**（模块自身或已登记的 fileDependency） | `modifiedFiles` 含它，触发新一轮 | `checkSnapshotValid` 失效（timestamp/hash 变或 `safeTime>startTime`） | 该模块 codegen etag（hash）变 | **必须重建 + 重解析**该模块；下游依赖按图重连；其余模块复用 | [FileSystemInfo.js:2829](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2829)、[NormalModule.js:1576](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1576) |
| **仅下游 chunk 归属/ID 变化，模块源码未变** | 可能因别处改动触发一轮 | snapshot 仍有效 → **不重建** | codegen etag 含 `hash`，随 chunk/hash 变而变 | **只需重新生成 + 重写出资产**（模块本身复用） | [Compilation.js:3639-3640](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3639-L3640) |
| **loader 额外登记的依赖变化**（正确 `addDependency`） | 该依赖在上一轮进了 `compilation.fileDependencies` → watcher 监听 → 触发 | 该依赖在模块 snapshot 内 → `checkSnapshotValid` 失效 | 同源码改动 | **必须重建 + 重解析**该模块 | [NormalModule.js:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)、[:1315-1321](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1315-L1321) |
| **loader 读了却未登记依赖**（缺 `addDependency`） | 该文件不在 watcher 列表 → **不触发** | 即便触发，snapshot 也不含它 → 判有效 | — | **错误复用（问题 A）**；根因是依赖未登记，非缓存 bug | 反证：[Watching.js:329-340](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L329-L340) 只监听 compilation 汇总的依赖 |
| **缺失依赖出现**（原先 import 的文件从无到有） | 该路径作为 missingDependency 被监听 → 触发 | snapshot 的 `missingExistence` 经 `checkExistence` 判失效 | 重建后重新解析该 import | **必须重建 + 重解析**（可能新增图节点/新 async chunk） | [FileSystemInfo.js:2803](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2803)、[:295](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L295) |
| **build 依赖 / 配置 / webpack 版本变化** | 下一轮照常 | 单模块 snapshot 未必变 | `checkSnapshotValid(buildSnapshot)` 失败或 `version` 不匹配 → **丢弃整个 pack** | **整轮重建**（持久化缓存全失效，问题 B 的合理表现） | [PackFileCacheStrategy.js:1197-1202](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1197-L1202)、[:1206-1225](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1206-L1225) |
| **构建过程中再次 invalid** | running 时只置 `invalid=true` 并合并 collected（[:431-433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L431-L433)），`_done` 再失效分支直接 `_go`（[:281-298](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L281-L298)） | 构建期改动因 `safeTime>startTime` 在**下一轮**判失效 | — | **本轮完成后立即再跑一轮**；受影响文件下一轮重建（避免用到途中被改内容） | [Watching.js:431-433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L431-L433)、[FileSystemInfo.js:2829](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2829) |
| **managed/immutable 路径下的文件改动** | 取决于 watcher 是否覆盖 | `checkManaged` 归入“假定不变” → 正常校验跳过 | managed 项由 `managedItemInfo`（版本）判定 | **默认复用**；若误配 managedPaths 会导致**错误复用（问题 A）** | [FileSystemInfo.js:2265-2306](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2265-L2306) |

**排查口诀**：
- 命中问题 **A（stale）**：先看该文件是否在 `compilation.fileDependencies/contextDependencies/missingDependencies`（决定是否被监听）→ 再看是否落入该模块的 snapshot（决定 `checkSnapshotValid` 是否管它）→ 再排除 managedPaths/immutablePaths 误配。多为**依赖未登记**或**managedPaths 误配**，而非缓存核心逻辑错误。
- 命中问题 **B（over-rebuild）**：先看是否 build 依赖/`version` 触发了整 pack 作废（`_openPack` 的丢弃分支）→ 再看是否非绝对路径依赖（[NormalModule.js:1262-1312](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1262-L1312) 会 warn）导致 snapshot 反复失效 → 再看是否有 `safeTime>startTime` 的写入型 loader 造成每轮自失效。

---

## 9. 模块被构建之前：factorize / resolve / createModule → loader-runner / parser / generator

线上问题常出在“rule 匹配、resolve、loader 三者之间”——模块还没真正 build 出来的那一段。本节补上 §3.3 `handleModuleCreation` 里 `factorizeModule` 之后、`_buildModule` 之前的全过程，仍沿用三域与对象所有权。

> 版本注记：`NormalModuleFactory` 在本版本**不使用 AsyncQueue**（构造函数导入只有 tapable，[NormalModuleFactory.js:10-16](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L10-L16)），factorize/resolve 全靠 hook `callAsync` 串联（与 §3.3 里 `Compilation` 侧的 `factorizeQueue` 是两回事——队列在 Compilation，工厂本身不排队）。`NormalModule` 用**单个 `createData` 对象**构造，非位置参数。

### 9.1 `NormalModuleFactory` 的 hooks 与 `create` 流水线

工厂 hooks（构造函数 [NormalModuleFactory.js:274-311](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L274-L311)）：`resolve`（AsyncSeriesBailHook，[:276](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L276)）、`resolveForScheme`/`resolveInScheme`（HookMap，[:278-284](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L278-L284)）、`factorize`（[:286](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L286)）、`beforeResolve`（[:288](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L288)）、`afterResolve`（[:290](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L290)）、`createModule`（[:292](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L292)）、`module`（SyncWaterfallHook，[:294](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L294)）、`createParser`/`parser`/`createGenerator`/`generator`（HookMap，[:296-306](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L296-L306)）。构造时还编译 `this.ruleSet = ruleSetCompiler.compile([...defaultRules, ...rules])`（[:313-320](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L313-L320)）与 `parserCache`/`generatorCache`（[:326-328](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L326-L328)）。

`create(data, callback)`（[:869-953](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L869-L953)）：建 `resolveData`（含 `createData:{}`、`cacheable:true`，[:882-895](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L882-L895)）→ `beforeResolve.callAsync`（[:896](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L896)，返回 `false` 则产出 ignored 模块）→ `factorize.callAsync`（[:931](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L931)）→ 包成 `ModuleFactoryResult`（[:941-950](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L941-L950)）。这条链由 §3.3 的 `Compilation.factorizeQueue` 触发（`_factorizeModule` → `factory.create`）。

### 9.2 request 与 resource 如何确定（inline loader / matchResource / rule 匹配）

默认 `resolve` tap（[:419-853](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L419-L853)）是最容易踩坑的地方：

1. **matchResource**：`MATCH_RESOURCE_REGEX = /^([^!]+)!=!/`（[:120](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L120)），命中即把 `xxx!=!` 前缀取出作为“用于 rule 匹配的伪资源”，剩余部分是真正的 `requestWithoutMatchResource`（[:456-478](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L456-L478)）。
2. **inline loader 前缀**（决定禁用哪些自动 loader）：`-!`→`noPreAutoLoaders`（[:486](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L486)），`!`→`noAutoLoaders`（[:487](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L487)），`!!`→`noPrePostAutoLoaders`（[:488](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L488)）。
3. **切分 request**：去前缀后按 `/!+/` split，最后一段 pop 成 `unresolvedResource`（**resource**），其余是 inline loader 元素（[:489-505](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L489-L505)）。
4. **解析 loaders 与 resource**：loader 用 `getResolver("loader")`（[:437](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L437)），inline loaders 经 `resolveRequestArray`（[:749-760](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L749-L760)）解析；resource 走 `defaultResolve` → normal resolver + `resolveResource`（[:765-811](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L765-L811)），得 `resourceData`。scheme（`data:`/`http:` 等）走 `resolveForScheme`/`resolveInScheme`（[:814-848](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L814-L848)），此时不做 inline 解析。
5. **rule 匹配**：`this.ruleSet.exec({ resource, realResource, resourceQuery, issuer, compiler, issuerLayer, ... })`（[:593-610](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L593-L610)）。结果按 effect 类型分桶：`use`→`useLoaders`、`use-post`→`useLoadersPost`、`use-pre`→`useLoadersPre`（[:611-628](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L611-L628)），其余 effect（如 `type`、parser/generator options）经 `cachedCleverMerge` 并入 `settings`（[:629-644](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L629-L644)）。上面三个 no*AutoLoaders 标志在此 gate 掉相应桶。
6. **组装最终 loader 顺序**（`continueCallback`，[:655-713](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L655-L713)）：`allLoaders` = `postLoaders` → （无 matchResource 时）inline `loaders` + `normalLoaders`（[:661-664](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L661-L664)）／（有 matchResource 时）`normalLoaders` + inline `loaders`（[:666-669](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L666-L669)）→ `preLoaders`（[:671-672](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L671-L672)）。数组顺序是 **post → normal/inline → pre**；配合 loader-runner 的“normal 阶段反向执行”（§9.4），**实际 normal 执行顺序变成 pre → normal → inline → post**。`userRequest` 字符串在 [:562-569](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L562-L569) 拼出（含 `matchResource!=!` 前缀）。

> 排查提示：`request`（含 loader 串）、`userRequest`（人类可读、含 matchResource）、`rawRequest`（原始）、`resource`（真正读盘的文件+query+fragment）是四个不同字段，rule/resolve 类 bug 常是把它们搞混。

### 9.3 createModule：构造 `NormalModule`，parser/generator 获取

默认 `factorize` tap（[:340-418](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L340-L418)）在 `resolve` 完成后：`resolve` 返回 `false`→ignored、返回 `Module`→直接用（[:350-361](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L350-L361)）→ `afterResolve.callAsync`（[:363](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L363)）→ `createModule.callAsync(createData, resolveData, ...)`（[:379](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L379)）。若无插件产出模块：先试 `createModuleClass.for(settings.type)`（[:389-394](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L389-L394)），再 `new NormalModule(createData)`（[:397-403](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L397-L403)）。最后 `module` SyncWaterfallHook 可**替换/装饰**模块（返回值即最终模块，[:406-410](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L406-L410)）。

`createData` 字段（request/userRequest/rawRequest/loaders/resource/matchResource/settings/type/parser/parserOptions/generator/generatorOptions/resolveOptions/layer）在 [:684-708](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L684-L708) 用 `Object.assign` 填好。其中 **parser/generator 实例**来自按 type 二级缓存的 `getParser(type, settings.parser)` / `getGenerator(type, settings.generator)`（[:703-706](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L703-L706)）：`getParser`（[:1245-1261](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L1245-L1261)）miss 时 `createParser` 触发 `createParser.for(type)` + `parser.for(type)` hook（[:1268-1280](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L1268-L1280)）；generator 同构（[:1287-1324](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L1287-L1324)）。**同 type + 同 options 的 parser/generator 被复用**——这是 build-time 对象，随工厂（每轮编译新建，§3.2）生灭。

### 9.4 `NormalModule.build`：调 loader-runner、parser，产出 dependencies

`build(options, compilation, resolver, fs, callback)`（[NormalModule.js:1175-1370](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1175-L1370)）：

1. **复位**：`_source=null`、`_ast=null`、`error=null`、`clearDependenciesAndBlocks()`、`buildMeta={}`，`buildInfo` 初始 `cacheable:false`（[:1176-1196](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1176-L1196)）；记 `startTime = compiler.fsStartTime || Date.now()`（[:1198](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1198)，接 §8.3）。
2. **`_doBuild`**（[:916-1087](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L916-L1087)）：建 loaderContext（[:917](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L917)）；**在这里**才 `new LazySet()` 三组依赖并置 `cacheable = true`（[:991-994](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L991-L994)，注意与 `build` 顶部的 `cacheable:false` 不同，真正基线在此）；`hooks.beforeLoaders.call`（[:997](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L997)）；**`runLoaders({ resource, loaders, context, processResource }, cb)`**（[:1013-1023](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1013-L1023)）。`processResource`（[:1023-1041](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1023-L1041)）不走 loader-runner 默认读盘，而是经 `hooks.readResource.for(scheme).callAsync`——这是 webpack 接管资源读取的点。
3. **回填依赖与 cacheable**（runLoaders 完成回调，[:1043-1085](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1043-L1085)）：`fileDependencies.addAll(result.fileDependencies)` 等（[:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)）、每个 loader 路径进 `buildDependencies`（[:1076-1082](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1076-L1082)）、`buildInfo.cacheable = buildInfo.cacheable && result.cacheable`（[:1083](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1083)）。
4. **产出 source/ast**（`processResult`，[:930-987](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L930-L987)）：`hooks.processResult` waterfall 允许插件改写 `[source, sourceMap, ast]`（[:945-948](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L945-L948)）→ `this._source = createSource(...)`（[:973-978](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L973-L978)）、`this._ast = extraInfo.webpackAST ?? null`（[:980-985](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L980-L985)）。
5. **parser 解析**（回到 `build`）：`this.parser.parse(this._ast || source, { source, module: this, compilation, options })`（[:1357-1363](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1357-L1363)）——**副作用填充 `this.dependencies`/`this.blocks`**（即 §7.1 的 Dependency/AsyncDependenciesBlock）。若 `noParse` 命中则跳过 parser 并置 `buildInfo.parsed=false`（[:1346-1351](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1346-L1351)）。
6. **收尾 + 建快照**：`handleParseResult` 排序依赖（[:1229-1241](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1229-L1241)）→ `handleBuildDone` 里 `createSnapshot(startTime, ...)` 存 `buildInfo.snapshot`（[:1315-1329](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1315-L1329)，接 §8.3）。

generator 不在 build 阶段跑：`this.generator.generate(...)` 在 `codeGeneration({...})`（[:1445-1519](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1445-L1519)，调用点 [:1494-1505](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1494-L1505)）里、seal 阶段才执行，产出各 sourceType 的输出 `Source`（§3.5 第 7 步）。**parser 在 build（造图），generator 在 seal（造码）——两者跨越 build-time 内部的“图 vs 码”边界。**

### 9.5 pitch 从左到右、normal 反向执行、pitch 提前返回跳过什么（契约级）

loader-runner（`4.3.0`，本 checkout 未安装 `node_modules`，故以下为**包契约级**说明，非源码行号）：给定 `loaders`（§9.2 组装的 post→normal/inline→pre 数组），

- **pitch 阶段：从左到右**遍历（数组头到尾）依次调用每个 loader 的 `pitch(remainingRequest, precedingRequest, data)`。pitch 从左到右，是为了让靠左（更“外层”、更接近 post 端）的 loader 有机会在读资源前先介入、并把数据通过 `data` 传给对应的 normal 阶段。
- **资源读取**：所有 pitch 都返回 `undefined` 时，在数组**最右端**（resource 侧）用 `processResource`（webpack 版即 `readResource` hook，§9.4）读入 resource 内容作为初始 source。
- **normal 阶段：从右到左**遍历，依次把 source 交给每个 loader 的默认导出函数处理。反向执行，是因为最靠近资源（最右）的 loader 应最先看到原始内容、最外层（最左）的最后收尾——形成“洋葱”式包裹。
- **pitch 提前返回的短路**：若某个 loader 的 `pitch` 返回了**非 `undefined`** 值，loader-runner **立即停止继续向右的 pitch**，**跳过资源读取（不再执行 `processResource`/`readResource`）**，也**跳过该 loader 右侧的所有 loader（包括它们的 pitch 和 normal）**，把这个返回值当作 source，直接从**该 loader（含）向左**进入 normal 阶段。这就是“为什么加了某个 loader 后，右边的 loader 和文件读取都没跑”的根因。

webpack 侧把结果收进 `result.{result,cacheable,fileDependencies,contextDependencies,missingDependencies}`（§9.4 第 3 点），其中 `result[0..2]` = `[source, sourceMap, ast]`。

### 9.6 loaderContext 登记依赖 / cacheable / 返回值 / 抛错的影响

loaderContext 对象在 `_createLoaderContext`（[NormalModule.js:594-844](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L594-L844)）构造。**注意归属**：只有 `addBuildDependency`（[:811-818](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L811-L818)）由 webpack 自身定义；`addDependency`/`addContextDependency`/`addMissingDependency`/`cacheable`/`getDependencies` 等由 **loader-runner 注入到 context**（本 checkout 无源码，故无行号）。webpack 侧只能看到它们在 `getResolveContext`（[:606-625](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L606-L625)，把 resolver 发现的路径回灌）与 `runLoaders` 结果回填（[:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)）处的调用点。各行为对前一轮快照/构建/错误传播的影响：

- **登记 file/context/missing dependency**：经 loader-runner 收集 → build 回调 `addAll` 进 `buildInfo` 三组 LazySet（[:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)）→ `handleBuildDone` 写入本模块 `snapshot`（[:1315-1329](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1315-L1329)）。**下一轮** `needBuild` 用 `checkSnapshotValid` 校验这些路径（§8.3）。missingDependency 记录“当时不存在”，其从无到有会令快照失效（§8.5 表“缺失依赖出现”）。**loader 少登记一个 `addDependency`，就直接导致前一轮快照缺该文件 → 改它不失效 → 问题 A。**
- **声明 cacheable(false)**：loader-runner 把它并入 `result.cacheable`，webpack 在 [:1083](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1083) 做 `&&` 合并。若最终 `buildInfo.cacheable === false`，`handleBuildDone` 不建 snapshot（[:1251-1254](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1251-L1254) 附近的 cacheable/snapshotOptions 判定），且 `needBuild` 因 `!cacheable` 恒重建（[:1552](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1552)）——**每轮必重建（问题 B 的合理来源）**。
- **返回 source/map/AST**：`[source, sourceMap, ast]` 经 `processResult` → `_source`/`_ast`（[:973-985](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L973-L985)）。若返回了 `webpackAST`，parser 直接吃 AST 跳过重新解析（[:1357](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1357) 的 `this._ast || source`）。source 非 Buffer/String 会成 `ModuleBuildError`（[:953-966](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L953-L966)）。
- **抛错**：loader 抛出 → `processResult` 包成 `new ModuleBuildError(err, { from: <loader 路径> })`（[:936-942](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L936-L942)）→ `_doBuild` 回调 → `build` 里 `markModuleAsErrored(err)`（[:1204-1208](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1204-L1208)），它设 `this.error`、`addError`（[:1093-1098](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1093-L1098)）。**关键：build 错误不经 `callback(err)` 冒泡**，而是记在模块上并 `callback()` 无错返回——所以一个 loader 失败不会中断整轮编译，错误随 `Stats` 汇总。parser 抛错则走 `handleParseError` → `ModuleParseError`（[:1214-1227](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1214-L1227)）。`emitWarning`/`emitError`（[:725-744](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L725-L744)）走 `ModuleWarning`/`ModuleError`，非致命。带 `error` 的模块在 `codeGeneration` 里用 `generateError` 或 `throw new Error(...)` 占位（[:1479-1493](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1479-L1493)）。

### 9.7 loader 与 compiler plugin 能观察/改变的边界

用源码调用点区分“谁能看到什么、改什么”：

| 关注点 | loader（在 loaderContext 内） | compiler/compilation plugin（tap hook） | 边界依据 |
| --- | --- | --- | --- |
| rule 匹配 / loader 选择 | 不参与（loader 已被选定才运行） | 可 tap `NormalModuleFactory.hooks.resolve/afterResolve`、改 `ruleSet` 前的 rules | [NMFactory.js:593-644](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L593-L644) |
| 替换/装饰模块实例 | 不能 | `factory.hooks.createModule` / `module`（waterfall） | [NMFactory.js:379-410](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L379-L410) |
| 扩展 loaderContext | 依赖别的插件注入 | `NormalModule.getCompilationHooks(compilation).loader.call(loaderContext, module)` | [NormalModule.js:837-841](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L837-L841) |
| 登记 file/context/missing dep | ✅ `addDependency` 等（loader-runner 注入） | 只能间接（改 loaderContext 或读 `buildInfo`） | [NormalModule.js:606-625](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L606-L625) |
| 改写 loader 返回的 source/map/ast | ✅ 直接返回 | `hooks.processResult`（waterfall） | [NormalModule.js:945-948](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L945-L948) |
| loader 执行前后 | pitch/normal 自身 | `hooks.beforeLoaders`/`beforeParse`/`beforeSnapshot` | [NormalModule.js:997](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L997)、[:1336](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1336)、[:1245](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1245) |
| 观察/改依赖图（dependencies/blocks） | 不能（parser 内部产出） | tap parser hook（§7.1）或 `compilation.hooks.finishModules` 等 | §7.1 / [Compilation.js:2984](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2984) |
| 资源读取本身 | 通过 pitch 短路可跳过 | `hooks.readResource.for(scheme)` | [NormalModule.js:1026-1040](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1026-L1040) |

一句话边界：**loader 活在 `_doBuild` 的 loaderContext 里，只能操作单个资源→source 的转换与依赖登记；compiler/compilation plugin 活在工厂与 `NormalModule.getCompilationHooks` 的 hook 上，能决定选哪些 loader、换掉模块、扩展 loaderContext、改写结果、以及观察构建后的整张图。** 两者在 `runLoaders` 的调用点（[NormalModule.js:1013](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1013)）交接。

---

## 10. 事故复盘推演：多入口 + 共享依赖 + 动态 import + splitChunks + filesystem cache + watch

本节把 §1–§9 的机制串成**一次可用于事故复盘的完整时间线**。每一步显式指出沿用了前文的哪个对象 / 图关系 / 依赖记录 / hook / runtime 产物，以及“为什么复用或失效”。凡是无法从当前版本源码逐行确认的（如 `SplitChunksPlugin` 具体抽取判据、`version` salt 组成、loader-runner 内部），一律以 **未证实** 标注，不润色成事实。

### 10.0 场景配置

```js
// webpack.config.js（web target）
module.exports = {
  mode: "production",
  entry: { entryA: "./src/entryA.js", entryB: "./src/entryB.js" },
  output: { filename: "[name].[contenthash].js", path: __dirname + "/dist" },
  module: {
    rules: [{ test: /\.js$/, use: [{ loader: "./my-loader.js" }] }]
  },
  optimization: {
    splitChunks: { chunks: "all", minChunks: 2 }, // 让 shared 可被抽取
    runtimeChunk: "single"
  },
  cache: { type: "filesystem" },
  watch: true
};
// entryA.js: import {s} from "./shared"; import("./lazy");
// entryB.js: import {s} from "./shared";
// my-loader.js: 读取并 addDependency 一个外部 config（如 ./ext.json）
```

- `entryA`、`entryB` 都静态依赖 `shared`；`entryA` 另有动态 `import("./lazy")`。
- `my-loader` 处理每个 `.js`，并在 loaderContext 上登记一个**外部文件** `ext.json`（`addDependency`）。
- 开了 `splitChunks`、`runtimeChunk:"single"`、`cache:{type:"filesystem"}`、`watch:true`。

装配期（§1–§2，同步）：`createCompiler` 依次跑规范化/默认值、`new Compiler`、用户与内建插件。其中 `WebpackOptionsApply` 装上 `SplitChunksPlugin`（[WebpackOptionsApply.js:502-503](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L502-L503)）、`RuntimeChunkPlugin`（[:506-507](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L506-L507)）、`EntryOptionPlugin`（两个 `EntryPlugin`，§2.1）、`RuntimePlugin`，以及 filesystem cache 的 `IdleFileCachePlugin(new PackFileCacheStrategy(...))`（[:711-714](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L711-L714)）。因 `watch:true`，`webpack()` 走 `compiler.watch(...)`（§1、§5.1），`Watching` 构造末尾 `process.nextTick` 触发首轮 `_invalidate`（[Watching.js:74-76](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L74-L76)）。

### 10.1 冷启动构建 → 首次写出

**T0 开一轮（watch/cache 域交接 build-time）**：`_go` 设 `compiler.fsStartTime`（[Watching.js:121](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L121)）→ `cache.endIdle`（此时 `PackFileCacheStrategy._openPack` 尝试读磁盘 pack；冷启动无 pack 或 `version` 不匹配 → 返回空 Pack，[PackFileCacheStrategy.js:1197-1202](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1197-L1202)）→ `watchRun` → `compiler.compile`（§3.2）。**沿用**：`Compiler`/`Cache`（长命，§6）。**为何不复用**：磁盘无有效 pack，全部模块从零构建。

**T1 make（build-time，造图）**：`EntryPlugin` 的 `make` tap 对 `entryA`/`entryB` 各调 `compilation.addEntry`（§2.1、§3.3）。`make` 是 `AsyncParallelHook`，两入口并行展开。每个模块经 §9 的 `NormalModuleFactory.create`（factorize→resolve→createModule）确定 `resource`/`loaders`，再 `NormalModule.build` 跑 `my-loader`（§9.4）。`my-loader` 通过 loaderContext `addDependency(ext.json)`（loader-runner 注入，**契约级**），回填进 `buildInfo.fileDependencies`（[NormalModule.js:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)），并在 build 末尾进该模块 snapshot（[:1315-1329](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1315-L1329)）。parser 产出 Dependency/Block：`shared` 是 `HarmonyImportSideEffectDependency`（两入口各一条边指向**同一个 Module**），`import("./lazy")` 是含 `ImportDependency` 的 `AsyncDependenciesBlock`（§7.1）。**沿用**：`ModuleGraph`（构造即在，[Compilation.js:1057](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1057)），逐步填充。

**T2 finish（build-time）**：`finishModules`（[Compilation.js:2984](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L2984)）。`ModuleGraph` 完整、`ChunkGraph` 仍 `undefined`（§3.4）。

**T3 seal → chunk 图（build-time，造 chunk）**：`new ChunkGraph(...)`（[Compilation.js:3063](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3063)）。按 entries 造 `entryA`/`entryB` 两入口 chunk 与 `Entrypoint`，`buildChunkGraph`（[:3227](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3227)）铺模块：裸算法下 `shared` 会被 `connectChunkAndModule` 连进**两个** entry chunk（§7.3），`lazy` 经 `iteratorBlock`→`addChunkInGroup`（[:3897](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3897)）得独立 async chunk。

**T4 splitChunks（build-time，优化 chunk）**：`SplitChunksPlugin` tap `compilation.hooks.optimizeChunks`（[optimize/SplitChunksPlugin.js:833](file:///e:/newGsb/questions/GSB-013/Thor/lib/optimize/SplitChunksPlugin.js#L833)），在 seal 的 `optimizeChunks` while 循环（[Compilation.js:3239](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3239)）里运行。因 `shared` 被 ≥2 个 chunk 引用且满足 `minChunks:2`，它被抽进一个独立的 split chunk，从两个 entry chunk 移除。`runtimeChunk:"single"` 由 `RuntimeChunkPlugin` 造一个公共 runtime chunk。**具体抽取判据（minSize/cacheGroups 等）未逐行读，标注 未证实**（§13）；确定的是它挂在 `optimizeChunks` 且改动 `ChunkGraph` 的 module↔chunk 连接。

**T5 code generation + runtime（build-time→output-runtime）**：`module.codeGeneration`（seal 第 7 步，§3.5）里 `ImportDependency.Template` 生成 `__webpack_require__.e(chunkId).then(...)` 并登记 `RuntimeGlobals.ensureChunk`；静态 import 生成 `__webpack_require__(id)` + `require`（§7.4）。`processRuntimeRequirements`（[Compilation.js:3328](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3328)）把 `ensureChunk`→`EnsureChunkRuntimeModule`、`ensureChunkHandlers`→`JsonpChunkLoadingRuntimeModule`（§7.5），经 `addRuntimeModule`（[:3843](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3843)）打进 runtime chunk。`createHash` 用 `[contenthash]` 定文件名。**runtime 产物**：`.e`/`.f.j` 装载器即“写入产物的 runtime”。

**T6 首次写出（output 域）**：`shouldEmit` 不短路 → `Compiler.emitAssets`（§4）→ `hooks.emit` → 逐个 `writeFile` 到 `dist/`。首次全部落盘（`_assetEmittingWrittenFiles` 为空）。

**T7 收尾（watch/cache 域）**：`_done`（§8.2）`storeBuildDependencies(compilation.buildDependencies)` → `cache.beginIdle()`（[Watching.js:326-327](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L326-L327)）→ idle 定时器触发 `PackFileCacheStrategy.afterAllStored`，把 pack + `buildSnapshot`/`resolveBuildDependenciesSnapshot` + `version` 序列化到磁盘（§8.4）→ 按 `compilation.fileDependencies/contextDependencies/missingDependencies` **重挂 watcher**（[:329-340](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L329-L340)）。此时 watcher 监听：所有源文件、`ext.json`（因 my-loader 登记）、`lazy`、`shared` 等。

### 10.2 推演一：入口源码变化（改 `entryA.js`）

**触发**：watcher change 回调 → `_invalidate` → `_mergeWithCollected`（[Watching.js:83-100](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L83-L100)）；非 running 则 `_go`。`_go` 设新 `fsStartTime`、`compiler.modifiedFiles={entryA.js}`（[:155](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L155)），`cache.endIdle`（内存 pack 仍在，`version`/`buildSnapshot` 未变 → **保留**，§8.4）。

**单模块判定（§8.3）**：`needBuild` 对每个模块 `checkSnapshotValid`。`entryA` 的 snapshot 因其文件 timestamp/hash 变（或 `safeTime>fsStartTime`）而**失效** → 重建 + 重解析（重跑 my-loader + parser）。`shared`/`lazy`/`entryB` 的 snapshot 仍有效 → **不重建**（`stillValidModule`），且持久缓存中它们的 build 结果按 etag=null + snapshot 复用（[Compilation.js:1433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1433)）。

**图/chunk/写出**：`entryA` 重解析后若依赖集不变，`ModuleGraph` 出边不变；seal 重新造 `ChunkGraph`（每轮全新，§3.4），splitChunks 重跑。`entryA` 内容变 → 其 chunk 的 `[contenthash]` 变 → codegen etag（`${hash}|...`，[Compilation.js:3639-3640](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3639-L3640)）变 → **重新生成**该 chunk；`shared`/`lazy` 若 hash 未变则 codegen 结果复用。写出阶段 `emitAssets` 的 `compareBeforeEmit` 让**未变文件跳过写盘**，仅变化的 `entryA.[hash].js`（及可能受 hash 传播影响的 runtime chunk）落盘。**结论**：局部重建 + 局部重写，符合预期。

### 10.3 推演二：loader 登记的外部文件变化（改 `ext.json`）

**触发**：`ext.json` 在上一轮被 `my-loader` 的 `addDependency` 登记 → 进了 `compilation.fileDependencies` → §10.1 T7 重挂 watcher 时**被监听**。改它 → 正常触发新一轮。

**为何失效/正确**：`ext.json` 在**引用它的那些模块**（所有 `.js`，因规则 `test:/\.js$/`）的 snapshot 内 → 这些模块 `checkSnapshotValid` 失效 → 全部重建 + 重解析。**沿用**：`buildInfo.fileDependencies`→snapshot 记录（§8.3、§9.6）。**关键因果**：正是 loader 那一句 `addDependency` 让 `ext.json` 既被 watch 又进 snapshot；**若 my-loader 漏了 addDependency**，`ext.json` 既不被监听也不进 snapshot → 改它无任何反应 = 问题 A（stale reuse），根因在 loader 未登记，不在缓存核心（§8.5 表“loader 读了却未登记依赖”行）。

### 10.4 推演三：构建过程中二次 invalid

**场景**：§10.2/§10.3 的某轮正在 running 时，又有文件改动。`_invalidate` 因 `this.running` 为真，**只**合并进 collected 并置 `this.invalid=true`（[Watching.js:431-433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L431-L433)），不立刻开新轮。

**本轮如何收尾**：`onCompiled`/`_done` 中若发现 `this.invalid`，`_done` 的**再失效分支**（[:281-298](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L281-L298)）`storeBuildDependencies` 后直接 `_go()`，**不进 idle**，立即用上一轮合并进 collected 的变更开下一轮。**正确性护栏**：构建途中被改的文件，其 `safeTime > 本轮 fsStartTime`，在**下一轮** `checkSnapshotValid` 必判失效（[FileSystemInfo.js:2829](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2829)）→ 保证不会把“读到一半的旧内容”当成已消化。**沿用**：`compiler.fsStartTime` 作为每轮读盘起点（§8.3 第 3 点）。

### 10.5 推演四：下一次重启命中持久缓存（filesystem cache）

**场景**：进程退出后重启 `webpack --watch`（同一 webpack 版本、同一配置、`node_modules`/loader 未变）。

**命中路径**：`cache.endIdle` → `PackFileCacheStrategy._openPack` 读磁盘 pack。三重校验（§8.4）：① `packContainer.version === version`（[PackFileCacheStrategy.js:1197-1202](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1197-L1202)）；② `checkSnapshotValid(buildSnapshot)`（build 依赖=loader/config/webpack 未变 → 有效，[:1206-1225](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1206-L1225)）；③ resolve 快照有效（[:1228-1268](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1228-L1268)）。三者皆过 → 加载旧 pack。

**复用什么**：`_modulesCache.get(identifier, null)`（[Compilation.js:1433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1433)）取回上次的 `NormalModule`（含 `_source`、`buildInfo.snapshot`）；`needBuild` 再用 snapshot 对当前磁盘 `checkSnapshotValid`——未改的文件有效 → **跳过 build（连 loader 都不跑）**。codegen 缓存按真 etag 复用（[:3639-3640](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3639-L3640)）。**为何仍要 seal**：`ChunkGraph`/`CodeGenerationResults`/`assets` 都是 build-time-only（§7 结尾、§6），不序列化，故每次重启仍重新 seal，但基于复用的模块，速度快。

**何时反而整轮重做（问题 B 的合理来源）**：若升级了 webpack、改了 `webpack.config.js`、或换了 loader 版本 → `version` 不匹配或 `buildSnapshot` 失效 → 整个 pack 作废（§8.4）→ 冷启动式全量重建。这是**预期行为**，不是 bug。`version` salt 的精确组成 **未证实**（§13）。

### 10.6 复盘速查：每步沿用与复用/失效理由

| 时刻 | 域 | 沿用的前文对象/记录 | 复用 or 失效 | 依据 |
| --- | --- | --- | --- | --- |
| 冷启动 make | build-time | `ModuleGraph`（构造即在）、`NormalModuleFactory`（每轮新建） | 全量构建（无 pack） | §3.3/§9、[PackFileCacheStrategy.js:1197](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1197) |
| 冷启动 seal | build-time | `ChunkGraph`（seal 新建）、splitChunks 于 `optimizeChunks` | 首次生成 | [Compilation.js:3063](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3063)/[:3239](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3239) |
| 首次写出 | output | `compilation.assets`、`emitAssets` | 全部落盘 | §4 |
| 改 entryA | build-time | `entryA.buildInfo.snapshot` | 失效重建；其余复用 | [NormalModule.js:1576](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1576) |
| 改 ext.json | build-time | 各 `.js` 模块 snapshot 内的 fileDependency | 失效重建（因 loader 登记过） | §9.6、[NormalModule.js:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075) |
| 构建中二次 invalid | watch/cache | `this.invalid`、`collected`、`fsStartTime` | 本轮完成后再跑一轮 | [Watching.js:431-433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L431-L433)/[:281-298](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L281-L298) |
| 重启命中缓存 | watch/cache→build-time | 磁盘 pack、`buildSnapshot`、模块 snapshot | 命中则跳过 build，仍重 seal | §8.4、[Compilation.js:1433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1433) |

---

## 11. 全文一致性审校（顺序与所有权）

对用户点名的五条链路做交叉核对，确认全文无自相矛盾；结论均可回到源码。

1. **public API → `Compiler`**（§1）：`webpack()` → `createCompiler` 顺序固定为 规范化 → baseDefaults → `new Compiler` → NodeEnvironmentPlugin → 用户插件 → `applyWebpackOptionsDefaults` → `WebpackOptionsApply.process` → `initialize`，全同步（[webpack.js:65-97](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L65-L97)）。**所有权**：`Compiler` 由调用方持有，长命。全文一致。
2. **`Compiler` → `Compilation`**（§3.2–§3.3）：`compile` 每次 `new Compilation`（[Compiler.js:1317](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1317)），`Compilation` 短命。§5.1/§8/§10 反复强调“每轮全新 Compilation”，与 §3.2 一致，无矛盾。
3. **异步块 → chunk/runtime**（§7）：`AsyncDependenciesBlock`（build-time，挂 `module.blocks`）→ `buildChunkGraph` 造 async chunk → `ImportDependency.Template` 生成 `.e(chunkId)` → `RuntimeModule` 落地。§7.6 的“只存在于构建期 / 翻译进 runtime / 跨编译存活”三分法与 §6 生命周期表、§10.1 T5 一致。
4. **watch/cache**（§5、§8、§10）：`Watching` 长命、每轮触发 `compiler.compile`；`Cache` hook 驱动、idle 落盘。§5.3 的 run-idle 握手（非 watch）与 §8.1/§8.2 的 watch 握手是**两条并列路径**（一个走 `Compiler.run`，一个走 `Watching._go`），均设/清 `fsStartTime`、均 `beginIdle/endIdle`——已在 §10.1/§10.4 明确区分，不矛盾。
5. **loader/plugin 边界**（§9.7）：loader 活在 `_doBuild` 的 loaderContext；plugin 活在工厂与 compilation hook；交接点 `runLoaders`（[NormalModule.js:1013](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1013)）。§10.2/§10.3 对 my-loader `addDependency` 的因果描述与 §9.6 一致。

**已消除/澄清的潜在冲突点**：
- “`factorizeQueue`”一词：§3.3 指 **`Compilation` 侧**的 `AsyncQueue`；§9.1 明确 **`NormalModuleFactory` 本身不排队**（无 AsyncQueue，靠 hook callAsync）。两处指的是不同对象，§9 版本注记已点明，避免误读。
- “`cacheable`”基线：§9.4 已澄清 `build` 顶部 `buildInfo.cacheable:false` 只是占位，真正基线 `true` 在 `_doBuild`（[NormalModule.js:991-994](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L991-L994)），与 §8.3 `needBuild` 的 `!cacheable` 判定不冲突。
- 章节顺序：本轮已把物理顺序整理为 §0→§9 单调递增（§7 场景、§8 watch 失效、§9 build 前），§13 为未证实清单；所有 `§x.y` 交叉引用已同步更新。

---

## 12. 关键结论证据索引（按文件 / symbol / hook）

下表为复盘时可直接跳转的证据锚点。**已证实**=已在当前 checkout 逐行读到；**未证实**=见 §13。

| 结论 | 文件:行 / symbol / hook | 状态 |
| --- | --- | --- |
| `webpack()` 装配顺序（含默认值必须晚于用户插件） | [webpack.js:65-97](file:///e:/newGsb/questions/GSB-013/Thor/lib/webpack.js#L65-L97) `createCompiler` | 已证实 |
| `Compiler` 构造持有 `resolverFactory`/`cache` | [Compiler.js:266](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L266)/[:287](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L287) | 已证实 |
| `make` 是 AsyncParallelHook（多入口并行） | [Compiler.js:180](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L180) `hooks.make` | 已证实 |
| entry → `EntryPlugin` tap `make`→`addEntry` | [EntryPlugin.js:47-51](file:///e:/newGsb/questions/GSB-013/Thor/lib/EntryPlugin.js#L47-L51) | 已证实 |
| `compile` 每轮 `new Compilation` | [Compiler.js:1317](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L1317) `newCompilation` | 已证实 |
| `ModuleGraph` 构造即在 | [Compilation.js:1057](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1057) | 已证实 |
| `ChunkGraph` 仅 seal 创建（此前 undefined） | [Compilation.js:1059](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L1059)/[:3063](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3063) | 已证实 |
| `buildChunkGraph` 铺 module→chunk | [Compilation.js:3227](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3227)；[buildChunkGraph.js:1301](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L1301) | 已证实 |
| 共享模块可属多 chunk（SortableSet） | [ChunkGraph.js:201-202](file:///e:/newGsb/questions/GSB-013/Thor/lib/ChunkGraph.js#L201-L202)/[:343-348](file:///e:/newGsb/questions/GSB-013/Thor/lib/ChunkGraph.js#L343-L348) | 已证实 |
| async chunk available-modules 去重 | [buildChunkGraph.js:707-711](file:///e:/newGsb/questions/GSB-013/Thor/lib/buildChunkGraph.js#L707-L711) | 已证实 |
| splitChunks 挂 `optimizeChunks` | [optimize/SplitChunksPlugin.js:833](file:///e:/newGsb/questions/GSB-013/Thor/lib/optimize/SplitChunksPlugin.js#L833)；wiring [WebpackOptionsApply.js:502-503](file:///e:/newGsb/questions/GSB-013/Thor/lib/WebpackOptionsApply.js#L502-L503) | 已证实（tap 点）；抽取判据 未证实 |
| 动态 import codegen `.e().then()` | [ImportDependency.js:116](file:///e:/newGsb/questions/GSB-013/Thor/lib/dependencies/ImportDependency.js#L116)；[RuntimeTemplate.js:1009](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimeTemplate.js#L1009) | 已证实 |
| runtimeRequirements 扇出 → RuntimeModule | [Compilation.js:3328](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3328)/[:3843](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3843)；[RuntimePlugin.js:371-380](file:///e:/newGsb/questions/GSB-013/Thor/lib/RuntimePlugin.js#L371-L380) | 已证实 |
| 写盘与跳过（compareBeforeEmit） | [Compiler.js:676-1021](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compiler.js#L676-L1021) `emitAssets` | 已证实 |
| Watching 首轮 nextTick 自失效 | [Watching.js:74-76](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L74-L76) | 已证实 |
| 变更合并 / running 延后 / 再失效分支 | [Watching.js:83-100](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L83-L100)/[:431-433](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L431-L433)/[:281-298](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L281-L298) | 已证实 |
| 重挂 watcher 用 compilation 三组依赖 | [Watching.js:329-340](file:///e:/newGsb/questions/GSB-013/Thor/lib/Watching.js#L329-L340) | 已证实 |
| snapshot 时间护栏 `safeTime>startTime` | [FileSystemInfo.js:2829](file:///e:/newGsb/questions/GSB-013/Thor/lib/FileSystemInfo.js#L2829) | 已证实 |
| `needBuild` 决策阶梯 | [NormalModule.js:1540-1591](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1540-L1591) | 已证实 |
| loader 依赖回填进 snapshot | [NormalModule.js:1073-1075](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1073-L1075)/[:1315-1329](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1315-L1329) | 已证实 |
| 持久缓存三重校验（version/build/resolve） | [PackFileCacheStrategy.js:1197-1268](file:///e:/newGsb/questions/GSB-013/Thor/lib/cache/PackFileCacheStrategy.js#L1197-L1268) | 已证实（校验逻辑）；version 组成 未证实 |
| codegen 缓存 etag=`hash\|depTplHash` | [Compilation.js:3639-3640](file:///e:/newGsb/questions/GSB-013/Thor/lib/Compilation.js#L3639-L3640) | 已证实 |
| NMF factorize/resolve/createModule/module hook | [NormalModuleFactory.js:286-294](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L286-L294)/[:869-953](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L869-L953) | 已证实 |
| loader 顺序组装 post→normal/inline→pre | [NormalModuleFactory.js:655-713](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModuleFactory.js#L655-L713) | 已证实 |
| `runLoaders` 调用点 / build 错误不冒泡 | [NormalModule.js:1013](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1013)/[:1204-1208](file:///e:/newGsb/questions/GSB-013/Thor/lib/NormalModule.js#L1204-L1208) | 已证实 |
| pitch 左→右 / normal 右→左 / 短路 | loader-runner `4.3.0`（未装 node_modules） | 未证实（契约级） |

---

## 13. 待核实 / 未证实清单（后续深入方向）

以下为**未直接读源确认**的点，标为 **未证实**，避免误导：

- **未证实**：`SplitChunksPlugin` 在 `optimizeChunks` 阶段抽取共享模块（把 §7.3 中被复制的 `shared` 提成独立 chunk）的具体判据（`minChunks`/`minSize`/`cacheGroups`）尚未逐行读。
- **未证实**：`runtimeChunk` 选项如何决定 runtime code 落在独立 runtime chunk 还是并入 entry chunk（§7.5 里 `addRuntimeModule` 的目标 chunk 选择）。
- **未证实**：`PackFileCacheStrategy` 的 `version`/salt 精确由哪些输入（webpack 版本、config 指纹、`cache.version` 等）在**本文件上游**如何拼成，尚未逐行确认。
- **未证实（无法给行号）**：`loader-runner` 包（`package.json` 声明 `^4.2.0`，[yarn.lock](file:///e:/newGsb/questions/GSB-013/Thor/yarn.lock) 锁定 `4.3.0`）在**本 checkout 未安装 `node_modules`**，其 `runLoaders` / `iteratePitchingLoaders` / `iterateNormalLoaders` 以及 `loaderContext` 上注入的 `addDependency`/`addContextDependency`/`addMissingDependency`/`cacheable` 等方法的具体实现无法给出行号；§9 对 pitch/normal 语义的描述基于该包的公开契约与 webpack 侧调用点，标注为“契约级”而非源码级。
- **未证实**：`MultiCompiler`（[lib/MultiCompiler.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/MultiCompiler.js)）如何按 `setDependencies` 调度子 compiler 的先后与并行。
- **未证实**：`JavascriptParser`（[lib/javascript/JavascriptParser.js](file:///e:/newGsb/questions/GSB-013/Thor/lib/javascript/JavascriptParser.js)）内部 AST 遍历与各 `hooks.*` 触发次序的完整细节（§7.1 只覆盖了 import 相关 hook）。

> 需要时可用仓库现有 Jest（`yarn jest <file>`）与 TypeScript（`yarn type-check` 相关脚本，见 [package.json](file:///e:/newGsb/questions/GSB-013/Thor/package.json)）在不改生产代码的前提下核对上述行为。
