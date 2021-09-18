## Node

整个 AST 树的任何一个部分都是 Node，包括每条语句、每个变量等等。

### kind

[[SyntaxKind]] 类型的属性，是一个 `enum` 枚举类型，但只属于其中一项。

### flags

[[NodeFlags]] 类型的属性，是一个 `enum` 枚举类型，通常只属于其中一项，但也可能属于多个。

### getText

返回
- String

返回该节点的源代码字符串。

```TypeScript
let a = "a";
// let b = "b";

let c = {
  "c1": "c1",
  // "c2": "c2",

  "c3": "c3",
};
```

```TypeScript
let c = {
  "c1": "c1",
  // "c2": "c2",

  "c3": "c3",
};
```

内部的注释、空行会被保留。

### getFullText

返回
- String

返回上一个节点到该节点的完整源代码字符串。

```TypeScript
let a = "a";
// let b = "b";

let c = {
  "c1": "c1",
  // "c2": "c2",

  "c3": "c3",
};
```

```TypeScript
// let b = "b";

let c = {
  "c1": "c1",
  // "c2": "c2",

  "c3": "c3",
};
```

除内部的空行、注释以外，还包含了上一个节点结束后的注释和源代码。

### getChildren

返回
- Node[]

返回这个节点子节点列表。

### forEachChild

```TypeScript
function ts.forEach<T>(
  cbNode: (node: Node) => T | undefined,
  cbNodes?: (nodes: NodeArray<Node>) => T | undefined
): T | undefined
```

参数
- cbNode，一个函数
    - 参数：node，代表节点的 [[Node|NodeObject]]
    - 返回：T，任意类型；或者 undefined
- cbNodes?，可选参数，一个函数
    - 参数：nodes，代表节点数组的 [[#NodeArray|NodeArray]]
    - 返回：T，任意类型，但须和 cbNode 一样；或者 undefined

返回
- T，任意类型，须和 cbNode 跟 cbNodes 一样；或者 undefined

说明

和 [[API#forEachChild]] 提供的功能一样，区别在于这是自调用的。