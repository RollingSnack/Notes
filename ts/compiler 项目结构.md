## compiler 项目结构

---

[[TypeSciprt]] 涉及到编译器的核心代码在 `src/compiler` 目录下，目前主要包含：

```shell
+-- factory/
+-- transformers/
+-- binder.ts
+-- builder.ts
+-- builderPublic.ts
+-- builderState.ts
+-- builderStatePublic.ts
+-- checker.ts
+-- commandLineParser.ts
+-- core.ts
+-- corePublic.ts
+-- debug.ts
+-- diagonsticMessages.json
+-- emitter.ts
+-- moduleNameResolver.ts
+-- moduleSpecifier.ts
+-- parser.ts
+-- path.ts
+-- perfLogger.ts
+-- performance.ts
+-- performanceCore.ts
+-- program.ts
+-- resolutionCache.ts
+-- scanner.ts
+-- semver.ts
+-- sourcemap.ts
+-- symbolWalker.ts
+-- sys.ts
+-- tracing.ts
+-- transformer.ts
+-- tsbuild.ts
+-- tsbuildPublic.ts
+-- tsconfig.json
+-- tsconfig.release.json
+-- types.ts
+-- utilities.ts
+-- utilitiesPublic.ts
+-- visitorPublic.ts
+-- watch.ts
+-- watchPublic.ts
+-- watchUtilities.ts
```