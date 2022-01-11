---
Title: Gem - exploding string alternatives
Tags: [类型/译文, 类型/笔记, 编程语言/Python]
Original Link: https://nedbatchelder.com/blog/202112/gem_exploding_string_alternatives.html
Original Title: "Gem: exploding string alternatives"
Original Author: Ned Batchelder
---

# Gem - exploding string alternatives

---

> 译注：gem 指代美好、有益的事物；explode 形容原本作为整体的东西向外碎裂成了多个的过程；string alternatives 指的是字符串中的某些部分，在其中供有多个备选项用于代表这一部分。

这儿有个 Python 的 gem：一点点的 Python，但很好地利用了语言和标准库的强大力量。

它是个函数，能根据嵌含了可替换部分的模式（*pattern*）生成字符串，并列出他们。
它以一个字符串作为输入，可替换部分被大括号包裹住，通过在其中进行选择，生成所有的字符串。

```Python
>>> list(explode("{Alice,Bob} ate a {banana,donut}."))
[
    'Alice ate a banana.',
    'Alice ate a donut.',
    'Bob ate a banana.',
    'Bob ate a donut.'
]
```

下面是这个函数：

```Python
def explode(pattern: str) -> Iterable[str]:
    """
    Expand the brace-delimited possibilities in a string
    """
    seg_choices = []
    for segment in re.split(r"(\{.*?\})", pattern):
        if segment.startswith("{"):
            seg_choices.append(segment.strip("{}").split(","))
        else:
            seg_choices.append([segment])
    for parts in itertools.product(*seg_choices):
        yield "".join(parts)
```

我称之为 gem 因为它简洁却不刁钻，且利用了 Python 的工具来实现强大的效果。
让我们看看它是如何工作的。

**re.split：** 第一步是把字符串分成若干部分。
我用了 [`re.split()`](https://docs.python.org/3/library/re.html#re.split)：它接受一个正则表达式（*regular expression*），在任何模式匹配的地方切分字符串，然后返回一个列表，列表各项正是切分开的各部分。

此处所用之精妙在于：如果切分用的正则表达式有捕获组（*capturing group*）（括号内的部分），那么被捕获的字符串也会被包含在结果列表里。
模式是用大括号括起来的任何东西，我完整保留了它的括号，以确保我可以在切分后的列表中识别出它们。

对于我们的示例字符串，`re.split` 会返回如下片段：

```Shell
['', '{Alice,Bob}', ' ate a ', '{banana,donut}', '.']
```

有一个初始的空字符串看上去会令人担忧，但这不构成问题。

**分组（*Grouping*）：** 我使用该片段列表构建了另一个列表，由各个片段的可选项组成。
如果某片段以大括号开头，那么去掉大括号，并且基于逗号分片以得到一个可选项的列表。
不以大括号开头的片段，是字符串中不可替换的部分，所以将他们添加为单一选项的列表。
这样我们得到了一个统一的、由列表构成的列表，内层的列表是每个片段的可选结果。

对于我们的示例字符串，这是列表的各部分：

```Shell
[[''], ['Alice', 'Bob'], [' ate a '], ['banana', 'donut'], ['.']]
```

**itertools.product：** 为了生成所有的组合，我用了 [`itertools.product()`](https://docs.python.org/3/library/itertools.html#itertools.product)，它完成了这个函数大部分的繁重工作。
它接受许多可迭代对象作为参数，并从每个元素中选择一个，生成所有可能的组合。
我的 `seg_choices` 列表正是 `itertools.product` 所需要的参数，我对其运用了星号语法（*star syntax*）。

从 `itertools.product` 得到的值是按可选项构建而来的元组。
最后一步是使之连接在一起，并用 `yield` 把值提供给调用方。

*不错。*