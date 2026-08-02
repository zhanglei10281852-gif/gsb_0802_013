# Webpack 构建基础设施理解（第一版）

> 基于仓库 `webpack@5.99.9` 源码阅读。本文档面向接手维护的团队，按执行域组织，
> 所有判断均可回到具体源码路径与 symbol。无法从当前版本确认的内容标注为「未证实」。

---

## 0. 三个执行域

整套基础设施在运行期可分成三个彼此独立但有数据交接的执行域，后文沿用这些名称：

1. **构建期 compiler 域**：以 [Compiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js) 为根，负责读取配置、规范化默认值、创建工厂、调度一次或多次 `Compilation`。该域是同步创建、长期持有的对象，跨越多次构建。
2. **写入产物的 runtime 域**：指 `Compilation.seal` 之后到 `Compiler.emitAssets` 之间，在 Node 进程内把 `Source` 渲染为文件并写到 `outputFileSystem` 的阶段。这里的「runtime」不是浏览器里的 webpack runtime，而是构建主机上的产物写出逻辑，入口在 [Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L676-L1021)。
3. **watch/cache 域**：由 [Watching](file:///e:/newGsb/questions/GSB-013/Steve/lib/Watching.js) 与 [Cache](file:///e:/newGsb/questions/GSB-013/Steve/lib/Cache.js) 共同管理，负责文件监听、增量失效、空闲时落盘缓存。该域在 `compiler.watch()` 时才激活，非 watch 构建不进入。

三个域共享同一个 `Compiler` 实例与 `options`，但每次 `compile()` 产生的 `Compilation`、`ModuleGraph`、`ChunkGraph` 只在本次构建内有效。

---

## 1. 对外 `webpack()` 入口与 Compiler 创建

### 1.1 入口与 schema 校验

对外导出在 [lib/index.js](file:///e:/newGsb/questions/GSB-013/Steve/lib/index.js#L128-L132)，主函数通过 `lazyFunction(() => require("./webpack"))` 延迟加载，真正实现在 [lib/webpack.js](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js)。

调用 `webpack(options, callback?)` 时：

- 先用预编译 schema `../schemas/WebpackOptions.check.js` 做快速校验（[webpack.js#L129](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L129)），失败再回退到完整的 [validateSchema](file:///e:/newGsb/questions/GSB-013/Steve/lib/validateSchema.js)。
- `options` 为数组时走 [createMultiCompiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L44-L58)，每个元素调用一次 `createCompiler`，再用 [MultiCompiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/MultiCompiler.js) 包装并按 `dependencies` 设置编译顺序。
- 单配置走 [createCompiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L65-L97)。

`callback` 存在时，`webpack()` 内部会自动决定 `compiler.run()` 或 `compiler.watch()`，并在 run 完成后调用 `compiler.close()`（[webpack.js#L166-L174](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L166-L174)）。无 callback 时只返回 compiler，由调用方自行 `run`/`watch`。

### 1.2 配置规范化与默认值（两阶段）

`createCompiler` 内部分两次处理 options，顺序很关键：

**第一阶段：规范化 + 基础默认值（在 new Compiler 之前）**

1. [getNormalizedWebpackOptions(rawOptions)](file:///e:/newGsb/questions/GSB-013/Steve/lib/config/normalization.js#L127) 把用户配置转成内部统一形态：
   - `entry` 统一为 `{ [name]: { import: string[], ... } }`，字符串/数组简写为 `main`（[normalization.js#L480-L536](file:///e:/newGsb/questions/GSB-013/Steve/lib/config/normalization.js#L480-L536)）。
   - `cache: true` 转为 `{ type: "memory" }`，`cache: false` 保持 `false`，`cache: "filesystem"` 展开完整字段。
   - `module.parser`、`module.generator`、`module.rules`、`output.library`、`optimization.runtimeChunk`、`optimization.splitChunks`、`stats`、`snapshot` 等都被转为固定形状的对象。
   - 函数形式的 `entry` 被包成 `() => Promise.resolve().then(fn).then(getNormalizedEntryStatic)`。
2. [applyWebpackOptionsBaseDefaults(options)](file:///e:/newGsb/questions/GSB-013/Steve/lib/config/defaults.js#L162-L165) 只做两件事：`context` 默认为 `process.cwd()`，以及基础设施日志默认值。这一步刻意保持最小，因为 `context` 是 `new Compiler` 的必需参数。

**第二阶段：完整默认值（在用户插件 apply 之后）**

3. `new Compiler(context, options)` 创建实例（[webpack.js#L68-L71](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L68-L71)）。
4. [NodeEnvironmentPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/node/NodeEnvironmentPlugin.js) 立即 `apply(compiler)`，注入：
   - `inputFileSystem = new CachedInputFileSystem(gracefulFs, 60000)`
   - `outputFileSystem = gracefulFs`
   - `intermediateFileSystem = gracefulFs`
   - `watchFileSystem = new NodeWatchFileSystem(inputFileSystem)`
   - `infrastructureLogger`
   - 在 `beforeRun` 钩子里 `purge()` 输入缓存并记录 `fsStartTime`（[NodeEnvironmentPlugin.js#L60-L68](file:///e:/newGsb/questions/GSB-013/Steve/lib/node/NodeEnvironmentPlugin.js#L60-L68)）。
5. **用户插件在此处 apply**：遍历 `options.plugins`，函数插件以 `plugin.call(compiler, compiler)` 调用，对象插件调用 `plugin.apply(compiler)`（[webpack.js#L75-L84](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L75-L84)）。这意味着用户插件能在完整默认值计算之前 tap 任何 compiler hook，但此时 `options.output.path` 等字段可能仍为 `undefined`。
6. [applyWebpackOptionsDefaults(options, compilerIndex)](file:///e:/newGsb/questions/GSB-013/Steve/lib/config/defaults.js#L172) 计算其余所有默认值：`target`、`mode`（影响 devtool/cache/optimization 一大批默认）、`output`、`module`、`resolve`、`optimization`、`experiments`、`cache`、`snapshot`、`performance` 等。该函数返回 `{ platform }`，若存在则赋值给 `compiler.platform`。
7. 依次触发 `compiler.hooks.environment.call()`、`afterEnvironment.call()`（[webpack.js#L92-L93](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L92-L93)）。这两个是同步 hook，`NodeEnvironmentPlugin` 已在步骤 4 完成文件系统注入，因此插件在此可读取文件系统。
8. [new WebpackOptionsApply().process(options, compiler)](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L94) 根据最终 options 挂载一大批内置插件（见下节）。
9. 最后触发 `compiler.hooks.initialize.call()`（[webpack.js#L95](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L95)），此时 compiler 完全就绪。

> **要点**：用户插件在 `applyWebpackOptionsDefaults` 之前执行，而内置插件在其后执行。因此用户插件若想覆盖内置插件的 tap，需注意 tap 顺序——同一 hook 上后 tap 的后执行（同步 hook）或按注册顺序执行。

### 1.3 WebpackOptionsApply 挂载了什么

[WebpackOptionsApply.process](file:///e:/newGsb/questions/GSB-013/Steve/lib/WebpackOptionsApply.js#L78-L794) 是把「配置项」翻译成「插件 + hook tap」的总装配厂，关键动作包括：

- 设置 `compiler.outputPath`、`recordsInputPath`、`recordsOutputPath`、`name`。
- 按 `externals`/`externalsPresets` 挂载 [ExternalsPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/ExternalsPlugin.js) 及各 target 插件。
- 按 `output.chunkFormat` 挂载 array-push/commonjs/module 三种 chunk 格式插件。
- 按 `output.enabledChunkLoadingTypes`/`enabledWasmLoadingTypes`/`enabledLibraryTypes` 循环挂载对应 Enable 插件。
- devtool 为 source-map 系列时挂载 [SourceMapDevToolPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/SourceMapDevToolPlugin.js) 或 [EvalSourceMapDevToolPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/EvalSourceMapDevToolPlugin.js)；含 `eval` 时挂载 [EvalDevToolModulePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/EvalDevToolModulePlugin.js)。
- 无条件挂载 [JavascriptModulesPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/javascript/JavascriptModulesPlugin.js)、[JsonModulesPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/json/JsonModulesPlugin.js)、[AssetModulesPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/asset/AssetModulesPlugin.js)。
- `experiments.css` 挂载 [CssModulesPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/css/CssModulesPlugin.js)。
- **入口处理**：`new EntryOptionPlugin().apply(compiler)` 后立即 `compiler.hooks.entryOption.call(context, options.entry)`（[WebpackOptionsApply.js#L391-L396](file:///e:/newGsb/questions/GSB-013/Steve/lib/WebpackOptionsApply.js#L391-L396)）。`entryOption` 是 `SyncBailHook`，[EntryOptionPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/EntryOptionPlugin.js#L20-L25) tap 它并返回 `true` 短路，随后为每个 entry 的每个 `import` 创建一个 [EntryPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/EntryPlugin.js) 并 apply。每个 `EntryPlugin` 做两件事：在 `compilation` hook 上把 `EntryDependency` 映射到 `normalModuleFactory`；在 `make` hook 上 `tapAsync`，调用 `compilation.addEntry(...)`。
- 挂载 [RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimePlugin.js)（负责把 `RuntimeGlobals` 需求翻译成 `RuntimeModule`）。
- 挂载 CommonJs / Harmony / AMD / RequireContext / Import / ImportMeta / URL / Worker / System / Loader 等依赖插件。
- 按 `optimization.*` 标志挂载 SplitChunksPlugin、RuntimeChunkPlugin、ModuleConcatenationPlugin、SideEffectsFlagPlugin、FlagDependencyExportsPlugin、FlagDependencyUsagePlugin、InnerGraphPlugin、MangleExportsPlugin、NoEmitOnErrorsPlugin、RealContentHashPlugin、各种 id 插件、minimizer 等。
- 按 `cache.type` 挂载 MemoryCachePlugin / MemoryWithGcCachePlugin / IdleFileCachePlugin + PackFileCacheStrategy。
- 最后在 `compiler.resolverFactory.hooks.resolveOptions` 上为 `normal`/`context`/`loader` 三种 resolver 合并 `options.resolve`/`options.resolveLoader`，并触发 `afterPlugins`、`afterResolvers`。

### 1.4 Compiler 对象的持有关系

`new Compiler(context, options)`（[Compiler.js#L142-L325](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L142-L325)）构造时即确定：

- `this.hooks`：全部 hook 在构造函数里冻结创建，贯穿 compiler 整个生命周期。
- `this.resolverFactory = new ResolverFactory()`。
- `this.cache = new Cache()`。
- `this.options`、`this.context`、`this.requestShortener`。
- `this.running = false`、`this.watchMode = false`、`this.idle = false`。
- 文件系统字段初始为 `null`，由 `NodeEnvironmentPlugin` 填充。
- `_lastCompilation`/`_lastNormalModuleFactory` 初始为 `undefined`，用于跨构建缓存与清理。

Compiler 在 `webpack()` 返回后长期存在，直到调用 `close()`。MultiCompiler 持有子 compiler 数组。

---

## 2. 一次非 watch 构建的主链路

非 watch 路径为 `compiler.run(callback)` → `compiler.compile(onCompiled)` → `onCompiled` 里 `emitAssets` → `emitRecords` → `done`。整体在 [Compiler.run](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L474-L609)。

### 2.1 run 阶段

[Compiler.run](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L474) 的顺序：

1. 防重入：`this.running` 为 true 时回调 `ConcurrentCompilationError`。
2. 定义 `finalCallback`：设置 `idle=true`、`cache.beginIdle()`、`running=false`，触发 `failed`（有错时）或 `afterDone`。
3. 若 `this.idle`（上次 watch/run 结束后进入空闲），先 `cache.endIdle` 再 `run()`；否则直接 `run()`。
4. `run()` 内依次：
   - `hooks.beforeRun.callAsync(this)` — `NodeEnvironmentPlugin` 在此 purge 输入文件系统。
   - `hooks.run.callAsync(this)`
   - `this.readRecords(callback)` — 若配置了 `recordsInputPath`，读取并 JSON.parse 到 `this.records`。
   - `this.compile(onCompiled)`

`onCompiled(err, compilation)` 是编译完成后的回调：

- 先查 `hooks.shouldEmit.call(compilation)`，这是 `SyncBailHook`，返回 `false` 则跳过写产物，直接构造 `Stats` 并走 `done`。
- 否则 `process.nextTick` 后 `this.emitAssets(compilation, ...)`。
- `emitAssets` 成功后查 `compilation.hooks.needAdditionalPass.call()`（`SyncBailHook`）。若为真，设置 `compilation.needAdditionalPass = true`，触发 `done` → `additionalPass` → 再次 `this.compile(onCompiled)`，形成多轮编译。这是 `DllPlugin` 等需要二次编译的机制。
- 正常路径：`emitRecords` → 构造 `Stats` → `hooks.done.callAsync` → `cache.storeBuildDependencies` → `finalCallback(null, stats)`。

### 2.2 compile 阶段：工厂、Compilation、make

[Compiler.compile](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1310-L1355) 是单次编译的调度核心：

1. `const params = this.newCompilationParams()`：
   - 调用 [createNormalModuleFactory](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1277-L1290)，内部 `new NormalModuleFactory({ context, fs: inputFileSystem, resolverFactory, options: options.module, associatedObjectForCache: this.root, layers })`，保存到 `_lastNormalModuleFactory`，触发 `hooks.normalModuleFactory.call(nmf)`。
   - 调用 [createContextModuleFactory](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1292-L1296)，`new ContextModuleFactory(resolverFactory)`，触发 `hooks.contextModuleFactory.call(cmf)`。
2. `hooks.beforeCompile.callAsync(params)` — 异步，插件可在此修改 params。
3. `hooks.compile.call(params)` — 同步。
4. `const compilation = this.newCompilation(params)`：
   - [createCompilation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1259-L1262) 先 `_cleanupLastCompilation()`（清理上一轮 `_lastCompilation` 上模块的 ChunkGraph/ModuleGraph 弱引用并调用 `module.cleanupForCache()`），再 `new Compilation(this, params)`，存为 `_lastCompilation`。
   - [newCompilation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1268-L1275) 设置 `compilation.name`、`compilation.records = this.records`，然后依次触发 `this.hooks.thisCompilation.call(compilation, params)` 和 `this.hooks.compilation.call(compilation, params)`。**这两个 hook 是所有插件注册 Compilation 级 tap 的入口**，例如 JavascriptModulesPlugin、RuntimePlugin、EntryPlugin 都在此把 dependency factory、renderManifest、runtimeRequirement 等 tap 到新 compilation 上。`thisCompilation` 对子编译也触发，`compilation` 只在非子编译触发（未证实：当前代码未显式区分，需核对 MultiCompiler/child compiler 路径）。
5. `hooks.make.callAsync(compilation)` — `AsyncParallelHook`。这是图构建的真正入口。EntryPlugin 在 `make` 上 `tapAsync`，调用 `compilation.addEntry(context, dep, options, cb)`。多个 entry 对应的 tap 并行执行。
6. make 完成后 `hooks.finishMake.callAsync(compilation)`。
7. `process.nextTick` 后 `compilation.finish(callback)`。
8. finish 完成后 `compilation.seal(callback)`。
9. seal 完成后 `hooks.afterCompile.callAsync(compilation)`，最后 `callback(null, compilation)`。

### 2.3 make 阶段：模块图构建

`make` hook 触发后，[EntryPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/EntryPlugin.js#L47-L51) 调用 [Compilation.addEntry](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2320-L2328)。

`addEntry` → [_addEntryItem](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2355-L2427)：

1. 在 `this.entries`（`Map<name, EntryData>`）中登记或复用 entry 数据，把 `EntryDependency` 推入 `dependencies` 数组。
2. 触发 `hooks.addEntry.call(entry, options)`。
3. 调用 [addModuleTree](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2269-L2311)：
   - 根据 `dependency.constructor` 从 `this.dependencyFactories` 取出对应的 `ModuleFactory`（EntryDependency → NormalModuleFactory，由 EntryPlugin 在 `compilation` hook 里注册）。
   - 调用 [handleModuleCreation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1935-L2072)。

[handleModuleCreation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1935) 是模块创建与递归依赖处理的核心递归入口，步骤：

1. `this.factorizeModule(...)`（实际入 `factorizeQueue`）→ `factory.create({ contextInfo, resolveOptions, context, dependencies }, cb)`。
   - 对 NormalModuleFactory，这一步完成 resolver 解析、loader 处理、创建 `NormalModule` 实例。
2. factory 回调里拿到 `ModuleFactoryResult`，把 `fileDependencies`/`contextDependencies`/`missingDependencies` 合并到 compilation。
3. `this.addModule(newModule, cb)`（入 `addModuleQueue`）→ [_addModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1419-L1455)：按 `module.identifier()` 去重，先查 `_modulesCache`，命中则 `cacheModule.updateCacheModule(module)` 并复用；否则放入 `this._modules` Map 与 `this.modules` Set，并通过 `ModuleGraph.setModuleGraphForModule(module, this.moduleGraph)` 关联。
4. 对每个 dependency 调用 `moduleGraph.setResolvedModule(originModule, dependency, module)`，建立 Dependency → Module 的映射；`moduleGraph.setIssuerIfUnset(module, originModule)`。
5. 调用 [_handleModuleBuildAndDependencies](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2083-L2163)：
   - 构建期循环检测：若 `checkCycle` 且 originModule 正在 build，用 `creatingModuleDuringBuild` WeakMap 追踪，发现环抛 `BuildCycleError`。
   - `this.buildModule(module)`（入 `buildQueue`）→ [_buildModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1492-L1554)：
     - `module.needBuild({ compilation, fileSystemInfo, valueCacheVersions }, cb)` 判断是否需要构建（缓存/snapshot 失效判断）。
     - 不需要则触发 `hooks.stillValidModule.call(module)` 直接返回。
     - 需要则 `hooks.buildModule.call(module)`，调用 `module.build(options, compilation, resolver, fs, cb)`。
     - 对 [NormalModule.build](file:///e:/newGsb/questions/GSB-013/Steve/lib/NormalModule.js#L1175)：重置 `buildMeta`/`buildInfo`/dependencies/blocks，调用 `_doBuild`（运行 loader 链、读源码），然后 parser 解析源码生成 dependencies 与 blocks，排序后 `_initBuildHash`，最后 snapshot。
     - 成功后 `_modulesCache.store(identifier, null, module)`，触发 `hooks.succeedModule.call(module)`。
   - build 成功后 `this.processModuleDependencies(module, cb)`（入 `processDependenciesQueue`）→ [_processModuleDependencies](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1593)：
     - 遍历 module 的 `dependencies` 与 `blocks`（AsyncDependenciesBlock），对每个 dependency 调用 `moduleGraph.setParents(dep, block, module, index)`。
     - 按 factory 类型与 resource identifier 分组，为每组再次调用 `handleModuleCreation`，递归构建子模块。
     - AsyncDependenciesBlock 对应的子模块通过 chunk group 在 seal 阶段分配到异步 chunk。

整个模块图构建由四个 [AsyncQueue](file:///e:/newGsb/questions/GSB-013/Steve/lib/util/AsyncQueue.js) 串成有父子关系的并行队列（构造见 [Compilation.js#L1063-L1093](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1063-L1093)）：

```
processDependenciesQueue (parallelism = options.parallelism || 100)
  └─ addModuleQueue (parent: processDependencies)
       └─ factorizeQueue (parent: addModule)
            └─ buildQueue (parent: factorize)
```

`rebuildQueue` 独立存在，供 watch 增量重建使用。队列父子关系保证子队列任务完成前父队列不会完成，从而 `make` hook 的 `AsyncParallelHook` 在所有模块（含传递依赖）构建完毕后才 resolve。

### 2.4 finish 阶段

[Compilation.finish](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2782-L3029)：

1. 清空 `factorizeQueue`。
2. 若 `profile`，计算各模块的并行度因子并输出诊断。
3. `_computeAffectedModules(this.modules)`（cache unaffected 相关）。
4. `hooks.finishModules.callAsync(modules, cb)` — 异步，插件可在此做全模块分析。
5. `moduleGraph.freeze("dependency errors")` 后遍历所有模块，报告 dependency 级别的 errors/warnings，以及模块自身的 errors/warnings；随后 `moduleGraph.unfreeze()`。

### 2.5 seal 阶段：chunk 图、优化、代码生成、产物

[Compilation.seal](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3050-L3428) 是最长的同步起点 + 异步瀑布。关键顺序：

**ChunkGraph 创建与 chunk 初始划分**

1. `const chunkGraph = new ChunkGraph(this.moduleGraph, this.outputOptions.hashFunction)`，赋值给 `this.chunkGraph`（[Compilation.js#L3063-L3067](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3063-L3067)）。若开启 backCompat，对所有已有模块调用 `ChunkGraph.setChunkGraphForModule`。
   - **ChunkGraph 此时才创建**，因为它依赖模块图完成。构造函数见 [ChunkGraph.js#L250-L277](file:///e:/newGsb/questions/GSB-013/Steve/lib/ChunkGraph.js#L250-L277)，内部用 `WeakMap<Module, ChunkGraphModule>` 与 `WeakMap<Chunk, ChunkGraphChunk>` 存储。
2. `hooks.seal.call()`。
3. `while (hooks.optimizeDependencies.call(this.modules)) {}` — `SyncBailHook`，返回真值则重复调用（插件可多轮优化），然后 `afterOptimizeDependencies`。
4. `hooks.beforeChunks.call()` → `moduleGraph.freeze("seal")`。
5. 遍历 `this.entries`，为每个 entry name：
   - `addChunk(name)` 创建初始 chunk。
   - `new Entrypoint(options)`，连接 chunk group 与 chunk。
   - 通过 `moduleGraph.getModule(dep)` 找到 entry dependency 对应的模块，`chunkGraph.connectChunkAndEntryModule(chunk, module, entrypoint)`。
   - `assignDepths(entryModules)` 计算模块深度。
6. 处理 `dependOn`/`runtime` 选项的 entry 间关系。
7. [buildChunkGraph(this, chunkGraphInit)](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js) 执行模块到 chunk 的分配算法，遍历模块依赖与 async blocks，创建异步 chunk groups，填充 `chunkGraph._modules`/`_chunks`。
8. `hooks.afterChunks.call(this.chunks)`。

**优化瀑布**

9. `hooks.optimize.call()`。
10. `while (hooks.optimizeModules.call(modules)) {}` → `afterOptimizeModules`。
11. `while (hooks.optimizeChunks.call(chunks, chunkGroups)) {}` → `afterOptimizeChunks`。
12. `hooks.optimizeTree.callAsync(chunks, modules, cb)` → `afterOptimizeTree`。
13. `hooks.optimizeChunkModules.callAsync(chunks, modules, cb)` → `afterOptimizeChunkModules`。
14. `hooks.shouldRecord.call()` 决定是否记录 records。
15. module id 阶段：`reviveModules` → `beforeModuleIds` → `moduleIds` → `optimizeModuleIds` → `afterOptimizeModuleIds`。
16. chunk id 阶段：`reviveChunks` → `beforeChunkIds` → `chunkIds` → `optimizeChunkIds` → `afterOptimizeChunkIds`。
17. `assignRuntimeIds()`。
18. `_computeAffectedModulesWithChunkGraph()`、`sortItemsWithChunkIds()`。
19. 若 shouldRecord：`recordModules`、`recordChunks`。
20. `hooks.optimizeCodeGeneration.call(modules)`。

**Hash 与代码生成**

21. `hooks.beforeModuleHash` → `createModuleHashes()` → `afterModuleHash`。
22. `hooks.beforeCodeGeneration.call()` → [codeGeneration(callback)](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3468-L3500)：
    - 创建 `new CodeGenerationResults(outputOptions.hashFunction)` 赋给 `this.codeGenerationResults`。
    - 遍历所有模块，按 `chunkGraph.getModuleRuntimes(module)` 与 module hash 分组生成 jobs（相同 hash 的多个 runtime 合并）。
    - [_runCodeGenerationJobs](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3508-L3607) 用 `asyncLib.eachLimit(jobs, options.parallelism, ...)` 并行执行 [_codeGenerationModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3622)：
      - 先查 `_codeGenerationCache`（MultiItemCache）。
      - 未命中则调用 `module.codeGeneration({ chunkGraph, moduleGraph, dependencyTemplates, runtimeTemplate, runtime, codeGenerationResults, compilation: this })`，结果存入 `codeGenerationResults` 与缓存。
      - 对 NormalModule，[codeGeneration](file:///e:/newGsb/questions/GSB-013/Steve/lib/NormalModule.js#L1445) 调用 generator 生成各 source type（javascript/css/asset），并通过 dependencyTemplates 渲染依赖。
      - 有 `codeGenerationDependencies` 的模块会被延迟到下一轮迭代，避免循环依赖死锁；全部延迟则报错。
23. `afterCodeGeneration`。
24. `beforeRuntimeRequirements` → [processRuntimeRequirements](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3708-L3787)：
    - 从 `codeGenerationResults` 取出每个 module 每个 runtime 的 runtimeRequirements，经 `additionalModuleRuntimeRequirements` 与 `runtimeRequirementInModule` HookMap 扩充后写入 chunkGraph。
    - （后续同方法内还会遍历 chunk/tree 层级，触发 `runtimeRequirementInTree`，RuntimePlugin 在此把 RuntimeGlobals 翻译成 RuntimeModule 加入 chunk。未证实：tree 级处理在同方法后半段，本次未逐行读完，但 RuntimePlugin 的 tap 模式与 module 级对称。）
25. `afterRuntimeRequirements`。
26. `beforeHash` → [createHash()](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L4321) 返回 `codeGenerationJobs`（runtime module 需在 runtime requirements 确定后才参与 code generation）→ `afterHash`。
27. `_runCodeGenerationJobs(codeGenerationJobs, cb)` 执行 runtime modules 的代码生成。
28. 若 shouldRecord：`recordHash`。

**产物生成**

29. `clearAssets()` → `beforeModuleAssets` → [createModuleAssets()](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L4877)（模块级 asset，如 asset module 的资源）。
30. `hooks.shouldGenerateChunkAssets.call()` 不为 false 时：`beforeChunkAssets` → [createChunkAssets(callback)](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L4914-L5070)：
    - `asyncLib.forEachLimit(this.chunks, 15, (chunk, cb) => { ... })` 并行处理每个 chunk。
    - 调用 [getRenderManifest](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L4906-L4908)，即 `hooks.renderManifest.call([], options)`（`SyncWaterfallHook`）。JavascriptModulesPlugin、CssModulesPlugin 等在此 push `RenderManifestEntry`，每个 entry 含 `render()`、`filenameTemplate`/`filename`、`pathOptions`、`info`、`identifier`、`hash`。
    - 对每个 manifest entry，查 `_assetsCache`，未命中则调用 `fileManifest.render()` 生成 `Source`，用 `CachedSource` 包装，然后 `this.emitAsset(file, source, assetInfo)` 放入 `this.assets`/`this.assetsInfo`，并加入 `chunk.files`，触发 `hooks.chunkAsset.call(chunk, file)`。
31. `cont()` 内：
    - `hooks.processAssets.callAsync(this.assets, cb)` — 资产处理的统一 hook，按 `Compilation.PROCESS_ASSETS_STAGE_*` 阶段执行。旧的 `optimizeChunkAssets`/`additionalAssets` 等 deprecated hook 通过拦截器转发到此 hook（见 [Compilation.js#L512-L644](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L512-L644)）。
    - `afterProcessAssets`。
    - `this.assets` 被冻结（或 backCompat 下用 soon-frozen 代理）。
    - [summarizeDependencies()](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L4203) 汇总 file/context/missing/build dependencies。
    - 若 shouldRecord：`hooks.record.call(this, records)`。
    - `hooks.needAdditionalSeal.call()` 为真时 `unseal()` 后递归 `this.seal(callback)`（与 additionalPass 不同，这是同一轮编译内重封）。
    - 否则 `hooks.afterSeal.callAsync(cb)`，完成后 `finalCallback()`。

### 2.6 emit 阶段（runtime 域）

seal 完成后回到 [Compiler.run 的 onCompiled](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L525-L580)，进入 [Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L676-L1021)：

1. `hooks.emit.callAsync(compilation, cb)` — 异步，插件可在此增删 assets。
2. `outputPath = compilation.getPath(this.outputPath, {})`，`mkdirp(outputFileSystem, outputPath, emitFiles)`。
3. `emitFiles` 内：
   - `const assets = compilation.getAssets()`，并 `compilation.assets = { ...compilation.assets }`（拷贝一份）。
   - `asyncLib.forEachLimit(assets, 15, ({ name, source, info }, cb) => { ... })` 并行写文件：
     - 处理 query string、immutable 标记。
     - 必要时 `mkdirp` 子目录。
     - 通过 `_assetEmittingSourceCache`（WeakMap<Source, CacheEntry>）与 `_assetEmittingWrittenFiles`（Map<path, generation>）做增量写出判断：
       - 若同一 Source 已写到同一路径且 generation 匹配，且文件在上一轮存在，可直接跳过。
       - `options.output.compareBeforeEmit` 为真时，先 `stat` 目标文件，存在则比较 size 和内容（`readFile` + `Buffer.equals`），相同则跳过以保持 mtime；不同才 `writeFile`。
       - immutable 文件已存在时直接跳过。
     - `outputFileSystem.writeFile(targetPath, content, cb)`，成功后 `compilation.emittedAssets.add(file)`，并 `hooks.assetEmitted.callAsync(file, { content, source, outputPath, compilation, targetPath }, cb)`。
     - 大小写冲突检测：若两个 asset 解析到同一大小写不敏感路径但 source 不同，报错防止竞争。
   - 所有文件写完后 `hooks.afterEmit.callAsync(compilation, cb)`。
4. 回到 `onCompiled`，继续 `emitRecords` → `done` → `storeBuildDependencies`。

---

## 3. 核心对象的创建、持有与可用时机

| 对象 | 创建者 | 创建位置 | 持有者 | 生命周期 | 何时可用 |
|------|--------|----------|--------|----------|----------|
| [Compiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L137) | `createCompiler` | [webpack.js#L68](file:///e:/newGsb/questions/GSB-013/Steve/lib/webpack.js#L68) | 调用方 / MultiCompiler | `webpack()` 到 `close()` | `environment` hook 之后文件系统就绪；`initialize` hook 时完全就绪 |
| [NormalModuleFactory](file:///e:/newGsb/questions/GSB-013/Steve/lib/NormalModuleFactory.js) | `Compiler.createNormalModuleFactory` | [Compiler.js#L1279](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1279) | `compiler._lastNormalModuleFactory`；通过 `CompilationParams` 传入 Compilation | 单次 compile | `normalModuleFactory` hook 触发时；Compilation 构造时通过 `params` 可用 |
| [ContextModuleFactory](file:///e:/newGsb/questions/GSB-013/Steve/lib/ContextModuleFactory.js) | `Compiler.createContextModuleFactory` | [Compiler.js#L1293](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1293) | 同上 | 单次 compile | 同上 |
| [Compilation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L481) | `Compiler.createCompilation` | [Compiler.js#L1261](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1261) | `compiler._lastCompilation`；run/watch 回调临时持有 | 单次 compile（seal 后仍存在到 Stats 生成与 emit） | `thisCompilation`/`compilation` hook 触发时；seal 前可修改 modules/chunks，seal 后 assets 冻结 |
| [ModuleGraph](file:///e:/newGsb/questions/GSB-013/Steve/lib/ModuleGraph.js#L128) | `Compilation` 构造函数 | [Compilation.js#L1057](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1057) | `compilation.moduleGraph` | 单次 Compilation | Compilation 构造后立即可用；make 阶段填充；seal 时 `freeze("seal")`；`unseal()` 时 unfreeze |
| [ChunkGraph](file:///e:/newGsb/questions/GSB-013/Steve/lib/ChunkGraph.js#L245) | `Compilation.seal` | [Compilation.js#L3063](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3063) | `compilation.chunkGraph` | 单次 Compilation 的 seal 阶段 | seal 开始时创建，此前为 `undefined`；unseal 后重新 seal 会创建新实例（未证实：unseal 是否新建 ChunkGraph，从代码看 unseal 清空 chunks 但未置空 chunkGraph，重入 seal 会 new 一个覆盖） |
| [CodeGenerationResults](file:///e:/newGsb/questions/GSB-013/Steve/lib/CodeGenerationResults.js) | `Compilation.codeGeneration` | [Compilation.js#L3470](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3470) | `compilation.codeGenerationResults` | seal 内 | codeGeneration 阶段创建，createChunkAssets 时使用 |
| [Cache](file:///e:/newGsb/questions/GSB-013/Steve/lib/Cache.js#L53) | `Compiler` 构造函数 | [Compiler.js#L287](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L287) | `compiler.cache` | 与 Compiler 相同 | 构造后可用；实际存取由 MemoryCachePlugin/IdleFileCachePlugin 等在 `cache.hooks.get/store` 上 tap |
| [Watching](file:///e:/newGsb/questions/GSB-013/Steve/lib/Watching.js#L27) | `Compiler.watch` | [Compiler.js#L466](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L466) | `compiler.watching` | `watch()` 到 `close()` | 构造后 `process.nextTick` 触发首次 `_invalidate()` |
| [Stats](file:///e:/newGsb/questions/GSB-013/Steve/lib/Stats.js) | `Compiler.run` 的 onCompiled / Watching._done | [Compiler.js#L517](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L517) | 回调参数、`done` hook 参数 | 一次构建结束后 | seal 完成、startTime/endTime 设置后构造 |

**ModuleGraph 与 ChunkGraph 的分工**：

- [ModuleGraph](file:///e:/newGsb/questions/GSB-013/Steve/lib/ModuleGraph.js) 在 Compilation 构造时即创建，记录模块间依赖关系：`_dependencyMap`（Dependency → ModuleGraphConnection）、`_moduleMap`（Module → ModuleGraphModule，含 incoming/outgoing connections、issuer、exports info、depth、pre/postOrderIndex）。make 阶段通过 `setResolvedModule`、`setParents`、`addConnection` 等填充。
- [ChunkGraph](file:///e:/newGsb/questions/GSB-013/Steve/lib/ChunkGraph.js) 在 seal 时创建，持有 ModuleGraph 引用，记录模块与 chunk 的归属关系：`_modules`（Module → ChunkGraphModule，含该模块属于哪些 chunks、entryInChunks、runtimeInChunks、各 runtime 的 hash/id/runtimeRequirements）、`_chunks`（Chunk → ChunkGraphChunk，含该 chunk 包含的 modules、entryModules、runtimeModules、runtimeRequirements）。模块图回答「谁依赖谁」，chunk 图回答「谁打进哪个包」。

**Compilation 上的队列与缓存字段**（构造于 [Compilation.js#L1063-L1093](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1063-L1093)）：

- `processDependenciesQueue`、`addModuleQueue`、`factorizeQueue`、`buildQueue`、`rebuildQueue`。
- `_modulesCache`、`_codeGenerationCache`、`_assetsCache`（未在截取段内直接看到定义，由 CacheFacade 包装 `compiler.cache`，未证实具体初始化行号）。

---

## 4. watch/cache 域

### 4.1 watch 启动

[Compiler.watch(watchOptions, handler)](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L459-L468)：

- 防重入后设置 `running=true`、`watchMode=true`。
- `this.watching = new Watching(this, watchOptions, handler)`，返回该实例。

[Watching 构造函数](file:///e:/newGsb/questions/GSB-013/Steve/lib/Watching.js#L33-L77)：规范化 `aggregateTimeout`（默认 20ms），`process.nextTick(() => { if (this._initial) this._invalidate(); })` 启动首次构建。

### 4.2 单次 watch 构建流程

[Watching._go](file:///e:/newGsb/questions/GSB-013/Steve/lib/Watching.js#L109-L239) 与 `Compiler.run` 结构类似，但有以下区别：

- 暂停上一个 watcher（`this.watcher.pause()`），保留为 `pausedWatcher`，从中取出聚合的 `fileTimestamps`/`contextTimestamps`/`changes`/`removals`。
- 设置 `compiler.modifiedFiles`、`compiler.removedFiles`、`compiler.fileTimestamps`、`compiler.contextTimestamps`、`compiler.fsStartTime`。
- 触发的是 `compiler.hooks.watchRun.callAsync(this.compiler, ...)` 而非 `beforeRun`/`run`。
- 不调用 `readRecords`（首次由 `_needRecords` 控制只读取一次）。
- 直接 `this.compiler.compile(onCompiled)`。
- `onCompiled` 中若 `this.invalid` 为 true（构建期间又有文件变化），直接 `_done(null, compilation)` 而不 emit，从而立即触发下一轮。
- emit 完成后同样检查 `needAdditionalPass`。

[Watching._done](file:///e:/newGsb/questions/GSB-013/Steve/lib/Watching.js#L255-L346)：

- 若 `this.invalid` 且未 suspended/blocked，先 `storeBuildDependencies` 然后再次 `_go()`（不等 idle）。
- 否则构造 Stats，触发 `compiler.hooks.done`，调用用户 `handler`，`storeBuildDependencies`，`cache.beginIdle()`，设置 `compiler.idle = true`，然后 `process.nextTick` 调用 `this.watch(fileDependencies, contextDependencies, missingDependencies)` 重新建立文件监听。
- `compiler.hooks.afterDone` 在最后触发。

### 4.3 增量重建

watch 模式下，文件变化后 `Watching._invalidate` 聚合变更，下次 `_go` 把 `modifiedFiles`/`removedFiles` 传入 Compilation。模块级别通过 [Compilation.rebuildModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2434-L2481) 入 `rebuildQueue`：

- 保存旧 dependencies/blocks。
- `module.invalidateBuild()`、`buildQueue.invalidate(module)`。
- 重新 `buildModule`，然后 `processDependenciesQueue.invalidate(module)`、`moduleGraph.unfreeze()`、重新 `processModuleDependencies`，再 `removeReasonsOfDependencyBlock` 清理旧依赖。
- 完成后 `hooks.finishRebuildingModule`。

### 4.4 Cache 子系统

[Cache](file:///e:/newGsb/questions/GSB-013/Steve/lib/Cache.js#L53-L69) 本身只定义 hooks：`get`（AsyncSeriesBailHook）、`store`（AsyncParallelHook）、`storeBuildDependencies`、`beginIdle`、`endIdle`、`shutdown`。具体策略由插件 tap 这些 hook：

- `cache.type === "memory"` 且无 `maxGenerations`：[MemoryCachePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/cache/MemoryCachePlugin.js)。
- 有 `maxGenerations`：MemoryWithGcCachePlugin（未读源码，未证实具体策略）。
- `cache.type === "filesystem"` + `store: "pack"`：[IdleFileCachePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/cache/IdleFileCachePlugin.js) 包装 [PackFileCacheStrategy](file:///e:/newGsb/questions/GSB-013/Steve/lib/cache/PackFileCacheStrategy.js)，在 `beginIdle`/`endIdle`/`shutdown` 时序列化与反序列化。
- 始终挂载 [ResolverCachePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/cache/ResolverCachePlugin.js)。

Compilation 通过 [CacheFacade](file:///e:/newGsb/questions/GSB-013/Steve/lib/CacheFacade.js) 访问缓存，`compiler.getCache(name)` 返回以 `${compilerPath}${name}` 为命名空间的 facade（[Compiler.js#L331-L337](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L331-L337)）。`_modulesCache`、`_codeGenerationCache`、`_assetsCache` 均基于此。

`Compiler.run`/`Watching._done` 在构建结束时调用 `cache.beginIdle()`，下次构建开始时若 `idle` 为 true 先 `cache.endIdle()`。filesystem cache 借此在空闲时落盘。

---

## 5. 关键 hook 速查与异步边界

### 5.1 Compiler hooks（创建于 [Compiler.js#L143-L217](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L143-L217)）

初始化期（同步，`createCompiler` 内按序）：
`environment` → `afterEnvironment` →（`WebpackOptionsApply.process`）→ `initialize`

每次 compile：
- `beforeRun`（AsyncSeries，run 模式；watch 用 `watchRun`）
- `run`（AsyncSeries）
- `readRecords`（AsyncSeries，内部）
- `beforeCompile`（AsyncSeries，params）
- `compile`（Sync，params）
- `thisCompilation`/`compilation`（Sync，在 newCompilation 内）
- `normalModuleFactory`/`contextModuleFactory`（Sync，工厂创建时）
- `make`（**AsyncParallel**，compilation）— 图构建入口，可并行
- `finishMake`（AsyncSeries，compilation）
- `afterCompile`（AsyncSeries，compilation）
- `shouldEmit`（SyncBail，compilation）— 返回 false 短路跳过 emit
- `emit`（AsyncSeries，compilation）
- `assetEmitted`（AsyncSeries，file, info）
- `afterEmit`（AsyncSeries，compilation）
- `done`（AsyncSeries，stats）
- `afterDone`（Sync，stats）
- `additionalPass`（AsyncSeries）— 在 needAdditionalPass 时触发，之后重新 compile
- `failed`（Sync，error）

watch 相关：`watchRun`、`invalid`、`watchClose`、`shutdown`。

**异步边界注意**：`make` 是 `AsyncParallelHook`，多个 entry 的 addEntry 并行执行；但其内部队列有父子依赖，保证所有传递模块处理完才完成。`optimizeTree`、`optimizeChunkModules`、`processAssets` 是异步瀑布中少数 AsyncSeries hook，插件可做异步工作。

### 5.2 Compilation hooks（创建于 [Compilation.js#L707-L993](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L707-L993)）

模块构建期：`buildModule`、`failedModule`、`succeedModule`、`stillValidModule`、`rebuildModule`、`finishRebuildingModule`。

入口：`addEntry`、`failedEntry`、`succeedEntry`。

finish：`finishModules`（AsyncSeries）。

seal 内按序（同步，除标注外）：
`seal` → `optimizeDependencies`(SyncBail, while 循环) → `afterOptimizeDependencies` → `beforeChunks` → `afterChunks` → `optimize` → `optimizeModules`(SyncBail, while) → `afterOptimizeModules` → `optimizeChunks`(SyncBail, while) → `afterOptimizeChunks` → `optimizeTree`(**AsyncSeries**) → `afterOptimizeTree` → `optimizeChunkModules`(**AsyncSeriesBail**) → `afterOptimizeChunkModules` → `shouldRecord`(SyncBail) → `reviveModules` → `beforeModuleIds` → `moduleIds` → `optimizeModuleIds` → `afterOptimizeModuleIds` → `reviveChunks` → `beforeChunkIds` → `chunkIds` → `optimizeChunkIds` → `afterOptimizeChunkIds` → `optimizeCodeGeneration` → `beforeModuleHash` → `afterModuleHash` → `beforeCodeGeneration` → `afterCodeGeneration` → `beforeRuntimeRequirements` → `afterRuntimeRequirements` → `beforeHash` → `afterHash` → `beforeModuleAssets` → `shouldGenerateChunkAssets`(SyncBail) → `beforeChunkAssets` → `processAssets`(**AsyncSeries**) → `afterProcessAssets` → `needAdditionalSeal`(SyncBail) → `afterSeal`(**AsyncSeries**)。

render：`renderManifest`（SyncWaterfall，各模块类型插件追加 RenderManifestEntry）。

runtime requirements：`additionalModuleRuntimeRequirements`、`runtimeRequirementInModule`(HookMap)、`additionalChunkRuntimeRequirements`、`runtimeRequirementInChunk`(HookMap)、`additionalTreeRuntimeRequirements`、`runtimeRequirementInTree`(HookMap)、`runtimeModule`。

hash：`fullHash`、`chunkHash`、`contentHash`、`moduleHash`(通过 beforeModuleHash/afterModuleHash)。

asset：`moduleAsset`、`chunkAsset`、`assetPath`(SyncWaterfall)、`processAssets`、`processAdditionalAssets`、`needAdditionalPass`(SyncBail)。

**短路 hook**：`optimizeDependencies`/`optimizeModules`/`optimizeChunks` 是 `SyncBailHook` 且被 `while` 包裹，返回真值会重复调用，插件可借此实现多轮定点优化。`shouldEmit`、`shouldGenerateChunkAssets`、`shouldRecord`、`needAdditionalSeal`、`needAdditionalPass` 返回 false/true 直接改变控制流。

### 5.3 多次编译的两种机制

- **additionalPass**（Compiler 级）：`compilation.hooks.needAdditionalPass` 返回 true → `compiler.hooks.done` → `additionalPass` → 再次 `compiler.compile()`。产生全新 Compilation，旧 Compilation 的 Stats 仍会传给 done。
- **needAdditionalSeal**（Compilation 级）：seal 末尾 `hooks.needAdditionalSeal` 返回 true → `unseal()`（清空 chunks/chunkGroups/entrypoints/assets，但保留 modules 与 moduleGraph）→ 递归 `this.seal(callback)`。同一 Compilation 内重封，模块不重新构建。

---

## 6. 子编译器与 MultiCompiler

- [Compiler.createChildCompiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L1167-L1249) 创建独立 Compiler，但共享 `inputFileSystem`、`resolverFactory`、`cache`、文件时间戳与 records 子树；`outputFileSystem` 为 null（由父 compilation 通过 `runAsChild` 收集 assets）。hook taps 除 `make/compile/emit/afterEmit/invalid/done/thisCompilation` 外从父 compiler 复制。子编译通过 `compiler.runAsChild(callback)` 触发（[Compiler.js#L615-L663](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compiler.js#L615-L663)），完成后把子 compilation 的 assets 通过 `parentCompilation.emitAsset` 注入，并把子 entry chunks 返回。
- [MultiCompiler](file:///e:/newGsb/questions/GSB-013/Steve/lib/MultiCompiler.js) 持有子 compiler 数组，按 `dependencies` 拓扑排序执行；`run`/`watch` 等待依赖编译器完成后再启动依赖方。未逐行阅读，未证实其并发控制细节。

---

## 7. 待深入与未证实项

- MultiCompiler 的并发/依赖调度未逐行阅读。
- `Compilation.unseal()` 后重 seal 是否新建 ChunkGraph：源码显示 seal 开头无条件 `new ChunkGraph`，但 unseal 未把 `this.chunkGraph` 置空，第二次 seal 会覆盖旧引用；旧 ChunkGraph 的 WeakMap 条目是否被显式清理未证实。
- NormalModuleFactory 的 `factory.create` 内部 resolver/loader 流水线未在本次展开。
- `processRuntimeRequirements` 后半段的 chunk/tree 级处理与 runtime module 添加顺序未逐行读完。
- MemoryWithGcCachePlugin、PackFileCacheStrategy 的序列化格式与 GC 策略未展开。
- `compilation.hooks.thisCompilation` 与 `compilation` 在子编译器场景下的触发差异未在 createChildCompiler 中看到显式区分，需进一步核对。
