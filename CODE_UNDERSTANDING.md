# Webpack 构建基础设施理解（事故复盘一致版本）

本文基于当前仓库 `webpack@5.99.9` 的源码阅读，按事故复盘需要串联 public API、`Compiler`、`Compilation`、loader/factory、模块图、chunk/runtime、watch、filesystem cache 和持久化复用。文中明确区分已由源码证实的调用顺序、对象所有权和未逐行展开的实现细节。

后续沿用以下三个执行域名称：

1. **构建期 compiler 域**：Node.js 进程内的 `Compiler`、`Compilation`、工厂、队列、hook 和缓存协调对象。
2. **写入产物的 runtime 域**：最终写入 bundle 的运行时代码。它不是 Node.js 进程中的 `Compiler`，而是由 `RuntimePlugin`、chunk format/render 插件和各模块 generator 在构建期生成的 `Source`。
3. **watch/cache 域**：文件监听、增量失效、跨编译复用、idle store/restore 和 `Cache` hook 层。它会复用构建期 compiler 域对象，但调度边界不同于一次性 `run()`。

## 1. 公共入口与 `Compiler` 创建

### 1.1 `require("webpack")` 到 `webpack()`

公共入口是 [index.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/index.js#L128-L132)。`module.exports` 是一个 lazy function，首次调用时才 `require("./webpack")`，同时通过 getter 暴露 `validate`、`version`、插件类和核心类。

真正的函数在 [webpack.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/webpack.js#L121-L193)：

- `webpack(options, callback)` 内部先执行 `create()`。
- `create()` 先用预编译 schema `webpackOptionsSchemaCheck` 快速检查；失败时再通过 [validateSchema.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/validateSchema.js) 和 [WebpackOptions.json](file:///e:/newGsb/questions/GSB-013/Tony/schemas/WebpackOptions.json) 做完整校验。
- `options` 为数组时走 `createMultiCompiler()`；单配置时走 `createCompiler()`。
- 传了 `callback` 时：
  - `watch: true` 调 `compiler.watch(watchOptions, callback)`；
  - 否则调 `compiler.run(...)`，完成后再 `compiler.close(...)`，最后把 `Stats` 交回 callback。
- 未传 callback 时直接返回 `Compiler` 或 `MultiCompiler`；若设置了 `watch` 但没有 callback，会触发弃用提示。

多配置入口在 [webpack.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/webpack.js#L44-L58) 中为每个配置创建一个子 `Compiler`，再用 [MultiCompiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/MultiCompiler.js) 组合；本文主线只展开单配置。

### 1.2 配置规范化、默认值与插件应用顺序

单配置创建逻辑在 [webpack.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/webpack.js#L65-L97)，顺序非常关键：

1. `getNormalizedWebpackOptions(rawOptions)` 把用户配置转为规范化形态。
   - 入口在 [normalization.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/config/normalization.js#L127-L189)。
   - `entry` 未传时转为 `{ main: {} }`；函数式 entry 会包成返回规范化静态 entry 的 Promise；静态 entry 由 `getNormalizedEntryStatic()` 处理。
   - `cache: true` 转为 `{ type: "memory" }`；`cache: false` 保留为 `false`；`cache.type` 只接受当前实现覆盖的类型。
2. `applyWebpackOptionsBaseDefaults(options)` 设置创建 compiler 前必须存在的基础值。
   - 见 [defaults.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/config/defaults.js#L162-L165)。
   - 当前包括 `context = process.cwd()` 和 infrastructure logging 默认值。
3. `new Compiler(context, options)` 创建 compiler。
4. `new NodeEnvironmentPlugin(...).apply(compiler)` 接入 Node 文件系统和基础设施日志。
   - 见 [NodeEnvironmentPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/node/NodeEnvironmentPlugin.js#L38-L69)。
   - 它设置 `inputFileSystem`、`outputFileSystem`、`intermediateFileSystem`、`watchFileSystem`，并在 `beforeRun` 中 purge cached input file system。
5. 应用用户 `options.plugins`：函数插件以 `plugin.call(compiler, compiler)` 调用，对象插件调 `plugin.apply(compiler)`。
6. `applyWebpackOptionsDefaults(options, compilerIndex)` 补齐完整默认值，包括 target、mode 派生项、devtool、watch、parallelism、cache、output、module、externals、resolve、optimization 等。
   - 主函数见 [defaults.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/config/defaults.js#L172-L363)。
   - 这意味着用户插件在**完整默认值应用之前**执行；它拿到的是已规范化但尚未完全补默认值的 `compiler.options`。
7. 根据解析出的 platform 设置 `compiler.platform`。
8. 同步触发 `compiler.hooks.environment` 和 `compiler.hooks.afterEnvironment`。
9. `new WebpackOptionsApply().process(options, compiler)` 注册内建基础设施和选项驱动插件。
10. 同步触发 `compiler.hooks.initialize`，然后返回 compiler。

`WebpackOptionsApply.process()` 是内建能力的总装配点，见 [WebpackOptionsApply.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/WebpackOptionsApply.js#L78-L793)。它会：

- 把 `output.path`、records path、`name` 写到 `Compiler`；
- 按 `externals`、target presets、chunk format、library、devtool、experiments、module type、optimization、performance、cache 等选项注册插件；
- 固定注册 `JavascriptModulesPlugin`、`JsonModulesPlugin`、`AssetModulesPlugin`、`EntryOptionPlugin`、`RuntimePlugin`、依赖语法插件、stats 插件、路径模板插件、records 插件和缓存插件等；
- 调 `EntryOptionPlugin().apply(compiler)` 后立即同步触发 `entryOption` hook，见 [WebpackOptionsApply.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/WebpackOptionsApply.js#L391-L398)；
- 末尾同步触发 `afterPlugins`，再把 `options.resolve`、`options.resolveLoader` 接入 `resolverFactory.hooks.resolveOptions`，最后触发 `afterResolvers`，见 [WebpackOptionsApply.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/WebpackOptionsApply.js#L753-L792)。

`entryOption` 是 `SyncBailHook`。内建 [EntryOptionPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/EntryOptionPlugin.js#L20-L24) tap 后返回 `true`，因此同一轮后续 tap 不会再执行。它把每个静态 entry description 转成 `EntryPlugin`；函数式 entry 则改为应用 [DynamicEntryPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/DynamicEntryPlugin.js)。

## 2. 核心对象由谁创建、持有、何时可用

### 2.1 `Compiler`

`Compiler` 构造函数定义在 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L137-L325)。它由 [webpack.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/webpack.js#L68-L71) 的 `createCompiler()` 创建，并在返回前被插件系统完整配置。

构造时即可用的长期对象包括：

- `hooks`：包含 `initialize`、`environment`、`beforeRun`、`run`、`compile`、`make`、`emit`、`done` 等；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L143-L217)。
- `resolverFactory = new ResolverFactory()`；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L265-L266)。
- `cache = new Cache()`；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L287-L287)。
- `requestShortener`、并发状态、watch 状态、asset emitting cache 等。

文件系统字段初始为 `null`，由 `NodeEnvironmentPlugin` 在用户插件应用前填入。

### 2.2 `CompilationParams`、`NormalModuleFactory`、`ContextModuleFactory`

每次 `Compiler.compile()` 都先调用 `newCompilationParams()`，见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1298-L1304)：

- `createNormalModuleFactory()` new 出 [NormalModuleFactory](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js)，并同步触发 `normalModuleFactory` hook；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1277-L1290)。
- `createContextModuleFactory()` new 出 [ContextModuleFactory](file:///e:/newGsb/questions/GSB-013/Tony/lib/ContextModuleFactory.js)，并同步触发 `contextModuleFactory` hook；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1292-L1296)。

因此这些 factory 是**每次 compile 新建并由当次 compilation 使用**，不是整个 `Compiler` 生命周期单例。`Compiler` 会通过 `_lastNormalModuleFactory` 保活上次对象，并在下一次创建前 cleanup。

### 2.3 `Compilation` 和 `ModuleGraph`

`Compiler.createCompilation()` 先清理上一次 compilation，再 `new Compilation(this, params)`，见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1255-L1262)。`newCompilation()` 随后设置 `name`、`records`，同步触发 `thisCompilation` 和 `compilation`，见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1268-L1275)。

`Compilation` 构造函数见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L993-L1209)。它持有：

- 反向引用 `compiler`；
- `resolverFactory`、`inputFileSystem`；
- 新的 `FileSystemInfo`；
- `mainTemplate`、`chunkTemplate`、`runtimeTemplate`、`moduleTemplates`；
- 新的 `moduleGraph = new ModuleGraph()`；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1057-L1057)。
- 模块构建队列：`processDependenciesQueue`、`addModuleQueue`、`factorizeQueue`、`buildQueue`、`rebuildQueue`；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1063-L1093)。
- `modules`、`chunks`、`chunkGroups`、`entrypoints`、`assets`、`errors`、`warnings`、依赖集合和 compilation 级 cache facade。

`ModuleGraph` 构造函数见 [ModuleGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L128-L161)。它由 `Compilation` 构造，随后在模块创建和依赖处理过程中持续写入：

- `setParents()` 记录 dependency 所属 block/module；见 [ModuleGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L176-L187)。
- `setResolvedModule()` 创建 `ModuleGraphConnection`，维护 incoming/outgoing connections；见 [ModuleGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L213-L243)。
- `seal()` 创建 chunk 前会 `freeze("seal")`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3083-L3086)。

### 2.4 `ChunkGraph`

`ChunkGraph` 不是在 `Compilation` 构造时创建，而是在 `Compilation.seal()` 开始处创建：

- `const chunkGraph = new ChunkGraph(this.moduleGraph, this.outputOptions.hashFunction)`；
- `this.chunkGraph = chunkGraph`；
- back-compat 模式下会把它关联到已存在模块；

见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3062-L3072)。其构造函数和内部 WeakMap 状态见 [ChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/ChunkGraph.js#L245-L260)。

因此在 `make` 和 `finish` 阶段，可靠可用的是 `compilation.moduleGraph`；`compilation.chunkGraph` 要到 `seal()` 才存在。

### 2.5 `CodeGenerationResults`、`Stats`

- `CodeGenerationResults` 在 `Compilation.codeGeneration()` 开始创建，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3467-L3499)。
- `Stats` 在编译产物完成、准备调用 `Compiler.hooks.done` 时创建；非 watch 路径见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L536-L577)，watch 路径见 [Watching.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L303-L345)。

## 3. 一次非 watch 构建主链路

### 3.1 `Compiler.run()`：构建外层异步边界

入口是 [Compiler.run](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L474-L609)。

主要顺序：

1. 若 `this.running` 已为 true，立即返回 `ConcurrentCompilationError`。
2. 设置 `running = true`，记录 `startTime`。
3. 如果 compiler 处于 idle，先 `cache.endIdle()`，否则直接进入 `run()`。
4. 异步依次执行：
   - `beforeRun`；
   - `run`；
   - `readRecords`；
   - `this.compile(onCompiled)`。
5. `compile` 成功后进入 `onCompiled`：
   - `shouldEmit` 是 `SyncBailHook`，返回 `false` 会短路资源写出，但仍创建 `Stats` 并执行 `done`/`afterDone`；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L514-L523)。
   - 否则 `process.nextTick` 后执行 `emitAssets()`。
   - emit 后检查 `compilation.hooks.needAdditionalPass`。若为 true，先触发 `done`，再异步触发 `additionalPass` 并重新 `this.compile(onCompiled)`；见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L533-L551)。这是多轮编译边界。
   - 正常路径继续 `emitRecords()`、创建 `Stats`、执行 `done`、`cache.storeBuildDependencies()`，最后 `finalCallback()` 和 `afterDone`。

### 3.2 `Compiler.compile()`：从 factory 到 compilation

核心在 [Compiler.compile](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1310-L1355)。

顺序为：

1. `newCompilationParams()` 创建 `NormalModuleFactory` 和 `ContextModuleFactory`。
2. 异步 `beforeCompile(params)`。
3. 同步 `compile(params)`。
4. `newCompilation(params)` 创建 `Compilation`，同步触发 `thisCompilation`、`compilation`。
5. 异步 `make(compilation)`。
6. 异步 `finishMake(compilation)`。源码中该 hook 实例化为 `AsyncSeriesHook`，见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L181-L183)；其上方 JSDoc 写成 `AsyncParallelHook`，以当前源码实例化为准确认。
7. `process.nextTick` 后执行：
   - `compilation.finish()`；
   - `compilation.seal()`；
   - `afterCompile(compilation)`；
   - callback 返回 compilation。

`make` 是 `AsyncParallelHook`，见 [Compiler.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L179-L180)。入口插件和其他添加模块的插件可以并行 tap；不要假设它们按注册顺序串行完成。

### 3.3 `make`：入口依赖如何启动模块图构建

静态 entry 由 [EntryPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/EntryPlugin.js#L33-L52) 接入：

- 在 `compilation` hook 中把 `EntryDependency` 映射到 `normalModuleFactory`；
- 预先创建 `EntryDependency`；
- 在 `compiler.hooks.make.tapAsync` 中调用 `compilation.addEntry(context, dep, options, callback)`。

`addEntry()` 只是把参数规整为 options 后转 `_addEntryItem()`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L2320-L2328)。`_addEntryItem()` 做三件事：

1. 记录或合并 entry data；
2. 同步触发 `addEntry` hook；
3. 调 `addModuleTree()`；成功后触发 `succeedEntry`，失败触发 `failedEntry`；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L2355-L2427)。

`addModuleTree()` 根据 dependency constructor 从 `compilation.dependencyFactories` 查 module factory，然后进入 `handleModuleCreation()`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L2269-L2311)。

### 3.4 factorize、add、build、process dependencies

`handleModuleCreation()` 是模块创建和递归依赖的汇合点，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1935-L2072)。它通过四组队列调度：

1. `factorizeQueue` 进入 `_factorizeModule()`。
2. `addModuleQueue` 进入 `_addModule()`。
3. `buildQueue` 进入 `_buildModule()`。
4. `processDependenciesQueue` 进入 `_processModuleDependencies()`。

`_factorizeModule()` 调 factory 的 `create()`。对普通模块，这是 [NormalModuleFactory.create](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L869-L953)，顺序为：

- 构造 `resolveData`；
- 异步 bail hook `beforeResolve`，返回 `false` 会忽略该请求；
- 异步 bail hook `factorize`；
- 内建 tap 再依次进入 `resolve`、`afterResolve`、`createModule`，然后通过 `createModuleClass` 或默认 `new NormalModule(createData)` 创建模块，最后经过同步 waterfall `module` hook；见 [NormalModuleFactory.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L340-L418)。

`_addModule()` 负责按 identifier 去重并从 compilation module cache restore，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1419-L1455)。首次加入的模块进入 `this.modules` 和 `this._modules`。

`_buildModule()` 负责真正构建模块，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1492-L1554)：

1. `module.needBuild(...)` 判断是否需要构建；
2. 不需要时触发 `stillValidModule`；
3. 需要时同步触发 `buildModule`，加入 `builtModules`；
4. 调 `module.build(options, compilation, resolver, fs, callback)`；
5. 成功后把模块存入 `_modulesCache`，触发 `succeedModule`；失败触发 `failedModule`。

构建完成后，`_handleModuleBuildAndDependencies()` 调 `processModuleDependencies()`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L2083-L2162)。`_processModuleDependencies()` 会遍历 module/block 的 dependencies，按 factory 分组，设置 dependency parent，再对每组递归调用 `handleModuleCreation()`；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1593-L1664)。循环依赖通过“依赖处理已在进行中则直接返回”避免死锁，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L2151-L2154)。

### 3.5 `finish()`：模块图收尾

`finish()` 在 `make` 和 `finishMake` 之后执行，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L2782-L3027)。

关键点：

- 清空 `factorizeQueue`；
- profile 开启时汇总模块性能数据；
- `_computeAffectedModules()`；
- 异步 `finishModules(modules)`；
- 临时 freeze `moduleGraph`，报告 dependency errors/warnings，并收集模块自身 errors/warnings；
- callback 返回。

### 3.6 `seal()`：chunk 图、优化、hash、代码生成、资产

`seal()` 是最长的同步/异步混合阶段，入口见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3049-L3427)。

主顺序如下。

#### A. 创建 `ChunkGraph` 并冻结模块图

- new `ChunkGraph(moduleGraph, hashFunction)`；
- 同步 `seal`；
- `while (optimizeDependencies.call(modules)) {}`。这是 `SyncBailHook`：tap 返回真值会让循环再跑一轮，返回假值退出；
- `afterOptimizeDependencies`；
- `beforeChunks`；
- `moduleGraph.freeze("seal")`。

#### B. 从 entries 创建初始 chunk/entrypoint 并构建 chunk graph

- 为每个 named entry 创建 chunk 和 `Entrypoint`；
- 把 entry dependency 解析到的 module 连接为 entry module；
- 处理 `dependOn`、`runtime` 选项；
- 调 `buildChunkGraph(this, chunkGraphInit)`；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3083-L3227)。

`buildChunkGraph()` 本身分两阶段，见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1301-L1357)：

1. `visitModules(...)` 从 entrypoints 和模块依赖图访问模块，创建 async chunk groups 并建立 chunk/module 关系；
2. `connectChunkGroups(...)` 连接 chunk group 父子关系；
3. 合并 chunk runtime；
4. cleanup 未连接 groups。

随后同步触发 `afterChunks`。

#### C. 优化树、ID、records 和 hash

顺序包括：

- `optimize`；
- `while (optimizeModules.call(modules)) {}`；
- `afterOptimizeModules`；
- `while (optimizeChunks.call(chunks, chunkGroups)) {}`；
- `afterOptimizeChunks`；
- 异步 `optimizeTree`；
- 异步 `optimizeChunkModules`；
- IDs：`beforeModuleIds`、`moduleIds`、`optimizeModuleIds`、`afterOptimizeModuleIds`、`beforeChunkIds`、`chunkIds`、`optimizeChunkIds`、`afterOptimizeChunkIds`；
- `assignRuntimeIds()`；
- `_computeAffectedModulesWithChunkGraph()`；
- records hooks；
- `beforeModuleHash`、`createModuleHashes()`、`afterModuleHash`。

#### D. 代码生成和 runtime requirements

- `beforeCodeGeneration`；
- `codeGeneration()`；
- `afterCodeGeneration`；
- `beforeRuntimeRequirements`；
- `processRuntimeRequirements()`；
- `afterRuntimeRequirements`。

`codeGeneration()` 从 `chunkGraph.getModuleRuntimes(module)` 获取模块所属 runtime，按模块 hash 合并相同 hash 的 runtime jobs，再调用 `_runCodeGenerationJobs()`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3467-L3499)。每个 job 通过 `_codeGenerationModule()` 使用 compilation code generation cache，最终调用 `module.codeGeneration({ chunkGraph, moduleGraph, dependencyTemplates, runtimeTemplate, runtime, codeGenerationResults, compilation })`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3621-L3681)。

#### E. full hash 与延后的 code generation jobs

`createHash()` 在 `beforeHash` 和 `afterHash` 之间调用，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3331-L3335)。

需要特别注意：`createHash()` 可能返回 `codeGenerationJobs`，用于依赖 chunk hash/full hash 的 runtime module；这些 jobs 在 `afterHash` 之后才由 `_runCodeGenerationJobs()` 执行，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3337-L3422)。因此不能简单认为 `afterHash` 时所有 runtime module 的 code generation 都已完成。

#### F. 资产生成

- `clearAssets()`；
- `beforeModuleAssets`；
- `createModuleAssets()`：把 module `buildInfo.assets` 中的产物通过 `emitAsset()` 加入 compilation；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4877-L4900)。
- `shouldGenerateChunkAssets` 是 `SyncBailHook`，返回 `false` 会跳过 chunk asset render；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3408-L3421)。
- 否则 `beforeChunkAssets` 后调用 `createChunkAssets()`。
- `createChunkAssets()` 对每个 chunk 调 `getRenderManifest()`，后者执行 `renderManifest` waterfall hook；manifest entry 的 `render()` 返回 `Source`，随后 `emitAsset()` 写入 compilation.assets，并记录 chunk files；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4914-L5070)。
- 所有资产进入 `processAssets` AsyncSeriesHook 和 `afterProcessAssets`。旧式 asset hooks 被映射到这个阶段化 hook。
- `summarizeDependencies()` 汇总文件依赖。
- `needAdditionalSeal` 是 `SyncBailHook`，返回真值会 `unseal()` 后递归重新 `seal(callback)`；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3392-L3395)。
- 最后异步 `afterSeal`。

`Compilation.emitAsset()` 只写入内存中的 `compilation.assets` 和 `assetsInfo`，并不写磁盘；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4611-L4634)。

### 3.7 `Compiler.emitAssets()`：从内存资产到磁盘

资源写出由 [Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L676-L1021) 完成：

1. 异步 `emit(compilation)`。
2. 通过 `compilation.getPath(this.outputPath, {})` 解析输出目录并 `mkdirp`。
3. `compilation.getAssets()` 获取内存资产快照；同时把 `compilation.assets` 复制为普通对象。
4. 并发限制为 15，对每个 asset：
   - 去掉 `?` query 后得到实际 target file；
   - 检查大小写/query 相似路径冲突；
   - 根据 `compareBeforeEmit` 和 immutable 信息决定是否 stat/readFile 比较；
   - 调 `outputFileSystem.writeFile(targetPath, content)`；
   - 写入成功后加入 `compilation.emittedAssets`，异步触发 `assetEmitted(file, info)`。
5. 所有文件写完后更新 `_assetEmittingPreviousFiles`，异步触发 `afterEmit(compilation)`。

因此写入产物 runtime 域的 JavaScript 字符串/Buffer 在这里才真正落盘。

## 4. 模块构建前：factory、resolve、rule 与 loader-runner

前面主链路只说到 `_factorizeModule()` 调 factory，factory 返回模块后进入 `_addModule()` 和 `_buildModule()`。线上 rule、resolve、loader 的问题大多发生在这两步之间：`NormalModuleFactory` 还没有创建 `NormalModule` 时解析 request/resource/loaders；创建后由 [NormalModule.build](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1175-L1370) 调 loader-runner，再 parse 和生成 snapshot。

### 4.1 `create()`：从 dependency 到 `resolveData`

[NormalModuleFactory.create](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L869-L953) 是 `_factorizeModule()` 的入口。它从 `ModuleFactoryCreateData` 中取：

- `dependencies[0].request` 作为 `request`；
- `dependencies[0].category` 作为 `dependencyType`；
- `data.context` 或 factory context 作为 `context`；
- `resolveOptions`、`contextInfo`、assertions。

它新建三个空集合：`fileDependencies`、`missingDependencies`、`contextDependencies`，随后构造 [ResolveData](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L881-L895)：

- `request`：原始请求，例如 `"./shared"`、`"style-loader!css-loader!./a.css"` 或带 scheme 的 `"data:..."`；
- `dependencies`：同一 factory 请求携带的 dependencies；
- `createData`：后续创建 NormalModule 的数据；
- `cacheable`：默认 true，若 `beforeResolve` 或 `factorize` 抛错会在错误回调中置为 false。

执行顺序是：

1. 异步 bail hook `beforeResolve`；返回 false 表示 ignore，可带 `ignoredModule`。
2. 异步 bail hook `factorize`。
3. 内建 `factorize` tap 再调 `resolve`、`afterResolve`、`createModule`、`createModuleClass`，最后同步 waterfall `module` 返回创建好的模块。

这些 hook 的类型见 [NormalModuleFactory.hooks](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L274-L311)。

### 4.2 `resolve`：request、resource、loaders 如何确定

内建 [resolve hook](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L419-L853) 是 rule 和 loader 解析核心。

#### A. 解析前缀和 inline loader 前缀

它先识别 request 上的 scheme、`!=!` matchResource 和 inline loader 前缀：

- `-!`：禁用 pre/post loaders，但保留 normal loaders；
- `!`：禁用普通 auto loaders；
- `!!`：禁用 pre/post/normal auto loaders；
- `resource!=!loader!resource`：设置 `matchResource`，用于把某份虚拟资源当作 rule 匹配对象，但实际读取另一个 resource；见 [normalization](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L449-L514)。

没有这些前缀时，request 按 `!+` split，最后一个元素是 `unresolvedResource`，前面的元素是 inline loader elements，见 [NormalModuleFactory.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L483-L505)。

#### B. 解析 inline loaders 和 resource

- inline loaders 通过 `loaderResolver` 解析，结果进入 `loaders`；
- 普通 resource 通过 normal resolver 解析；resolve context 中传入的 file/missing/context dependencies 会被 enhanced-resolve 填充；见 [resolveResource](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L963-L1029)。
- 带 scheme 的 request 走 `resolveForScheme`；context scheme 走 `resolveInScheme`。

解析成功后得到：

- `resourceData.resource`：真正读取的资源，等于 resolved path + query + fragment；
- `resourceData.path/query/fragment`：拆分后的 resource 各部分；
- `resourceData.data`：enhanced-resolve 返回的 resolve data。

#### C. 执行 rules 并合并 use/pre/post loaders

若没有 matchResource，普通资源继续执行 `ruleSet.exec()`，输入包括：

- `resource`、`realResource`、`resourceQuery`、`resourceFragment`；
- `scheme`、`mimetype`、`dependency`、`issuer`、`issuerLayer`、`compiler` 等；见 [ruleSet.exec](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L591-L610)。

rules 返回的 effect 被分到：

- `useLoadersPost`；
- `useLoaders`（normal）；
- `useLoadersPre`；
- 以及 type、sideEffects、parser、generator、resolve、layer 等 settings。

这些 loader 都用 loader resolver 解析。若 request 带 `!!`/`!`/`-!`，对应 auto loaders 会被跳过，见 [use 合并](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L617-L628)。

#### D. 最终 loader 顺序

所有 loader 解析完成后，[continueCallback](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L655-L713) 组装最终数组。对非 matchResource：

- post loaders；
- request 内联 loaders；
- rule normal loaders；
- pre loaders。

对 matchResource，则把 rule normal loaders 放在内联 loaders 前面，因为 normal rules 作用于 matchResource。这个数组就是 `NormalModule.loaders`，也是 loader-runner normal 阶段的输入顺序。

### 4.3 创建 NormalModule、parser、generator

当 `afterResolve` 没有返回 false，`createModule` 没有返回自定义模块时，factory 会按 `settings.type` 查 `createModuleClass` hook；默认没有自定义模块类时 new [NormalModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L337-L380)。创建数据在 [createData](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L683-L708) 中设置，关键字段包括：

- `request`：所有 loader 和最终 resource 字符串化后的完整请求；
- `userRequest`：用户意图请求，可能包含 matchResource 前缀；
- `rawRequest`：dependency 原始 request；
- `loaders`；
- `resource`；
- `matchResource`；
- `resourceResolveData`；
- `parser`/`parserOptions` 和 `generator`/`generatorOptions`。

parser/generator 通过 `NormalModuleFactory.getParser/getGenerator` 获取并缓存，见 [NormalModuleFactory.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L1256-L1326)。JavaScript parser 和 generator 由 [JavascriptModulesPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyModulesPlugin.js#L49-L137) 注册。

### 4.4 `NormalModule.build()`：重置状态并进入 `_doBuild()`

每次需要构建时，[build](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1175-L1370) 会先清空模块自身状态：

- `_source`、`_ast`、error/warnings；
- `clearDependenciesAndBlocks()` 清掉上一轮 `Dependency` 和 `AsyncDependenciesBlock`；
- 重建空 `buildMeta` 和新的 `buildInfo`，初始 `cacheable:false`。

这很重要：如果模块因 snapshot 失效而 rebuild，parser 生成的 dependency/block 不会复用上一轮对象；会重新 parse 并重新写入新的模块图边。但模块对象本身可能来自 `Compilation/modules` cache，因此 `build()` 必须显式清理旧状态。

随后调用 [_doBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L916-L1087)。

### 4.5 loader context 与 hook 边界

`_doBuild()` 先通过 [_createLoaderContext](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L594-L844) 创建 loader context。它包含 webpack 注入的 API：

- `getOptions(schema)`：读取当前 loader options 并可选校验；
- `emitWarning`/`emitError`；
- `resolve`/`getResolve`；
- `emitFile`；
- `addBuildDependency`；
- `addDependency`/`addContextDependency`/`addMissingDependency` 通过 resolve context 的 Set adder 进入 loader-runner 的依赖集合；见 [getResolveContext](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L603-L625)；
- `fs`、`utils`、`mode`、`sourceMap`、hash 参数。

loader 相关的 compilation hook 在 [NormalModule.getCompilationHooks](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L275-L331)：

- `loader`：同步，loaderContext 创建后、runLoaders 前调用，可给 context 加字段或包装 API；
- `beforeLoaders`：同步，可检查/修改 `this.loaders`；
- `readResource`/`readResourceForScheme`：可替换资源读取；
- `processResult`：同步 waterfall，可改写 loader-runner 返回的 result；
- `beforeParse`、`beforeSnapshot`、`needBuild`。

这些是 loader/plugin 能观察或改变模块构建的边界。compiler 级插件还能通过 `NormalModuleFactory.hooks.beforeResolve/resolve/afterResolve/createModule/module` 影响模块创建，但那时 loader-runner 尚未执行。

### 4.6 loader-runner：pitch 与 normal 顺序

`_doBuild()` 调用 [runLoaders](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1013-L1086)，传入：

- `resource: this.resource`；
- `loaders: this.loaders`；
- `context: loaderContext`；
- `processResource`：实际资源读取逻辑。默认读取 `loaderContext.resource`，但 scheme 可通过 `hooks.readResource` 覆盖；见 [processResource](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1023-L1041)。

根据 [LoaderContext 类型](file:///e:/newGsb/questions/GSB-013/Tony/declarations/LoaderContext.d.ts#L98-L240) 和 loader-runner 的 loader 数组约定：

- pitch 阶段按 loader 数组从左到右执行，即 pre -> inline -> normal -> post；
- normal 阶段按相反方向执行，即 post -> normal -> inline -> pre；
- 每个 loader 在 pitch 阶段可以读写 `loaderContext.loaders`；
- `loaderContext.remainingRequest`、`previousRequest`、`currentRequest`、`resourcePath`、`resourceQuery`、`resourceFragment` 会随当前 loaderIndex 变化。

为什么 normal 阶段反向执行？因为 pipeline 是嵌套的：pitch 从数组左侧先获得控制权，normal 从最右侧 loader 开始逐层把资源交给左侧 loader；最终由 loaderIndex 0（数组最左侧）把结果返回给 webpack。在 webpack 的最终数组里，最左侧是 post loader，因此 post loader 的 normal 阶段最后执行并把结果交给 webpack，这与 [processResult 检查 final loader](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L953-L965) 一致。

若某 pitch loader 向 `callback` 返回非空内容，loader-runner 会把该内容作为该位置结果，跳过其右侧 loaders 的 normal 执行，也跳过对剩余 resource 的读取；具体实现属于 loader-runner 包，当前仓库未包含其源码，标为“实现来自 loader-runner，按 webpack 类型和 runLoaders 调用点确认”。在 webpack 侧能确认的是：runLoaders 返回的 result 直接经 `processResult` 进入 NormalModule；若没有 result，`buildInfo.cacheable` 会被置 false 并报 `No result from loader-runner processing`，见 [NormalModule.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1043-L1059)。

### 4.7 loader 返回 source/map/AST、错误、cacheable 与依赖

runLoaders 完成后，webpack：

1. 清理 loaderContext 上对 compiler/compilation/module/fs 的引用，避免 IC 泄漏；见 [NormalModule.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1043-L1050)。
2. 若 result 不存在，标记 cacheable=false 并进入错误处理。
3. 把 loader-runner 返回的 `fileDependencies`、`contextDependencies`、`missingDependencies` 加入 `buildInfo`；见 [NormalModule.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1061-L1075)。
4. 把每个 loader 的路径加入 `buildDependencies`；见 [NormalModule.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1076-L1082)。
5. `buildInfo.cacheable = buildInfo.cacheable && result.cacheable`。
6. 用 result 的 content/sourceMap/additionalData 调 `processResult`。

[processResult](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L930-L987) 要求最终 content 是 string 或 Buffer，否则产生 `ModuleBuildError`。它随后：

- 用 [createSource](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L872-L905) 把内容和 source map 包成 `RawSource`、`OriginalSource` 或 `SourceMapSource`，写入 `this._source`；
- 若第三个参数含 `webpackAST`，写入 `this._ast`，parser 可直接使用而无需重新 parse。

loader 抛错、`callback(err)` 或 `async()(err)` 都会变成 [ModuleBuildError](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L930-L944)。loader 还可 `emitError/emitWarning`，这些成为模块级 `ModuleError/ModuleWarning`，不会直接中断 build。

关于 cache/snapshot：

- `this.cacheable(false)` 或 `result.cacheable=false` 会导致 [needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592) 直接返回 true，下一轮必然 rebuild，也不会创建 snapshot。
- 若 loader 声明的依赖不稳定，例如漏掉 `addDependency`，snapshot 不包含该文件，文件变化时旧 source/AST 可能被复用；这是“改了却复用旧结果”的常见原因。
- `addBuildDependency(loaderOrConfig)` 影响 filesystem pack 恢复；loader 文件变化会让 buildDependencies snapshot 失效，整个 pack 不恢复，从而强制重新解析和构建。
- `addMissingDependency()` 记录构建时不存在的路径；该路径后来出现会使 existence 变化，模块 rebuild。
- `addContextDependency()` 记录目录；目录 timestampHash/hash 变化会使模块 rebuild。

### 4.8 parser 和 generator 的边界

loader-runner 返回后，NormalModule：

1. 调 `hooks.beforeParse`；
2. 检查 `module.noParse`；若命中，不 parse，保留 source，直接进入 hash 和 snapshot；见 [noParse](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1343-L1352)。
3. 否则用 `this.parser.parse(this._ast || source, { source, current: this, module: this, compilation, options })` 解析，见 [parse call](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1354-L1367)。

parser 期间生成的 `Dependency`、`AsyncDependenciesBlock`、buildMeta/buildInfo 都挂在该 NormalModule 上，并在后续 `processDependencies` 中变成 `ModuleGraphConnection`。若 parse 抛错，会生成 `ModuleParseError`。

generator 不在 `build()` 中执行；它在 code generation 阶段被 `module.codeGeneration()` 调用。也就是说：

- build 阶段负责产出 source/AST、dependencies、buildMeta/buildInfo 和 snapshot；
- seal/codegen 阶段才使用 generator 把 source+dependency templates 转成最终 runtime source；
- 若只改 generator/template 而模块 source 和 dependencies 未变，模块可能不 rebuild，但 codegen/asset cache 的 etag 应变化，导致重新生成或写出资产；否则说明 etag 未覆盖该变化。

### 4.9 factory/loader 问题与三个执行域

用前几节的三个执行域看这一段：

- **构建期 compiler 域**：NormalModuleFactory、resolver、rules、loaders、NormalModule、parser 生成的 dependencies/blocks、snapshot 都在这里；它们不会出现在浏览器 runtime 中。
- **写入产物的 runtime 域**：generator、dependency templates 和 runtime modules 把构建期 dependencies 翻译成 `__webpack_require__`、`__webpack_require__.e`、namespace promise 等 emitted source；loader 生成的 source 只是这一阶段的输入。
- **watch/cache 域**：snapshot 记录 loader 登记的 file/context/missing dependencies 和 buildDependencies；filesystem cache 还额外受 cache version 与 buildDependencies 控制。

因此排查线上问题时要先定位发生在哪个边界：resolve/rule 错误通常在 `beforeResolve/resolve/afterResolve`；loader 结果或依赖登记错误通常在 `loader/beforeLoaders/readResource/processResult`；旧结果复用要检查 `cacheable`、snapshot、buildDependencies 和 codegen/asset etag，而不是只看文件 mtime。

## 5. 具体场景：两个入口、一个共享模块、一个普通动态 `import()`

本节固定一个最小场景，说明“源码里的一条 import”怎样穿过第一版定义的三个执行域：

- `entry-a.js`：静态 `import "./shared.js"`，随后调用 `import("./async.js")`。
- `entry-b.js`：静态 `import "./shared.js"`。
- `shared.js`：普通共享模块。
- `async.js`：只被普通动态 `import()` 引用的异步模块。

默认 web target 会启用异步 chunk 所需的 chunk loading。两个入口通常各自产生初始 runtime chunk；若没有 `optimization.splitChunks` 或共享 runtime 配置，`shared.js` 会同时进入两个入口初始 chunk，而不是自动抽成一个独立共享 chunk。`async.js` 则会形成一个由入口 A 父子关系指向的异步 chunk。

### 4.1 parser 阶段：源码 import 变成构建期 Dependency/Block

**静态 `import "./shared.js"` 属于构建期 compiler 域。** JS parser 由 [HarmonyImportDependencyParserPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L109-L147) 处理：

- `parser.hooks.import` 为 import 语句生成一个清空原语句范围的 `ConstDependency`，再生成 [HarmonyImportSideEffectDependency](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportSideEffectDependency.js)，并通过 `parser.state.module.addDependency()` 加入当前模块的 `dependencies`。
- `parser.hooks.importSpecifier` 记录 imported binding 与 source 的 tag 信息；真正访问导入绑定时，`expression`、`expressionMemberChain`、`callMemberChain` 等 hook 再生成 [HarmonyImportSpecifierDependency](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportSpecifierDependency.js)。

因此在 `entry-a.js` 和 `entry-b.js` 中，指向 `shared.js` 的静态 import 最终都是当前 `Module.dependencies` 上的边；它不会创建 `AsyncDependenciesBlock`。[DependenciesBlock](file:///e:/newGsb/questions/GSB-013/Tony/lib/DependenciesBlock.js#L29-L64) 明确区分普通 `dependencies` 和用于 code-splitting 的 `blocks`。

**普通动态 `import("./async.js")` 同时产生一个异步边界。** [ImportParserPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportParserPlugin.js#L47-L335) 在 `parser.hooks.importCall` 中解析 magic comment：

- 默认 `mode` 来自 parser options 的 `dynamicImportMode`；普通字符串参数且不是 `eager` 或 `weak` 时，会 new [AsyncDependenciesBlock](file:///e:/newGsb/questions/GSB-013/Tony/lib/AsyncDependenciesBlock.js#L24-L41)；
- block 的 `groupOptions` 来自 `webpackChunkName`、`webpackPrefetch`、`webpackPreload`、`webpackFetchPriority` 等注释；
- 再 new [ImportDependency](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportDependency.js#L25-L139)，把它 `depBlock.addDependency(dep)`，最后 `parser.state.current.addBlock(depBlock)`；见 [ImportParserPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportParserPlugin.js#L281-L300)。

所以本例中：

- `entry-a`、`entry-b` 的根 `DependenciesBlock.dependencies` 都有指向 `shared` 的 Harmony dependency；
- `entry-a` 的根 `DependenciesBlock.blocks` 有一个 `AsyncDependenciesBlock`；
- 该 block 的 `dependencies` 里有指向 `async` 的 `ImportDependency`。

这些对象只存在于构建期 compiler 域。浏览器不会执行 `Dependency` 或 `AsyncDependenciesBlock` 类；它们在后续图构建和 code generation 中被翻译成 chunk 关系和运行时代码。

### 4.2 factorize/build 阶段：Dependency 变成 `ModuleGraphConnection`

依赖处理仍由第一版主链路中的 `handleModuleCreation()`、`_factorizeModule()`、`_addModule()`、`_buildModule()`、`_processModuleDependencies()` 队列完成，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1935-L2162)。

关键转换是：

1. factory 根据 dependency 创建 `shared` 和 `async` 模块；普通 JS 走 [NormalModuleFactory](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L869-L953)。
2. `handleModuleCreation()` 在模块 add 后调用 [ModuleGraph.setResolvedModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L213-L243)。它为每条 dependency 创建 `ModuleGraphConnection`：
   - 对 `entry-a -> shared`、`entry-b -> shared`，这是两条 incoming connection，目标都是同一个 `shared` Module；
   - 对 `entry-a -> async`，connection 的 dependency 属于 `AsyncDependenciesBlock`，不是根 block。
3. `_processModuleDependencies()` 遍历 `block.dependencies` 和 `block.blocks`。它调用 [ModuleGraph.setParents](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L176-L187) 记录 dependency 的 parent block/module，再按 factory 分组递归创建后续模块；见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1593-L1664)。

构建结束但尚未 seal 时，图中已有一个 `ModuleGraph`：

- 节点：`entry-a`、`entry-b`、`shared`、`async`；
- 静态边：`entry-a -> shared`、`entry-b -> shared`；
- 异步 block 边：`entry-a.blocks[] -> AsyncDependenciesBlock -> async`；
- 此时还没有 `ChunkGraph`，也没有 chunk。

### 4.3 seal/`buildChunkGraph`：从模块图到初始 chunk 和异步 chunk

`Compilation.seal()` 开始时创建 [ChunkGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3062-L3072)。随后它为每个 entry 创建初始 chunk 和 `Entrypoint`，把入口模块连接为 entry module，再调用 [buildChunkGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1301-L1357)。

对本例要抓住三个动作。

第一，两个入口各自入队初始模块。入口信息来自 seal 阶段创建的 `chunkGraphInit`，`visitModules()` 为没有 parent 的 entrypoint 设置 `minAvailableModules = 0n`，并把入口模块加入队列，见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L380-L437)。队列中的 `ADD_AND_ENTER_MODULE` 会调用 [ChunkGraph.connectChunkAndModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L813-L834)，因此：

- chunk A 连接 `entry-a`；
- chunk B 连接 `entry-b`。

第二，静态依赖中的 `shared` 按当前 chunk group 继续处理。`processBlock()` 通过 `getBlockModules()` 读取模块图连接，并用 `minAvailableModules` 判断父级 chunks 是否已经包含某模块；见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L675-L751)。由于 chunk A 和 chunk B 是两个独立入口初始 group，彼此初始时没有父子关系：

- 处理 `entry-a` 的静态 `shared` dependency 时，`shared` 不在 A 的父级可用模块集合中，于是连接进 chunk A；
- 处理 `entry-b` 的同一静态 dependency 时，`shared` 也不在 B 的父级可用模块集合中，于是连接进 chunk B。

这就是“两个入口共享一个模块”在默认图构建中的结果：**同一个 Module 被两个初始 chunk 同时连接**，但没有自动形成第三个共享 chunk。若要让它抽成独立 chunk，需要 splitChunks、runtime chunk 共享或其他改变图/可用模块集合的优化；这些优化发生在 `buildChunkGraph()` 之后的 optimize hooks 中。

第三，动态 `import()` 的 `AsyncDependenciesBlock` 触发异步 chunk group。处理根 block 的子 block 时，`iteratorBlock()` 发现该 block：

- 若启用 `asyncChunks` 和 chunk loading，就调用 `compilation.addChunkInGroup()` 创建一个新的 `ChunkGroup` 和其中的异步 chunk；
- 相同 `webpackChunkName` 会通过 `namedChunkGroups` 复用已有 group；
- 该连接被记录到 `blockConnections`，并把目标 group 放入 `queueConnect`；见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L483-L669)。

`processConnectQueue()` 和 `processChunkGroupsForMerging()` 会把父 group 的 resulting available modules 合并进异步 group。由于初始 chunk A 已经包含 `shared`，`shared` 对 A 的异步 group 是可用模块；异步 block 再被遍历时，`processBlock()` 遇到已在父级可用集合中的模块会放入 `skippedItems`，不重复连接进异步 chunk，见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L686-L744)。而 `async` 不在父级可用集合中，所以连接到新异步 chunk。

最后 `connectChunkGroups()` 根据 `blockConnections` 做两件事：

- [ChunkGraph.connectBlockAndChunkGroup](file:///e:/newGsb/questions/GSB-013/Tony/lib/ChunkGraph.js#L1315-L1331) 把 `AsyncDependenciesBlock` 映射到生成的 chunk group；
- `connectChunkGroupParentAndChild()` 把入口 A 的初始 group 与异步 group 连为父子；见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1238-L1272)。

因此本例默认结果是：

- 初始 chunk A：`entry-a` + `shared`，父 group 是 entrypoint A；
- 初始 chunk B：`entry-b` + `shared`，父 group 是 entrypoint B；
- 异步 chunk C：`async`，父 group 是 entrypoint A；
- `AsyncDependenciesBlock` 通过 `ChunkGraph.getBlockChunkGroup()` 指向 C 的 group，运行时模板据此知道要加载哪个 chunk。

### 4.4 code generation：静态 import 与动态 import 的翻译方式不同

code generation 由 [Compilation.codeGeneration](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3467-L3499) 调度。每个普通 JS 模块最终调用 [JavascriptGenerator.generate](file:///e:/newGsb/questions/GSB-013/Tony/lib/javascript/JavascriptGenerator.js#L98-L111)。它：

- 遍历 `module.dependencies` 和 `module.blocks`；
- 为每个 dependency 从 `dependencyTemplates` 找 template；
- 把 `runtimeRequirements` 放进 template context；
- 用 `ReplaceSource` 改写原始源码；见 [JavascriptGenerator.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/javascript/JavascriptGenerator.js#L130-L247)。

#### 静态 import：翻译成本模块内的 harmony import 语句

静态 `shared` import 的 template 在 [HarmonyImportDependency.Template.apply](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportDependency.js#L270-L370)。它会先检查 connection 在当前 runtime 是否 active，然后调用 dependency 的 `getImportStatement()`。`RuntimeTemplate.importStatement()` 生成：

- `var <importVar> = __webpack_require__(<moduleId>);`
- 必要时追加 compat default export 代码；
- 同时把 `RuntimeGlobals.require` 加入 runtime requirements；见 [RuntimeTemplate.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeTemplate.js#L790-L850)。

`HarmonyImportSpecifierDependency.Template` 再把源码中使用的导入绑定替换成对已导入 namespace/default/named export 的成员访问，见 [HarmonyImportSpecifierDependency.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportSpecifierDependency.js#L325-L400)。

所以静态 import 在 emitted runtime 中不是浏览器原生跨 chunk 加载；它被翻译成当前 chunk 内的模块注册/require 访问。因为本例默认把 `shared` 同时放入两个初始 chunk，chunk A 和 B 都能同步 `__webpack_require__(sharedId)`。

#### 动态 import：翻译成 `__webpack_require__.e(chunkId).then(...)`

动态 `async` import 的 template 是 [ImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportDependency.js#L107-L137)。它通过 `moduleGraph.getParentBlock(dep)` 找回 parser 阶段创建的 `AsyncDependenciesBlock`，然后调用 [RuntimeTemplate.moduleNamespacePromise](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeTemplate.js#L602-L739)。

这个模板的关键步骤是：

1. 通过 `chunkGraph.getModuleId(module)` 取得 `async` 的模块 id；
2. 调用 [RuntimeTemplate.blockPromise](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeTemplate.js#L986-L1042)；
3. `blockPromise()` 用 [ChunkGraph.getBlockChunkGroup](file:///e:/newGsb/questions/GSB-013/Tony/lib/ChunkGraph.js#L1315-L1321) 找到异步 group，过滤掉 runtime chunk，生成 `__webpack_require__.e(<chunkId>)` 或 `Promise.all([...])`，并把 `RuntimeGlobals.ensureChunk` 加入 runtime requirements；
4. 根据 `async` 的 exports type，再追加 `__webpack_require__.then(...)`、`createFakeNamespaceObject(...)` 等逻辑。

因此入口 A 里的 `import("./async.js")` 在 emitted runtime 中会变成类似：

```js
__webpack_require__.e(/* import() */ <asyncChunkId>).then(__webpack_require__.bind(__webpack_require__, <asyncModuleId>))
```

如果目标模块的 exports type 需要 fake namespace，还会继续 `.then(m => __webpack_require__.nco(m, type))`。具体形状由 `module.getExportsType()` 决定，见 [RuntimeTemplate.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeTemplate.js#L672-L739)。

### 4.5 runtime requirements 与 `RuntimeModule`：浏览器如何真正装载异步 chunk

动态 import template 只声明它需要 [RuntimeGlobals.ensureChunk](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeGlobals.js#L76-L81)。真正的 `__webpack_require__.e` 和脚本加载逻辑由 runtime 域的 `RuntimeModule` 生成。

`seal()` 中 `processRuntimeRequirements()` 会遍历模块和树级 requirements，并触发：

- `additionalModuleRuntimeRequirements`；
- `runtimeRequirementInModule`；
- `additionalTreeRuntimeRequirements`；
- `runtimeRequirementInTree`。

[RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimePlugin.js#L371-L383) 监听 `runtimeRequirementInTree.for(RuntimeGlobals.ensureChunk)`：

- 如果当前 chunk 有异步子 chunk，则进一步要求 `RuntimeGlobals.ensureChunkHandlers`；
- 总是向 chunk 添加 [EnsureChunkRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/runtime/EnsureChunkRuntimeModule.js)。

[EnsureChunkRuntimeModule.generate](file:///e:/newGsb/questions/GSB-013/Tony/lib/runtime/EnsureChunkRuntimeModule.js#L26-L65) 生成两种形态：

- 需要加载非初始 chunk 时：`__webpack_require__.f = {}; __webpack_require__.e = function(chunkId) { return Promise.all(Object.keys(__webpack_require__.f).reduce(...)); }`；
- 所有引用 chunk 都已在当前文件中时：`__webpack_require__.e = () => Promise.resolve()`。这在多入口场景中可能出现。

默认 web/jsonp 输出中，异步脚本加载 handler 由 [JsonpChunkLoadingRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/web/JsonpChunkLoadingRuntimeModule.js#L90-L217) 生成：

- `installedChunks` 记录未加载、加载中、已加载状态；
- `__webpack_require__.f.j(chunkId, promises)` 检查 chunk 是否有 JS，创建 Promise，计算 URL，并调用 `__webpack_require__.l(url, loadingEnded, ...)`；
- [LoadScriptRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/runtime/LoadScriptRuntimeModule.js#L58-L160) 生成创建 `<script>`、处理 onload/onerror、去重 in-progress URL 的逻辑。

异步 chunk 自己的包裹格式由 [ArrayPushCallbackChunkFormatPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/javascript/ArrayPushCallbackChunkFormatPlugin.js#L43-L137) 生成：

- 非初始 chunk 输出为 `(globalThis["webpackChunk..."] = globalThis["webpackChunk..."] || []).push([chunkIds, modules, runtime])`；
- 其中的 modules 包含 `async` 模块工厂；
- runtime chunk 中的 [JsonpChunkLoadingRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/web/JsonpChunkLoadingRuntimeModule.js#L417-L466) 生成 `webpackJsonpCallback`，把 moreModules 写入 `__webpack_require__.m`/module factories，并把 `installedChunks[chunkId]` 置为已加载、resolve 正在等待的 Promise。

因此浏览器执行顺序是：

1. 加载入口 A initial chunk；
2. 用户代码执行到 `import("./async.js")` 翻译出的 `__webpack_require__.e(asyncChunkId)`；
3. `__webpack_require__.e` 调用 `__webpack_require__.f.j`，创建 script 标签加载异步 chunk；
4. 异步 chunk 的 `webpackChunk.push([...])` 执行，注册 `async` 模块工厂并 resolve chunk promise；
5. `.then(__webpack_require__.bind(__webpack_require__, asyncModuleId))` 执行模块工厂，返回模块 namespace；
6. 若异步模块还静态依赖了已在父 chunk 可用的模块，注册工厂中同步 `__webpack_require__()` 即可取到，无需再次下载。

### 4.6 三个执行域中的所有权边界

**只存在于构建期 compiler 域：**

- parser、`Dependency` 子类、`AsyncDependenciesBlock`、`ModuleGraphConnection`；
- `Compilation.modules`、构建队列、factory、resolver；
- `ModuleGraph` 的 connection/export info/profile 等内部结构；
- `ChunkGraph` 中 block-to-group、runtime requirement bookkeeping 等大部分调度数据。

这些对象不会原样写进 bundle。它们只决定模块解析、chunk 切分、id 分配、runtime requirements 和 source 替换。

**被翻译进写入产物的 runtime 域：**

- 静态 import -> harmony import init fragment 和 `__webpack_require__(moduleId)`；
- 动态 import -> `AsyncDependenciesBlock` 对应 chunk id 的 `__webpack_require__.e(chunkId)` Promise；
- module/chunk id、module factories、`installedChunks`、`__webpack_require__.f.j`、`__webpack_require__.l`、jsonp callback；
- 必要时由 `RuntimePlugin` 和输出格式插件生成的 `RuntimeModule` 源码。

**watch/cache 域复用的是构建期结构：**

- 文件变化后，parser/build/cache 可能复用模块或依赖信息；
- 每次 `compile()` 仍会新建 `Compilation`、`ModuleGraph` 和 `ChunkGraph`；
- watch 模式可根据 `modifiedFiles/removedFiles` 和 cache 让某些模块 `stillValidModule`，但 chunk 关系仍在当次 `seal()` 中重新建立。

### 4.7 共享模块和异步块形成/复用 chunk 的条件

本例可作为判断基线：

- 两个独立 initial entry 同时静态依赖 `shared`：同一 Module 会有两条 incoming connections，默认连接进两个初始 chunk；不会仅因“被两个入口共享”就自动抽出。
- 初始 entry A 的动态 `import()` 指向 `async`：只要 `output.asyncChunks` 和 chunk loading 未禁用，就会创建异步 `ChunkGroup` 和 chunk；见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L569-L633)。
- 如果异步边界的目标模块已经在父级 available modules 中，则在该异步 group 中被跳过，不重复放入异步 chunk；见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L707-L744)。
- 相同 `webpackChunkName` 的异步 block 会复用同名 `ChunkGroup`；复用 initial group 会报 `AsyncDependencyToInitialChunkError`，见 [buildChunkGraph.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L580-L630)。
- 若 `webpackMode: "eager"`，parser 不创建 `AsyncDependenciesBlock`，而是生成 `ImportEagerDependency`，因此不会产生按需 chunk；见 [ImportParserPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportParserPlugin.js#L265-L273)。
- 若 `webpackMode: "weak"`，parser 使用 `ImportWeakDependency` 且也不创建异步 block；见 [ImportParserPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportParserPlugin.js#L273-L281)。
- `runtimeChunk`、module concatenation、usedExports/providedExports 等优化也可能在 `seal()` 后续 hooks 中改变 chunk 成员和生成代码；其中 `SplitChunksPlugin` 的关键改写见 5.8，其他优化插件仍需单独展开。


### 5.8 `splitChunks` 如何改写共享模块的 chunk 归属

[SplitChunksPlugin.apply](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L822-L840) 在 `thisCompilation` 中注册，并在 `compilation.hooks.optimizeChunks` 的 `STAGE_ADVANCED` 执行。因此它发生在 [buildChunkGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1301-L1357) 建立初始 chunk graph 之后、code generation 之前。

已证实的关键动作是：

- 插件遍历模块和模块所在 chunks，通过 `getCacheGroups(module, { moduleGraph, chunkGraph })` 取得 cache groups，见 [SplitChunksPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L1240-L1291)。
- 满足 `minChunks`、`minSize`、request 限制等条件的模块集合会形成候选 chunk info，见 [SplitChunksPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L1131-L1364)。
- 生成 chunk 时，可以按 name 复用 `compilation.namedChunks`，也可以按 `reuseExistingChunk` 复用已有 chunk；否则 `compilation.addChunk()` 创建新 chunk，见 [SplitChunksPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L1394-L1547)。
- 对每个源 chunk 调 [Chunk.split](file:///e:/newGsb/questions/GSB-013/Tony/lib/Chunk.js#L542-L551)，它把新 chunk 插入该 chunk 所属的所有 chunk groups，并合并 runtime。
- 随后通过 `chunkGraph.connectChunkAndModule(newChunk, module)` 和 `chunkGraph.disconnectChunkAndModule(chunk, module)` 移动模块，见 [SplitChunksPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L1572-L1589)。

因此，5.1 到 5.7 描述的是 `buildChunkGraph` 后的初始图；若配置命中 splitChunks cache group，被两个入口共享的 `shared` 可以从两个 initial chunk 移到一个共享 initial chunk。它不改变 `ModuleGraph` 中 `entry-a -> shared`、`entry-b -> shared` 的 connection，只改变 `ChunkGraph` 中模块属于哪些 chunk，以及 chunk group 中 chunk 的顺序。动态 import 仍以前文 `AsyncDependenciesBlock -> ChunkGroup` 为边界；如果异步目标模块已经在父级可用模块集合中，仍不会重复放入异步 chunk。


## 6. 写入产物的 runtime 如何形成

“产物 runtime”与构建期 `compiler` 不是同一层。构建期由插件决定需要哪些 runtime requirements，generator/render 再把它们转成 bundle 源码。

关键入口是 [RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimePlugin.js#L106-L260)。它在 `compilation` hook 中：

- 注册 `RuntimeRequirementsDependency` template；
- 对 `runtimeRequirementInModule`、`runtimeRequirementInTree` 建立 requirement 依赖关系；
- 当 tree 需要 `__webpack_require__`、publicPath、module cache、chunk loading 等能力时，向 chunk 添加对应 `RuntimeModule`。

JavaScript 渲染由 [JavascriptModulesPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/javascript/JavascriptModulesPlugin.js#L263-L557) 接入：

- 在 `NormalModuleFactory.hooks.createParser/createGenerator` 上为 JS module types 提供 `JavascriptParser` 和 `JavascriptGenerator`；
- 在 `Compilation.hooks.renderManifest` 中为 chunk 添加 render manifest entry：
  - runtime chunk 走 `renderMain()`；
  - 非 runtime JS chunk 走 `renderChunk()`；
  - hot update chunk 也有单独分支；
- 在 `chunkHash` 和 `contentHash` 中把 bootstrap/runtime module 内容纳入 hash；
- `renderModule()` 从 `codeGenerationResults.get(module, chunk.runtime)` 读取 generator 产出的 `javascript` source；见 [JavascriptModulesPlugin.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/javascript/JavascriptModulesPlugin.js#L582-L606)。

所以最终 bundle runtime 是 `RuntimeModule`、普通模块生成结果、dependency templates、bootstrap/render 代码共同组成的 `Source`，再由 `Compilation.createChunkAssets()` 放入 `compilation.assets`，最后由 `Compiler.emitAssets()` 写出。

## 7. watch/cache 域的失效与复用边界

前几节的 `Compiler`、`Compilation`、parser、`ModuleGraph`、`ChunkGraph` 和 emitted runtime 都属于单次构建中的对象。watch/cache 域要解决的是：这些对象在下一轮构建里哪些能跨轮保留，哪些必须重新解析、重新建图、重新生成或重新写出。

### 7.1 invalidation 从文件事件到新 compile

watch 模式下，[Watching](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js) 并不直接持有原生文件句柄；它通过 `compiler.watchFileSystem.watch()` 委托给 [NodeWatchFileSystem](file:///e:/newGsb/questions/GSB-013/Tony/lib/node/NodeWatchFileSystem.js#L30-L189)。每次 `Watching.watch()` 都：

1. 把上一轮 watcher 放到 `pausedWatcher`，新创建一个 Watchpack；见 [Watching.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L354-L394)。
2. Watchpack aggregated 后，NodeWatchFileSystem 先 `pause()` 新 watcher，purge 掉 changed/removed paths 在 enhanced-resolve/cached input file system 中的缓存，再收集 `fileTimeInfoEntries`、`contextTimeInfoEntries`、`changes`、`removals` 回调；见 [NodeWatchFileSystem.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/node/NodeWatchFileSystem.js#L78-L109)。
3. `Watching` 收到回调后进入 `_invalidate()`。

[Watching._invalidate](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L420-L442) 的核心规则是：

- 如果 `suspend()` 或插件设置的 blocker 生效，只缓存变化，不启动编译；`resume()` 时再 `_invalidate()`。
- 如果当前 `running` 为 true，不立即重入 `compile()`，而是用 [_mergeWithCollected](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L83-L100) 合并 changed/removed，并设置 `this.invalid = true`。
- 如果当前没有编译运行，则直接 `_go()`。

[Watching._go](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L109-L239) 启动新一轮时：

- 若有旧 watcher，调它的 `pause()`，使其暂停上报；
- 设置 `compiler.fsStartTime = Date.now()`；
- 设置 `compiler.fileTimestamps`、`compiler.contextTimestamps`、`compiler.modifiedFiles`、`compiler.removedFiles`；
- 如果 compiler 处于 idle，先 `cache.endIdle()`；
- 首次需要 records 时 `readRecords()`；
- 异步触发 `compiler.hooks.watchRun`；
- 调用 `compiler.compile(onCompiled)`。

这解释了第一个排查重点：**文件改了却看起来复用旧结果时，先确认该文件是否进入了 `compiler.modifiedFiles`/时间戳，以及它是否是当前模块的 `fileDependencies`、`contextDependencies`、`missingDependencies` 或 build dependencies。** 没有登记为依赖的文件变化不会让对应模块 snapshot 失效。

编译运行中再次 invalid 时，[Watching._done](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L281-L345) 会在 store build dependencies 前检查 `this.invalid`；若已失效，当前 compilation 的结果不会走用户 handler/done 汇报，而是直接重新 `_go()`。这会形成“正在 emit 或 done 前又改文件，当前轮结果被丢弃”的行为。

### 7.2 依赖集合和 snapshot：watch 保存的事实

`Compilation` 在构造时创建四类集合，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1183-L1189)：

- `fileDependencies`：已读取文件；
- `contextDependencies`：需要监听目录内容变化的目录；
- `missingDependencies`：构建时缺失、但未来出现后可能影响解析的路径；
- `buildDependencies`：构建配置、loader、插件等影响构建过程的文件。

普通模块构建时，[NormalModule.build](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L989-L1083) 会新建 `buildInfo.fileDependencies/contextDependencies/missingDependencies`，把 loader-runner 返回的 `result.fileDependencies`、`result.contextDependencies`、`result.missingDependencies` 加进去；如果有 loaders，还会把每个 `loader.loader` 加入 `buildDependencies`。

构建和 parse 完成后，[handleBuildDone](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1243-L1332) 调 `compilation.fileSystemInfo.createSnapshot()`，把这些 file/context/missing dependencies 转成 [Snapshot](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L271-L306)，并清空原始集合，只保留 `buildInfo.snapshot`。因此模块跨编译复用时，下一轮不是重新扫描整个模块，而是把 snapshot 中的事实交给 FileSystemInfo 校验。

[FileSystemInfo.createSnapshot](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2169-L2643) 会保存以下事实中的一种或多种：

- file timestamp、file hash，或 timestamp + hash；
- context resolved timestamp、context hash，或二者组合；
- missing path 的 existence；
- managed paths/immutable paths/package manager 信息；
- child snapshots。

默认模块 snapshot 选项来自 `compilation.options.snapshot.module`；hash/timestamp 模式由 options 决定。本文不把默认具体模式写死，因为它可能受配置和版本影响，当前版本以 [createSnapshot 的 mode](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2201-L2202) 和 snapshot options 为准。

下一轮 `_buildModule()` 调 [NormalModule.needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592)，判定顺序为：

1. `_forceBuild` 为 true：重建；
2. 模块上次有 error：重建；
3. `buildInfo.cacheable` 为 false：重建；
4. 没有 snapshot：重建；
5. `valueDependencies` 与 `valueCacheVersions` 不匹配：重建；
6. [FileSystemInfo.checkSnapshotValid](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2803-L3249) 校验 snapshot；
7. 最后还允许 `NormalModule.getCompilationHooks().needBuild` 异步强制重建。

Snapshot 校验的关键比较在 [FileSystemInfo.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2803-L2903)：

- `checkExistence()` 比较 missing/path 是否存在；
- `checkFile()` 比较文件 existence、safeTime/startTime 以及 timestamp；tsh/hash 模式下 timestamp 不可靠时可 fallback 到 hash；
- `checkContext()` 比较目录 existence、safeTime/startTime 以及 `timestampHash`；
- hash snapshot 直接比较 hash。

safeTime/startTime 是避免 timestamp race 的关键：当前条目的 `safeTime` 晚于 snapshot startTime 时，FileSystemInfo 认为它可能在读取后变化，从而判定 invalid；见 [checkFile](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2824-L2858) 和 [checkContext](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2866-L2903)。

### 7.3 从模块到 compilation：依赖如何进入下一轮 watch

`Compilation.seal()` 接近完成时调用 [summarizeDependencies](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4203-L4219)：

- 先合并 child compilations 的四类依赖；
- 再对每个 module 调 [NormalModule.addCacheDependencies](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1621-L1646)。

如果模块已有 snapshot，就从 snapshot 中展开 file/context/missing iterables；否则从 `buildInfo` 中取未转换的集合。buildDependencies 总是从 `buildInfo.buildDependencies` 加入 compilation。

成功编译后，`Watching` 在 `_done()` 中：

- 调 `compiler.cache.storeBuildDependencies(compilation.buildDependencies)`；
- `cache.beginIdle()`；
- 下一个 tick 用 `compilation.fileDependencies`、`contextDependencies`、`missingDependencies` 调 `watch()`，见 [Watching.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L311-L345)。

因此：

- **fileDependencies** 决定已读文件变化是否触发模块重建；
- **contextDependencies** 决定目录中新增/删除/重命名文件是否触发；
- **missingDependencies** 决定之前不存在的文件/目录后来出现时是否触发；
- **buildDependencies** 不用于监听普通模块文件变化，而用于决定整个持久化 cache 是否可恢复。

“什么都没改却整轮重做”的常见原因之一，是目录 context 或 missing path 的 timestamp/hash 被外部工具刷新、snapshot safeTime 判定保守、或者 buildDependencies 被改变；这时即使源码内容没变，snapshot/value dependency 也可能判定需要 rebuild。

### 7.4 `CacheFacade` identifier/etag：编译期对象复用的 key

每次 `Compiler.compile()` 新建 [Compilation](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L993-L1209)，但它通过 [Compiler.getCache](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L327-L337) 获得带前缀的 [CacheFacade](file:///e:/newGsb/questions/GSB-013/Tony/lib/CacheFacade.js#L196-L300)。CacheFacade 把：

- cache name，例如 `Compilation/modules`；
- item identifier，例如 module identifier；
- etag，例如 null、module hash 或 content hash；

组合成底层 [Cache](file:///e:/newGsb/questions/GSB-013/Tony/lib/Cache.js#L53-L160) hook 的字符串 key。[ItemCacheFacade](file:///e:/newGsb/questions/GSB-013/Tony/lib/CacheFacade.js#L98-L160) 的 `get()`/`store()` 最终就是 `cache.get(name, etag)` 和 `cache.store(name, etag, data)`。

etag 可以是字符串或支持 `updateHash()` 的对象；[getLazyHashedEtag](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/getLazyHashedEtag.js#L19-L40) 会惰性调用 `obj.updateHash(hash)` 并做 base64 digest。

Compilation 中有三层最直接相关的 cache：

1. **Module cache**：`_modulesCache = getCache("Compilation/modules")`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1203-L1205)。`_addModule()` 用 `module.identifier()` 和 etag `null` 读取；成功 build 后用相同 identifier 存入，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1419-L1455) 和 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1529-L1539)。模块是否可用不是靠这里的 etag，而是靠前面的 `needBuild()`。
2. **Code generation cache**：`_codeGenerationCache = getCache("Compilation/codeGeneration")`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1203-L1205)。item key 是 `module.identifier()|runtime`，etag 是 `moduleHash|dependencyTemplates.getHash()`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3622-L3643)。
3. **Asset/render cache**：`_assetsCache = getCache("Compilation/assets")`，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1203-L1205)。`createChunkAssets()` 对 render manifest 的每个 entry，用 `fileManifest.identifier` 作为 key、`fileManifest.hash` 作为 etag 读取缓存；cache miss 才 render 并 store，见 [Compilation.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4945-L5052)。

所以复用边界是：

- 模块没有 rebuild，通常可以复用 module object 和其上的 parse/AST/dependency 结果；
- 模块 rebuild 了，但 module hash 或 dependency template hash 没变，理论上 code generation result 可复用；实际是否复用仍以 etag 命中为准；
- module/chunk/runtime/template/content hash 没变，asset `Source` 可复用；
- 即使 `Source` 复用，`Compiler.emitAssets()` 仍会根据磁盘状态、`compareBeforeEmit`、immutable 信息决定是否写盘；见第一版资源写出链路和 [Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L676-L1021)。

### 7.5 filesystem pack cache：保存和验证的事实

filesystem cache 由 `cache.type: "filesystem"` 的策略接入。核心类是 [PackFileCacheStrategy](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1064-L1547)，它把大量 CacheFacade item 存储到 pack content 中，并用一个 [PackContainer](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L37-L86) 保存元事实：

- `version`：cache version；
- `buildSnapshot`：build dependencies 的 snapshot；
- `buildDependencies`：未解析的构建依赖集合；
- `resolveResults`：build dependencies 解析结果；
- `resolveBuildDependenciesSnapshot`：解析 build dependencies 时自身依赖的 snapshot；
- `data`：实际 pack items。

恢复 pack 时，[_openPack](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1151-L1309) 依次验证：

1. index pack 文件能反序列化为 `PackContainer`；
2. `packContainer.version === version`；
3. `buildSnapshot` 仍有效；
4. `resolveBuildDependenciesSnapshot` 仍有效；如果失效，则用 `checkResolveResultsValid()` 再判断一次；
5. 都有效才恢复 pack metadata 和 items；否则返回空 Pack。

item 读取走 [restore](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1325-L1343)，Pack 内还会比较 item 的 etag，见 [Pack.get](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L161-L177)。因此即使 pack 可打开，某个 module/codegen/asset item 的 etag 不匹配也只会返回 miss。

每轮结束后，`Watching._done()` 或非 watch 的 `Compiler.run()` 调 `cache.storeBuildDependencies(compilation.buildDependencies)`。[IdleFileCachePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/IdleFileCachePlugin.js#L94-L103) 把它加入 pending idle task，最终由 [PackFileCacheStrategy.storeBuildDependencies](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1345-L1351) 合并到 `newBuildDependencies`。

真正写 pack 在 [afterAllStored](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1353-L1534)：

- 若有新的 build dependencies 或还没有 buildSnapshot，先 `resolveBuildDependencies()`；
- 为解析过程的依赖创建 `resolveBuildDependenciesSnapshot`；
- 为解析出的 files/directories/missing 创建 `buildSnapshot`；
- 序列化新的 `PackContainer` 到 `cacheLocation/index.pack...`；
- 更新内部 buildDependencies 集合。

[IdleFileCachePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/IdleFileCachePlugin.js#L178-L228) 负责 idle 调度：beginIdle 后等待 idle timeout，再分批执行 pending tasks，之后 `afterAllStored()`；新一轮编译开始时 `endIdle` 会取消 timer。因此“文件改了但磁盘 cache 仍旧结果”的排查还要区分：是 memory item 命中、disk pack 恢复成功，还是 buildDependencies snapshot 没有覆盖真正的配置/loader 变化。

### 7.6 失效边界与判定表

| 变化类型 | watch/snapshot 如何发现 | 模块解析/构建图 | Code generation | Asset/写出 | 主要依据 |
|---|---|---|---|---|---|
| 普通源码文件内容变化，且文件已在 `fileDependencies` | FileSystemInfo snapshot 比较 timestamp/hash/existence；`needBuild()` 返回 true | 受影响模块重新 parse，dependency 和 `ModuleGraph` connection 重建；后续模块按图传播 | module hash 改变，codegen cache etag 失效 | render manifest content hash 变化，asset cache miss 后重新 render；通常需要写盘 | [NormalModule.needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592), [FileSystemInfo.checkFile](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2824-L2858) |
| loader 额外登记的普通文件依赖变化 | loader-runner 结果进入 module 的 `fileDependencies`，最终转成 snapshot；校验失败则 rebuild | 重新跑 loader/parse，依赖结果可能变化 | module/build hash 通常变化，codegen etag 失效 | 受影响 chunk 的 content hash 变化则重新生成/写出 | [NormalModule.build](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1063-L1083), [handleBuildDone](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1314-L1332) |
| loader 或 loader 配置/构建依赖变化 | 属于 `buildDependencies`；filesystem pack 恢复前验证 `buildSnapshot` 和 resolve results | 若 buildDependencies 失效，pack 不恢复，模块/解析重新计算；即使模块文件内容没变，也不能安全复用旧构建过程 | pack 中 codegen items 不会作为有效 cache 恢复 | pack 中 asset items 也不会作为有效 cache 恢复 | [NormalModule.build](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1075-L1081), [PackFileCacheStrategy._openPack](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1203-L1286) |
| 缺失依赖出现 | 路径必须已在 `missingDependencies` 并被 watch；snapshot 比较 existence，从 missing 变 existing 会 invalid | 相关模块需要重新解析；常见于可选文件、扩展路径、生成文件出现 | 若解析结果影响模块依赖或 hash，则 codegen 失效 | 受影响 asset 重新生成/写出 | [FileSystemInfo.createSnapshot](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2555-L2583), [checkExistence](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2803-L2816) |
| context directory 中新增/删除文件，但入口文件本身没变 | 路径必须在 `contextDependencies`；context snapshot 比较 `timestampHash`/hash，变化则 rebuild | 重新解析目录，可能发现新模块或移除旧模块 | 解析结果影响 graph/hash 时 codegen 失效 | chunk graph/content hash 变化则重新生成 asset | [FileSystemInfo.checkContext](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2866-L2903), [Watching.watch](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L354-L394) |
| 编译过程中再次 invalid | `Watching.running` 时只合并 changed/removed 并设 `invalid=true` | 当前 compile 继续完成，但 `_done()` 发现 invalid 后立即重新 `_go()` | 当前 codegen/asset 可能已计算，但结果不汇报；下一轮重新 compile/判定 cache | 当前 emit/done 可能被跳过或结果不汇报；下一轮重新 emit | [Watching._invalidate](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L420-L442), [Watching._done](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L281-L300) |
| 只有未影响 module hash/dep template hash 的内部变化 | 可能 rebuild，但如果 hash 结果一致 | 依赖或内部数据可能更新，但 codegen etag 仍匹配 | code generation result 可从 cache 复用 | 若 manifest identifier/content hash 不变，asset `Source` 可复用；写盘仍由 emit 比较逻辑决定 | [code generation cache](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3636-L3644), [asset cache](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4951-L5052) |
| 输出 asset 内容没变但磁盘文件被删或 mtime 变化 | 这不是 module snapshot 问题，而是 emit 阶段状态问题 | 不需要重新建图 | 不需要重新 codegen，可复用内存 asset source | `emitAssets()` 根据 stat/readFile/immutable/compareBeforeEmit 判断是否重新 writeFile | [Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L893-L982) |

### 7.7 两类问题的排查顺序

**“文件改了却复用旧结果”优先检查：**

1. 文件事件是否进入 Watchpack，并在 `NodeWatchFileSystem` aggregated 回调中 purge 缓存；
2. `compiler.modifiedFiles`/`removedFiles` 是否包含该路径；
3. 该路径是否被模块加入 `fileDependencies`/`contextDependencies`/`missingDependencies`；若是 loader 生成的依赖，检查 loader 是否正确调用 `this.addDependency/addContextDependency/addMissingDependency`；
4. `NormalModule.needBuild()` 是否因为 cacheable/snapshot/valueDependencies 返回 false；
5. filesystem cache 的 buildDependencies/version 是否让旧 pack 被恢复，但 item etag 实际应变化；
6. 动态 import 对应异步块是否复用旧 chunk asset；需要看 render manifest hash 和 codegen etag，而不是只看文件时间。

**“什么都没改却整轮重做”优先检查：**

1. 是否有 context dependency 的目录 timestampHash 被编辑器、临时文件、代码生成器刷新；
2. 是否有 missing dependency 在不存在/存在之间抖动，导致解析路径变化；
3. buildDependencies 是否包含不稳定路径、软链、loader 生成文件或配置文件；
4. `fsStartTime`/safeTime 是否因文件系统时间精度而保守判定 snapshot invalid；
5. 插件是否在 `watchRun`、`normalModuleFactory`、`needBuild` 等 hook 中强制 invalidate 或修改 value cache versions；
6. 是否是编译中连续保存触发 `Watching.invalid`，导致上一轮结果被丢弃。这不是复用错误，而是调度层主动重编译。

## 8. 事故复盘场景：多 entry、共享依赖、动态 import、splitChunks、filesystem cache、watch

为了让前述链路可用于复盘，固定一个确定性配置场景（这里是说明，不是仓库中的实际配置文件）：

- `entry.app = "./src/app.js"`，`entry.admin = "./src/admin.js"`。
- `app.js` 和 `admin.js` 都静态 `import "./shared"`。
- `app.js` 还调用普通动态 `import("./lazy")`。
- 一个自定义 loader 处理 `*.component.html`；该 loader 通过 `this.addDependency(externalFile)` 登记外部数据文件。
- `optimization.splitChunks.cacheGroups.shared` 明确配置为 `chunks: "all"`、`name: "shared"`、`minChunks: 2`、`enforce: true`，用于把 `shared.js` 抽成初始共享 chunk。
- `cache.type = "filesystem"`，`cache.cacheLocation` 固定，`cache.buildDependencies.config` 包含 webpack 配置和自定义 loader；未额外配置 `optimization.runtimeChunk`。

### 8.1 冷启动构建与首次写出

1. **public API 到 Compiler。** `webpack(options)` 经 [webpack.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/webpack.js#L121-L193) 规范化配置、创建 [Compiler](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L137-L325)，再由 [WebpackOptionsApply](file:///e:/newGsb/questions/GSB-013/Tony/lib/WebpackOptionsApply.js#L78-L793) 注册内建插件。这里创建的是长期存在的 compiler；watch/cache 都挂在它上面。
2. **compile 创建当次图对象。** [Compiler.compile](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1310-L1355) 为当次构建创建 `NormalModuleFactory`、`ContextModuleFactory` 和新的 `Compilation`。[Compilation](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L993-L1209) 持有新的 [ModuleGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L128-L161)，但 `ChunkGraph` 要到 seal 才创建。
3. **入口和模块 factory。** [EntryPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/EntryPlugin.js#L33-L52) 在 `make` 中 `addEntry()`，随后进入 [handleModuleCreation](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1935-L2072) 的 factorize/add/build/processDependencies 队列。普通模块由 [NormalModuleFactory.create](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L869-L953) 解析 request、resource、rules 和 loaders，再创建 [NormalModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L337-L380)。
4. **首次 build。** 冷启动时模块通常没有有效 snapshot，[needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592) 返回 true。[NormalModule.build](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1175-L1370) 清空旧状态后调用 loader-runner；自定义 loader 读取外部文件时必须调用 `addDependency`，否则该外部文件不会进入 snapshot。parser 为静态 import 生成 Harmony dependencies，为动态 `import("./lazy")` 生成 `AsyncDependenciesBlock` 和 `ImportDependency`。
5. **ModuleGraph。** `_processModuleDependencies()` 遍历 dependencies/blocks，调用 [ModuleGraph.setResolvedModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L213-L243) 建边。此时图中有 `app -> shared`、`admin -> shared`、`app -> async block -> lazy` 等关系。
6. **初始 ChunkGraph。** `seal()` 创建 [ChunkGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3062-L3072)，并由 [buildChunkGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1301-L1357) 为 app/admin 创建 initial chunks，为 lazy 创建 async chunk group。若没有 splitChunks，`shared` 会在两个 initial chunks 中；本场景配置了 splitChunks，下一步会改写它。
7. **splitChunks 改写。** [SplitChunksPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L822-L840) 在 `optimizeChunks` STAGE_ADVANCED 运行，把满足 cache group 的 `shared` 移到名为 `shared` 的新 initial chunk；[Chunk.split](file:///e:/newGsb/questions/GSB-013/Tony/lib/Chunk.js#L542-L551) 把该 chunk 插入 app/admin 的 chunk groups，然后 [connect/disconnect](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L1572-L1589) 修改 ChunkGraph。
8. **codegen 和 runtime。** [codeGeneration](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3467-L3681) 按 module/runtime 生成 `Source`；静态 import 被翻译为 harmony import/`__webpack_require__`，动态 import 由 [ImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportDependency.js#L107-L137) 调 [RuntimeTemplate.moduleNamespacePromise](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeTemplate.js#L602-L739)，进一步通过 [blockPromise](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimeTemplate.js#L986-L1042) 生成 `__webpack_require__.e(lazyId)`。[RuntimePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimePlugin.js#L371-L383) 根据 `ensureChunk` requirement 添加 [EnsureChunkRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/runtime/EnsureChunkRuntimeModule.js#L26-L65)。
9. **assets 与首次写出。** [createChunkAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4914-L5070) 生成内存 assets；[Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L676-L1021) 写磁盘。默认 `output.compareBeforeEmit` 为 true，见 [defaults.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/config/defaults.js#L1148-L1148)，因此内容未变的输出文件可跳过 writeFile。
10. **watch 与持久化。** 成功后 [summarizeDependencies](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4203-L4219) 汇总 file/context/missing/build dependencies；[Watching._done](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L303-L345) 调 `cache.storeBuildDependencies`，并用 file/context/missing dependencies 重新 watch。idle 后 [IdleFileCachePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/IdleFileCachePlugin.js#L178-L228) 触发 [PackFileCacheStrategy.afterAllStored](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1353-L1534)，把 pack、version、buildSnapshot 和 item etags 写入 cacheLocation。

### 8.2 watch 中入口源码变化

当 `app.js` 变化：

- Watchpack aggregated 后，[NodeWatchFileSystem](file:///e:/newGsb/questions/GSB-013/Tony/lib/node/NodeWatchFileSystem.js#L78-L109) purge 相关缓存并回传 changes/removals；[Watching._invalidate](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L420-L442) 在未运行时进入 `_go()`。
- `app.js` 已在 `fileDependencies` 中，[FileSystemInfo.checkSnapshotValid](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2803-L3249) 判定 snapshot 失效，[NormalModule.needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592) 返回 true。
- `app` 模块 rebuild 时会 [clearDependenciesAndBlocks](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1180-L1184)，因此旧 dependency/block 不会残留；重新 parse 后生成新的 Harmony dependency 和新的 `AsyncDependenciesBlock`。
- `shared`、`lazy` 若自身 snapshot 有效，则不重新跑 loader/parser；它们的 Module 对象和 buildInfo 可复用于当次 Compilation。图关系仍由 `app` 的新 dependencies 重新连接。
- SplitChunksPlugin 重新运行；若 import 关系未变，`shared` 仍移动到 `shared` initial chunk，`lazy` 仍是 async chunk。
- `app` 的 module hash 改变，使 [code generation cache](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3636-L3644) etag 失效；`shared`、`lazy` 的 codegen etag 不变时可复用。asset cache 以 [render manifest hash](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4951-L5052) 判定，app asset 重新 render，未变 chunk asset 可复用 `Source`。
- emit 阶段默认 compareBeforeEmit，因此未变的 shared/lazy/runtime 文件可能不重写；变化的 app 文件会写盘。

### 8.3 loader 登记的外部文件变化

如果自定义 loader 正确调用 `this.addDependency(externalFile)`：

- `externalFile` 进入 loader-runner 返回的 `result.fileDependencies`，随后被 [NormalModule._doBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1063-L1083) 加入 `buildInfo.fileDependencies`，再经 [createSnapshot](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1314-L1332) 保存。
- 即使 `app.js` 本身没变，`externalFile` 的 timestamp/hash/existence 变化也会让 snapshot 失效，`app` rebuild，loader 重新执行。
- rebuild 后如果生成内容和 dependency graph 与上一轮一致，模块仍可能重新构建但 codegen/asset etag 不变，从而复用后续产物；这是“重建但不重写资产”的正常分层。

如果 loader 没有调用 `addDependency`：

- `externalFile` 不在 `fileDependencies` 或 snapshot 中；Watchpack 也不会监听它。
- `app` 的 snapshot 可能继续有效，[stillValidModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1519-L1526) 复用旧模块，旧 source/AST/dependency 会继续参与图和代码生成。
- 这正是“文件改了却复用旧结果”的典型 loader 边界问题。它不是 ChunkGraph 或 runtime 复用，而是模块 build 失效事实缺失。

### 8.4 构建过程中再次 invalid

若文件在一次编译运行中再次变化：

- [Watching._invalidate](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L431-L434) 不重入 `compile()`，只合并 changed/removed 并设置 `this.invalid = true`。
- 当前 Compilation 可能已经走完 build、seal、codegen，甚至已经进入或完成 [emitAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L676-L1021)；webpack 不会仅因 `invalid` 标志在中途停止当前 emit。
- [Watching._done](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L281-L300) 在汇报用户 callback/done 前检查 `invalid`；若已失效，它不会汇报当前 Stats，而是立即再次 `_go()`。
- 因此事故现象可能是：磁盘上出现了当前轮写出的文件，但用户没有收到对应 Stats，下一轮编译很快又覆盖它们。这属于 watch 调度层的“丢弃结果”，不是文件缓存错误。

### 8.5 下一次重启命中持久缓存

进程重启后：

- compiler 仍是同一套配置，但每次 [Compiler.compile](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1310-L1355) 仍会创建新的 NormalModuleFactory 和 Compilation。
- filesystem cache 插件在编译早期恢复 [PackFileCacheStrategy](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1151-L1309)。只有 [PackContainer.version](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L37-L86)、buildSnapshot、resolveBuildDependenciesSnapshot 都有效时，pack metadata 和 items 才可恢复。
- `_addModule()` 通过 [Compilation/modules cache](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L1419-L1455) 恢复模块对象；恢复出的模块仍要经过 [needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592)。snapshot 有效的模块触发 `stillValidModule`，不跑 loader/parser；snapshot 失效的模块才 rebuild。
- 若 `shared`、`lazy` 和未变入口的 snapshot 有效，模块构建阶段被跳过；SplitChunksPlugin 仍会在 seal 中基于恢复后的模块和依赖重新计算 ChunkGraph。
- codegen/asset cache item 还要再比较 etag：module hash、dependency template hash、render manifest hash 没变才复用 [code generation](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3622-L3681) 和 [asset Source](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4945-L5052)。
- `emitAssets()` 再根据磁盘状态和 `compareBeforeEmit` 决定是否 writeFile。因此“命中缓存”不等于“一定写盘”，也不等于“完全不运行 seal”；它表示可安全复用模块构建、codegen 或 asset 结果。
- 如果 webpack 配置、loader 文件或其他 buildDependencies 变化，[buildSnapshot 校验失败](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1203-L1286)，整个 pack 不恢复，表现为接近冷启动的全量重建。

### 8.6 复盘时的对象所有权速查

- **Compiler 域长期持有**：`Compiler`、`resolverFactory`、`Cache`、watch 状态、输出文件发射缓存。
- **每次 Compilation 新建**：`NormalModuleFactory`、`ContextModuleFactory`、`Compilation`、`ModuleGraph`、`ChunkGraph`、`CodeGenerationResults`、assets map。
- **可跨 Compilation/进程恢复**：NormalModule 对象和 buildInfo/snapshot 通过 `Compilation/modules` cache 与 filesystem pack 恢复；是否可用由 snapshot/valueDependencies/`needBuild` hook 决定。
- **只存在于构建期**：Dependency、AsyncDependenciesBlock、ModuleGraphConnection、ChunkGraph bookkeeping；它们不直接进入浏览器 runtime。
- **进入 runtime 域**：generator 输出、dependency templates、RuntimeModule 生成的 `__webpack_require__`、`__webpack_require__.e`、jsonp callback、module factories 等字符串。
- **watch/cache 域负责**：changed/removed 集合、snapshot、CacheFacade identifier/etag、PackFileCacheStrategy 的 version/buildSnapshot/item etag。


## 9. 证据索引：真实路径、symbol 与 hook

| 主题 | 文件路径与 symbol/hook | 已证实结论 |
|---|---|---|
| public API 到 Compiler | [webpack.js](file:///e:/newGsb/questions/GSB-013/Tony/lib/webpack.js#L121-L193), [WebpackOptionsApply.process](file:///e:/newGsb/questions/GSB-013/Tony/lib/WebpackOptionsApply.js#L78-L793) | 配置校验/规范化、插件应用、内建设施注册、watch/run 分流顺序。 |
| Compiler 生命周期 | [Compiler.run](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L474-L609), [Compiler.compile](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L1310-L1355) | beforeRun/run/readRecords、compile、make、finishMake、finish/seal/afterCompile、emit/done 的顺序。 |
| Compilation/ModuleGraph 所有权 | [Compilation constructor](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L993-L1209), [ModuleGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/ModuleGraph.js#L128-L243) | Compilation 每轮新建；ModuleGraph 保存 parent/resolved module/connections；seal 时 freeze。 |
| ChunkGraph 创建 | [Compilation.seal](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3049-L3427), [ChunkGraph constructor](file:///e:/newGsb/questions/GSB-013/Tony/lib/ChunkGraph.js#L245-L260) | ChunkGraph 在 seal 开始创建，记录 chunk/module/runtime/block 关系。 |
| 模块工厂 | [NormalModuleFactory.create](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L869-L953), [NormalModuleFactory hooks](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L274-L311) | beforeResolve/factorize/resolve/afterResolve/createModule/module 的顺序和 hook 类型。 |
| request/resource/loader | [NormalModuleFactory.resolve](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModuleFactory.js#L419-L853) | inline loader 前缀、matchResource、scheme、resolver、rules、pre/normal/post loader 合并顺序。 |
| loader 执行边界 | [NormalModule._doBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L916-L1087), [NormalModule._createLoaderContext](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L594-L844) | loaderContext API、readResource hook、loader-runner 结果如何变成 source/sourceMap/AST 和依赖集合。 |
| 模块是否重建 | [NormalModule.needBuild](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1540-L1592), [FileSystemInfo.checkSnapshotValid](file:///e:/newGsb/questions/GSB-013/Tony/lib/FileSystemInfo.js#L2803-L3249) | forceBuild/error/cacheable/valueDependencies/snapshot/needBuild hook 的判定顺序。 |
| 静态 import | [HarmonyImportDependencyParserPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportDependencyParserPlugin.js#L109-L219), [HarmonyImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/HarmonyImportDependency.js#L270-L370) | parser 生成 side-effect/specifier dependency，template 生成 harmony import 和 `__webpack_require__`。 |
| 动态 import / async block | [ImportParserPlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportParserPlugin.js#L47-L335), [AsyncDependenciesBlock](file:///e:/newGsb/questions/GSB-013/Tony/lib/AsyncDependenciesBlock.js#L24-L115), [ImportDependency.Template](file:///e:/newGsb/questions/GSB-013/Tony/lib/dependencies/ImportDependency.js#L107-L137) | 动态 import 创建 async block/dependency，block 映射到 chunk group，template 生成 namespace promise。 |
| 初始和异步 chunk 图 | [buildChunkGraph](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1301-L1357), [visitModules](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L380-L1208), [connectChunkGroups](file:///e:/newGsb/questions/GSB-013/Tony/lib/buildChunkGraph.js#L1216-L1273) | initial chunks、async groups、available modules、skipped modules、父子 chunk group 连接规则。 |
| splitChunks | [SplitChunksPlugin.apply](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L822-L840), [chunk split/connect](file:///e:/newGsb/questions/GSB-013/Tony/lib/optimize/SplitChunksPlugin.js#L1394-L1589), [Chunk.split](file:///e:/newGsb/questions/GSB-013/Tony/lib/Chunk.js#L542-L551) | optimizeChunks advanced 阶段移动模块到新 chunk 并修改 chunk groups。 |
| codegen/asset cache | [Compilation.codeGeneration](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L3467-L3681), [createChunkAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4914-L5070) | codegen key/etag 和 render manifest identifier/hash 如何控制复用。 |
| runtime loading | [RuntimePlugin ensureChunk](file:///e:/newGsb/questions/GSB-013/Tony/lib/RuntimePlugin.js#L371-L383), [EnsureChunkRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/runtime/EnsureChunkRuntimeModule.js#L26-L65), [JsonpChunkLoadingRuntimeModule](file:///e:/newGsb/questions/GSB-013/Tony/lib/web/JsonpChunkLoadingRuntimeModule.js#L90-L468) | `__webpack_require__.e`、`__webpack_require__.f`、installedChunks、script 加载和 webpack chunk callback。 |
| watch invalidation | [NodeWatchFileSystem](file:///e:/newGsb/questions/GSB-013/Tony/lib/node/NodeWatchFileSystem.js#L30-L189), [Watching](file:///e:/newGsb/questions/GSB-013/Tony/lib/Watching.js#L83-L442) | changed/removed 汇总、watcher pause/resume、running 时只标记 invalid、新一轮 `_go()`。 |
| dependency summary | [Compilation.summarizeDependencies](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compilation.js#L4203-L4219), [NormalModule.addCacheDependencies](file:///e:/newGsb/questions/GSB-013/Tony/lib/NormalModule.js#L1621-L1646) | file/context/missing/build dependencies 如何汇总给下一轮 watch 和 cache。 |
| CacheFacade/etag | [CacheFacade](file:///e:/newGsb/questions/GSB-013/Tony/lib/CacheFacade.js#L98-L300), [getLazyHashedEtag](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/getLazyHashedEtag.js#L19-L40) | cache name/identifier/etag 组合为底层 Cache key，etag 惰性 hash。 |
| filesystem pack | [PackFileCacheStrategy._openPack](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1151-L1309), [afterAllStored](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/PackFileCacheStrategy.js#L1353-L1534), [IdleFileCachePlugin](file:///e:/newGsb/questions/GSB-013/Tony/lib/cache/IdleFileCachePlugin.js#L58-L228) | pack 恢复前验证 version/buildSnapshot/resolve snapshot，idle 时解析 build dependencies 并序列化 pack。 |
| 资源写出 | [Compiler.emitAssets](file:///e:/newGsb/questions/GSB-013/Tony/lib/Compiler.js#L676-L1021), [config defaults compareBeforeEmit](file:///e:/newGsb/questions/GSB-013/Tony/lib/config/defaults.js#L1148-L1148) | 内存 asset 到磁盘、immutable/compareBeforeEmit、assetEmitted/afterEmit 边界。 |


## 10. hook 短路、异步边界和多轮编译要点

- `compiler.hooks.shouldEmit`：`SyncBailHook`，返回 `false` 跳过 `emitAssets()`，但不跳过 `done`。
- `compiler.hooks.entryOption`：`SyncBailHook`，内建 `EntryOptionPlugin` 返回 `true` 后短路。
- `NormalModuleFactory.hooks.beforeResolve`：`AsyncSeriesBailHook`，返回 `false` 可忽略模块请求。
- `Compilation.hooks.optimizeDependencies/optimizeModules/optimizeChunks`：均为 `while (hook.call(...)) {}`，tap 返回真值会触发新一轮优化。
- `Compilation.hooks.optimizeTree` 和 `optimizeChunkModules`：异步边界，错误会包装成对应 hook 名的 `HookWebpackError`。
- `compiler.hooks.make`：`AsyncParallelHook`，不要假设入口插件串行完成。
- `compiler.hooks.finishMake`：当前源码中是 `AsyncSeriesHook`。
- `Compilation.hooks.needAdditionalSeal`：返回真值会 `unseal()` 并递归 `seal()`。
- `compilation.hooks.needAdditionalPass` 与 `compiler.hooks.additionalPass`：会在 emit 后重新 `compile()`，形成额外编译轮次。
- `Compilation.hooks.afterHash` 位于部分延后 code generation jobs 之前；涉及 full hash runtime module 时要特别注意。
- watch 模式下 `Watching.invalid` 可以让正在进行的 emit/done 结果不汇报，直接进入下一轮编译。

## 11. 未证实范围

以下内容本轮没有逐行确认，后续接手时不要把本文当作这些细节的最终结论：

- loader-runner 包内部 pitch 提前返回、跳过后续 loader 的具体实现；当前仓库只通过 webpack 调用点和类型声明确认边界；
- `JavascriptModulesPlugin.renderMain()`/`renderChunk()` 中 bootstrap、init fragments、startup 拼接的完整生成细节；
- `JavascriptModulesPlugin.renderMain()`/`renderChunk()` 中 bootstrap、init fragments、startup 拼接的完整细节；
- `MultiCompiler` 的并行/依赖调度细节；
- 除 `SplitChunksPlugin` 外，其他内建 optimization 插件如何逐字段修改 module/chunk graph；
- 默认 snapshot 选项在具体配置下最终选择 timestamp、hash 还是 tsh 的值；这由 snapshot options 和运行时解析决定，本文只确认 `createSnapshot()` 的 mode 分支。
