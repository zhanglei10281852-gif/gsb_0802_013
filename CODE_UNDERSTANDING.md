# CODE_UNDERSTANDING — webpack 构建基础设施（第一版）

> 适用版本：本仓库 `package.json` 声明 `webpack@5.99.9`，工具链为 Yarn 1.22.22、Jest 29、TypeScript 5.8（见 `package.json` 的 `packageManager`/`devDependencies`）。
> 本文所有结论均来自对当前 checkout 源码的逐行阅读；行号引用自读到的当前版本。无法从源码确认的内容统一标注 **未证实**。
> 约定：本文及后续维护讨论中，把整套代码划分为三个执行域——**构建期 compiler 域**、**写入产物的 runtime 域**、**watch / cache 域**（定义见第 1 节）。

---

## 1. 三个执行域（本文术语约定）

| 域                    | 覆盖范围                                                                                                                                                                                                                       | 主要文件                                                                                                |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| 构建期 compiler 域    | 从 `webpack()` 创建 `Compiler`，到一次 `compile()` 内 `Compilation` 完成 `seal()` 的全部对象与 hook 流程：配置规范化、插件应用、make（模块图构建）、seal（chunk 图构建、代码生成、hash、assets 生成）                          | `lib/webpack.js`、`lib/Compiler.js`、`lib/Compilation.js`、`lib/WebpackOptionsApply.js`、`lib/config/*` |
| 写入产物的 runtime 域 | `Compilation` 完成后，把 `compilation.assets` 落到 `outputFileSystem` 的一侧：`Compiler.emitAssets` / `emitRecords` / `readRecords`，以及控制它的 `shouldEmit`、`emit`、`assetEmitted`、`afterEmit`、`needAdditionalPass` 回路 | `lib/Compiler.js`（`emitAssets` 等）                                                                    |
| watch / cache 域      | `Watching` 驱动的增量重建回路（`watchRun`/`invalid`/`watchClose`），与 `lib/Cache.js` 为中心的缓存体系（memory / filesystem 策略、`beginIdle`/`endIdle`/`shutdown` 生命周期）                                                  | `lib/Watching.js`、`lib/Cache.js`、`lib/cache/*`                                                        |

注意术语冲突：webpack 源码内部还有 "runtime" 一词（`RuntimeSpec`、`RuntimeGlobals`、`RuntimeModule`、runtime chunk），那是 **seal 阶段** 产物组织结构的概念，属于构建期 compiler 域；与本文"写入产物的 runtime 域"不是一回事。后文单独提及时会写全名以示区分。

---

## 2. 入口链：`webpack()` → 配置规范化与默认值 → `Compiler` 创建 → 插件应用

### 2.1 公开入口

- `package.json` 的 `main` 是 `lib/index.js`；它用 `mergeExports` 把 `lib/webpack.js` 导出的 `webpack` 函数与一批 lazy getter（`Compiler`、`Compilation`、`Hook` 等）合并成公开 API。
- `bin/webpack.js` 是 CLI shim，负责探测并转发到 `webpack-cli`（仅确认其前半部分的探测逻辑，**转发细节未逐行核对**）。

### 2.2 `webpack(options, callback)`（`lib/webpack.js`）

`webpack` 函数体（`lib/webpack.js:121-193`）分 `create()` 和启动两步：

1. **Schema 校验**（`create()` 内，`lib/webpack.js:128-136`）：先用预编译校验器 `schemas/WebpackOptions.check.js`（`webpackOptionsSchemaCheck`）快速检查；只有它报错时才回退到 `lib/validateSchema.js` 的 `validateSchema`（基于 `schema-utils` 的 `validate`，附带 `DID_YOU_MEAN`/`REMOVED` 友好提示，见 `lib/validateSchema.js:77`），并触发 `DEP_WEBPACK_PRE_COMPILED_SCHEMA_INVALID` 弃用警告。
2. **创建编译器**：
   - `options` 为数组 → `createMultiCompiler`（`lib/webpack.js:44-58`）：逐项 `createCompiler(options, index)`，包进 `MultiCompiler`，再按各子配置的 `dependencies` 调 `compiler.setDependencies` 建立编译顺序约束。
   - 单对象 → `createCompiler(webpackOptions)`。
3. **启动**（有 `callback` 时，`lib/webpack.js:160-176`）：任一配置 `watch: true` → `compiler.watch(watchOptions, callback)`；否则 `compiler.run(...)`，在回调里**先 `compiler.close()` 再把 `err || err2` 交给用户 callback**（即 `webpack()` 的单次构建总是"run 完即 close"）。
4. 无 `callback`：只创建并返回 compiler；若配了 `watch` 则发 `DEP_WEBPACK_WATCH_WITHOUT_CALLBACK` 弃用警告（`lib/webpack.js:181-190`）。
5. `create()` 抛错时：经 `process.nextTick` 把错交给 callback 并返回 `null`（`lib/webpack.js:177-180`）。

### 2.3 `createCompiler`（`lib/webpack.js:65-97`）——顺序固定，共 6 步

1. **`getNormalizedWebpackOptions(rawOptions)`**（`lib/config/normalization.js:127`）：把用户配置克隆成 `WebpackOptionsNormalized` 结构（逐节深拷贝，`cloneObject`/`nestedConfig` 等辅助）。要点：`entry` 为函数时被包装成 `Promise.resolve().then(fn).then(getNormalizedEntryStatic)`（`lib/config/normalization.js:187`），即动态 entry 在此被规范化成"每次调用返回静态 entry 的 Promise"。
2. **`applyWebpackOptionsBaseDefaults(options)`**（`lib/config/defaults.js:162`）：只补两项基础默认——`context` 默认 `process.cwd()`、`infrastructureLogging` 默认值。**此时 `Compiler` 还不存在**。
3. **`new Compiler(options.context, options)`**（`lib/Compiler.js:142`）：
   - 构造函数冻结创建全部 compiler hooks（`lib/Compiler.js:143-217`，完整清单见 3.7 节）；
   - 建 `this.resolverFactory = new ResolverFactory()`（`:266`）、`this.cache = new Cache()`（`:287`，此时只是 hook 中枢，策略后挂）；
   - 文件系统字段（`inputFileSystem` 等）先置 `null`。
4. **`new NodeEnvironmentPlugin({ infrastructureLogging }).apply(compiler)`**（`lib/webpack.js:72-74`，实现 `lib/node/NodeEnvironmentPlugin.js:38-69`）：
   - `compiler.inputFileSystem = new CachedInputFileSystem(fs, 60000)`（enhanced-resolve 的缓存封装，60s）；
   - `compiler.outputFileSystem = fs`、`compiler.intermediateFileSystem = fs`（graceful-fs）；
   - `compiler.watchFileSystem = new NodeWatchFileSystem(inputFileSystem)`；
   - 创建 `infrastructureLogger`；并 tap `compiler.hooks.beforeRun`：每次 run 前 `inputFileSystem.purge()` 清读缓存（`NodeEnvironmentPlugin.js:60-68`）。
5. **应用用户插件**（`lib/webpack.js:75-84`）：`plugins` 数组中函数项以 `plugin.call(compiler, compiler)` 调用，对象项调 `plugin.apply(compiler)`。注意时机：**用户插件先于 `applyWebpackOptionsDefaults` 和 `WebpackOptionsApply` 执行**——所以用户插件能读到的是"已规范化 + 基础默认"的配置，`output` 等完整默认值此时尚未补齐。
6. **`applyWebpackOptionsDefaults(options, compilerIndex)`**（`lib/config/defaults.js:172`）：补全所有默认值——`target` 经 `lib/config/target.js`（`getDefaultTarget`/`getTargetProperties`/`getTargetsProperties`）解析出 `targetProperties`；按 `mode` 判定 `development`/`production`；静态 entry 的 `import` 默认 `["./src"]`（`:193-201`）；`devtool`（development 默认 `"eval"`）、`cache`（development 默认 `{ type: "memory" }`，否则 `false`，`:220-222`）；再依次 `applyExperimentsDefaults` → `applyCacheDefaults` → `applySnapshotDefaults` → `applyOutputDefaults` → `applyModuleDefaults` → `applyExternalsPresetsDefaults` → `applyLoaderDefaults` → `externalsType` → `applyNodeDefaults` → `performance` → `applyOptimizationDefaults` → `resolve`/`resolveLoader`（`cleverMerge` 合并默认值与用户值，`:332-348`）。默认值写入工具是局部 helper `D`/`F`/`A`（`lib/config/defaults.js:96/111/129`）。返回 `{ platform }`，随后赋给 `compiler.platform`（`lib/webpack.js:89-91`）。
7. **环境与内置插件**（`lib/webpack.js:92-95`）：
   - `compiler.hooks.environment.call()` → `compiler.hooks.afterEnvironment.call()`（均为无参 `SyncHook`）；
   - `new WebpackOptionsApply().process(options, compiler)`（见 2.4）；
   - `compiler.hooks.initialize.call()`，返回 compiler。

### 2.4 `WebpackOptionsApply.process`（`lib/WebpackOptionsApply.js:78`）——内置插件装配

顺序要点（只列主链路上关键的）：

1. 回写 `compiler.outputPath = options.output.path`、`recordsInputPath/recordsOutputPath`、`compiler.name`（`:79-82`）。
2. 按配置条件应用：`ExternalsPlugin`、各 `externalsPresets` 目标插件（`NodeTargetPlugin` 等）、devtool 插件（`SourceMapDevToolPlugin` / `EvalDevToolModulePlugin`）。
3. 无条件应用模块种类插件：`JavascriptModulesPlugin`、`JsonModulesPlugin`、`AssetModulesPlugin`（`:310-312`）；再按 `experiments.*` 应用 wasm/css/lazyCompilation/buildHttp 插件。
4. **入口接线**（`:391-396`）：`new EntryOptionPlugin().apply(compiler)` 后立刻 `compiler.hooks.entryOption.call(options.context, options.entry)`。
   - `entryOption` 是 `SyncBailHook`（`lib/Compiler.js:216`）；`EntryOptionPlugin` 的 tap **恒返回 `true`**（`lib/EntryOptionPlugin.js:21-24`）→ bail 短路，后续 entryOption tap 不再执行。
   - 静态 entry：`EntryOptionPlugin.applyEntryOption` 为每个 entry 的每个 `import` 各 `new EntryPlugin(context, entry, options).apply(compiler)`（`lib/EntryOptionPlugin.js:33-54`）；函数 entry → `DynamicEntryPlugin`。
5. `RuntimePlugin`、依赖语法插件族（`HarmonyModulesPlugin`、`CommonJsPlugin`、AMD、`NodeStuffPlugin`、`APIPlugin` 等）、`DefaultStatsFactoryPlugin`/`DefaultStatsPresetPlugin`/`DefaultStatsPrinterPlugin`、按 `optimization.*` 的一串优化插件。
6. **缓存策略装配**（`:654-752`）：`cache.type === "memory"` → `MemoryCachePlugin`（或带 GC 代数上限的 `MemoryWithGcCachePlugin`）；`"filesystem"` → `AddBuildDependenciesPlugin` + 内存层插件 + `IdleFileCachePlugin(new PackFileCacheStrategy({...}))`（`:709-741`）。末尾 `new ResolverCachePlugin().apply(compiler)`（`:753`）。
7. `compiler.hooks.afterPlugins.call(compiler)`（`:760`）；断言 `inputFileSystem` 非空；用 `cleverMerge` 把 `options.resolve`/`resolveLoader` 合入 `resolverFactory` 的 `normal`/`context`/`loader` 三类 `resolveOptions` 并注入 `fileSystem`（`:764-791`）；最后 `compiler.hooks.afterResolvers.call(compiler)`（`:792`）。

### 2.5 `MultiCompiler` 分支

`MultiCompiler.run`（`lib/MultiCompiler.js:616-635`）经 `_runGraph`/`runWithDependencies`（`:330`）按 `dependencies` 约束调度：就绪的子 compiler 并行 `compiler.run(callback)`（`asyncLib.map`）。`MultiCompiler.watch`（`:580-610`）同理逐个 `compiler.watch` 并组合成 `MultiWatching`。`_runGraph`/`validateDependencies` 的内部调度算法**未逐行核对**。

---

## 3. 构建期 compiler 域：一次非 watch 构建主链路

### 3.1 `Compiler.run`（`lib/Compiler.js:474-608`）

- 重入保护：`this.running` 为真则回调 `ConcurrentCompilationError`。
- 若 `this.idle`（上一轮已 idle）：先 `this.cache.endIdle(...)` 再继续（`:598-607`）——这是与 cache 域的交接点。
- 主序列（`run()` 内部，`:582-596`）：
  1. `hooks.beforeRun.callAsync(this)` →
  2. `hooks.run.callAsync(this)` →
  3. `this.readRecords(...)`（读 records JSON，`:1097`）→
  4. `this.compile(onCompiled)`。
- `onCompiled`（`:509-580`）：
  - `hooks.shouldEmit.call(compilation) === false`（`SyncBailHook` 短路）：跳过整个 emit，直接 `new Stats(compilation)` + `hooks.done.callAsync` → `finalCallback(null, stats)`。
  - 否则 `process.nextTick` 内 `this.emitAssets(compilation, ...)`（第 4 节）→ `compilation.hooks.needAdditionalPass.call()`（`SyncBailHook`）为真：置 `compilation.needAdditionalPass = true`，先走完本轮 `done` hook，再 `hooks.additionalPass.callAsync` → **`this.compile(onCompiled)` 在同一 `Compiler` 上再来一轮完整编译**（`:533-552`）。
  - 常规路径：`emitRecords` → `new Stats(compilation)` → `hooks.done.callAsync(stats, ...)` → `this.cache.storeBuildDependencies(compilation.buildDependencies, ...)` → `finalCallback(null, stats)`。
- `finalCallback`（`:486-498`）：`cache.beginIdle()` + `idle = true` + `running = false`；有错先 `hooks.failed.call(err)`；调用户 callback；最后 `hooks.afterDone.call(stats)`。
- `webpack()` 包装层随后 `compiler.close(...)`（`lib/Compiler.js:1361-1378`）：若仍有 watching 先关；`hooks.shutdown.callAsync` → 清 `_lastCompilation`/`_lastNormalModuleFactory` 引用 → `cache.shutdown(callback)`。

### 3.2 `Compiler.compile`（`lib/Compiler.js:1310-1355`）——Compilation 的诞生地

1. `newCompilationParams()`（`:1298-1304`）：
   - `createNormalModuleFactory()`（`:1277-1290`）：先 `_cleanupLastNormalModuleFactory()`，再 `new NormalModuleFactory({...})`，`hooks.normalModuleFactory.call(normalModuleFactory)`；
   - `createContextModuleFactory()`（`:1292-1296`）：同理触发 `hooks.contextModuleFactory`。
   - 两个 factory 组成 `CompilationParams`（每次 `compile()` 都新建）。
2. `hooks.beforeCompile.callAsync(params)` → `hooks.compile.call(params)`（同步 `SyncHook`）。
3. `newCompilation(params)`（`:1268-1275`）：
   - `createCompilation`（`:1259-1262`）：先 `_cleanupLastCompilation()`（对上一个 compilation 的所有 module/chunk 调 `ChunkGraph.clearChunkGraphForModule` / `ModuleGraph.clearModuleGraphForModule` 清掉 back-compat 反查表，`:421-444`），然后 `new Compilation(this, params)` 并存入 `this._lastCompilation`；
   - `hooks.thisCompilation.call(compilation, params)` → `hooks.compilation.call(compilation, params)`。**插件挂 compilation 级 hook 的注册点就在这里**——`thisCompilation` 先于 `compilation` 触发，预期 tap 顺序因此稳定。例如 `EntryPlugin` 在 `compilation` hook 里把 `EntryDependency → normalModuleFactory` 写进 `compilation.dependencyFactories`（`lib/EntryPlugin.js:34-42`）。
4. `hooks.make.callAsync(compilation, ...)`（`make` 是 **`AsyncParallelHook`**，`lib/Compiler.js:180`）。
5. `hooks.finishMake.callAsync(compilation, ...)`。注意源码注释类型写的是 `AsyncParallelHook` 但实例是 **`AsyncSeriesHook`**（`lib/Compiler.js:181-182`），以实例为准。
6. `process.nextTick` 边界后：`compilation.finish(err)` → `compilation.seal(err)` → `hooks.afterCompile.callAsync(compilation, ...)` → `callback(null, compilation)` 回到 3.1 的 `onCompiled`。

### 3.3 `make` 阶段：模块图构建（entry → 全图）

- 触发源：`EntryPlugin` 在 `compiler.hooks.make` 上 `tapAsync`（`lib/EntryPlugin.js:47-51`），调 `compilation.addEntry(context, dep, options, cb)`；`DynamicEntryPlugin` 用 `tapPromise`（`lib/DynamicEntryPlugin.js:46`）。因 `make` 是 `AsyncParallelHook`，多个 entry 并行展开。
- `addEntry`（`lib/Compilation.js:2320`）→ `_addEntryItem`（`:2355`）：把依赖记入 `this.entries`（具名）或 `this.globalEntry`（无名），冲突的 entry options 直接报错；`hooks.addEntry.call(entry, options)` → `addModuleTree(...)`；完成后按结果触发 `hooks.succeedEntry` / `hooks.failedEntry`。
- `addModuleTree`（`:2269`）：按 `dependency.constructor` 从 `this.dependencyFactories` 取 `ModuleFactory`（取不到报 "No dependency factory available"），转 `handleModuleCreation`。
- `handleModuleCreation`（`:1935`）五步管线，全部由构造时创建的 5 条 `AsyncQueue` 承载（`:1064-1093`，父子关系：`factorizeQueue` ← `addModuleQueue` ← `processDependenciesQueue`，另有 `buildQueue`、`rebuildQueue`）：
  1. **factorize**：`factorizeModule` → `factorizeQueue` → `_factorizeModule`（`:2170`）→ `factory.create(...)`。`NormalModuleFactory.create`（`lib/NormalModuleFactory.js:869-953`）：`hooks.beforeResolve`（bail 返回 `false` 即忽略该依赖）→ `hooks.factorize`（`AsyncSeriesBailHook`；其内部 stage 100 的 tap 串起 `resolve` → `afterResolve` → `createModule` → `module`，`:340-418`；`resolve` 的 stage 100 tap 做 loader 与资源解析，`:419`）→ 产出 `Module`（默认 `new NormalModule`）。解析失败包装成 `ModuleNotFoundError`。
  2. **add**：`addModule` → `addModuleQueue`（以 `module.identifier()` 为 key 去重）→ `_addModule`（`:1419`）：先 `_modulesCache.get(identifier)` 尝试从缓存恢复模块（`updateCacheModule`），写入 `this._modules`/`this.modules`；`_backCompat` 时 `ModuleGraph.setModuleGraphForModule(module, this.moduleGraph)`。
  3. **接线**：`moduleGraph.setResolvedModule(originModule, dep, module)`、`setIssuerIfUnset`（`:2040-2052`）。
  4. **build**：`_handleModuleBuildAndDependencies`（`:2083`）→ `buildQueue` → `_buildModule`（`:1492`）：`module.needBuild(...)` 为假 → `hooks.stillValidModule.call(module)` 直接跳过；否则 `hooks.buildModule.call(module)` → `module.build(options, compilation, resolver, inputFileSystem, ...)` → 按结果 `hooks.succeedModule` / `hooks.failedModule`（含循环构建检测，报 `BuildCycleError`，`:2090-2130`）。
  5. **processDependencies**：`processModuleDependencies` → `processDependenciesQueue` → `_processModuleDependencies`（`:1593`）：对每个依赖 `moduleGraph.setParents` 后按 factory 分组，再对每组递归 `handleModuleCreation`——图就此滚雪球式展开。`options.bail` 时停止全部队列（`addModuleTree` 回调，`:2298-2303`）。
- 并行度：`options.parallelism`（默认 100，`lib/config/defaults.js` 中 `D(options, "parallelism", 100)`）。

### 3.4 `Compilation.finish`（`lib/Compilation.js:2782-3029`）

profile 汇总 → `_computeAffectedModules` → `hooks.finishModules.callAsync(modules)` → `moduleGraph.freeze("dependency errors")` 下逐模块 `reportDependencyErrorsAndWarnings` 汇总进 `this.errors`/`this.warnings` → `moduleGraph.unfreeze()`。

### 3.5 `Compilation.seal`（`lib/Compilation.js:3050-3428`）——固定序列

0. `finalCallback` 先清空 5 条队列（`:3055-3062`）。
1. **创建 `ChunkGraph`**：`this.chunkGraph = new ChunkGraph(this.moduleGraph, this.outputOptions.hashFunction)`（`:3063-3067`）；`_backCompat` 时给所有已有 module 设 `ChunkGraph.setChunkGraphForModule`。
2. `hooks.seal.call()`。
3. `while (hooks.optimizeDependencies.call(this.modules)) {}`（`SyncBailHook`：任何 tap 返回 truthy 就整体再来一轮）→ `hooks.afterOptimizeDependencies.call`。
4. **chunk 图构建**：`hooks.beforeChunks.call()` → `moduleGraph.freeze("seal")` → 遍历 `this.entries` 为每个入口 `addChunk(name)` + `new Entrypoint(options)`，`connectChunkGroupAndChunk`、`chunkGraph.connectChunkAndEntryModule(chunk, module, entrypoint)`、`assignDepths`；处理 `dependOn`/`runtime` 的合法性校验与 runtime chunk 复用（`:3088-3226`）→ **`buildChunkGraph(this, chunkGraphInit)`**（`lib/Compilation.js:3227`，实现 `lib/buildChunkGraph.js:1301`；内部 `extractBlockModules`（`:108`）/`visitModules`（`:247`）/`connectChunkGroups`（`:1216`），把模块按 block/连接状态分配到 chunk）→ `hooks.afterChunks.call(this.chunks)`。
5. **优化 hook 链**（`:3231-3309`）：`optimize` → `while (optimizeModules.call(modules))` → `afterOptimizeModules` → `while (optimizeChunks.call(chunks, chunkGroups))` → `afterOptimizeChunks` → `optimizeTree.callAsync`（异步）→ `afterOptimizeTree` → `optimizeChunkModules.callAsync` → `afterOptimizeChunkModules`。
6. **ID 分配**：`shouldRecord = hooks.shouldRecord.call() !== false` → `reviveModules` → `beforeModuleIds` → `moduleIds` → `optimizeModuleIds` → `afterOptimizeModuleIds` → `reviveChunks` → `beforeChunkIds` → `chunkIds` → `optimizeChunkIds` → `afterOptimizeChunkIds` → `assignRuntimeIds()` → `_computeAffectedModulesWithChunkGraph()` → `sortItemsWithChunkIds()`（顺带排序 errors/warnings，`:4193-4201`）→ `shouldRecord` 时 `recordModules`/`recordChunks` → `optimizeCodeGeneration.call(modules)`。
7. **模块 hash**：`beforeModuleHash` → `createModuleHashes()`（`:4221`：逐 module × 其所属 runtime 调 `module.updateHash`，`chunkGraph.setModuleHashes`；可用 `moduleMemCaches2` 命中）→ `afterModuleHash`。
8. **代码生成**：`beforeCodeGeneration` → `codeGeneration(callback)`（`:3468`：`this.codeGenerationResults = new CodeGenerationResults(...)`（`:3470`）；按 module × runtime 组装 jobs，同 hash 的 runtime 合并 → `_runCodeGenerationJobs`（`:3508`，`asyncLib.eachLimit(jobs, options.parallelism)`；处理 `codeGenerationDependencies` 导致的延迟轮次）→ `_codeGenerationModule`（`:3622`：先查 `_codeGenerationCache`（key 含模块 hash 与 dependencyTemplates hash），未命中才 `module.codeGeneration({...})`，结果写进 `codeGenerationResults`）→ `afterCodeGeneration`。
9. **运行时需求**：`beforeRuntimeRequirements` → `processRuntimeRequirements()`（`:3708`：先 module 级（`additionalModuleRuntimeRequirements` 等），再 chunk 级、再 tree 级（entry/async entrypoint 的 runtime chunk）；`runtimeRequirementInTree` / `runtimeRequirementInChunk` 是 `HookMap<SyncBailHook>`（`:793/:813`）；需要运行时模块时经 `addRuntimeModule(chunk, module)`（`:3843`）接入 chunk graph 并触发 `hooks.runtimeModule.call(module, chunk)`（`:3886`））→ `afterRuntimeRequirements`。
10. **整体 hash**：`beforeHash` → `createHash()`（`:4321`：`fullHash`/`chunkHash`/`contentHash` hooks 参与计算；返回因 runtime 模块等需要重新生成代码的 jobs）→ `afterHash` → `_runCodeGenerationJobs(codeGenerationJobs)` 补跑 → `shouldRecord` 时 `recordHash`。
11. **模块自有 assets**：`clearAssets()` → `beforeModuleAssets` → `createModuleAssets()`（`:4877`：遍历各 module 的 `buildInfo.assets` 经 `getPath` 落名后 `emitAsset`）。
12. **chunk assets**：`hooks.shouldGenerateChunkAssets.call() !== false`（`SyncBailHook`，`false` 才跳过）→ `beforeChunkAssets` → `createChunkAssets(callback)`（`:4914`：每 chunk `getRenderManifest(...)` → `hooks.renderManifest.call([], options)`（`SyncWaterfallHook`，`:4906-4908`；`JavascriptModulesPlugin` 在 `compilation.hooks.renderManifest` 上 tap 产出渲染项，`lib/javascript/JavascriptModulesPlugin.js:307`）→ 逐文件：命中 `_assetsCache` 复用，否则 `fileManifest.render()` 并用 `CachedSource` 包装 → `emitAsset(file, source, assetInfo)`（`:4611`，同名不同内容即报 "Conflict" 错误）→ `chunk.files`/`auxiliaryFiles` 记账 → `hooks.chunkAsset.call(chunk, file)`（`:5043`））。
13. **processAssets 管线**：`hooks.processAssets.callAsync(this.assets)`（实例是 `AsyncSeriesHook`，`:494`；构造时对其 `intercept`（`:513-644`）实现：按 tap 声明的 `stage` 排序执行，阶段常量为 `Compilation.PROCESS_ASSETS_STAGE_*`（如 `ADDITIONAL = -2000`（`:5646`）、`PRE_PROCESS = -1000`（`:5651`）、`SUMMARIZE = 1000`（`:5702`）、`OPTIMIZE_HASH = 2500`（`:5707`）、`OPTIMIZE_TRANSFER = 3000`（`:5712`））；每个 tap 执行后 `popNewAssets` 把新增 assets 交给 `processAdditionalAssets`；`additionalAssets` 选项的 tap 会被改写到 `processAdditionalAssets` 上）→ `afterProcessAssets` → **冻结 `this.assets`**（`_backCompat` 下是 `soonFrozenObjectDeprecation` 警告包装，否则 `Object.freeze`，`:3369-3382`）。
14. `summarizeDependencies()`（`:4203`：汇总子 compilation 与各 module 的 `fileDependencies`/`contextDependencies`/`missingDependencies`/`buildDependencies`——watch 与 cache 域依赖这些数据）→ `shouldRecord` 时 `record`。
15. `hooks.needAdditionalSeal.call()`（`SyncBailHook`）为真 → `unseal()`（`:3031`：清 chunks/chunkGroups/assets、moduleGraph 解冻）→ **`this.seal(callback)` 在同一 `Compilation` 上整体重跑**（`:3393-3396`）。
16. `hooks.afterSeal.callAsync` → `fileSystemInfo.logStatistics()` → 回到 `compile()` 的 `afterCompile`。

### 3.6 关键对象的创建者 / 持有者 / 可用时机

| 对象                                                                | 由谁创建                                                                                    | 谁持有                                                                                        | 何时可用 / 生命周期                                                                                                                          |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Compiler`                                                          | `createCompiler`（`lib/webpack.js:65`），或 `createChildCompiler`（`lib/Compiler.js:1166`） | 调用方 / `MultiCompiler.compilers`；`compiler.root` 指向根 compiler                           | `createCompiler` 全程同步返回；直到 `close()` 后 `cache.shutdown`                                                                            |
| `MultiCompiler`                                                     | `createMultiCompiler`（`lib/webpack.js:44`）                                                | 调用方                                                                                        | 聚合多个 `Compiler`，按 `dependencies` 调度                                                                                                  |
| `Compilation`                                                       | `Compiler.createCompilation`（`lib/Compiler.js:1259`）于**每次 `compile()`**                | `Compiler._lastCompilation`；`compilation.compiler` 回指                                      | `newCompilation` 内先触发 `thisCompilation`/`compilation` hooks；`afterCompile` 后交给 emit；下一轮 `compile()` 时 `_cleanupLastCompilation` |
| `CompilationParams`（`normalModuleFactory`/`contextModuleFactory`） | `Compiler.newCompilationParams`（`:1298`）每次 `compile()` 新建                             | 传入 `beforeCompile`/`compile`/`thisCompilation`/`compilation` hooks；存 `compilation.params` | 仅在本次编译内有效                                                                                                                           |
| `ModuleGraph`                                                       | `Compilation` 构造函数 `new ModuleGraph()`（`lib/Compilation.js:1057`）                     | `compilation.moduleGraph`                                                                     | 随 Compilation 创建即存在，make 期间被填充；`finish`/seal 中被 `freeze`/`unfreeze`；seal 起只读倾向（rebuild 时 `unfreeze`）                 |
| `ChunkGraph`                                                        | **`seal()` 开头** `new ChunkGraph(moduleGraph, hashFunction)`（`lib/Compilation.js:3063`）  | `compilation.chunkGraph`                                                                      | **seal 之前为 `undefined`**（构造时显式置 `undefined`，`:1059`）；make 阶段插件不能依赖它                                                    |
| `CodeGenerationResults`                                             | `codeGeneration()` 内 `new CodeGenerationResults(...)`（`lib/Compilation.js:3470`）         | `compilation.codeGenerationResults`                                                           | seal 第 8 步之后可用                                                                                                                         |
| `ResolverFactory`                                                   | `Compiler` 构造函数（`lib/Compiler.js:266`）                                                | `compiler.resolverFactory`（子 compiler 共享，`:1185`）                                       | `afterResolvers` 后配置完整；`resolverFactory.get("normal"\|"context"\|"loader")`                                                            |
| `Cache`                                                             | `Compiler` 构造函数 `new Cache()`（`lib/Compiler.js:287`）                                  | `compiler.cache`（子 compiler 共享，`:1191`）                                                 | 只是 hook 中枢；策略插件在 `WebpackOptionsApply` 阶段挂接；`getCache(name)` 产生 `CacheFacade`                                               |
| `Stats`                                                             | `new Stats(compilation)`：`run`/`Watching._done` 内                                         | 回调链                                                                                        | `done` hook 的参数                                                                                                                           |
| `Watching`                                                          | `Compiler.watch`（`lib/Compiler.js:466`）                                                   | `compiler.watching`                                                                           | 见第 5 节                                                                                                                                    |

补充：`_backCompat`（`experiments.backCompat !== false`，`lib/Compiler.js:303`）时，`ModuleGraph`/`ChunkGraph` 还会通过模块级静态 WeakMap 反查（`ModuleGraph.setModuleGraphForModule`（`lib/ModuleGraph.js:884`）、`ChunkGraph.setChunkGraphForModule`（`lib/ChunkGraph.js:1791`）），供老的 `module.moduleGraph`/`chunk.chunkGraph` 访问器使用；清理在 `_cleanupLastCompilation`。

### 3.7 `Compiler` hooks 全清单（`lib/Compiler.js:143-217`）

- 生命周期：`initialize`、`beforeRun`、`run`、`watchRun`、`failed`、`invalid`、`watchClose`、`shutdown`、`done`、`afterDone`、`additionalPass`。
- 编译：`beforeCompile`、`compile`、`thisCompilation`、`compilation`、`make`（`AsyncParallelHook`）、`finishMake`（实例为 `AsyncSeriesHook`）、`afterCompile`。
- emit：`shouldEmit`（`SyncBailHook`，`false` 短路）、`emit`、`assetEmitted`、`afterEmit`。
- 基础设施：`environment`、`afterEnvironment`、`afterPlugins`、`afterResolvers`、`entryOption`（`SyncBailHook`）、`normalModuleFactory`、`contextModuleFactory`、`readRecords`、`emitRecords`、`infrastructureLog`（`SyncBailHook`，返回非 `undefined` 即拦截日志，见 `lib/Compiler.js:360-364`）。

`Compilation` hooks 定义于 `lib/Compilation.js:709-993`，关键者已在 3.3–3.5 节按触发顺序列出。

---

## 4. 写入产物的 runtime 域（emit 侧）

### 4.1 `Compiler.emitAssets`（`lib/Compiler.js:675-1020`）

调用时机：`run` 的 `onCompiled` 与 `Watching` 的 `onCompiled` 中，`shouldEmit` 未短路时，经 `process.nextTick` 进入。

1. `hooks.emit.callAsync(compilation)`（`:1011`）——插件最后追加/修改 assets 的异步窗口。
2. `outputPath = compilation.getPath(this.outputPath, {})`；`mkdirp` 输出目录。
3. `emitFiles`：`asyncLib.forEachLimit(assets, 15, ...)` 并发 15 路写文件（`:692`）。每个 asset：
   - 处理 `?` query 后缀与 `immutable` 判定（hash 是否在文件名中）；
   - 目录含 `/` 或 `\` 先 `mkdirp`；
   - 去重与跳过逻辑，三份状态全在 `Compiler` 上（跨多次编译存活）：`_assetEmittingWrittenFiles`（目标路径 → 写入代数）、`_assetEmittingSourceCache`（`WeakMap<Source, CacheEntry>`）、`_assetEmittingPreviousFiles`（上一轮写出的路径集）；
   - `output.compareBeforeEmit` 为真且目标已存在：尺寸相同再读盘比对内容，一致则跳过写入（保持 mtime，避免触发 watcher）（`:892-923`）；
   - `outputFileSystem.writeFile(targetPath, content, ...)` → `compilation.emittedAssets.add(file)` → `hooks.assetEmitted.callAsync(file, { content, source, outputPath, compilation, targetPath })`（`:831`）；
   - 大小写不敏感冲突检测（`caseInsensitiveMap`），内容相同可共享，不同则报错（`:743-779`）；
   - 写完后把 compilation 里的 asset 替换成 `SizeOnlySource`（`updateAsset`），释放 `Source` 内存（`:848-886`）。
4. 全部完成 → `hooks.afterEmit.callAsync(compilation)`（`:1002`）。

### 4.2 records 与 Stats

- `readRecords`（`:1097`）在每次 `compile` 之前（run 域在 `hooks.run` 之后；watch 域在首次 `_go` 时）执行；`emitRecords`（`:1026`）在 emit 之后、`done` 之前，把 `this.records` JSON 写到 `recordsOutputPath`。
- `new Stats(compilation)` 在 `done` hook 前创建；`Stats` 内部走 `StatsFactory`/`StatsPrinter`（由 `DefaultStatsFactoryPlugin` 等在 `WebpackOptionsApply` 中注册）。**Stats 内部管线未逐行核对**。

### 4.3 控制回路（短路 / 多次执行）

- `shouldEmit === false`：整个第 4 节跳过，但仍走 `done`（`lib/Compiler.js:514-523`；watch 域同理 `lib/Watching.js:192-194`）。
- `needAdditionalPass` 为真：本轮 `done` 之后 `additionalPass` → 再次 `compile()`——**同一 `Compiler`、同一 `Cache`，全新 `Compilation`**（`lib/Compiler.js:533-552`）。
- `emit`/`afterEmit` 出错会使整个构建以错误收场（`finalCallback(err)` → `hooks.failed`）。

---

## 5. watch / cache 域

### 5.1 `Watching`（`lib/Watching.js`）

- 进入：`Compiler.watch(watchOptions, handler)`（`lib/Compiler.js:459-468`）：`running = true`、`watchMode = true`、`new Watching(this, watchOptions, handler)`。构造函数里 `process.nextTick(() => this._initial && this._invalidate())` 触发首轮构建（`lib/Watching.js:74-76`）；`aggregateTimeout` 默认 20ms（`:59-61`）。
- **`_go(...)`（`:109-238`）——每轮重建入口**：
  1. 暂停旧 watcher（`pausedWatcher`），把 `fileTimestamps`/`contextTimestamps`/`modifiedFiles`/`removedFiles` 写到 `compiler` 上（cache/快照失效依据）；
  2. 仅首轮 `readRecords`（`_needRecords`）；
  3. **`hooks.watchRun.callAsync(this.compiler)`**——注意：watch 域**不走 `beforeRun`/`run` hooks**，对应物是 `watchRun`；
  4. `compiler.compile(onCompiled)`：与 run 域共享 3.2–3.5 的全部流程；
  5. `onCompiled`：`this.invalid` 为真（构建期间又有文件变化）→ 直接 `_done(null, compilation)`，由 `_done` 立即再 `_go()`（`:190`、`:280-300`）；`shouldEmit === false` → `_done`；否则 `emitAssets` → `emitRecords` → `needAdditionalPass` 分支（与 run 域一致）→ `_done`。
- **`_done(err, compilation)`（`:254-345`）**：出错 → `hooks.failed` + `cache.beginIdle` + handler；成功 → `new Stats` → `hooks.done.callAsync` → `handler(null, stats)` → `cache.storeBuildDependencies` → `cache.beginIdle()`、`compiler.idle = true` → `process.nextTick` 内 `this.watch(fileDependencies, contextDependencies, missingDependencies)` 重新挂起文件监听 → `hooks.afterDone`。**监听的文件集就是上一轮 Compilation 汇总的三类依赖**。
- **`watch(files, dirs, missing)`（`:353-394`）**：`compiler.watchFileSystem.watch(...)`（`NodeWatchFileSystem`，底层 watchpack）。变化回调 → `_invalidate(...)`；invalid 回调 → `hooks.invalid.call(fileName, changeTime)`（每轮只报一次，`_invalidReported` 闸口）。
- **`_invalidate(...)`（`:419-441`）**：suspended/blocked → 只合并收集变更；**正在构建** → 合并变更并置 `this.invalid = true`（当前轮次作废，完成后再来一轮）；空闲 → 直接 `_go()`。
- `invalidate(callback)`（`:400`）：手动触发重建（会触发 `hooks.invalid.call(null, Date.now())`）。
- `close(callback)`（`:458-524`）：关 watcher、`storeBuildDependencies`、`hooks.watchClose.call()`、复位 compiler 的 watch 相关字段。`Compiler.close` 会先递归关闭存活中的 `watching` 再走 `shutdown`（`lib/Compiler.js:1361-1368`）。
- 多次编译与 run 域的差异：同一 `Compiler`/`Cache` 复用；每轮 `compile()` 仍新建 `Compilation`/`ModuleGraph`/`ChunkGraph`（`createCompilation` 先 `_cleanupLastCompilation` 清 back-compat 反查表，`lib/Compiler.js:421-444`）。

### 5.2 `Cache` 体系（`lib/Cache.js` + `lib/cache/*`）

- `Compiler` 构造时 `new Cache()`：本身只定义 hooks——`get`/`store`/`beginIdle`/`endIdle`/`shutdown`/`storeBuildDependencies`（`lib/Cache.js:60-68`），方法仅是转发 `callAsync`（`get :78`、`store :112`、`storeBuildDependencies :127`、`beginIdle :137`、`endIdle :145`、`shutdown :155`）。**真正存数据的是策略插件**（2.4 节第 6 步挂接）。
- `compiler.getCache(name)` → `CacheFacade`（`compilerPath + name` 作前缀，`lib/Compiler.js:331-337`）。Compilation 内建三个命名缓存：`Compilation/modules`（`_modulesCache`，模块恢复）、`Compilation/assets`（`_assetsCache`，chunk 渲染结果）、`Compilation/codeGeneration`（`_codeGenerationCache`）（`lib/Compilation.js:1203-1205`）。
- 生命周期节拍：`beginIdle`（构建结束/出错：`run` 的 `finalCallback`、`Watching._done`）→ 持久化策略（`IdleFileCachePlugin` + `PackFileCacheStrategy`）在 idle 窗口落盘；`endIdle`（下一轮 `run`/`_go` 开头）→ 回到工作状态；`shutdown`（`compiler.close`）。`storeBuildDependencies` 在 `done` 后保存构建依赖，`filesystem` 缓存用它判定缓存可否复用。**策略插件内部的序列化/还原流程未逐行核对**。
- 增量判断的另一翼：`FileSystemInfo`（`Compilation` 构造时创建，`lib/Compilation.js:1007`）+ `snapshot` 配置（`applySnapshotDefaults`）负责文件时间戳/哈希快照；watch 域经 `compiler.modifiedFiles` 等字段把变更喂给这轮 Compilation。

---

## 6. 关键 hook 语义备忘（短路、异步边界、多次执行）

**短路（bail）语义**——容易读错的点：

- `hooks.entryOption`（`SyncBailHook`）：`EntryOptionPlugin` 恒返回 `true` → 内置实现之后，其他 entryOption tap 不执行。
- `hooks.shouldEmit`（`SyncBailHook`）：判定方式是 `=== false`；返回其他 falsy/undefined 不短路。
- `hooks.optimizeDependencies` / `optimizeModules` / `optimizeChunks`（`SyncBailHook`）：`while (hook.call(...)) {}`——**任一 tap 返回 truthy 就整体重跑一轮**，直到没人返回 truthy。
- `hooks.shouldRecord` / `shouldGenerateChunkAssets`（`SyncBailHook`）：判定是 `!== false`，即只有明确返回 `false` 才跳过。
- `compilation.hooks.needAdditionalPass` / `needAdditionalSeal`（`SyncBailHook`）：truthy 即触发"再编译一次" / "unseal 后重 seal"。
- `NormalModuleFactory` 的 `beforeResolve`/`factorize`/`resolve`/`afterResolve`/`createModule`（`AsyncSeriesBailHook`）：返回 `false` 表示忽略该依赖；`resolve` 返回 `Module` 实例直接作为结果短路。
- `runtimeRequirementInTree` / `runtimeRequirementInChunk`：`HookMap<SyncBailHook>`，按 runtime 名字分发。

**异步边界**：

- `compiler.hooks.make` 是 `AsyncParallelHook`：所有 entry（以及任何 tap 了 make 的插件）**并行**展开模块图，共享 5 条 `AsyncQueue` 的并发控制。
- `compile()` 内 `finishMake` 之后经 `process.nextTick` 才 `finish`/`seal`（`lib/Compiler.js:1331`）；`run`/`Watching` 的 `onCompiled` 里 `emitAssets` 前同样隔一个 `process.nextTick`。
- `webpack()` 创建期异常经 `process.nextTick` 异步回 callback（`lib/webpack.js:178`）。
- `Watching`：构造后首轮 `_invalidate`（`:74-76`）与 `_done` 后重挂 watcher（`:328-339`）都在 `process.nextTick`。
- `emitAssets` 写文件并发上限 15；`_runCodeGenerationJobs` 并发为 `options.parallelism`。

**多次编译**：

- `needAdditionalPass`：同一 `Compiler` 再 `compile()`，全新 `Compilation`。
- `needAdditionalSeal`：**同一 `Compilation`** 上 `unseal()` 后重跑 `seal()` 全序列（chunk/ChunkGraph 全部重建）。
- watch 每轮：新 `Compilation`、新 `ModuleGraph`、新 `ChunkGraph`；`Compiler` 与 `Cache` 复用；`emitAssets` 的去重状态（`_assetEmittingWrittenFiles` 等）跨轮保留，保证重复内容不重写。
- `_cleanupLastCompilation` 仅清 back-compat 反查表，不影响 `Stats` 对旧 Compilation 的引用（`close()` 注释，`lib/Compiler.js:1371-1375`）。

**已知源码瑕疵**（阅读时确认，供维护留意）：`finishMake` 的 JSDoc 类型标注（`AsyncParallelHook`）与实例（`AsyncSeriesHook`）不一致（`lib/Compiler.js:180-182`）。

---

## 7. 场景追踪：一条 `import` 如何变成图与浏览器装载逻辑

本章用一个具体场景把 parser → Dependency → `AsyncDependenciesBlock` → `ModuleGraph` → `buildChunkGraph` → `ChunkGraph` → code generation → runtime requirements → `RuntimeModule` 串成一条链。场景（目标 web、默认配置）：

```text
src/a.js      import { shared } from "./shared"; console.log("a", shared);
              import(/* webpackChunkName: "lazy-chunk" */ "./lazy").then(m => m.lazy());
src/b.js      import { shared } from "./shared"; console.log("b", shared);
src/shared.js export const shared = "SHARED";
src/lazy.js   export function lazy() { return "LAZY"; }
配置: entry: { a: "./a.js", b: "./b.js" }, mode: "development"
```

对象所有权沿用 3.6 节；三个执行域沿用第 1 节。本章末尾（7.7）汇总"哪些结构只在构建期、哪些信息被翻译进 emitted runtime"。

### 7.1 parser 阶段（make 内 `module.build` 时）：源码 → `Dependency` / `AsyncDependenciesBlock`

- **静态 `import { shared } from "./shared"`**：`HarmonyModulesPlugin` 在 `compiler.hooks.compilation` 上注册工厂/模板并挂 parser 插件（`lib/dependencies/HarmonyModulesPlugin.js:53-142`）。`HarmonyImportDependencyParserPlugin` 工作：
  - `parser.hooks.import` tap → 每条 import 语句创建一个 `HarmonyImportSideEffectDependency`（`lib/dependencies/HarmonyImportDependencyParserPlugin.js:109-130`）；
  - `parser.hooks.importSpecifier` tap → 每个被使用的说明符（如 `shared`）创建一个 `HarmonyImportSpecifierDependency`（`:134-216`）；
  - 均通过 `parser.state.module.addDependency(...)` 挂到**当前模块的 `module.dependencies`**。
- **动态 `import("./lazy")`**：`ImportPlugin` 把 `ImportDependency → normalModuleFactory` 与 `ImportDependency.Template` 注册进 `compilation.dependencyFactories/dependencyTemplates`（`lib/dependencies/ImportPlugin.js:35-42`），并给三类 js 模块挂 `ImportParserPlugin`。后者 tap `parser.hooks.importCall`（`lib/dependencies/ImportParserPlugin.js:47`）：参数是字符串字面量且 `webpackMode` 为默认 `"lazy"` 时——
  - 创建 `new AsyncDependenciesBlock({ ...groupOptions, name: chunkName }, loc, request)`（`:282-289`；`webpackChunkName` 注释写入 `groupOptions.name`，`:108-118`）；
  - 块内创建唯一的 `new ImportDependency(request, range, exports, attributes)`（`:290-295`），`dep.optional = Boolean(parser.scope.inTry)`（`:297`，try/catch 内的 import 出错只警告）；
  - `depBlock.addDependency(dep)` + `parser.state.current.addBlock(depBlock)`（`:298-299`）→ block 进入 **`module.blocks`**（`DependenciesBlock` 结构，`lib/DependenciesBlock.js`；`AsyncDependenciesBlock` 定义 `lib/AsyncDependenciesBlock.js:24`，持有 `groupOptions`）。
  - 变体：`webpackMode: "eager"` → 直接 `ImportEagerDependency`、**不产生 block、不产生异步 chunk**（`:266-272`）；`"weak"` → `ImportWeakDependency`；非字面量参数 → `ContextDependencyHelpers.create(ImportContextDependency, ...)`（`:306-333`，走 contextModuleFactory）。
- 归属：`Dependency`/`AsyncDependenciesBlock` 归模块所有，随模块在 make 阶段产生；**纯构建期结构**，不会原样出现在产物里。

### 7.2 make 阶段：`Dependency` → `ModuleGraph`

- `Compilation._processModuleDependencies`（`lib/Compilation.js:1593`）用队列遍历模块及其嵌套 blocks（`:1854-1867`）：对 block 里每个 dep 先 `moduleGraph.setParents(dep, currentBlock, module, index)`（`:1672`）——这一步记录 dep 的父 block 与父模块，正是后面 code generation 里 `moduleGraph.getParentBlock(dep)`（`ImportDependency.Template`，7.4）与 `getParentModule` 的数据来源；然后按 factory 分组，逐组递归 `handleModuleCreation`（3.3 节）。
- 每组依赖经 `NormalModuleFactory` 解析、构建后，`moduleGraph.setResolvedModule(originModule, dep, module)` 建立 **`ModuleGraphConnection`**（`lib/ModuleGraphConnection.js`）。
- 本场景结果：`shared.js` 只存在**一个模块实例**（`_addModule` 按 `module.identifier()` 去重，`lib/Compilation.js:1420-1424`），图中有 a→shared、b→shared 两条 connection；a 的 `module.blocks[0]`（AsyncDependenciesBlock）内有 a→lazy 的 connection，且该 dep 的 parent block 指向这个 block。
- 至此一切都还在 `ModuleGraph`（构建期，`Compilation` 构造时创建）里，**`ChunkGraph` 尚不存在**（seal 开头才创建，3.5 节）。

### 7.3 seal 阶段：`buildChunkGraph` → `ChunkGraph`、chunk 的形成与复用条件

- seal 为 a、b 各建 Chunk + `Entrypoint`；无 `dependOn`/`runtime` 时 `entrypoint.setRuntimeChunk(chunk)`（`lib/Compilation.js:3096-3098`），即**入口 chunk 自己就是 runtime chunk**（`Chunk.hasRuntime()` 判定见 `lib/Chunk.js:445-455`）。
- `buildChunkGraph(this, chunkGraphInit)`（`lib/buildChunkGraph.js:1301`）从入口 BFS：
  - 到达模块时 `chunkGraph.connectChunkAndModule(chunk, module)`（`:829`）。`shared` 从 a、b 两个入口分别可达 → **同时被 connect 到 a 的 chunk 和 b 的 chunk**（默认配置下两入口各持一份，见 7.6 的条件讨论）。
  - 到达 `AsyncDependenciesBlock` 时 `iteratorBlock(b)`（`:488`）：
    - 若父 chunk group 的 `chunkLoading`/`asyncChunks` 为 false → **不建异步 chunk**，block 并入当前 chunk group 继续遍历（`:569-578`）；
    - 否则按名字复用或新建：`namedChunkGroups.get(chunkName)` 命中即复用已有 ChunkGroup（`:580`；这就是"多个 `import()` 用同一 `webpackChunkName` 会合并"的依据），未命中则 `compilation.addChunkInGroup(b.groupOptions || b.chunkName, module, b.loc, b.request)`（`:582`，实现 `lib/Compilation.js:3897`）新建 ChunkGroup + Chunk；具名块指向已存在的 **initial** chunk 会报 `AsyncDependencyToInitialChunkError`（`:613-620`）；
    - 父子关系暂存 `blockConnections`，最后 `connectChunkGroupParentAndChild`（`:1267`）并 `chunkGraph.connectBlockAndChunkGroup(block, chunkGroup)`（`:1264`；存储于 `ChunkGraph._blockChunkGroups`，`lib/ChunkGraph.js:1328-1331`）——`getBlockChunkGroup(block)` 是 7.4 生成 `__webpack_require__.e` 的查询入口；
    - 异步块内模块若已在父链可用，经 `minAvailableModules` 位掩码机制跳过（`skippedItems`，`:894-910` 一带），不重复进异步 chunk。
- 本场景产物：`ChunkGraph` 上共有 3 个 chunk——`a`（runtime chunk）、`b`（runtime chunk）、`lazy-chunk`（普通异步 chunk，名字来自 magic comment）；`ChunkGraph`/`Chunk`/`ChunkGroup` 同样是**纯构建期结构**。

### 7.4 code generation：`Dependency` → 运行时代码片段与 runtime requirements

`Compilation.codeGeneration` 对每个模块调 `module.codeGeneration()`，模块再用 `dependencyTemplates` 对每个 Dependency 调 `Template.apply(dep, source, context)`；`context.runtimeRequirements` 是**要被填充的需求集合**（3.5 节第 8 步）。

- **静态 import**：`HarmonyImportDependency.Template.apply`（`lib/dependencies/HarmonyImportDependency.js:279`）→ `getImportStatement` → `runtimeTemplate.importStatement(...)`（`lib/RuntimeTemplate.js:790`）：生成 `var shared__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/* moduleId */)` 之类的声明 + 兼容调用，以 `InitFragment`（`STAGE_HARMONY_IMPORTS`）形式注入模块头部；`HarmonyImportSpecifierDependency.Template.apply`（`lib/dependencies/HarmonyImportSpecifierDependency.js:325-348`）把源码中对 `shared` 的引用范围替换为 importVar 上的属性访问（`propertyAccess(ids)`）。模块 id 由 seal 的 `moduleIds` hook 分配（默认 development `"named"`、production `"deterministic"`，`lib/config/defaults.js:1523-1531`）。
- **动态 import**：`ImportDependency.Template.apply`（`lib/dependencies/ImportDependency.js:116-136`）把 `import(...)` 表达式整体替换为 `runtimeTemplate.moduleNamespacePromise({...})` 的输出：
  - `blockPromise`（`lib/RuntimeTemplate.js:986`）：`chunkGraph.getBlockChunkGroup(block)` 取不到 chunk 或 chunks 为空 → 退化为 `Promise.resolve()`；恰一个可加载 chunk → 生成 `__webpack_require__.e(chunkId)` 并 **`runtimeRequirements.add(RuntimeGlobals.ensureChunk)`**（`:1009`）；多个 chunk → `Promise.all([...__webpack_require__.e(...)])`；
  - 命名空间语义：`.then(__webpack_require__.bind(__webpack_require__, moduleId))` 并 `runtimeRequirements.add(RuntimeGlobals.require)`（`:690-691`）；被引模块是 CJS 互操作时用 `RuntimeGlobals.createFakeNamespaceObject`（`:701`）。
- **本场景实测**（development）：a.js 中该表达式生成——
  `__webpack_require__.e(/*! import() | lazy-chunk */ "lazy-chunk").then(__webpack_require__.bind(__webpack_require__, /*! ./lazy */ "./lazy.js"))`
- 关键转折：**chunk 边界（block）在这一步被翻译成"chunkId 字符串 + RuntimeGlobals 函数调用"，同时把"需要哪些 runtime 函数"声明进 runtimeRequirements**。生成结果存 `CodeGenerationResults`（构建期），文本本身将进入 emitted runtime。

### 7.5 runtime requirements → `RuntimeModule`：需求级联

`Compilation.processRuntimeRequirements`（`lib/Compilation.js:3708`）按 module → chunk → tree 三级收集需求；tree 级对每个 `chunkGraphEntries`（entrypoint 的 runtime chunk，`:3685-3697`）的需求集合逐项调 `hooks.runtimeRequirementInTree.for(r).call(chunk, set, context)`。JS `Set` 迭代允许边迭代边新增，因此需求会**级联触发**。本场景在 a 的 runtime chunk 上：

1. codegen 声明的 `RuntimeGlobals.ensureChunk` 命中 `RuntimePlugin` 的 tap（`lib/RuntimePlugin.js:371-383`）：`chunk.hasAsyncChunks()` 为真 → 向集合追加 `RuntimeGlobals.ensureChunkHandlers`，并 `compilation.addRuntimeModule(chunk, new EnsureChunkRuntimeModule(set))`。该模块生成 `__webpack_require__.e = chunkId => Promise.all(Object.keys(__webpack_require__.f).reduce(...))`（`lib/runtime/EnsureChunkRuntimeModule.js:35-53`）。
2. 新加入的 `ensureChunkHandlers` 命中 `JsonpChunkLoadingPlugin` 的 handler（`lib/web/JsonpChunkLoadingPlugin.js:42-55`，每 chunk 只挂一次）：`addRuntimeModule(chunk, new JsonpChunkLoadingRuntimeModule(set))`，并追加 `publicPath`/`loadScript`/`getChunkScriptFilename` 三个需求（`:69-76`）。
3. 这三个需求再命中 `RuntimePlugin` 对应 taps → `PublicPathRuntimeModule`、`LoadScriptRuntimeModule`（`__webpack_require__.l`，`lib/RuntimePlugin.js:395-410`）、`GetChunkFilenameRuntimeModule`（生成 `__webpack_require__.u = chunkId => ...` 的 chunkId→URL 映射，`lib/runtime/GetChunkFilenameRuntimeModule.js:19`）。
4. `RuntimeGlobals.require` 等经 `GLOBALS_ON_REQUIRE` 补 `requireScope`（`lib/RuntimePlugin.js:130-141`）。

`addRuntimeModule`（`lib/Compilation.js:3843`）把 runtime module 接到 runtime chunk（`connectChunkAndRuntimeModule`）并触发 `hooks.runtimeModule.call(module, chunk)`（`:3886`）。**b 的 runtime chunk 没有任何人声明 `ensureChunk`，所以 b.js 不含 chunk 装载运行时**（实测确认，见 7.8）。

`JsonpChunkLoadingRuntimeModule.generate`（`lib/web/JsonpChunkLoadingRuntimeModule.js:75`）生成 JSONP 装载逻辑：`installedChunks` 注册表（以 `getInitialChunkIds` 播种初始 chunk，`:123/135-143`）、`__webpack_require__.f.j`（Promise + `url = __webpack_require__.p + __webpack_require__.u(chunkId)` + 插入 `<script>`，`:147-180`）、`webpackJsonpCallback` 并劫持全局 `chunkLoadingGlobal` 数组的 `push`（`:463-465`）。挂接来源：`output.chunkLoading` 默认 web→`"jsonp"`，`WebpackOptionsApply` 应用 `EnableChunkLoadingPlugin`（`lib/WebpackOptionsApply.js:221-222`）→ `JsonpChunkLoadingPlugin`（`lib/javascript/EnableChunkLoadingPlugin.js:78-81`）。

### 7.6 chunk → 文件（写入产物的 runtime 域）与共享/复用条件

- `createChunkAssets` 对每个 chunk 走 `hooks.renderManifest` → `JavascriptModulesPlugin` 的 tap（`lib/javascript/JavascriptModulesPlugin.js:307`）：runtime chunk → `renderMain`（`:771`），非 runtime chunk → `renderChunk`（`:700`）。
  - `renderMain`：`var __webpack_modules__ = ({...模块工厂...})`（`:848`）→ bootstrap 头（`__webpack_require__` 函数本体由 `renderRequire`（`:1407-1491`）在需要时生成，含 module cache）→ **runtime modules 渲染**（`chunkGraph.getChunkRuntimeModulesInOrder(chunk)` → `Template.renderRuntimeModules`，`:871-880`，此调用触发 runtime module 的 codeGeneration）→ startup：默认逐个执行入口模块（`getChunkEntryModulesWithChunkGroupIterable`，`:1205-1322`）。
  - 非 runtime chunk 的文件外壳由 `ArrayPushCallbackChunkFormatPlugin`（`lib/javascript/ArrayPushCallbackChunkFormatPlugin.js:43-78`，按 `output.chunkFormat` 在 `lib/WebpackOptionsApply.js:194-195` 应用）包裹成 `(globalThis["webpackChunk..."] = ... || []).push([[chunkIds], { ...模块工厂... }])`——与 7.5 的 `webpackJsonpCallback` 对应。
- **共享模块的形成/复用条件**（实测对照，7.8）：
  - 默认 `optimization.splitChunks.chunks: "async"`（`lib/config/defaults.js:1573`）：splitChunks 只处理异步 chunk，两个入口间的静态共享**不去重**——`shared` 的模块工厂同时打进 a.js 和 b.js（同一 module id、各自独立的 module cache）。
  - `splitChunks.chunks: "all"` 且体积过 `minSize`（开发默认 10000 字节，`:1576`；实测需 `minSize: 0` 才对玩具模块生效）：`default` cacheGroup（`minChunks: 2`，`:1585-1590`）把 `shared.js` 抽到独立 chunk（实测产出 `shared_js.js`，同样是 push 格式的非 runtime chunk）；`defaultVendors`（`test: node_modules`，`:1591-1596`）处理来自 node_modules 的共享。`SplitChunksPlugin` 挂在 `compilation.hooks.optimizeChunks`（`lib/optimize/SplitChunksPlugin.js:833`，即 3.5 节第 5 步的循环里）。
  - 抽包后的连带效应（实测）：b 的入口 chunk 现在需要在执行入口前确保 `shared_js` 已加载，b.js 因此**也获得了 chunk 装载运行时**（`installedChunks` 出现）——抽取共享 chunk 会把原本"自包含"的入口变成多 chunk 入口。
  - 异步侧复用：同一 `webpackChunkName` 合并（7.3）；`webpackMode: "eager"` 不产生异步 chunk；`reuseExistingChunk: true` 允许抽包时并入已存在 chunk。
- 最后经第 4 节的 `emitAssets` 写盘。

### 7.7 哪些只在构建期、哪些被翻译进 emitted runtime

| 只存在于构建期 compiler 域 | 被翻译进 emitted runtime 的信息 |
| --- | --- |
| `Dependency` 对象（`HarmonyImportSideEffectDependency`/`HarmonyImportSpecifierDependency`/`ImportDependency`） | 依赖的**替换文本**：`__webpack_require__(moduleId)` 声明、`importVar.x` 属性访问、`__webpack_require__.e(chunkId).then(...)` |
| `AsyncDependenciesBlock`（`groupOptions`/`webpackChunkName`） | block→chunk 的关系：调用点里的 **chunkId 字符串**；`webpackChunkName` 只以注释与（命名的）chunk id/文件名存在 |
| `ModuleGraph` / `ModuleGraphConnection` | 模块 id→工厂函数的 `__webpack_modules__` 表；exports 使用信息（重命名、namespace 包装、`createFakeNamespaceObject` 的 bit 标记） |
| `Chunk`/`ChunkGroup`/`ChunkGraph` | `__webpack_require__.u` 的 chunkId→URL 映射；`installedChunks` 初始表；`chunkLoadingGlobal` 的 push 协议 |
| `CodeGenerationResults`、`runtimeRequirements` 集合 | `RuntimeModule` 生成的 `__webpack_require__.*` 函数族（`.e`/`.f.j`/`.l`/`.p`/`.u`…）；`__webpack_require__` 本体与 module cache（`renderRequire`） |
| `AsyncQueue`、`NormalModuleFactory`、`ResolverFactory` 等 | —（不进入产物） |

浏览器执行路径（本场景，web target）：加载 a.js → runtime modules 初始化 `installedChunks`/`__webpack_require__.f.j` → startup 执行 `./a.js` → 静态 `shared` 经 `__webpack_require__` 命中本 bundle 内副本 → 执行到 `import()`：`__webpack_require__.e("lazy-chunk")` → `f.j` 查 `installedChunks`，未装载则建 Promise 并插入 `<script src=__webpack_require__.p + __webpack_require__.u("lazy-chunk")>` → `lazy-chunk.js` 执行 push → `webpackJsonpCallback` 把工厂并入 `__webpack_modules__` 并 resolve → `.then(__webpack_require__.bind(__webpack_require__, "./lazy.js"))` 取到命名空间执行 `m.lazy()`。

### 7.8 本章运行期核对（临时脚本，仓库外临时目录；工具见第 9 节）

- 默认配置（`splitChunks.chunks: "async"`）：产物 `a.js`、`b.js`、`lazy-chunk.js` 三个文件。a.js 同时含 `shared` 模块工厂、`__webpack_require__.e(/*! import() | lazy-chunk */ "lazy-chunk").then(__webpack_require__.bind(__webpack_require__, /*! ./lazy */ "./lazy.js"))` 调用点、`__webpack_require__.f.j = ` 与 `installedChunks`；b.js 含 `shared` 模块工厂但无任何 chunk 装载运行时；`lazy-chunk.js` 为 `(...webpackChunk...).push([[...], ...])` 包裹格式。→ 证实 7.3/7.4/7.5/7.6 的默认行为。
- `splitChunks: { chunks: "all", minSize: 0 }`：多出 `shared_js.js`（含 `shared` 工厂、push 格式），a.js/b.js 不再含 `shared` 工厂；b.js 新出现 `installedChunks`（抽包后入口需先确保共享 chunk 装载）。→ 证实 7.6 的条件讨论。
- 默认 `minSize`（development 10000）会阻止玩具尺寸模块被抽包，这是 `chunks: "all"` 下"没有反应"的常见原因。
- 未做：production 模式、`optimization.runtimeChunk: true`、context import（非字面量）的运行期核对（结论仍属静态阅读）。

---

## 8. 未证实 / 待核对清单

以下内容本轮**未逐行核对或无法从当前版本确认**，后续按需要补读：

1. `bin/webpack.js` 转发 `webpack-cli` 的完整逻辑（只读文件头 60 行）。
2. `MultiCompiler._runGraph` / `validateDependencies` 的调度细节；`MultiWatching` 行为。
3. `IdleFileCachePlugin` / `PackFileCacheStrategy` 的 store/restore 内部流程与序列化格式（`lib/serialization/*`）。
4. `JavascriptModulesPlugin` 的 `renderManifest` tap 产出的具体 `render()` 实现（`renderChunk`/`renderMain` 未读）；`MainTemplate`/`ChunkTemplate`/`ModuleTemplate` 的实际角色（`Compilation` 构造时创建，`lib/Compilation.js:1039-1049`）。
5. `Stats`/`StatsFactory`/`StatsPrinter` 输出链（仅确认注册点与 `new Stats(compilation)` 调用点）。
6. `lib/config/target.js` 的 browserslist 解析细节（只确认函数名与调用点）。
7. `processRuntimeRequirements` 中 module 级收集段（`:3708-3800` 前段）的逐行逻辑。
8. `lib/index.js` 的全部 lazy 导出清单（只读头部 120 行）。
9. 除第 9 节已核对的运行期行为外，其余调用顺序结论来自静态阅读。

---

## 9. 核对记录

- [x] 源码逐行阅读：`lib/webpack.js`、`lib/Compiler.js`、`lib/Compilation.js`（主链路）、`lib/config/{normalization,defaults}.js`、`lib/WebpackOptionsApply.js`、`lib/EntryOptionPlugin.js`、`lib/EntryPlugin.js`、`lib/DynamicEntryPlugin.js`、`lib/node/NodeEnvironmentPlugin.js`、`lib/Watching.js`、`lib/Cache.js`、`lib/NormalModuleFactory.js`（主链路段）、`lib/buildChunkGraph.js`（结构）、`lib/ModuleGraph.js`/`lib/ChunkGraph.js`（结构与静态反查）、`lib/validateSchema.js`、`lib/MultiCompiler.js`（部分）、`lib/javascript/JavascriptModulesPlugin.js`（hook 挂点）。
- [x] `yarn install --frozen-lockfile`（Yarn 1.22.22，lockfile 未变；安装后 `git status` 仍只有本文件一个改动）。
- [x] Jest：`node node_modules/jest-cli/bin/jest test/Defaults.unittest.js` —— 42 tests / 41 snapshots 全部通过（核对 `lib/config/defaults.js` 的默认值行为与本仓库快照一致）。
- [x] 运行期冒烟（临时脚本，仓库外临时目录构建 `entry.js` + 普通 JS + JSON，`mode: development`、`cache: false`，构建产物同样写到临时目录）：
  - 实测 hook 触发次序为 `beforeRun → run → beforeCompile → compile → thisCompilation → compilation → make → finishMake → compilation.seal → compilation.processAssets → compilation.afterSeal → afterCompile → shouldEmit → emit → assetEmitted → afterEmit → done`，与第 3、4 节描述一致；`afterDone` 在用户 callback 之后触发（`lib/Compiler.js:496-497`），探测脚本在其前打印故未捕获。
  - 实测 `compilation` hook 触发时 `compilation.moduleGraph` 已存在、`compilation.chunkGraph` 为 `undefined`；`compilation.hooks.seal` 触发时 `chunkGraph` 已创建——与 3.6 节"可用时机"一致。
  - 产物 `bundle.js` 实际写出（emit 域工作正常），`stats.hasErrors() === false`。
  - 创建期 hooks（`environment`/`afterEnvironment`/`afterPlugins`/`afterResolvers`/`initialize`）在 `webpack()` 返回前已触发，返回后再 tap 捕获不到——与 2.3 节顺序一致。
- [x] 第 7 章（场景追踪）源码阅读：`lib/dependencies/{ImportPlugin,ImportParserPlugin,ImportDependency,HarmonyModulesPlugin,HarmonyImportDependency,HarmonyImportSpecifierDependency,HarmonyImportDependencyParserPlugin}.js`、`lib/AsyncDependenciesBlock.js`、`lib/RuntimeTemplate.js`（`importStatement`/`moduleNamespacePromise`/`blockPromise`）、`lib/RuntimePlugin.js`、`lib/runtime/{EnsureChunkRuntimeModule,GetChunkFilenameRuntimeModule}.js`、`lib/web/{JsonpChunkLoadingPlugin,JsonpChunkLoadingRuntimeModule}.js`、`lib/javascript/{EnableChunkLoadingPlugin,ArrayPushCallbackChunkFormatPlugin,JavascriptModulesPlugin}.js`（render/renderMain/renderRequire）、`lib/buildChunkGraph.js`（`iteratorBlock`/队列处理段）、`lib/optimize/SplitChunksPlugin.js`（hook 挂点）、`lib/Chunk.js`（`hasRuntime`）。
- [x] 第 7 章场景运行期冒烟：两入口 + 共享模块 + 带 `webpackChunkName` 的动态 `import()`，development 默认配置与 `splitChunks: { chunks: "all", minSize: 0 }` 各构建一次，结果见 7.8（全部通过，临时脚本已删除）。
- 未做：watch 模式与 filesystem 缓存的运行期核对（覆盖第 5 节，仍属静态阅读结论）。
