## JS 的导出

### 语句定义

```JavaScript
(269) ExportAssignment
    export = x;

(270) ExportDeclaration

(271) NamedExports

(272) NamespaceExport

(273) ExportSpecifier
```

### 举例

```TypeScript
export const c = "c";  // 跟随语句定义直接导出

export { a, b as d };  // a, b 已经提前定义了, d 作为别名
export { Hey as Hi } from "mod";  // re-export 把 mod 的 Hey 直接作为 Hi 导出
export * as ns from "mod";  // 命名空间 re-export

export * from "mod";  // 合并 re-export，多个 mod 共享命名空间，因此可能冲突

export = a;  // a 已经提前定义，配合 import = require() 使用
export = { a, b };  // a 和 b 已经提前定义了

export default a;
```

对应的 AST 结构

```JavaScript
- (300) SourceFile
  -- (235) VariableStatement                   | export const c = "c";
    --- (93) ExportKeyword                     | export
    --- (253) VariableDeclarationList          | const c = "c"
      ---- (252) VariableDeclaration           | c = "c"
        ----- (79) Identifier                  | c
        ----- (10) StringLiteral               | "c"

  -- (270) ExportDeclaration                   | export { a, b as d };
    --- (271) NamedExports                     | { a, b as d }
      ---- (273) ExportSpecifier               | a
        ----- (79) Identifier                  | a
      ---- (273) ExportSpecifier               | b as d
        ----- (79) Identifier                  | b
        ----  (79) Identifier                  | d
  -- (270) ExportDeclaration                   | export { Hey as Hi } from "mod";
    --- (271) NamedExports                     | { Hey as Hi }
      ---- (273) ExportSpecifier               | Hey as Hi
        ----- (79) Identifier                  | Hey
        ----- (79) Identifier                  | Hi
    --- (10) StringLiteral                     | "mod"
  -- (270) ExportDeclaration                   | export * as ns from "mod";
    --- (272) NamespaceExport                  | * as ns
      ---- (79) Identifier                     | ns
    --- (10) StringLiteral                     | "mod"
  -- (270) ExportDeclaration                   | export * from "mod";
    --- (10) StringLiteral                     | "mod"

  -- (269) ExportAssignment                    | export = a;
    --- (79) Identifier                        | a
  -- (269) ExportAssignment                    | export = { a, b };
    --- (203) ObjectLiteralExpression          | { a, b }
      ---- (292) ShorthandPropertyAssignment   | a
        ----- (79) Identifier                  | a
      ---- (292) ShorthandPropertyAssignment   | b
        ----- (79) Identifier                  | b

  -- (269) ExportAssignment                    | export default a;
    --- (79) Identifier                        | a
```