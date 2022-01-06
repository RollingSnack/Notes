---
Title: this 指向问题
Tags: [类型/笔记, 编程语言/JavaScript]
---

# this 指向问题

![[this]]

## 引例

假设有如下一段代码，`foo` 和 `obj.foo` 引用了一段内容相同的 [[函数]]。
函数的内部使用了 `this` [[关键字]]。

两者调用后产生的返回值却不相同。

```JavaScript
var obj = {
  foo: function () { return this.bar; },
  bar: 1
};

var foo = obj.foo;
var bar = 2;

obj.foo();  // return 1
foo();  // return 2
```

## 基本原理

### 引用传递

要思考 `this` 的具体指向，需要从 [[内存]] 的角度入手。

以引例的代码为例。
`obj.foo` 是函数，是引用数据类型，因此存放的是一段指向函数内容的内存地址；`obj.bar` 是数字，存放的是值本身。

```ascii
  +-------------------------------------------------+
  |                       global                    |
  |  +-------+                             +-----+  |
  |  |  obj  |                             | bar |  |
  |  |-------|                             +-----+  |
  |  |  foo--+--------------+                       |
  |  |       |              v                       |
  |  │  bar  |     +--------+--------+     +-----+  |
  |  +-------+     | function () {}  +<----+ foo |  |
  |                +-----------------+     +-----+  |
  +-------------------------------------------------+
```

函数运行时，函数内的 `this` 会根据上下文环境，指向直接调用它的 [[对象]]。

因此在 `obj.foo()` 调用中，`this` 指向 `obj` 对象。

但当 `obj.foo` 被赋值给了全局的 `foo` 变量后，实际是对内存地址进行了赋值，全局的 `foo` 会越过 `obj` 直接指向函数主体。
再调用 `foo()` 时，不再由 `obj` 发起，因此不存在直接调用的对象，故而指向默认的 `global` 全局环境。

### 对象嵌套

`this` 的逻辑始终是：指向**直接**调用函数的对象。

如下代码让 `obj2` 调用 `obj1`，然后间接调用 `obj1.foo()`。
然而最近一个调用函数的对象依旧是 `obj1`，因此返回的也是 `obj1` 中的 `bar`，输出 1。

```JavaScript
var obj1 = {
  foo: function () { return this.bar; },
  bar: 1
};
var obj2 = {
  obj1,
  bar: 2
};

obj2.obj1.foo();  // return 1
```

```ascii
  +------------------------------------------------------+
  |                                        global        |
  |  +--------+      +--------+                          |
  |  |  obj2  |      |  obj1  |                          |
  |  |--------|      |--------|      +----------------+  |
  |  |  obj1--+----->+   foo--+----->+ function () {} |  |
  |  |        |      |        |      +----------------+  |
  |  |  bar=2 |      |  bar=1 |                          |
  |  +--------+      +--------+                          |
  +------------------------------------------------------+
```

### 词法环境

`this` 是基于 [[词法环境]] 而来的。
[[JavaScript]] 中的词法环境有两种：全局环境 `global` 和由函数定义而来的函数环境。

同级的函数环境之间彼此不可见。
子函数允许引用父函数的变量，但当子函数中存在和父函数同名变量时，子函数会覆盖之。

`this` 同理，每个函数都有各自的 `this` 变量，彼此在各自的词法环境中独立；根据同名覆盖原则，即使是子函数也不会突破自己的词法环境去调用父函数的 `this`。

有如下代码，构造了“对象-函数-函数”的三层嵌套。
其中 `funcA` 内部构造了 `funcB` 的函数主体，并将 `funcB` 的返回值作为自己的结果立即返回。
调用时，从最外层的对象 `obj` 发起对 `funcA` 的调用，返回的是立即执行的 `funcB` 的返回值。

```JavaScript
var obj = {
  foo: 1,
  funcA: function () {
    var foo = 2;
    function funcB () {
      var foo = 3;
      return this.foo;
    }
    return funcB();
  }
};
var foo = 4;

obj.funcA();  // return 4
```

根据 [[#对象嵌套]] 部分的原理，不存在直接调用 `funcB` 的对象，而且由于词法环境的隔离，最终返回指向了默认的全局环境的 4。


```ascii
  +---------------------------------------------------------+
  |        global                                           |
  |                      +-------------------+              |
  |                      |    funcA () {}    |              |
  |                      |-------------------|              |
  |                      |  +-------------+  |              |
  |  +-------+           |  | funcB () {} |  |   +-------+  |
  |  |  obj  |           |  |-------------|  |   | foo=4 |  |
  |  |-------|           |  |    foo=3    |  |   +-------+  |
  |  | foo=1 |           |  |    this?    |  |              |
  |  |       | this->obj |  +-------------+  |              |
  |  | funcA-+---------->+       foo=2       |              |
  |  +-------+           |                   |              |
  |                      |     this->obj     |              |
  |                      +-------------------+              |
  +---------------------------------------------------------+
```

词法环境的隔离并非意味着完全不可访问其他环境的 `this`。
当确定环境后，`this` 值也被确定下来，和其他对象一样可以被赋值。

```JavaScript
var obj = {
  foo: 1,
  funcA: function () {
    var that = this;
    function funcB () {
      return that.foo;
    }
    return funcB();
  }
}
var foo = 4;

obj.funcA();  // return 1
```

此外，还应当注意：**对象是不能创建函数环境的**。

由于 [[类]] 的 [[实例]] 也是对象，因此类方法和其他普通函数一样，`this` 不总是跟着实例走，而是取决于如何被调用的。实例方法尽量只由实例访问是优先的选择。

如果有必要的话可以在构造函数内使用 [[this 指向问题#显式绑定 | 显示绑定]] 的方法绑定 `this`。

```JavaScript
class MyClass {
  constructor() {
    this.instance_func = this.instance_func.bind(this);
  }
  instance_func() {
    return this.foobar;
  }
  util_func() {
    return this.foobar;
  }
  get foobar() {
    return 1;
  }
}

var my_obj = new MyClass();
var obj = { foobar: 2 }

obj.util_func = my_obj.util_func;
obj.util_func();  // return 2 (obj.foobar)
obj.instance_func = my_obj.instance_func;
obj.instance_func();  // return 1 (my_obj.foobar)
```

## 绑定方式

有默认绑定、隐式绑定、显式绑定和 new 绑定四种。

优先级为：**new 绑定 / 显式绑定 > 隐式绑定 > 默认绑定**。（new 绑定和显式绑定不会共存）

### 默认绑定

如果在没有被任何对象调用的方法中使用了 `this`，`this` 将会处在全局环境中。

### 隐式绑定

某一对象调用的方法中使用 `this`。
虽然没有声明 `this` 的指向，但根据上下文可以判断 `this` 指向了调用该方法的该对象。

### 显式绑定

在 JavaScript 中，函数也是对象，对象就有方法，存在一些方法可用来切换函数执行的上下文环境，从而控制 `this` 的指向。

**`apply`** 方法传入两个参数，第一个是显式绑定的 `this` 对象，第二个是传给 `func` 函数的参数。
调用 `apply` 后会立刻执行 `func`，同时返回 `func` 中定义的返回值。

```JavaScript
func.apply(obj, [argsArray])
```

**`call`** 方法相当于 `apply` 的语法糖，第一个参数同样是显式绑定的 `this` 对象，剩下的所有参数都是传递给 `func` 函数的参数。
调用 `call` 后也会立刻执行 `func`，同时返回 `func` 中定义的返回值。

```JavaScript
func.apply(obj, arg1, arg2, ...)
```

**`bind`** 方法不直接调用函数，而是修改其中的 `this` 值之后，返回这个函数的拷贝。
`func` 在调用 `bind` 以后不会立刻执行。
可以将返回的拷贝给另一个变量，将来通过该变量调用函数时也不会改变 `this` 的指向。

```JavaScript
func.bind(obj[, arg1[, arg2[, ...]]])
```

```JavaScript
function foo () {
  return this.bar;
}

var bar = 1;
var obj = {
  bar: 2
};

var foobar = foo.bind(obj);  // this 指向 obj
console.log(foo());  / 1
console.log(foobar());  // 2
```

这个例子中，使用 `bind` 显式指定 `this` 的指向为 `obj` ，然后开辟新的内存空间，拷贝了一份带有 `this` 指向的函数。

原函数没有被修改，因此直接调用 `foo()` 仍然会返回全局环境的 1；且仅复制了函数，而不是在 `obj` 中创建真实的引用变量，此时的 `obj` 对象内仍不存在 `foo` 变量，如果调用 `obj.foo()` 会报错。

```ascii
  +-------------------------------------------------------------+
  |                           global                            |
  |  +-------+                                                  |
  |  |  obj  |                   +-------------+     +-------+  |
  |  |-------|                   |  foo () {}  |     | bar=1 |  |
  |  | bar=2 |                   +------+------+     +-------+  |
  |  +---+---+                          :                       |
  |      :                              :                       |
  |      :                              v                       |
  |      :   bind: this->obj    +-------+------+                |
  |      +-- -- -- -- -- -- -- >+ foobar () {} |                |
  |                             +--------------+                |
  +-------------------------------------------------------------+
```


### new 绑定

使用 `new` 关键字也可以绑定 `this` 的指向。

被 `new` 修饰的函数会被改写成构造函数，调用后创建并返回一个新对象。
此时，构造函数内的 `this` 将会指向该新对象。

```JavaScript
function foo (bar) {
  this.bar = bar;
}
var foobar1 = new foo(1);
console.log(foobar1.bar);  // 1
var foobar2 = new foo(10);
console.log(foobar2.bar);  // 10
```

## 箭头函数

严格来说 [[箭头函数]] 没有 `this`，但是使用 `this` 并不会报错。
因为根据词法环境的规则，箭头函数的 `this` 引用自它的父函数，也就是说 `this` 依赖于父函数的环境。

```JavaScript
function foo () {
  return () => {
    return this.bar;
  }
}

var obj1 = {
  bar: 1
};
var obj2 = {
  bar: 2
};

var foobar = foo.call(obj1);
foobar.call(obj2);  // return 1
```

使用显式绑定将 `foo` 的环境绑定到 `obj1` 上，然后将 `foo` 返回的箭头函数赋值给声明的 `foobar` 变量。

`foorbar` 也是个函数，不妨再用显式绑定的方法，尝试把 `foobar` 绑定到 `obj2` 上。
然而，结果却是 `obj2` 绑定失败了，`foobar` 函数的返回结果仍然是 `obj1` 环境中的 `bar`。

这正是因为箭头函数不存在 `this`，显式绑定找不到 `foobar` 中应该被赋值的 `this`，从而导致第二次绑定没有成功。

```ascii
  +--------------------------------------------------------------+
  |                                             global           |
  |  +------+                                                    |
  |  | obj1 |   call: this->obj1                                 |
  |  +------+< -- -- -- -- -- -- -+                              |
  |  |  bar |                     :                              |
  |  +------+            +--------+-------+                      |
  |                      |   foo () {}    |                      |
  |                      |----------------|                      |
  |                      |  +==========+  |  return  +--------+  |
  |                      |  | () => {} +- +- -- -- ->| foobar |  |
  |                      |  +==========+  |          +----+---+  |
  |  +------+            +----------------+               :      |
  |  | obj2 |                                             :      |
  |  +------+< -- -X -- -- -- -- -- -- -- -- -X -- -- -- -+      |
  |  |  bar |        Failed: call: this->obj2                    |
  |  +------+                                                    |
  +--------------------------------------------------------------+
```

根据上述逻辑，可以写一个能够修改 `this` 的版本。

每次显式绑定时，都绑定外层的 `foo` 的环境，这样可以间接控制箭头函数的环境。

```JavaScript
foo.call(obj1)();  // return 1
foo.call(obj2)();  // return 2
```

```ascii
  +------------------------------------------+
  |                         global           |
  |  +------+                                |
  |  | obj1 |   call: this->obj1             |
  |  +------+< -- -- -- -- -- -- -+          |
  |  |  bar |                     :          |
  |  +------+            +--------+-------+  |
  |                      |   foo () {}    |  |
  |                      |----------------|  |
  |                      |  +==========+  |  |
  |                      |  | () => {} |  |  |
  |                      |  +==========+  |  |
  |  +------+            +--------+-------+  |
  |  | obj2 |                     :          |
  |  +------+< -- -- -- -- -- -- -+          |
  |  |  bar |   call: this->obj2             |
  |  +------+                                |
  +------------------------------------------+
```

将箭头函数以对象的 [[字面量]] 形式创建，由于对象不创建函数环境，故而 `this` 不会指向这个对象，而是会根据词法环境规则被设置。

```JavaScript
var obj = {
  foo: () => { return this.bar; },
  bar: 1
};
var bar = 2;

obj.foo();  // return 2

var foobar = obj.foo;
foobar.call(obj);  // return 2
```

```ascii
  +----------------------------------------------+
  |                 global                       |
  |  +-------+                        +-------+  |
  |  |  obj  |                        | bar=2 |  |
  |  |-------|                        +-------+  |
  |  |  foo--+------------+                      |
  |  |       |            v                      |
  |  | bar=1 |      +=====+====+     +--------+  |
  |  +-------+      | () => {} +<----+ foobar |  |
  |                 +==========+     +--------+  |
  +----------------------------------------------+
```

## 浏览器行为

### window 对象

全局对象在不同的环境中指向通常也不同。

在 [[浏览器]] 中，这个全局对象是 `window` 对象。
它表示一个包含 [[DOM]] 文档的窗口。

### DOM 事件处理

在 [[事件]] 监听下，函数被用作事件处理函数，它的 `this` 会指向触发事件的 DOM 元素。

```HTML
<input type="button" value="button" id="btn">
```
```JavaScript
function what_is_this(e) {
  console.log(this === e.currentTarget)  // true
}
var btn = document.getElementById('btn');
btn.addEventListener('click', what_is_this, false);
```

作为 `on-event` [[内联事件]] 处理函数时，`this` 也会指向监听器所在的 DOM 元素。

```HTML
<button onclick="alert(this.tagName.toLowerCase());">  <!-- button -->
  Show this
</button>
```

不过，内联事件处理函数同样遵循词法环境的规则，只有最外层的 `this` 才会指向元素。

```HTML
<button onclick="alert((function(){return this})());">  <!-- window -->
  Show inner this
</button>
```

## 严格模式

在 [[严格模式]] 和非严格模式下，`this` 关键字的表现会有一些差别。

总的一点是，`this` 在非严格模式下，总是指向一个对象；在严格模式下可以是任意值。

### 全局环境

全局环境中（即在任何函数体之外），`this` 都指向全局对象。

这一特性和严格模式开启与否无关。

### 函数环境

非严格模式下，`this` 在未设置时会指向全局对象。

严格模式下，如果函数内部的 `this` 未设置，那么会变为 `undefined`。

```JavaScript
function func () {
  "use strict";
  return this;
}

func();  // undefined
```

### 类环境

在类的内部总是严格模式。
调用一个值为 `undefined` 的 `this` 方法会抛出错误。

## 总结

核心在于理解引用传递的概念和判断词法环境的作用范围。

1. 在全局环境中 `this` 指向全局对象。
2. 在函数内部，`this` 的值取决于函数被调用的方式。
    1. 当函数作为对象的方法被调用时，`this` 是调用该函数的直接对象。
    2. 当函数用作构造函数时(new 关键字)，`this` 被绑定到构造出的新对象。
    3. 当函数被用作浏览器的事件处理函数时，`this` 指向触发事件的 DOM 元素。
3. 箭头函数本身不存在 `this`，使用时其与封闭词法环境的 `this` 保持一致。
4. 可以使用 `apply` / `call` / `bind` 显式改变 `this` 的指向。
5. 严格模式下，未设置的 `this` 会指向 `undefined`。
