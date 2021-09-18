## API

### createSourceFile

```TypeScript
source = ts.createSourceFile(
  fileName: string,
  sourceText: string,
  languageVersion: ScriptTarget,
  setParentNode?: boolean,
  scriptKind?: ScriptKind
): SourceFile;
```

参数
- fileName，代表文件名的字符串，没什么太大影响
- sourceText，源文件的字符串，主要用于创建的参数
- languageVersion // TODO
- setParentNode?，可选参数，是否在节点中同时保留父节点的信息
- scriptKind // TODO

返回
- [[#SourceFile]]

### forEachChild

```TypeScript
function ts.forEach<T>(
  node: Node,
  cbNode: (node: Node) => T | undefined,
  cbNodes?: (nodes: NodeArray<Node>) => T | undefined
): T | undefined
```

参数
- node，代表节点的 [[Node|NodeObject]]
- cbNode，一个函数
    - 参数：node，代表节点的 [[Node|NodeObject]]
    - 返回：T，任意类型；或者 undefined
- cbNodes?，可选参数，一个函数
    - 参数：nodes，代表节点数组的 [[#NodeArray|NodeArray]]
    - 返回：T，任意类型，但须和 cbNode 一样；或者 undefined

返回
- T，任意类型，须和 cbNode 跟 cbNodes 一样；或者 undefined

说明

为 node 的每个子节点，执行 cbNode 操作；根据参数，如果可以执行 cbNodes 那么就优先执行 cbNodes。

### isVariableStatement

```TypeScript
const variable1 = "hey"
let variable2 = "hello"
```

返回
- Boolean

识别变量赋值的语句。
