## Program

在编译器中，Program 形容的是包含固定的 SourceFile 集合以及编译选项 CompilerOptions 的实体。

### getSourceFiles

```TypeScript
const sourceFiles = program.getSourceFiles();
```

返回：
- SourceFile[]，程序中所有的 [[SourceFile]] 构成的数组

### getGlobalDiagnostics

```TypeScript
program.getGlobalDiagnostics();
```

为程序分析生成全局变量、局部变量，将会作为 `locals` 属性添加到 AST 中。