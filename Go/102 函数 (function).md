---
Title: 102 函数 (function)
Tags: [Document, Go]
---

# 102 函数 (function)

---

## 定义

```Go
func functionName(parameter_list) (return_value_list) {
    //function body
}
```

[[Go]] 语言使用 [[关键字]] `func` 声明一个 [[函数]]。

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

### 可变参数

上述例子中的参数是 [[固定参数]]，不过 Go 也接受 [[可变参数]]。
可变是指函数的参数个数不固定，在参数名后面加 `...` 用作标识。

```Go
func sum(nums ... int) int {
    ans := 0
    for _, num := range nums {
        ans += num
    }
    return ans
}
```

此时传入的 `nums` 参数，会自动变成 Go 语言中的 [[切片]]。

如果同时使用了固定参数和可变参数，那么可变参数需要放在最后。

```Go
func sum(base int, nums ... int) int {
    ans := base
    for _, num := range nums {
        ans += num
    }
    return ans
}
```

注意，由于传入的参数 `nums` 变成了切片类型的，和参数 `base` 不同，因此也需要显式指出 `base` 的类型。

### 值传递和引用传递

Go 在默认的情况下，使用的是 [[值传递]]。

不过如果部分数据存放的就是地址而不是具体的值，那么会表现得像是使用了[[引用传递]]，如切片。
这部分的讨论，将会在 [[后面]] 讨论到数据类型是再详细说明。

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

## 返回值

关键字 `return` 用于标识 [[返回值]]。

### 多返回值

函数允许同时具有多个返回值，但此时必须使用括号将返回值包裹起来。

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

### 返回值命名

定义函数时可以直接为返回值命名，格式和参数一样。
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