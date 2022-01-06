---
Title: 使用 `print()` 调试的技巧
Tags: [类型/译文, 类型/教程, 编程语言/Python, 开发/调试]
Original Link: https://adamj.eu/tech/2021/10/08/tips-for-debugging-with-print/
Original Title: Tips for debugging with print()
Original Author: Adam Johnson
---

# 使用 `print()` 调试的技巧

---

如果你在为调试时使用 `print()` 而感到窘困，请不要担心——这没有任何问题！
只需在正确位置做些检查，许多漏洞就能被解决。
尽管我很喜欢使用调试器，但首先，我更经常使用 `print()` 语句。

这里有五个技巧，可以充分利用 `print()` 来调试。

> 译注：五个技巧，原文最早的版本只有五个。

## 1 搭配 f-string 和 `=` 调试

我们调试时经常使用 `print()` 确认变量的值，例如：

```Python
>>> print("widget_count =", widget_count)
widget_count = 9001
```

在 Python 3.8+，我们可以使用 f-string 搭配一个 `=` 说明符，少打字而实现相同的效果。
这个说明符将会打印变量名、“=”、`repr()` 及其值：

```Python
>>> print(f"{widget_count=}")
widget_count=9001
```

少打字 *就是胜利！*

`=` 不会将我们局限在变量名。
我们可以使用任意表达式：

```Python
>>> print(f"{(widget_count / factories)=}")
(widget_count / factories)=750.0833333333334
```

如果你更喜欢在 `=` 边留有空格，可以把它们加到 f-string 中去，然后它们就会在输出中体现：

```Python
>>> print(f"{(widget_count / factories) = }")
(widget_count / factories) = 750.0833333333334
```

清爽！

## 2 搭配 emoji 让输出更 “时尚”

搭配 emoji 让你的调试语句从其他输出中脱颖而出。

```Python
print("👉 spam()")
```

然后，你还可以使用终端的 “查找” 功能跳转到调试的输出。

这里有一些用于调试还不错的 emoji，它们也可能有助于表达相关的情绪：

- 👉 “到达了此处”
- ❌ “不符合预期”
- ✅ “总之跑通了”
- 🥲 “努力中”
- 🤡 “[小丑](https://codewithoutrules.com/softwareclown/)竟是我自己”
- 🤯 “WTF”

为了更快地输入 emoji，针对你的操作系统使用快捷键：

- Windows: **Windows Key + .**
- macOS: **Control + Command + Space**
- Ubuntu: **Control + .**
- Other Linuxes: 🤷‍♂️ 请查阅文档

## 3 使用 `rich` 和 `pprint` 漂亮地打印

[Rich](https://rich.readthedocs.io/en/latest/introduction.html) 是一个终端格式化的库。
它捆绑了许多用于美化终端输出的工具，可以通过 `pip install rich` 安装它。

Rich 的 `print()` 函数对于调试对象很有用。
它能齐整地缩进大型嵌套数据结构，并添加语法高亮：

```Python
from rich import print as rprint
>>> rprint(luke)
{
    'id': 1,
    'name': 'Luke Skywalker',
    'films': [3, 4, 5, 6, 7, 8],
    'birth_year': '19BBY'
}
```

帅。

用 `from rich import print` 替换内建的 `print()` 函数。
这 *通常* 是安全的，因为 Rich 版本设计之初就是用于直接替换的，但它的确意味着传递给 `print()` 的所有内容都已格式化。
为了精确保留我们应用的非调试输出，我们可以用一个导入别名，就像 `from rich import print as rprint`一样，保留 `rprint()` 用于调试。

有关更多信息，请参阅 [the Rich quick start guide](https://rich.readthedocs.io/en/latest/introduction.html#quick-start)。

如果你不能随意安装 Rich，你可以使用 Python 的 [`pprint()` 函数](https://docs.python.org/3/library/pprint.html) 替代。
额外的 “p“ 代表 “漂亮”。
`pprint()` 也缩进数据结构，尽管没有颜色和样式。

```Python
>>> from pprint import pprint
>>> pprint(luke)
{'birth_year': '19BBY',
 'films': [3, 4, 5, 6, 7, 8],
 'id': 1,
 'name': 'Luke Skywalker'}
```

便利。

## 4 使用 `locals()` 调试所有局部变量

局部变量是在当前函数中定义的所有变量。
我们可以使用 [`locals()` 内建函数](https://docs.python.org/3/library/functions.html#locals) 抓取所有局部变量，它们将被放到一个字典里返回。
这使得一次性调试多个变量很方便，与 Rich 和 pprint 结合后更是如此。

我们可以像这样使用 `locals()`：

```Python
from rich import print as rprint

def broken():
    numerator = 1
    denominator = 0
    rprint("👉", locals())
    return numerator / denominator
```

当我们运行这段代码，我们可以在异常前看到一个字典，包含了变量的值：

```Python
>>> broken()
👉 {'numerator': 1, 'denominator': 0}
Traceback (most recent call last):
 ...
ZeroDivisionError: division by zero
```

还有 [`globals()` 内建函数](https://docs.python.org/3/library/functions.html#globals)，它返回所有全局变量，也即，那些定义在模块范围的变量，如导入的变量和类。
`globals()` 对于调试的帮助不大，因为全局变量通常不会改变，但了解它还是有好处的。

## 5 搭配 `vars()` 调试一个对象的所有属性

[`vars()` 内建函数](https://docs.python.org/3/library/functions.html#vars) 返回一个对象的属性组成的字典。
当我们一次性调试许多属性时，这很有帮助：

```Python
>>> rprint(vars(widget))
{'id': 1, 'name': 'Batara-Widget'}
```

出色。

（不带参数的 `vars()` 等价于 `locals()`，大约能少打一半的字。）

`vars()` 通过访问对象的 [`__dict__` 属性](https://docs.python.org/3/library/stdtypes.html#object.__dict__) 来工作。
大多数 Python 对象都有它，它被用作对象的（可写）属性的字典。
我们也可以直接使用 `__dict__`，尽管这要求打更多的字:

```Python
>>> rprint(widget.__dict__)
{'id': 1, 'name': 'Batara-Widget'}
```

`vars()` 和 `__dict__` 不适用于 *每个* 对象。
如果一个对象使用了 [`__slots__`](https://docs.python.org/3/reference/datamodel.html#object.__slots__)，或者它由 C 语言内建，那么它就没有 `__dict__` 属性了。

## 6 调试时用你的文件名及行号轻松返回

**更新 (2021-10-09):** 感谢 [Lim H 的技巧](https://twitter.com/limdauto/status/1447929546163081223)。

许多终端允许我们从输出打开中的文件名。
并且当冒号和行号跟随着文件名时，文本编辑器支持打开文件且定位到给定行。
例如，这个输出可以使我们打开 `example.py` 的第 12 行：

```shell
./example.py:12
```

在 macOS 上的 iTerm 中，我们可以按住 command 键单击打开该文件（[“smart selection”](https://iterm2.com/documentation-smart-selection.html)）。
其他终端请查阅文档。

我们可以在我们的 `print()` 中使用这项能力来轻松再打开我们尝试调试中的代码：

```Python
print(f"{__file__}:12 {widget_count=}")
```

这里使用了 Python 的魔术变量 [`__file__`](https://docs.python.org/3/reference/import.html#file__) 来获取当前的文件名。
我们需要自己提供行号。

运行它我们能看到：

```shell
$ python example.py
/Users/me/project/example.py:12 widget_count=9001
```

并且在 iTerm 中，我们可以按住 command 键单击这一行的开头，直接跳回中断的代码处。

[我听说](https://twitter.com/limdauto/status/1447929546163081223) 如果想在 VSCode 中打开，你可以输出一个类似 `vscode://file/<filename>` 的链接。

## ✨Bonus✨ 7 试试 icecream

**更新 (2021-10-09):** 感谢 Malcolme Greene 提醒我这个包。

[icecream 包](https://github.com/gruns/icecream) 提供了一个便利的调试快捷函数 `ic()`。
这个函数结合了一些我正在研究的工具。
不带有参数地调用 `ic()`，可以得到有关它在何时何地被调用的细节：

```Python
from icecream import ic

def main():
    print("Starting main")
    ic()
    print("Finishing")

if __name__ == "__main__":
    main()
```

```shell
$ python example.py
Starting main
ic| example.py:6 in main() at 11:28:27.609
Finishing
```

带参数地调用 `ic()`，它将检查并打印每个表达式（通过检查源码）及其结果：

```Python
from icecream import ic

def main():
    a = 1
    b = 2
    ic(a, b, a / b)

if __name__ == "__main__":
    main()
```

```shell
$ python example.py
ic| a: 1, b: 2, a / b: 0.5
```

如 Rich 那样，这包含了一些语法高亮。

icecream 还具有一些其他便利的功能，比如作为一个内建包来安装以使得你不需要导入它。
查阅 [它的文档](https://github.com/gruns/icecream) 获得更多信息。

## 结语

愿你能把你的 bug 都 `print()` 走。

—Adam