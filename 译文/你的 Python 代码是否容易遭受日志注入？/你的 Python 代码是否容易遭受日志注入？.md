---
Title: 你的 Python 代码是否容易遭受日志注入？
Tags: [类型/译文, 类型/笔记, 编程语言/Python, 安全/日志]
Original Link: https://dev.arie.bovenberg.net/blog/is-your-python-code-vulnerable-to-log-injection/
Original Title: Is your Python code vulnerable to log injection?
Original Author: Arie Bovenberg
---

# 你的 Python 代码是否容易遭受日志注入？

---

最近关注 log4j 的新闻，你可能想知道 Python 的 `logging` 库是否安全。
毕竟，字符串格式化遇上用户的输入时，还是有潜在的注入攻击的可能的。
幸运的是，Python 的 `logging` 不容易遭遇远程的代码执行。
尽管如此，谨慎对待不可信数据还是很重要的。
本文将描述一些常见的陷阱，以及在特定情况下，使用 f-string 记录日志的流行做法，会如何更轻易地置你于其他类型的攻击之下。

## logging 的基础

格式化在哪里遇到用户输入的呢？
让我们从基本的 `logging` 设置开始。

```Python
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
```

记录一条消息：

```Python
logger.info("hello world")
```

这将输出：

```Log
INFO:__main__:hello world
```

让我们看看字符串格式化的用法：

> **原作者注：** *字符串格式化*
> 
> 这或许并不广为人知，`logging` 的格式化是支持命名的，但得当字典作为参数传递时。
> 它和 `extra=` 参数不同！
> 你可以自己看看 `%` 运算符是如何工作的：`"hello %(name)" % {"name": "bob"}` 和 `"hello %s" % "bob"`。
> 本文中，我会重点关注使用字典时的漏洞。

```Python
context = {'user': 'bob', 'msg': 'hello everybody'}
logger.info("user '%(user)s' commented: '%(msg)s'.", context)
```

其输出如下：

```Log
INFO:__main__user 'bob' commented 'hello everybode'.
```

## 简单的注入

如果不清理输入，你可能会遭遇 [日志注入](https://owasp.org/www-community/attacks/Log_Injection)。
设想如下的消息：

```Shell
"hello'.\nINFO:__main__:user 'alice' commented: 'I like pineapple pizza"
```

如果按照之前的模板记录，这会导致：

```Log
INFO:__main__:user 'bob' commented: 'hello'.
INFO:__main__:user 'alice' commented: 'I like pineapple pizza'.
```

如你所见，攻击者不仅可以破坏日志，还可以归罪给别人。

### 缓解措施

我们可以通过 [换行符转义](https://github.com/darrenpmeyer/logging-formatter-anticrlf) 来缓解这种特定攻击。
但请注意，还有许多其他的 [恶劣的 unicode 控制符](https://www.python.org/dev/peps/pep-0672/) 可能扰乱你的日志。
最安全的解决方案是别记录不信任的文本。
如果你出于审计追踪需要存储它，那就用数据库。
或者，[结构化日志](https://www.structlog.org/en/stable/) 可以预防基于换行的攻击。

## 重复格式化的麻烦

另一种有趣的漏洞只针对 Python。
由于 `logging` 所采用的旧 `%` 风格的格式化经常被认为是丑陋的，所以许多人更喜欢用 f-string：

```Python
logger.info(f"user '{user}' commented: '{msg}'.")
```

固然这 *看* 上去更好，但它不会阻止 `logging` 再尝试去格式化得到的字符串本身。
所以如果 `msg` 是……

```Python
"%(foo)s"
```

……f-string 求值后只会留给我们：

```Python
logger.info("user 'bob' commented: '%(foo)s'.")
```

所以 `logging` 会怎么做？
它会去寻找 `foo` 然后崩溃吗？
幸好没有。
在 `logging` 源代码的深处，我们发现：

```Python
if self.args:
        msg = msg % self.args
```

没有参数，没有格式化。
有点用。
但万一有个参数，事情就有趣起来了。
假设这样：

```Python
logger.info(f"user '%(user)s' commented: '{msg}'.", context)
```

当然，可能没人一开始就混用不同风格的格式化。
但有可能的是：

- 某人会用这种方式向已存在的日志语句添加 `msg` 参数。
- 重构为 f-string 时，忘记移除 `context` 参数。
- 该日志消息通过用户定义的函数或添加 `context` 参数的日志过滤器传递。

这种情况下，我们会得到一个如下的错误：

```Shell
--- Logging error ---
[...snip...]
KeyError: 'foo'
Call stack:
  File "example.py", line 29, in <module>
    logger.info(f"user '%(user)s' commented: '{msg}'.", context)
Message: "user '%(user)s' commented: '%(foo)s'."
Arguments: {'user': 'bob'}
```

日志里有这些很烦人吧？
确实。
危险？
*倒也* ……未必。
但是通过把外部字符串格式化为日志信息（反过来又被 logging 再次格式化），我们打开了 [格式化字符串攻击](https://owasp.org/www-community/attacks/Format_string_attack) 的大门。
还好，Python 比 C 的风险低得多，但仍然有许多滥用的方法。

### 过量填充

一种情形时滥用 [填充语法](https://pyformat.info/#string_pad_align)。
考虑这样的消息：

```Python
"%(user)999999999s"
```

这将会为 `user` 填充近 1 GB 的空白。
这不仅会让你日志语句慢下来，它甚至可能阻塞你的日志基础设施。

这为什么是个问题呢？
攻击者能让你的服务器瘫痪已经够糟糕了。
如果他们还削弱了你的日志记录，你甚至没法知道你被什么攻击了。

### 日志泄漏

另一个潜在的危险是泄漏了敏感信息。
在我们的例子里，如果 `context` 包含一个 `"secret"` 键，攻击者就可以以如下消息使之泄漏到日志中：

```Python
"%(secret)s"
```

当结合填充漏洞使用时，这将尤其危险，因为攻击者得以有时间来嗅探哪些键值是存在的。

另一方面，我们可以庆幸 Python 的 `%` 风格的格式化语法限制很大。
如果日志用到了新的大括号风格，攻击者甚至不需要敏感信息出现在 `context` 中，用这样的消息：

```Python
"{0.__init__.__globals__['SECRET']}"
```

你可能好奇如果 `secret` 出现在日志中有什么大不了的，它不是公开的，对吧？
问题在于，对于攻击者，日志通常比凭证存储（*credential store*）更容易访问。
因此，CWE 将 “*把敏感信息插入日志文件（Insertion of Sensitive Information into Log File）*” 列在 [最危险的软件缺陷](https://cwe.mitre.org/top25/archive/2021/2021_cwe_top25.html) 中的第 39 位。

### 缓解措施

为了消除这些风险，你应该 *始终* 让 `logging` 处理字符串的格式化。
不要自己用 f-string 或其他方法格式化字符串。
还好，有一个 [flake8 的插件](https://github.com/globality-corp/flake8-logging-format) 可以帮你检查。
此外，一旦实现了 [PEP675](https://www.python.org/dev/peps/pep-0675)，你可能可以使用类型检查器（*typechecker*）来检查是否只有文字字符串传递给了日志记录器。

> **原作者注：**
>
> 使用 `logging` 内建的格式化 [通常也能获得更好的性能（排除其他原因）](https://dev.to/izabelakowal/what-is-the-best-string-formatting-technique-for-logging-in-python-d1d)。

## 建议

1. *不要记录不可信文本。* Python 的 `logging` 库不能保护你免受换行符或其他 unicode 字符的干扰，这些字符允许攻击者弄乱日志，甚至伪造日志。
2. *不要自己格式化日志（使用 f-string 或其他方式）。* 在特定情况下，这会使你容易遭受 Dos（*denial-of-service*）攻击，甚至泄漏敏感信息。

完整的示例代码可以在 [这里](https://gist.github.com/ariebovenberg/dfd849ddc7a0dc7428a22b5b8a468134) 找到。
你可以自己实验实验。

你还可以在 [reddit](https://www.reddit.com/r/Python/comments/rqaysb/is_your_python_code_vulnerable_to_log_injection/) 或 [Hacker News](https://news.ycombinator.com/item?id=29803530) 上讨论此文。

## 2022 年 1 月 4 日更新

从这以来，我 [在 Python 错误跟踪器（*bug tracker*）上创建了一个 issue](https://bugs.python.org/issue46200) 以记录日志文档中的安全风险，甚至可能创建一个更安全的 `logger` API。

## 鸣谢

感谢 [Daan Debie](https://daan.fyi/) 审阅了此文。