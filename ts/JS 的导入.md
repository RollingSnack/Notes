## JS 的导入

### 语句定义

```JavaScript
(263) ImportEqualsDeclaration extends DeclarationStatement, JSDocContainer:  // 等式 import 语句
    import x = require("mod");
    import x = M.x;

(264) ImportDeclaration extends Statement:  //  import 语句

(265) ImportClause extends NamedDeclaration:  //  import 导入的内容
    import d from "mod";
        //  name: d
        //  namedBinding: undefined
    import * as ns from "mod";
        //  name: undefined
        //  namedBinding: NamespaceImport({ name: ns })
    import d, * as ns from "mod";
        //  name: d
        //  namedBinding: NamespaceImport({ name: ns })
    import { a, b as x } from "mod";
        //  name: undefined
        //  namedBinding: NamedImports({ elements: [{ name: a }, { name: x, propertyName: b}]})
    import d, { a, b as x } from "mod";
        //  name: d
        //  nameBinding: NamedImports({ elements: [{ name: a }, { name: x, propertyName: b}]})

(266) NamespaceImport extends NamedDeclaration:  //  作为 namespace 导入

(267) NamedImports extends Node:  //  具名导入

(268) ImportSpecifier extends NamedDeclaration:  //  import 的 identifier 指示
```

对于 ImportClause 来说，有两个重要属性（可以为空）

```TypeScript
readonly name?: Identifier;  //  Default binding
```

name 指向的是默认导出导入。

name 实际上就是默认导出的另一个别名。

```TypeScript
readonly namedBinding?: NamedImportBindings;
```

nameBinding 则指向其他类型的导入，因此形成一个列表；此时也有两种情况：命名空间以及具名导入。

命名空间实际上只是一个储存所有导入内容的对象，具名导入则直接一一对应。

### 举例

```JavaScript
import { MyNode, AnotherNode } from "./node";
import type { MyNode as type1 } from "./node";
import * as node1 from "./node";
import node2 from "./node";

import "./node";  // 只执行，不导入内容
var node3 = import("./node");

import node4 = require("./node");
```

TypeScript 的 Compiler 产生的 AST 结构及对应源代码内容。

```shell
- (300) SourceFile                        | 略
  -- (264) ImportDeclaration              | import { MyNode, AnotherNode } from "./node";
    --- (265) ImportClause                | { MyNode, AnotherNode }
      ---- (267) NamedImports             | { MyNode, AnotherNode }
        ----- (268) ImportSpecifier       | MyNode
          ------ (79) Identifier          | MyNode
        ----- (268) ImportSpecifier       | AnotherNode
          ------ (79) Identifier          | AnotherNode
    --- (10) StringLiteral                | "./node"
  -- (264) ImportDeclaration              | import type { MyNode as type1 } from "./node";
    --- (265) ImportClause                | type { MyNode as type1 }
      ---- (267) NamedImports             | { MyNode as type1 }
        ----- (268) ImportSpecifier       | MyNode as type1
          ------ (79) Identifier          | MyNode
          ------ (79) Identifier          | type1
    --- (10) StringLiteral                | "./node"
  -- (264) ImportDeclaration              | import * as node1 from "./node";
    --- (265) ImportClause                | * as node1
      ---- (266) NameSpaceImport          | * as node1
          ------ (79) Identifier          | node1
    --- (10) StringLiteral                | "./node"
  -- (264) ImportDeclaration              | import node2 from "./node";
    --- (265) ImportClause                | node2
      ---- (79) Identifier                | node2
    --- (10) StringLiteral                | "./node"

  -- (264) ImportDeclaration              | import "./node";
    --- (10) StringLiteral                | "./node"
  -- (235) FirstStatement                 | var node3 = import("./node");
    --- (253) VariableDeclarationList     | var node3 = import("./node")
      ---- (252) VariableDeclaration      | node3 = import("./node")
        ----- (79) Identifier             | node3
        ----- (206) CallExpression        | import("./node")
        ------ (100) ImportKeyword        | import
        ------ (10) StringLiteral         | "./node"

  -- (263) ImportEqualsDeclaration        | import node4 = require("./node");
    --- (79) Identifier                   | node4
    --- (275) ExternalModuleReference     | require("./node")
      ---- (10) StringLiteral             | "./node"
  -- (1) EndOfFileToken                   |
```

### 264 ImportDeclaration 属性

```JavaScript
decorators: undefined
end: <number>
flags: 0
importClause: <Node>  -  (265)  //  undefined
kind: 264
locals: undefined
localSymbol: undefined
modifierFlagsCache: 0
modifiers: undefined
moduleSpecifier: <Token>  -  (10)
nextContainer: undefined
parent: <SourceFile>
pos: <number>
symbol: undefined
transformFlags: 0
```

### 265 属性

```JavaScript
end: <number>
flags: 0
isTypeOnly: false
kind: 265
modifierFlagsCache: 0
name: undefined  //  例4 是 <Identifier> 79
namedBindings: <Node>  -  (267)  //  例3 显示的是 NameSpaceImport (266)  例4 是 undefined
parent: <Node>  -  (264)
pos: <number>
symbol: <Symbol>  -  (2097152)  // SymbolFlags.Alias  例4 才有
transformFlags: 0
```

### 266 属性

```JavaScript
end: <number>
flags: 0
kind: 266
modifierFlagsCache: 536870912
name: <Identifier>  -  (79)  //  例3 显示的是 as 之后的 Identifier
parent: <Node>  -  (265)
pos: <number>
symbol: <Symbol>
transformFlags: 0
```

### 267 属性

```JavaScript
elements: <Node>(Array)  -  (268)  // 是 268 的个数，而不是 79 的个数
end: <number>
flags: 0
kind: 267
modifierFlagsCache: 0
parent: <Node>  -  (265)
pos: <number>
transformFlags: 0
```

### 268 属性

```JavaScript
end: <number>
flags: 0
kind: 268
modifierFlagsCache: 536870912  // ModifierFlags.HasComputedFlags
name: <Identifier>  -  (79)  //  例2 显示的是 as 之后的 Identifier
parent: <Node>  -  (267)
pos: <number>
propertyName: undefined
symbol: <Symbol>
```

### 79 属性

```JavaScript
// IdentifierObject
end: <number>
escapedText: <string>
flags: 0
flowNode: <FlowFlags>  //  例6 显示 2050，其他是 2 (2: Start, 2048: Referenced)
kind: 79
modifierFlagsCache: 0
originalKeyworkKind: undefined
parent: <Node>  -  (268)  //  <Node>  -  (266)  //  <Node>  -  (265)
pos: <number>
text: <function>
transformFlags: 0
```

### 10 属性

```JavaScript
// TokenObject
end: <number>
flags: 0
hasExtendeUnicodeEscape: false
kind: 10
modifierFlagsCache: 0
parent: <Node>  -  (264)
pos: <number>
singleQuote: undefined
text: <string>
transformFlags: 0
```

### Symbol 属性

```JavaScript
declarations: <Node>(Array)  -  (265?)
escapedName: <string>  //  as 之后的
flags: 2097152  //  SymbolFlags.Alias
name: <function>  //  as 之后的
parent: undefined
transformFlags: 0
```

### 206 CallExpression 属性

```JavaScript
arguments: <Token>(Array)
end: <number>
expression: <Token>  -  (100)
flags: 0
kind: 206
modifierFlagsCache: 0
parent: <Node>  -  (252)
transformFlags: 4194304  //  TypeFormatFlags.InFirstTypeArgument?
typeArguments: undefined
```