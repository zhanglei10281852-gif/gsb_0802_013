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

## 7. 从一条 import 到浏览器装载：静态依赖、动态 import() 与 chunk 形成

本节用一个具体场景把 parser → Dependency → ModuleGraph → buildChunkGraph → ChunkGraph → code generation → runtime requirements → RuntimeModule → emitted runtime 串起来。

### 7.1 场景设定

三个源码文件，两个入口：

```js
// entry-a.js
import { foo } from "./shared.js";
import("./async.js").then(m => m.bar());
console.log(foo);

// entry-b.js
import { foo } from "./shared.js";
console.log(foo);

// shared.js
export const foo = "shared";

// async.js
export const bar = () => "async";
```

`entry-a` 和 `entry-b` 都静态依赖 `shared.js`；`entry-a` 还通过普通动态 `import("./async.js")` 引用 `async.js`。

### 7.2 Parser 阶段：Dependency 与 AsyncDependenciesBlock 的产生

NormalModule 构建时，`module.build()` 调用 parser 解析 AST。[HarmonyImportDependencyParserPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependencyParserPlugin.js) 和 [ImportParserPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js) 在 parser hooks 上注册了 tap，把语法节点翻译成 Dependency 对象。

**静态 `import { foo } from "./shared.js"`** 产生两类 Dependency，全部直接挂在 module 的 `dependencies` 数组上（不是 block）：

1. `parser.hooks.import`（[HarmonyImportDependencyParserPlugin.js#L109-L133](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L109-L133)）：
   - 先创建一个 [ConstDependency](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ConstDependency.js)，把整行 `import ...` 语句替换为空字符串（或 ASI 分号），通过 `parser.state.module.addPresentationalDependency(clearDep)` 添加为展示性依赖。
   - 再创建 [HarmonyImportSideEffectDependency](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportSideEffectDependency.js)（继承自 [HarmonyImportDependency](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js)），通过 `parser.state.module.addDependency(sideEffectDep)` 添加。这个 dependency 负责在运行时触发 `__webpack_require__("./shared.js")` 以执行 shared 模块的副作用。

2. `parser.hooks.importSpecifier`（[#L134-L147](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L134-L147)）：不创建 Dependency，而是用 `parser.tagVariable(name, harmonySpecifierTag, { source, ids, sourceOrder, attributes })` 给导入的本地变量名打标签。后续代码中引用该变量时，标签触发表达式 hook。

3. 当 `foo` 被使用时（`console.log(foo)`），`parser.hooks.expression.for(harmonySpecifierTag)`（[#L192-L219](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L192-L219)）触发，创建 [HarmonyImportSpecifierDependency](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportSpecifierDependency.js)，记录 `ids: ["foo"]`、`source: "./shared.js"`、`sourceOrder`、range 等，`addDependency` 挂到 module 上。如果是成员访问 `obj.foo`，走 `expressionMemberChain`；如果是调用 `foo()`，走 `callMemberChain`。这些 specifier dependency 负责把源码中的 `foo` 替换为对 shared 模块导出属性的访问表达式。

因此 `entry-a.js` 和 `entry-b.js` 各自在 `dependencies` 数组里都有一个 `HarmonyImportSideEffectDependency` 指向 shared，以及使用处的 `HarmonyImportSpecifierDependency`。两个入口的 side-effect dependency 是不同的 Dependency 实例，但它们指向同一个 Module（解析后由 `_addModule` 去重）。

**动态 `import("./async.js")`** 的处理在 [ImportParserPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js)。默认 `webpackMode` 是 `"lazy"`（[#L51](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js#L51)），当 `param.isString()` 时（[#L262-L301](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js#L262-L301)）：

1. 创建 [AsyncDependenciesBlock](file:///e:/newGsb/questions/GSB-013/Steve/lib/AsyncDependenciesBlock.js)，传入 `groupOptions`（含 `webpackChunkName`、`prefetchOrder`、`preloadOrder`、`fetchPriority` 等，无 magic comment 时为 `{ name: undefined }`）、`loc`、`request`。
2. 创建 [ImportDependency](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportDependency.js)（继承 `ModuleDependency`，`type` 为 `"import()"`，`category` 为 `"esm"`）。
3. `depBlock.addDependency(dep)` —— ImportDependency 挂在 **block 的 dependencies** 上，不是 module 的。
4. `parser.state.current.addBlock(depBlock)` —— block 挂在当前 DependenciesBlock（即 entry-a 模块）的 `blocks` 数组上。

这是静态依赖与动态依赖在数据结构上的根本区别：静态 import 的 dependency 直接在 module 的 `dependencies` 里；动态 import() 的 dependency 被包在一个 `AsyncDependenciesBlock` 里，block 在 module 的 `blocks` 里。`AsyncDependenciesBlock` 携带 `groupOptions`（chunk 命名、preload/prefetch 等），这是后续形成异步 chunk group 的依据。`mode: "eager"` 则不创建 block，直接用 `ImportEagerDependency` 加到当前 module（[#L265-L272](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js#L265-L272)）；`mode: "weak"` 用 `ImportWeakDependency`（[#L273-L280](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js#L273-L280)）；非字符串参数走 `ImportContextDependency`。

### 7.3 模块图构建：Dependency → ModuleGraphConnection

make 阶段，[handleModuleCreation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1935) 通过 `_factorizeModule` 让 NormalModuleFactory 解析每个 dependency 的 request，创建 Module，然后在 [_addModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1419-L1455) 中按 identifier 去重——entry-a 和 entry-b 对 shared 的 dependency 解析出的是同一个 NormalModule 实例。

关键映射发生在 [handleModuleCreation](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2038-L2047)：

```js
for (const dependency of dependencies) {
    moduleGraph.setResolvedModule(
        connectOrigin ? originModule : null,
        dependency,
        module
    );
}
moduleGraph.setIssuerIfUnset(module, originModule !== undefined ? originModule : null);
```

[ModuleGraph.setResolvedModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/ModuleGraph.js) 在 `_dependencyMap`（WeakMap<Dependency, ModuleGraphConnection>）中为每个 Dependency 建立一个 [ModuleGraphConnection](file:///e:/newGsb/questions/GSB-013/Steve/lib/ModuleGraphConnection.js)，记录 originModule、targetModule、dependency、active state 等。这意味着：

- entry-a 的 `HarmonyImportSideEffectDependency` → connection(originModule=entry-a, targetModule=shared)
- entry-b 的 `HarmonyImportSideEffectDependency` → connection(originModule=entry-b, targetModule=shared)
- entry-a 的 `HarmonyImportSpecifierDependency`(ids=["foo"]) → connection(originModule=entry-a, targetModule=shared)
- async block 内的 `ImportDependency` → connection(originModule=entry-a, targetModule=async)

在 [_processModuleDependencies](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1593) 中，每个 dependency 还会调用 `moduleGraph.setParents(dep, block, module, index)`（[#L1672](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L1672)），记录 dependency 属于哪个 DependenciesBlock 和 module。对于 async block 内的 ImportDependency，它的 parent block 是那个 AsyncDependenciesBlock，parent module 是 entry-a。

此时 [ModuleGraph](file:///e:/newGsb/questions/GSB-013/Steve/lib/ModuleGraph.js) 已经完整：`_moduleMap` 中 shared、async、entry-a、entry-b 各有一个 ModuleGraphModule，incomingConnections 汇集了所有指向它们的 connection，outgoingConnections 记录它们依赖谁。**但此时还没有 chunk 的概念**——ModuleGraph 只回答模块间依赖，不回答模块在哪个产物文件里。

### 7.4 buildChunkGraph：从模块图到 chunk 图

seal 阶段先 `new ChunkGraph(moduleGraph, hashFunction)`（[Compilation.js#L3063-L3067](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3063-L3067)），然后为每个 entry 创建初始 chunk 和 Entrypoint（[#L3089-L3151](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3089-L3151)）：

- entry-a → chunk `a` + Entrypoint，`chunkGraph.connectChunkAndEntryModule(chunk-a, entry-a, entrypoint)`
- entry-b → chunk `b` + Entrypoint，`chunkGraph.connectChunkAndEntryModule(chunk-b, entry-b, entrypoint)`

此时 shared 还不在任何 chunk 里。接下来调用 [buildChunkGraph(this, chunkGraphInit)](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L1301)，算法分两部分。

**PART ONE: visitModules（[buildChunkGraph.js#L247](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L247)）**

队列初始化为每个 entrypoint 的 entry module 推入 `ADD_AND_ENTER_MODULE`（[#L422-L431](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L422-L431)）。`processQueue`（[#L803-L888](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L803-L888)）处理队列项：

- `ADD_AND_ENTER_MODULE`：调用 `chunkGraph.connectChunkAndModule(chunk, module)`（[#L829](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L829)）把模块加入当前 chunk，更新 `maskByChunk` 位掩码，然后 fall through 到 ENTER_MODULE。
- `ENTER_MODULE`：设置 pre-order index，把 action 改为 `LEAVE_MODULE` 重新入队，然后 fall through 到 `PROCESS_BLOCK`。
- `PROCESS_BLOCK`：调用 [processBlock](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L675-L761)。

`processBlock` 是核心：

1. 通过 `getBlockModules(block, runtime)` 提取该 block 直接和间接引用的所有模块（[extractBlockModules](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L108) 遍历 block 的 dependencies 和子 blocks，解析 active connections）。
2. 对每个引用模块 refModule：
   - 如果已经在当前 chunk 中（`chunkGraph.isModuleInChunk`），跳过。
   - 如果 connection 不是 active（条件依赖、runtime 条件不满足），加入 skippedModuleConnections，可能跳过。
   - **关键**：如果 `isOrdinalSetInMask(minAvailableModules, refOrdinal)` 为 true（[#L707](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L707)），说明该模块已经在父级 chunk group 的某个 chunk 中可用，加入 skippedItems，不在当前 chunk 重复放置。
   - 否则入队 `ADD_AND_ENTER_MODULE`，把模块加入当前 chunk。
3. 遍历 `block.blocks`（即 AsyncDependenciesBlock），对每个调用 [iteratorBlock](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L488-L669)。

对场景的执行过程：

- 处理 entry-a 时，entry-a 加入 chunk-a。processBlock 遍历它的 dependencies，发现 shared 不在 chunk-a 也不在 minAvailableModules（初始为 0），于是把 shared 加入 chunk-a。遍历 blocks 发现 async block，进入 iteratorBlock。
- **iteratorBlock 处理 async block**（[#L569-L632](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L569-L632)）：因为 `chunkGroupInfo.asyncChunks` 和 `chunkGroupInfo.chunkLoading` 都为 true（默认配置），且无 chunkName：
  - 调用 [compilation.addChunkInGroup(groupOptions, module, loc, request)](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3897-L3935) 创建一个新的普通 ChunkGroup（非 Entrypoint）和一个新 chunk（async chunk）。
  - 创建对应的 ChunkGroupInfo，runtime 继承自父 chunkGroupInfo，`minAvailableModules` 初始为 undefined（后续由父级 merge）。
  - 在 `blockConnections` 中记录从 entry-a 的 chunkGroupInfo 到新 chunkGroup 的连接。
  - 在 `queueConnect` 中入队连接，后续会把父级的 `resultingAvailableModules` merge 到子级的 `minAvailableModules`。
  - 把 PROCESS_BLOCK 项（目标是新 chunk、新 chunkGroupInfo）入队，延迟处理。
- processQueue 继续处理 entry-b：entry-b 加入 chunk-b。它的 dependencies 也引用 shared，但此时 shared **已经在 chunk-a 中**——不过 chunk-a 和 chunk-b 是不同的 entry runtime，minAvailableModules 是按 chunkGroup 独立计算的。对 chunk-b 来说，它的 minAvailableModules 从 0 开始，shared 不在其中，所以 shared **也会被加入 chunk-b**。

  > 这就是"共享模块"在多入口下的默认行为：shared 模块被 **复制到每个入口 chunk**（每个入口 runtime 独立）。只有当 shared 被异步 chunk 引用、且已在父级 entry chunk 中时，才会被跳过（不重复打入异步 chunk）。要跨入口真正去重需要 splitChunks 或 runtimeChunk 配置。

- 处理 async chunk 的 PROCESS_BLOCK 时，async block 的 dependencies 指向 async 模块。此时 async chunkGroupInfo 的 `minAvailableModules` 已经从父级（entry-a）merge 了 resultingAvailableModules（即 chunk-a 中所有模块的位掩码，**包含 shared 和 entry-a**）。因此：
  - async 模块不在 minAvailableModules 中 → 加入 async chunk。
  - 如果 async.js 也 import 了 shared.js，shared 在 minAvailableModules 中 → **跳过**，不打入 async chunk，因为运行时从 chunk-a 中已可获取。

`minAvailableModules` 的 merge 逻辑在 [processChunkGroupsForMerging](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L947-L997)：子 chunkGroup 的 minAvailableModules 是所有父级 resultingAvailableModules 的**按位与**（intersection），即"所有父路径都已包含的模块"。这保证只有在所有父级都有的模块才被视为可用，避免在某个父路径缺失时重复打入或漏打。

**PART TWO: connectChunkGroups（[buildChunkGraph.js#L1216-L1273](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L1216-L1273)）**

遍历 `blockConnections`，对每个 async block 到 chunkGroup 的连接：

1. 先检查是否可以跳过：如果该 block 没有嵌套 blocks，且目标 chunkGroup 的所有模块在 `originChunkGroupInfo.resultingAvailableModules` 中已全部可用（[#L1246-L1257](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L1246-L1257)），则不建立连接——这种情况说明 async block 引用的模块全部已在父 chunk 中，不需要异步加载。
2. 否则调用 `chunkGraph.connectBlockAndChunkGroup(block, chunkGroup)` 建立 block → chunkGroup 映射，并通过 [connectChunkGroupParentAndChild](file:///e:/newGsb/questions/GSB-013/Steve/lib/GraphHelpers.js) 把 entry-a 的 ChunkGroup 设为 async chunkGroup 的 parent，async chunkGroup 成为 entry-a ChunkGroup 的 child。

最后 [cleanupUnconnectedGroups](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L1280-L1293) 删除没有 parent 的 chunkGroup（即没有被任何 async block 引用的孤儿 group）。

执行完后，ChunkGraph 状态：

- chunk-a：{ entry-a, shared }，是 Entrypoint
- chunk-b：{ entry-b, shared }，是 Entrypoint
- async chunk（如 `[request]` 命名或数字 id）：{ async }，普通 ChunkGroup，parent 是 entry-a 的 Entrypoint
- AsyncDependenciesBlock 通过 `chunkGraph.getBlockChunkGroup(block)` 可查到对应的 ChunkGroup

### 7.5 chunk 复用与命名条件

- **同名 chunk group 复用**：[addChunkInGroup](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3902-L3915) 先查 `namedChunkGroups.get(name)`，存在则直接复用（添加 origin），不创建新 chunk。因此两个动态 import 使用相同 `webpackChunkName` 会合并到同一个异步 chunk。visitModules 的 [iteratorBlock](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L580-L610) 也先查 `namedChunkGroups`。如果该名字是初始 chunk 名，会报 `AsyncDependencyToInitialChunkError`（[#L613-L620](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L613-L620)）。
- **模块去重**：`_addModule` 按 identifier 去重，全 compilation 只有一个 Module 实例。
- **模块在 chunk 间的分配**：由 `minAvailableModules` 位掩码决定。共享模块若在父级 chunk 可用则不重复打入子 chunk；但不同独立入口的初始 chunk 各自包含共享模块副本（除非 splitChunks 抽取）。
- **asyncChunks/chunkLoading 关闭时**：[iteratorBlock](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L569-L578) 走 `PROCESS_BLOCK` 分支，async block 的依赖被并入当前 chunk，不产生新 chunk（`import()` 退化为同步加载）。

### 7.6 Code generation：从 Dependency 到运行时表达式

seal 的 codeGeneration 阶段，每个模块的 [module.codeGeneration()](file:///e:/newGsb/questions/GSB-013/Steve/lib/NormalModule.js#L1445) 调用 generator，generator 用 DependencyTemplate 把源码中的 Dependency range 替换为运行时代码，并收集 runtimeRequirements。

**静态 import 的 side-effect dependency**（HarmonyImportSideEffectDependency）使用 [HarmonyImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L270-L370)：

1. 调用 `dep.getImportStatement(false, templateContext)`（[#L331](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L331)）。
2. [getImportStatement](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L116-L129) 调用 [runtimeTemplate.importStatement()](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L790-L853)。
3. `importStatement` 生成形如：

```js
/* harmony import */ var shared__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__("./shared.js");
```

   其中 `__webpack_require__` 对应 [RuntimeGlobals.require](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeGlobals.js)，通过 `runtimeRequirements.add(RuntimeGlobals.require)`（[RuntimeTemplate.js#L842](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L842)）声明需求。变量名由 [getImportVar](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L92-L109) 生成，格式为 `${userRequest}__WEBPACK_IMPORTED_MODULE_${n}__`，缓存在 module meta 的 importVarMap 中，保证同一模块只声明一次。
4. 该语句作为 [ConditionalInitFragment](file:///e:/newGsb/questions/GSB-013/Steve/lib/ConditionalInitFragment.js) 在 `InitFragment.STAGE_HARMONY_IMPORTS` 阶段、按 sourceOrder 排序注入模块顶部（[HarmonyImportDependency.js#L360-L368](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L360-L368)）。如果引用的是 async 模块，还会追加 [AwaitDependenciesInitFragment](file:///e:/newGsb/questions/GSB-013/Steve/lib/async-modules/AwaitDependenciesInitFragment.js) 和 `STAGE_ASYNC_HARMONY_IMPORTS` 的兼容语句（[#L336-L358](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L336-L358)）。

**静态 import 的 specifier dependency**（`foo` 的使用处）使用 [HarmonyImportSpecifierDependency.Template](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportSpecifierDependency.js#L316)：

1. [apply](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportSpecifierDependency.js#L325-L400) 调用 `_getCodeForIds`，后者调用 [runtimeTemplate.exportFromImport()](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L873)。
2. 对 `import { foo }` 且 shared 是 namespace 类型（纯 ESM），`foo` 被替换为 `shared__WEBPACK_IMPORTED_MODULE_0__.foo`（或经 mangle 后的导出名）。对 CJS 互操作则可能生成 `_compat_getDefault_export` 等包装。
3. `source.replace(range[0], range[1]-1, exportExpr)` 完成源码替换。

**动态 import() 的 ImportDependency** 使用 [ImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportDependency.js#L107-L137)：

1. `apply` 中通过 `moduleGraph.getParentBlock(dep)` 拿到那个 AsyncDependenciesBlock（[#L122-L124](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportDependency.js#L122-L124)）。
2. 调用 [runtimeTemplate.moduleNamespacePromise()](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L603-L750)（[#L125-L133](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportDependency.js#L125-L133)）。
3. `moduleNamespacePromise` 内部先调用 [blockPromise()](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L986-L1042)（[#L637](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L637)）：
   - `chunkGraph.getBlockChunkGroup(block)` 拿到 buildChunkGraph 阶段关联的 ChunkGroup。
   - 过滤出 `!chunk.hasRuntime()` 的 chunks（异步 chunk 本身不含 runtime）。
   - 如果只有一个 chunk，生成 `__webpack_require__.e(chunkId)`，并 `runtimeRequirements.add(RuntimeGlobals.ensureChunk)`（[#L1009](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L1009)）。
   - 如果多个 chunk，生成 `Promise.all([__webpack_require__.e(id1), __webpack_require__.e(id2)])`。
4. 然后根据 exportsType 拼接 `.then(__webpack_require__.bind(null, moduleId))`（namespace 类型，[#L690-L692](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L690-L692)）或 `.then(__webpack_require__.bind(...)).then(__webpack_require__.createFakeNamespaceObject(...))`（default-only 等）。
5. 最终 `import("./async.js")` 表达式被替换为类似：

```js
__webpack_require__.e(/* import() */ 123).then(__webpack_require__.bind(__webpack_require__, "./async.js"))
```

   并 `source.replace(range[0], range[1]-1, content)`（[ImportDependency.js#L135](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportDependency.js#L135)）。

这里的关键区分：**静态依赖生成 `__webpack_require__(id)`（同步、立即执行），动态依赖生成 `__webpack_require__.e(chunkId).then(__webpack_require__.bind(null, id))`（先确保 chunk 加载完成，再 require 模块）**。

### 7.7 Runtime requirements 汇总与 RuntimeModule 注入

code generation 过程中，每个模块的 `codeGeneration()` 返回的 `CodeGenerationResult` 包含一个 `runtimeRequirements` Set（[Module.codeGeneration](file:///e:/newGsb/questions/GSB-013/Steve/lib/Module.js) 的约定）。对本场景：

- entry-a、entry-b、shared 的 codegen 都添加了 `RuntimeGlobals.require`（来自静态 import）。
- entry-a 的 codegen 还添加了 `RuntimeGlobals.ensureChunk`（来自动态 import 的 blockPromise）。
- 若有 fetchPriority 注释，还会加 `RuntimeGlobals.hasFetchPriority`。

seal 阶段的 [processRuntimeRequirements](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3708) 遍历所有模块和 chunk：

1. 从 `codeGenerationResults.getRuntimeRequirements(module, runtime)` 取出模块声明的需求。
2. 触发 `additionalModuleRuntimeRequirements` hook 和 `runtimeRequirementInModule` HookMap，让插件补充需求。[RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimePlugin.js) 在后者上为许多 global 注册了传递依赖（例如 `require` 相关的 global 会补 `requireScope`）。
3. `chunkGraph.addModuleRuntimeRequirements(module, runtime, set)` 记录模块级需求。
4. （方法后半段）汇总到 chunk 级和 tree 级，触发 `additionalChunkRuntimeRequirements`、`runtimeRequirementInChunk`、`additionalTreeRuntimeRequirements`、`runtimeRequirementInTree`。RuntimePlugin 在 `runtimeRequirementInTree` 上为每个 `RuntimeGlobals` 注册了对应的 RuntimeModule 工厂。

RuntimeModule 的注入发生在 `runtimeRequirementInTree` 的 tap 中。以本场景为例：

- **`ensureChunk`**：[RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimePlugin.js#L372-L383) 为 `RuntimeGlobals.ensureChunk` 注册 tap，当 chunk tree 需要它时，向 chunk 添加 [EnsureChunkRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/EnsureChunkRuntimeModule.js)。
  - 如果 chunk tree 还需要 `ensureChunkHandlers`（即有 JSONP/ESM chunk loading 机制），[EnsureChunkRuntimeModule.generate()](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/EnsureChunkRuntimeModule.js#L26-L54) 生成 `__webpack_require__.e = function(chunkId) { return Promise.all(Object.keys(__webpack_require__.f).reduce(...)) }`，它遍历所有 chunk loading handler。
  - 如果不需要 handlers（所有引用 chunk 都已内联），生成空的 `__webpack_require__.e = () => Promise.resolve()`。
- **`ensureChunkHandlers` + chunk loading 类型**：由 [JsonpTemplatePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/web/JsonpTemplatePlugin.js) / [EnableChunkLoadingPlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/javascript/EnableChunkLoadingPlugin.js) 根据 `output.chunkLoading`（web 默认 `"jsonp"`）注册。它们在 `runtimeRequirementInTree` 上为 `ensureChunkHandlers` 添加 [JsonpChunkLoadingRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/web/JsonpChunkLoadingRuntimeModule.js)。
- **`loadScript`**：[RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimePlugin.js#L395-L410) 为 `RuntimeGlobals.loadScript` 注册 tap，添加 [LoadScriptRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/LoadScriptRuntimeModule.js)，生成通过 `<script>` 标签加载 JS chunk 的代码。
- **`publicPath`、`getChunkScriptFilename`**：由 [PublicPathRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/PublicPathRuntimeModule.js) 和 [GetChunkFilenameRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/GetChunkFilenameRuntimeModule.js) 提供，决定 chunk 的 URL。
- **`require`**：由 [MainTemplate](file:///e:/newGsb/questions/GSB-013/Steve/lib/MainTemplate.js)（或模块类型对应的 template）生成核心的 `__webpack_require__` 函数、module cache、module factories 等引导代码。这不是 RuntimeModule，而是 chunk 渲染时 template 的固定部分（未证实：webpack 5 中部分引导逻辑已迁移到 RuntimeModule，但 `__webpack_require__` 主体仍由 MainTemplate/JavascriptModulesPlugin 生成，需逐行核对）。

[JsonpChunkLoadingRuntimeModule.generate()](file:///e:/newGsb/questions/GSB-013/Steve/lib/web/JsonpChunkLoadingRuntimeModule.js#L75) 生成的核心运行时代码包括：

```js
// installedChunks 记录 chunk 加载状态：0=已加载, [resolve,reject,promise]=加载中, undefined=未加载
var installedChunks = { "a": 0 };  // 初始 chunk 标记为已加载

// __webpack_require__.f.j = JSONP chunk loading handler
__webpack_require__.f.j = function(chunkId, promises) {
    var installedChunkData = installedChunks[chunkId];
    if (installedChunkData !== 0) {
        if (installedChunkData) {
            promises.push(installedChunkData[2]);
        } else {
            var promise = new Promise((resolve, reject) => {
                installedChunkData = installedChunks[chunkId] = [resolve, reject];
            });
            promises.push(installedChunkData[2] = promise);
            var url = __webpack_require__.p + __webpack_require__.u(chunkId);  // publicPath + chunk filename
            __webpack_require__.l(url, ...);  // loadScript，创建 <script>
        }
    }
};
```

其中 `__webpack_require__.l` 就是 [LoadScriptRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/LoadScriptRuntimeModule.js#L58-L171) 生成的 `loadScript` 函数，它创建 `<script>` 元素、设置 `src`、`onload`/`onerror`、超时处理，并通过 `document.head.appendChild(script)` 注入。异步 chunk 文件本身以 JSONP 方式包裹（`window["webpackChunk"].push([[chunkId], {...modules}])`），加载后由 webpack runtime 的 push 回调注册模块并 resolve promise。

`__webpack_require__.u(chunkId)`（chunk 文件名映射）由 [GetChunkFilenameRuntimeModule](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/GetChunkFilenameRuntimeModule.js) 生成，通常是一个从 chunkId 到文件名的映射对象或函数。

### 7.8 哪些结构只存在于构建期，哪些被翻译进 runtime

| 结构 | 存在域 | 是否进入产物 | 说明 |
|------|--------|-------------|------|
| `AsyncDependenciesBlock` | 构建期 compiler 域 | 否 | 仅构建期数据结构，seal 后通过 `chunkGraph.getBlockChunkGroup(block)` 关联到 ChunkGroup，但 block 对象本身不序列化到产物 |
| `ModuleGraph` / `ModuleGraphConnection` | 构建期 | 否 | 模块依赖关系图，仅用于 chunk 分配、tree-shaking、codegen 决策；产物中不存在 |
| `ChunkGraph` | 构建期 | 否 | chunk 与模块的归属关系，用于决定 module id、chunk id、codegen 时查 moduleId/chunkId；产物中只有计算结果（id 数字/字符串） |
| `Compilation.entries` / EntryData | 构建期 | 否 | 入口配置，seal 时转化为 Entrypoint 和初始 chunk |
| Dependency 子类（HarmonyImportSideEffectDependency 等） | 构建期 | 部分 | Dependency 对象本身不进入产物，但其 range 被替换后生成的运行时代码（`__webpack_require__()`、`importVar.foo`）进入产物 |
| `ImportDependency` | 构建期 | 部分 | 不直接出现，但生成的 `__webpack_require__.e(chunkId).then(...)` 进入产物；关联的 AsyncDependenciesBlock 决定了 chunkId |
| `ChunkGroup` / `Entrypoint` | 构建期 | 间接 | 对象不进入产物，但 parent-child 关系决定了 `__webpack_require__.f` 要加载哪些 chunk，chunk 的 id/name/files 被写入 runtime 的 `installedChunks` 初始值和 chunk filename 映射 |
| `Chunk` | 构建期 + runtime 域 | 间接 | Chunk 对象不序列化，但 chunk 的 id 出现在 `installedChunks`、`__webpack_require__.e(id)` 调用、chunk filename 映射中；chunk 的 files 是 emit 的目标文件名 |
| `Module.id`（经 chunkGraph.getModuleId） | 构建期计算 | 是 | codegen 时 `chunkGraph.getModuleId(module)` 返回的数字/字符串直接内联到 `__webpack_require__("id")` 调用中 |
| RuntimeModule 生成的代码 | 构建期生成 → runtime 域执行 | 是 | EnsureChunkRuntimeModule、LoadScriptRuntimeModule、JsonpChunkLoadingRuntimeModule 等的 `generate()` 返回的字符串被拼入入口 chunk，在浏览器中执行 |
| `runtimeRequirements` Set | 构建期 | 否 | 仅决定哪些 RuntimeModule 被加入 chunk，不直接出现；产物中是它们生成的具体代码 |
| `CodeGenerationResults` | 构建期 | 否 | 缓存各模块各 runtime 的 codegen 结果，渲染 chunk 时取出 source 拼接 |
| emitted chunk 文件 | runtime 域 | 是 | 异步 chunk 作为独立 .js 文件被 emit，浏览器通过 `<script>` 加载；其内部是 `webpackChunk.push([[chunkId], modules])` 格式 |

### 7.9 浏览器执行时的装载时序

回到场景，产物通常为：

- `a.js`（入口 chunk，含 runtime + entry-a + shared）
- `b.js`（入口 chunk，含 entry-b + shared，若独立 runtime 则也有一套 runtime）
- `[id].js`（异步 chunk，含 async 模块）

浏览器加载 `a.js` 后：

1. webpack runtime bootstrap 执行：定义 `__webpack_require__`、`installedChunks`（标记 chunk-a 为 `0` 已加载）、`__webpack_require__.f.j`、`__webpack_require__.l`、`__webpack_require__.u`、`__webpack_require__.p`（publicPath）等。
2. `__webpack_require__` 入口模块 entry-a，执行其模块函数：
   - `var shared__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__("shared-id")` —— 同步执行 shared 模块（已在同一 chunk，直接从 module cache 取或执行）。
   - `__webpack_require__.e("async-chunk-id").then(__webpack_require__.bind(null, "async-id")).then(m => m.bar())`：
     a. `__webpack_require__.e` 调用 `Object.keys(__webpack_require__.f).reduce(...)`，触发 `.j` handler。
     b. `.j` 在 `installedChunks` 中查 async chunk id，未找到则创建 Promise，存入 `installedChunks[chunkId] = [resolve, reject, promise]`。
     c. 计算 URL：`__webpack_require__.p + __webpack_require__.u(chunkId)`。
     d. 调用 `__webpack_require__.l(url, ...)` → `loadScript` 创建 `<script src=url>`，append 到 head。
     e. 浏览器异步请求并执行 async chunk 文件。
     f. async chunk 文件执行 `window.webpackChunk.push([[chunkId], { "async-id": (module) => { ... } }])`。
     g. webpack runtime 的 push handler 把模块注册到 module factories，标记 `installedChunks[chunkId] = 0`，resolve promise。
     h. `.then(__webpack_require__.bind(null, "async-id"))` 执行 async 模块函数，拿到 namespace。
     i. `.then(m => m.bar())` 调用 `bar()`。

静态依赖（shared）在步骤 2 的 `__webpack_require__` 中同步完成，因为 shared 和 entry-a 在同一个 chunk 里。如果 async 模块也引用了 shared，由于 shared 已在 chunk-a 中且 async chunk 不含 shared，`__webpack_require__("shared-id")` 会直接从 webpack module cache 中获取已执行的 shared 模块——这正是 buildChunkGraph 阶段 `minAvailableModules` 跳过 shared 打入 async chunk 的运行时对应。

### 7.10 关键源码依据索引

- Parser 翻译静态 import：[HarmonyImportDependencyParserPlugin.js#L109-L219](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L109-L219)
- Parser 翻译动态 import()：[ImportParserPlugin.js#L262-L301](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportParserPlugin.js#L262-L301)
- AsyncDependenciesBlock 定义：[AsyncDependenciesBlock.js#L24-L41](file:///e:/newGsb/questions/GSB-013/Steve/lib/AsyncDependenciesBlock.js#L24-L41)
- Dependency → ModuleGraph 映射：[Compilation.js#L2038-L2047](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L2038-L2047)
- ChunkGraph 创建：[Compilation.js#L3063-L3067](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3063-L3067)
- buildChunkGraph 入口与两阶段：[buildChunkGraph.js#L1301-L1357](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L1301-L1357)
- processBlock 模块分配与 minAvailableModules 跳过：[buildChunkGraph.js#L675-L761](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L675-L761)
- iteratorBlock 创建异步 chunk group：[buildChunkGraph.js#L488-L669](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L488-L669)
- addChunkInGroup 复用/创建：[Compilation.js#L3897-L3935](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3897-L3935)
- connectChunkGroups 父子连接：[buildChunkGraph.js#L1216-L1273](file:///e:/newGsb/questions/GSB-013/Steve/lib/buildChunkGraph.js#L1216-L1273)
- 静态 import codegen：[RuntimeTemplate.importStatement](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L790-L853)、[HarmonyImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/HarmonyImportDependency.js#L270-L370)
- 动态 import codegen：[ImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Steve/lib/dependencies/ImportDependency.js#L107-L137)、[RuntimeTemplate.blockPromise](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L986-L1042)、[RuntimeTemplate.moduleNamespacePromise](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimeTemplate.js#L603-L750)
- Runtime requirements 处理：[Compilation.processRuntimeRequirements](file:///e:/newGsb/questions/GSB-013/Steve/lib/Compilation.js#L3708)
- RuntimePlugin 注册 RuntimeModule：[RuntimePlugin.js#L111-L495](file:///e:/newGsb/questions/GSB-013/Steve/lib/RuntimePlugin.js#L111-L495)
- EnsureChunkRuntimeModule：[EnsureChunkRuntimeModule.js#L26-L65](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/EnsureChunkRuntimeModule.js#L26-L65)
- JsonpChunkLoadingRuntimeModule：[JsonpChunkLoadingRuntimeModule.js#L75-L217](file:///e:/newGsb/questions/GSB-013/Steve/lib/web/JsonpChunkLoadingRuntimeModule.js#L75-L217)
- LoadScriptRuntimeModule：[LoadScriptRuntimeModule.js#L58-L171](file:///e:/newGsb/questions/GSB-013/Steve/lib/runtime/LoadScriptRuntimeModule.js#L58-L171)

---

## 8. 待深入与未证实项

- MultiCompiler 的并发/依赖调度未逐行阅读。
- `Compilation.unseal()` 后重 seal 是否新建 ChunkGraph：源码显示 seal 开头无条件 `new ChunkGraph`，但 unseal 未把 `this.chunkGraph` 置空，第二次 seal 会覆盖旧引用；旧 ChunkGraph 的 WeakMap 条目是否被显式清理未证实。
- NormalModuleFactory 的 `factory.create` 内部 resolver/loader 流水线未在本次展开。
- `processRuntimeRequirements` 后半段 chunk/tree 级 runtime requirement 汇总与 RuntimeModule 添加的完整顺序未逐行读完（第一版已列出，本次仍未展开）。
- MemoryWithGcCachePlugin、PackFileCacheStrategy 的序列化格式与 GC 策略未展开。
- `compilation.hooks.thisCompilation` 与 `compilation` 在子编译器场景下的触发差异未在 createChildCompiler 中看到显式区分，需进一步核对。
- webpack 5 中 `__webpack_require__` 主体引导代码的生成位置（MainTemplate vs JavascriptModulesPlugin vs RuntimeModule）未逐行核对，7.7 节中标注为未证实。
- 异步 chunk 文件的 JSONP wrapper（`window["webpackChunk"].push(...)`）由哪个 template/runtime module 生成未在本次展开（应在 JavascriptModulesPlugin 的 renderChunk 或 JsonpTemplatePlugin 中）。
- splitChunks/runtimeChunk 如何改变 7.4 节描述的默认 chunk 分配（共享模块抽取到独立 chunk、runtime 抽取到独立 chunk）未在本次追踪，因为场景设定为默认配置。
- 多入口共享模块在默认配置下被复制到每个入口 chunk 的结论基于 buildChunkGraph 的 minAvailableModules 按 chunkGroup 独立计算逻辑推断，未用实际构建产物验证。
