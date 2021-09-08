---
Title: 102 函数 (function)
Tags: [Document, Go]
---

# 102 函数 (function)

---

## 声明

```Go
func functionName(parameter_list) (return_value_list) {
    //function body
}
```

[[Go]] 语言使用 [[关键字]] `func` 声明一个 [[函数]]。

作为编译型语言，函数编写的顺序是无关紧要的。

## 参数

定义函数时，第一个括号内规定的是函数的 [[参数]]。

参数间使用逗号`,`分割，每个参数后面必须跟着该参数的 [[类型]]。

```Go
func add(a int, b int) int {
    return a + b
}
```

### 类型简写

函数的参数中，如果相邻变量的类型相同，那么就可以省略标注前者的类型。

```Go
func add(a, b int) int {
    return a + b
}
```

### 变长参数

上述例子中的参数是 [[固定参数]]，不过 Go 也接受 [[变长参数]]。
变长是指函数的参数个数不固定，在参数名后面加 `...` 用作标识。

```Go
func sum(nums ...int) int {
    ans := 0
    for _, num := range nums {
        ans += num
    }
    return ans
}
```

此时传入的 `nums` 参数，会自动变成 Go 语言中的 [[切片]]。

不过调用该函数时，不允许直接传递切片。
如果此前这些变长参数存放在切片中的话，那么同样也需要 `...` 来传递。

```Go
func f1(s ...string) {  // 传入的 s 视作切片
    f2(s...)  // ...传递可变参数
    f3(s)  // 直接传递切片
}

func f2(s ...string) { }  // 可变参数
func f3(s []string) { }  // 切片，固定参数
```

如果同时使用了固定参数和变长参数，那么变长参数需要放在最后。

```Go
func sum(base int, nums ...int) int {
    ans := base
    for _, num := range nums {
        ans += num
    }
    return ans
}
```

注意，由于传入的参数 `nums` 变成了切片类型的，和参数 `base` 不同，因此也需要显式指出 `base` 的类型。

此外，变长函数也可以接受长度为 0 的参数个数。

### 值传递和引用传递

Go 在默认的情况下，使用的是 [[值传递]]。

不过如果部分数据存放的就是地址而不是具体的值，那么会表现得像是使用了[[引用传递]]，如切片。
这部分内容，将会在 [[后面]] 讨论到数据类型是再详细说明。

```Go
func double(nums []int) {
    for i, num := range nums {
        nums[i] += num
    }
}

func main() {
    a := []int{1, 2, 3}  // 切片存放的是地址，输出 [1, 2, 3]
    double(a)  // 修改了切片内各地址指向的值，输出 [2, 4, 6]
}
```

此外，Go 有 [[指针]]，因此也可以使用 `&` 取地址，将地址传递给函数。
指针本身也是一个变量，也有自己的值（值是一个地址）和地址。

```Go
func addOne(a *int) {
    b := a  // b 拿到的是原始数据 a 的地址
    *b += 1  // 修改了原始数据的数值
}

func addOne(a int) {
    b := &a  // b 拿到的是副本 a 的地址
    *b += 1  // 修改副本数值，不会对原始数据产生影响
}
```

## 返回值

关键字 `return` 用于标识 [[返回值]]。

### 多返回值

函数允许同时具有多个返回值，但此时必须在定义处使用括号将返回值包裹起来。

```go
func calc(a, b int) (int int) {
    return a + b, a - b
}
```

接受返回值时也须有相对应的数量，或者一个返回值也不接受。
只接受部分返回值是不允许的。

```go
// 允许：accept both
x, y := calc(1, 2)
// 允许：accept nothing
calc(1, 2)

// 不允许：accept part
z := calc(1, 2)
```

### 命名的返回值

定义函数时可以使用 [[命名的返回值]]，格式和参数一样。
这样在函数中就可以不先声明而直接使用，返回时也可以直接使用关键字 `return`。

命名了却没有用到的返回值，会返回默认值。

```Go
func calc(a, b int) (sum, sub, zero int) {
    sum = a + b
    sub = a - b
    return
}
```

即使只有一个命名了的返回值，也必须使用括号。

```Go
func partCalc(a, b int) (sum int) {
    sum = a + b
    return
}
```

### 空白符

一些不需要的返回值，可以使用 [[空白符]] 匹配 `_`，空白符匹配到的数值会被舍弃。

```Go
func threeValue(a, b, c int) (int int int) {
    return a + 1, b, c - 1
}

func main() {
    addOne, _, subOne := threeValue(3, 4, 5)
}
```

## 函数体

[[函数体]] 使用大括号包裹起来。

左大括号 `{` 必须和函数的声明放在同一行。
这部分涉及到 [[编译器]] 的工作原理，如果函数声明之后不跟着大括号的话，编译器将会报错。

## 推迟执行 defer

使用关键字 `defer` 修饰的语句会推迟到函数返回前才执行。
真实的执行位置在 `return` 语句后；但注意，函数虽然执行了 `return` 语句，但是还没有正式返回。

### 释放资源

```Go
func main() {
    doDBOperations()
}

func connectDB() {
    fmt.Println("1. connect")
}

func closeDB() {
    fmt.Println("3. close")
}

func doDBoperations() {
    connectDB()
    defer closeDB()
    fmt.Println("2. operation")
    return  // 输出顺序：1 2 3
}
```

### 代码追踪

```Go
func trace(s string) {
    fmt.Println("in ", s)
    return s
}

func un(s string) {
    fmt.Println("out ", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("do a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("do b")
    a()
}

func main() {
    b()  // 输出顺序 in b -> do b -> in a -> do a -> out a -> out b
}
```

### 监控出入参

```Go
func add(num int) (addOne, addTwo int) {
    defer func() {
        fmt.Printf("add(%d) = %d, %d\n", num, addOne, addTwo)
    }()  // 匿名函数
    return num + 1, num + 2
    // defer 修饰的语句，真实的执行顺序在 return 语句之后，即此处
}

func main() {
    add(5)  // 输出 add(5) = 6, 7
}
```

### 带参数的推迟执行

`defer` 修饰的语句可以接受参数。
虽然执行推迟了，但是参数是提前传入其中的。

```Go
func main() {
    result := add(4)  // 输出 0
    fmt.Println(result)  // 输出 5
}

func add(num int) (addOne int) {
    defer fmt.Println(addOne)
    addOne = num + 1
    return
    // 此时执行 defer，使用的是执行 num + 1 前的 addOne
}
```

### 后进先出

有多个 `defer` 存在时，语句会按照 [[栈]] 的形式，后进先出，逆序执行

```Go
func f() {
    for i := 0; i < 5; i++ {
        defer fmt.Println(i)  // 输出 4 3 2 1 0
    }
}
```