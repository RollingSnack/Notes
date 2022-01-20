---
Title: pathlib (2) 纯路径类
Tags: [类型/笔记, 类型/系列, 编程语言/Python]
---

# pathlib (2) 纯路径类

![[Concepts/pathlib]]

## 两种路径风格

纯路径 [[类]] 不访问真实的 [[文件系统]]，只参与路径计算；因此无论在什么系统上都可以实例化 3 个纯路径类。

`PurePosixPath` 和 `PureWindowsPath` 分别表示两种路径风格，前者是 [[POSIX]] 的路径风格，后者是以 [[Windows]] 系统为代表的路径风格。

当使用 `PurePath` 时，会根据当前系统的路径风格，实例化为上述两种中具体的某一种 [[对象]]。

```Python
>>> PurePath("setup.py")
PurePosixPath('setup.py')  # 在 Unix 系统上运行时实例化
```

## 绝对路径和相对路径

文件系统以目录 [[树]] 的方式存储，树的根称为 [[根目录]]。
Windows 系统在根目录之上，还有 [[盘符]]，不过一个盘符下面也只有一个根目录与之对应。

> All paths can have a drive and a root. For POSIX paths, the drive is always empty.

根据 [PEP 428](https://www.python.org/dev/peps/pep-0428/) 的设计，所有路径对象都带有一个盘符 和一个根目录，只是 POSIX 下的盘符总是为空。

> A relative path has neither drive nor root.

只有既没有盘符也没有根目录的路径，才被称为 [[相对路径]]。

> A POSIX path is absolute if it has a root. A Windows path is absolute if it has both a drive _and_ a root. A Windows UNC path (e.g. \\host\share\myfile.txt) always has a drive and a root (here, \\host\share and \, respectively).

由于 POSIX 总是没有盘符的，因此 POSIX 路径只要包含根目录就是 [[绝对路径]]；而 Windows 系统中的绝对路径必须同时包含盘符和根目录才算。

此外，对于 Windows 来说，存在一类既不是绝对路径，也不是相对路径的路径对象；即没有盘符却有根目录，或有盘符而没有根目录的路径对象。

```Python
>>> PureWindowsPath("/Windows").is_absolute()
False
```

## 初始化的参数

初始化时，可以传入任意个 [[参数]]，它们是代表路径片段的 [[字符串]]，或者是另一个路径对象。

```Python
>>> PurePath("foo", "some/path", "bar")
PurePosixPath('foo/some/path/bar')

>>> PurePath(Path("foo"), Path("bar"))
PurePosixPath('foo/bar')
```

路径片段参数也可为空，此时返回的是当前路径 `.`。

```Python
>>> PurePath("")
PurePosixPath('.')
```

### 锚点规则

初始化传入多个路径参数时，需找出其中的 [[锚点]]，作为路径合并的基础。

如果全都是相对路径，那么锚点就是当前目录 `.`，后续的所有路径都会基于当前目录。

根目录和根目录之间没有串联的可能，所以当传入的参数是若干个包含根目录的路径时，只有最后一个包含根目录的路径可以被视作锚点，锚点及其之后的相对路径会被保留。

```Python
>>> PurePath("/etc", "init.d", "/usr", "lib64")
PurePosixPath('/usr/lib64')
```

当传入的参数是若干个带有盘符的路径时，只有最后一个带有盘符的路径及其之后的路径会被保留。

```Python
>>> PureWindowsPath("c:/Windows", "Files.c", "d:/Windows.old", "Files.d")
PureWindowsPath('d:/Windows.old/Files.d')
```

当多个参数内既有盘符又有根目录时，要优先考虑盘符的情况。
在满足盘符的条件后，根目录的选择可以脱离盘符进行运算。

```Python
>>> PureWindowsPath("c:/Windows", "/Files.c")
PureWindowsPath("c:/Files.c")
```

### / 和 . 和 .. 规则

路径片段中包含的假斜线 `//` 和当前路径 `.` 会直接被消除，同时路径末尾的斜杠也不会保留。

```Python
>>> PurePath("foo//bar")
PurePosixPath('foo/bar')

>>> PurePath("foo/./bar")
PurePosixPath('foo/bar')

>>> PurePath("foo/bar/")
PurePosixPath('foo/bar')
```

与当前路径不同，双点 `..` 会被保留。
这是因为纯路径类不涉及真实文件系统，不能正确处理 [[符号链接]] 的含义。

```shell
\-- Dir
    |-- LinkDir -> RealDirA/SubDir
    |-- RealDirA
    |   \-- SubDir
    \-- RealDirB
```

```Python
>>> PurePath("LinkDir/../SubDir")
PurePosixPath('LinkDir/../SubDir')  # 符号链接，实际能否访问，纯路径不考虑

>>> PurePath("RealDirA/../RealDirB")
PurePosixPath('RealDirA/../RealDirB')
```

## 路径计算

### 斜杠运算符

可以直接使用斜杠运算符 `/` 创建子路径。
它会返回一个新的路径对象，表达效果上更直观。

斜杠运算符的最终结果也遵循初始化参数里的各种先后规则。

```Python
>>> PurePath("local") / "bin"
PurePosixPath('local/bin')  # 字符串

>>> PurePath("local") / PurePath("lib")
PurePosixPath('local/lib')  # 路径对象

>>> "usr" / PurePath("local")
PurePosixPath('usr/local')  # 字符串在运算符左侧

>>> PurePath("/usr") / PurePath("/etc")
PurePosixPath('/etc')  # 取最后一个根目录
```

### 字符表示

路径的字符串表示会按照原始的文件系统路径风格显示，如 Windows 系统使用反斜杠 `\`。

```Python
>>> str(PurePosixPath("/etc"))
'/etc'

>>> str(PureWindowsPath("c:/Program Files"))
'c:\\Program Files'
```

`str()` 返回的是 [[Unicode]] 字符串，这在 Windows 下是规范表示法，但 POSIX 有时也会需要用 `bytes()` 方法得到 [[字节]] 表示。

### 排序和比较

相同风格的路径之间可以进行排序和比较。
它们的比较，相当于是对路径的各部分构成的字符串数组进行比较。
其中需要注意，POSIX 系统对大小写敏感，Windows 则不然。

```Python
>>> PurePosixPath('foo') == PurePosixPath('FOO')
False  # Poxis 系统对大小写敏感

>>> PureWindowsPath('foo') == PureWindowsPath('FOO')
True  # Windows 对大小写不敏感。

>>> PureWindowsPath('C:') < PureWindowsPath('d:')
True
```

跨风格的路径，必然是不相等的；尝试排序的话会抛出 `TypeError` [[异常]]。

```Python
>>> PureWindowsPath('foo') == PurePosixPath('foo')
False

>>> PureWindowsPath('foo') < PurePosixPath('foo')
Traceback (most recent call last): File "<stdin>", line 1, in <module>
TypeError: '<' not supported between instances of 'PureWindowsPath' and 'PurePosixPath'
```

### 哈希

此外，路径对象是可 [[哈希]] 的。
和比较一样，哈希也是对于路径的各部分构成的字符串数组而言的，所以也会有路径风格的差异。

```Python
>>> PureWindowsPath('FOO') in { PureWindowsPath('foo') }
True
```

## 路径属性

### parts 各个部分

返回一个 [[元组]]，各项是路径各部分的字符串，包括盘符和根路径（如果有的话）。

```Python
>>> PurePosixPath("/usr/bin/python3").parts
('/', 'usr', 'bin', 'python3')

>>> PureWindowsPath("c:/Windows/lib").parts
('c:\\', 'Windows', 'lib')  # Windows 的盘符和根目录会合并为一个项

>>> PureWindowsPath("/Windows/lib").parts
('\\', 'Windows', 'lib')  # 不包含盘符时，根目录会单独列出
```

### drive 盘符

返回一个字符串；不存在盘符时，返回空字符串。

[[UNC]] 路径也被认作一种驱动器。

```Python
>>> PureWindowsPath("//host/share").drive
'\\\\host\\share'  # 反斜杠转义
```

### root 根目录

返回一个字符串。
Windows 系统也有根目录，与盘符是无关的。

```Python
>>> PureWindowsPath("c:/Program Files").root
'\\'

>>> PureWindowsPath("c:Program Files").root
''  # 没有根目录，但有盘符
```

### anchor 锚点

返回一个字符串。

> A path which has either a drive _or_ a root is said to be anchored. Its anchor is the concatenation of the drive and root. Under POSIX, "anchored" is the same as "absolute".

一个盘符或者根目录被称为 [[锚点]]。
在 POSIX 系统中，锚点就相当于是绝对路径；而 Windows 系统中，锚点通常是盘符，盘符不存在时才是根目录。

```Python
>>> PureWindowsPath('c:/Program Files').anchor
'c:\\'

>>> PureWindowsPath('/Program Files').anchor
'\\'

>>> PurePosixPath('/etc').anchor
'/'
```

### parent 逻辑父路径

返回一个路径对象。

根目录的父路径仍然会返回根目录。

```Python
>>> PurePosixPath("/").parent
PurePosixPath('/')
```

相对路径可以到达的最顶层父路径是当前目录。

```Python
>>> PurePosixPath("d").parent
PurePosixPath('.')
```

### parents 父路径序列

返回一个序列，使用从 0 开始的 [[下标]]，可以相对当前路径从近到远，依次访问所有的父路径对象。

这是纯路径计算，因此不会涉及到符号链接和 `..` 的转换。

```Python
>>> p = PurePosixPath("link/../python3")
>>> p.parents[0]
PurePosixPath('/link/..')

>>> p.parents[1]
PurePosixPath('/link')

>>> p.parents[2]
PurePosixPath('.')
```

可以使用 `list` 可以对序列进行强制转化。

```Python
>>> p = PurePosixPath("/usr/lib/python3")
>>> list(p.parents)
[PurePosixPath('/usr/lib'), PurePosixPath('/usr'), PurePosixPath('/')]
```

### name 文件名

返回一个字符串。
`name` 指向 `parts` 除盘符和根目录外的最后一部分。

```Python
>>> PurePosixPath("lib/setup.py").name
'setup.py'

>>> PurePosixPath("/").name
''  # 根目录不属于 name
```

### suffix 和 suffixes 后缀

**`suffix`** 返回一个字符串，它指向 `name` 的最后一个 [[后缀]]，包含用于标识后缀的那个点。

```Python
>>> PurePosixPath("lib/library.tar.gz").suffix
'.gz'
```

文件名不存在，或者后缀不存在时，将返回空字符串。
文件名以点结尾时，也视作后缀不存在。

```Python
>>> PurePosixPath("/").name, PurePosixPath("/").suffix
('', '')

>>> PurePosixPath(".gitignore").name, PurePosixPath(".gitignore").suffix
('.gitignore', '')  # name 存在，但此处的点代表隐藏文件不代表后缀

>>> PurePosixPath("lib/library.tar.gz.").name, PurePosixPath("lib/library.tar.gz.").suffix
('library.tar.gz.', '')  # name 存在，但点位于最后一位，说明无后缀
```

**`suffixes`** 返回一个字符串 [[列表]]，路径可能有多个后缀，将按顺序返回列表；不存在时返回空列表。

顺序：指从左到右，`suffix` 在最后一个。

```Python
>>> PurePosixPath("lib/library.tar.gz").suffixes
['.tar', '.gz']
```

### stem 文件名主干

返回一个字符串。
`stem` 是去掉最后一个后缀的 `name`，相当于 `name - suffix`。

```Python
>>> PurePosixPath("lib/library.tar.gz").stem
'library.tar'
```

## is 判断方法

### is_absolute()

判断是否是绝对路径。

### is_relative_to(other)

判断是否是相对于 *other* 的相对路径。

允许传入多个参数，参数的使用规则和 [[#relative_to(other)]] 一致。

```Python
>>> PurePosixPath("a/b/c").is_relative_to("a", "b")
True

>>> PurePosixPath("a/b/c").is_relative_to("a", "a/b")
False
```

### is_reserved()

Windows 系统会预设部分路径作为 [[保留路径]]；POSIX 则不存在，总是返回 `False`。

```Python
>>> PureWindowsPath("nul").is_reserved()
True

>>> PurePosixPath("nul").is_reserved()
False
```

## as 转换方法

as 方法可以将路径对象按需求转换为各种风格的字符串返回。

### as_posix()

返回 POSIX 路径风格的字符串，实际上就是指使用正斜杠的路径字符串。

```Python
>>> PureWindowsPath("c:/Windows").as_posix()
'c:/Windows'
```

### as_uri()

返回 file [[URL]] 类型的字符串，必须是绝对路径才能转换，否则抛出 `ValueError`。

```Python
>>> PurePosixPath("/etc/init.d").as_uri()
'file:///etc/init.d'

>>> PureWindowsPath("c:/Windows").as_uri()
'file:///c:/Windows'
```

## with 修改方法

with 方法会返回一个新的路径对象，按要求修改部分路径。

### with_name()

返回一个新路径对象，带有修改后的 `name`；必须要有 `name` 才能修改，否则抛出 `ValueError`。

### with_stem()

返回一个新路径对象，带有修改后的 `stem`；必须要有 `stem` 才能修改，否则抛出 `ValueError`。

### with_suffix()

返回一个新路径对象，带有修改后的 `suffix`。
原本的路径对象可以没有 `suffix`，该方法会添加新的后缀；传入参数为空字符串时，该方法会删除原有的后缀。

注意：传入的参数需要标识后缀的点，否则会抛出 `ValueError`。

```Python
>>> PurePath("library.tar.gz").with_suffix(".bz2")
PurePosixPath('library.tar.bz2')

>>> PurePath("library.tar.gz").with_suffix("")
PurePosixPath('library.tar')

>>> PurePath("library").with_suffix(".tar")
PurePosixPath('library.tar')
```

## 其他方法
    
### joinpath(other)

返回一个新路径对象，将传入的多个参数和原路径对象拼接在一起。

传入参数的各种细节和斜杠运算符是一致的。

```Python
>>> PurePath("/a").joinpath("/b")
PurePosixPath('/b')
```

### relative_to(other)

返回一个新路径对象，计算原路径对象相对于 *other* 的相对路径版本。

*other* 可以是多个参数。
这些参数会自先按照初始化的规则，组合成一个类似路径对象的结构，然后才计算相对路径。

```Python
>>> PurePath("a/b/c").relative_to("a", "b")
PurePosixPath('c')  # 先组合成 a/b 再计算相对
```

如果不可计算，抛出 `ValueError`。
绝对路径和相对路径之间必然是不可计算的，即使它们在真实文件系统上可能是可计算的。

### match(pattern) 匹配判断

返回一个 [[布尔]] 变量。

将路径对象和 [[通配符]] *pattern* 进行匹配，如果匹配成功则返回 `True`，否则 `False`。
它会遵循相应平台的规则，例如 Windows 对大小写不敏感。

```Python
>>> PureWindowsPath("a.py").match("*.PY")
True
```

如果 *pattern* 是相对的，那么路径对象既可以是绝对路径也可以是相对路径，匹配从右向左匹配；如果 *pattern* 是绝对的，那么路径对象必须是绝对路径，且需要从左向右完全匹配。

```Python
>>> PurePosixPath("/etc/init.d").match("*.d")
True

>>> PurePosixPath("/etc/init.d").match("/*.d")
False
```

---

## 版本差异

- 3.9 新增方法
    - [[#is_relative_to(other)]]
    - [[#with_stem()]]

## 系列笔记

- [[pathlib (1) 面向对象]]
- [[pathlib (2) 纯路径类]] **<**
- [[pathlib (3) 路径类]]
- [[pathlib (4) 源码阅读 PurePath]]
