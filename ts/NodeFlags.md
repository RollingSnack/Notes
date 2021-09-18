## 常用

### 0 None

### 1 Let

带有 `let` 的变量声明。（不携带分号）

### 2 Const

带有 `const` 的变量声明。（不包含分号）

### 32 OptionalChain

指 `?.` 链的用法。

```TypeScript
const a = {
  b: 'b',
  c: {
    d: 'd'
  }
}

d = a.c?.d;  // d
f = a.e?.f;  // undefined
```

整条链前后都属于其范围，且有多个 `?.` 时，可以在问号处分割。

### 4096 DisallowInContext

不允许 `in-expressions` 的环境下解析 Node。
如 `for (const dir of dirs)` 环境下的 `dir`。k

### 16384 DecoratorContext

装饰器 `@` 用法。

不过该标志只识别装饰器的内容，如 `@promised` 只识别 `promised` 部分。

### 32768 AwaitContext

export = {}

### 32769

### 32770

### 32800

### 8388608

### 8388610

### 134217728

## 完整列表

| NodeFlags | Number |
|:---------:|:------:|
||None|0|
|Let|1|
|Const|2|
|NestedNamespace|4|
|Synthesized|8|
|Namespace|16|
|OptionalChain|32|
|ExportContext|64|
|ContainsThis|128|
|HasImplicitReturn|256|
|HasExplicitReturn|512|
|GlobalAugmentation|1024|
|HasAsyncFunctions|2048|
|DisallowInContext|4096|
|YieldContext|8192|
|DecoratorContext|16384|
|AwaitContext|32768|
|ThisNodeHasError|65536|
|JavaScriptFile|131072|
|ThisNodeOrAnySubNodesHasError|262144|
|HasAggregatedChildData|524288|
|JSDoc|4194304|
|JsonFile|33554432|
|BlockScoped|3|
|ReachabilityCheckFlags|768|
|ReachabilityAndEmitFlags|2816|
|ContextFlags|25358336|
|TypeExcludesFlags|40960|