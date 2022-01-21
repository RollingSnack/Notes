---
Title: pathlib (2) 纯路径类
Tags: [类型/笔记, 类型/系列/pathlib, 编程语言/Python]
---

# pathlib (2) 纯路径类

![[Concepts/pathlib]]

## 两种路径风格

纯路径 [[类]] 不访问真实的 [[文件系统]]，只参与路径计算；因此无论在什么系统上都可以实例化 3 个纯路径类。

`PurePosixPath` 和 `PureWindowsPath` 分别表示两种路径风格，前者是 [[POSIX]] 的路径风格，后者是以 [[Windows]] 系统为代表的路径风格。

可直接使用 `PurePath`，它会在运行时，根据实际的系统确定路径风格，并实例化为上述两种中具体的某一种 [[对象]]。

```Python
>>> PurePath("setup.py")
PurePosixPath('setup.py')  # 在 Unix 系统上运行时实例化
```

## 绝对路径和相对路径

文件系统使用 [[树]] 形目录存储文件数据，树的根称为 [[根目录]]。
Windows 系统在根目录之上，还有 [[盘符]]，一个盘符下面有且仅有一个根目录。

> All paths can have a drive and a root. For POSIX paths, the drive is always empty.

根据 [PEP 428](https://www.python.org/dev/peps/pep-0428/) 的设计，所有路径对象都带有一个盘符和一个根目录，只是 POSIX 下的盘符总是为空。

> A relative path has neither drive nor root.

既没有盘符也没有根目录的路径，才被称为 [[相对路径]]。

> A POSIX path is absolute if it has a root. A Windows path is absolute if it has both a drive _and_ a root. A Windows UNC path (e.g. \\host\share\myfile.txt) always has a drive and a root (here, \\host\share and \, respectively).

由于 POSIX 总是没有盘符的，因此 POSIX 路径只要包含根目录就是 [[绝对路径]]；而 Windows 系统中的绝对路径必须同时包含盘符和根目录才算。

此外，对于 Windows 来说，存在一类既不是绝对路径，也不是相对路径的路径对象，即，没有盘符却有根目录，或有盘符而没有根目录的路径对象。

```Python
>>> PureWindowsPath("/Windows").is_absolute()
False

>>> PureWindowsPath("c:windows").is_absolute()
False
```

## 初始化

初始化时，可以传入任意个 [[参数]]，它们是代表路径片段（*pathsegment*）的 [[字符串]]，或者是另一个路径对象。

```Python
>>> PurePath("foo", "some/path", "bar")
PurePosixPath('foo/some/path/bar')

>>> PurePath(Path("foo"), Path("bar"))
PurePosixPath('foo/bar')
```

路径片段参数也可为空，此时返回的是当前路径 `.`。

```Python
>>> PurePath()
PurePosixPath('.')
```

通常情况下，如果传入的参数中有多个绝对路径，以最后一个绝对路径作为 [[锚点]]。
只有锚点及其之后的路径会被保留。
此处逻辑是在模仿 [`os.path.join`](https://docs.python.org/3/library/os.path.html#os.path.join)。

```Python
>>> PurePath("/etc", "init.d", "/usr", "lib64")
PurePosixPath('/usr/lib64')
```

**注意：** Windows 有特殊情况，若参数携带了多个盘符和根目录，计算锚点时会将二者拆分，然后优先定位盘符，再在其后的路径中依据根目录确定锚点。
如下所示，只有第一个参数是绝对路径，但是锚点是分别根据最后两个参数得到的。

```Python
>>> PureWindowsPath("d:/Data", "/Documents", "c:Windows.old", "/Files.c")
PureWindowsPath("c:/Files.c")
```

路径片段中包含的假斜线 `//` 和当前路径 `.` 会直接被消除，同时路径末尾的斜杠也不会保留。

```Python
>>> PurePath("foo//bar")
PurePosixPath('foo/bar')

>>> PurePath("foo/./bar")
PurePosixPath('foo/bar')

>>> PurePath("foo/bar/")
PurePosixPath('foo/bar')
```

但是双点 `..` 符号会被保留。
这是因为纯路径类不涉及真实文件系统，不能正确处理 [[符号链接]] 的含义。

```Console
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

## 魔术方法（*magic method*）

### 斜杠运算符

可以直接使用斜杠运算符 `/` 创建子路径。

```Python
>>> PurePath("local") / "bin"
PurePosixPath('local/bin')  # 字符串

>>> PurePath("local") / PurePath("lib")
PurePosixPath('local/lib')  # 路径对象

>>> "usr" / PurePath("local")
PurePosixPath('usr/local')  # 字符串在运算符左侧

>>> PurePath("/usr") / PurePath("/etc")
PurePosixPath('/etc')  # 锚点改变
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
它们的比较，相当于是对路径各片段组成的字符串数组进行比较。

```Python
>>> PurePosixPath('foo/bar') < PurePosixPath('foo/car')
True # 字符串比较，foo == foo，bar < car
```

由于是数组比较，所以父目录会小于其下的子目录，以确保从小到大排序时，顺序是符合常理的。

```Python
>>> PurePosixPath('foo') < PurePosixPath('foo/bar')
True # 父目录小于子目录
```

其中需要注意，POSIX 系统对大小写敏感，Windows 则不然。

```Python
>>> PurePosixPath('foo') == PurePosixPath('FOO')
False  # POSIX 对大小写敏感

>>> PureWindowsPath('foo') == PureWindowsPath('FOO')
True  # Windows 对大小写不敏感。
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

此外，路径对象是 [[可哈希]]（*hashable*）的，而且也有路径风格的差异。

```Python
>>> PureWindowsPath('FOO') in { PureWindowsPath('foo') }
True
```

## 属性

### parts

返回一个字符串 [[元组]]，分别表示路径的各个部分，包括盘符和根路径（如果有的话）。

```Python
>>> PurePosixPath("/usr/bin/python3").parts
('/', 'usr', 'bin', 'python3')

>>> PureWindowsPath("c:/Windows/lib").parts
('c:\\', 'Windows', 'lib')  # Windows 的盘符和根目录会合并为一个项

>>> PureWindowsPath("/Windows/lib").parts
('\\', 'Windows', 'lib')  # 盘符或根目录某一项缺失时，另一项会单独列出
```

### drive 盘符

返回一个字符串；不存在盘符时，返回空字符串。

[[UNC]] 路径也被认作一种驱动器。

```Python
>>> PureWindowsPath("//host/share").drive
'\\\\host\\share'  # 反斜杠转义
```

### root

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

在 POSIX 系统中，锚点就相当于是绝对路径；而 Windows 系统中，同时存在盘符和根目录的话，二者共同组成锚点，否则其中之一会被作为锚点。
既没有盘符也没有根目录，那么锚点为空。

```Python
>>> PureWindowsPath('c:/Program Files').anchor
'c:\\'

>>> PureWindowsPath('/Program Files').anchor
'\\'

>>> PurePosixPath('/etc').anchor
'/'

>>> PurePosixPath('etc').anchor
''  # 锚点为空
```

### parent 和 parents 父路径

**`parent`** 返回一个路径对象，为当前路径的逻辑父路径。

根目录的父路径仍然会返回根目录。

```Python
>>> PurePosixPath("/").parent
PurePosixPath('/')
```

相对路径可以到达的最顶层的父路径是当前目录。

```Python
>>> PurePosixPath("d").parent
PurePosixPath('.')
```

**`parents`** 返回一个序列，使用从 0 开始的 [[下标]]，可以相对当前路径从近到远，依次访问所有的父路径对象。

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

parents 是一个没有对外暴露的类而不是列表，使用 `list` 可以对序列进行强制转化。

```Python
>>> p = PurePosixPath("/usr/lib/python3")
>>> list(p.parents)
[PurePosixPath('/usr/lib'), PurePosixPath('/usr'), PurePosixPath('/')]
```

### name

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
文件名以单个点结尾也视作后缀不存在。

```Python
>>> PurePosixPath("/").name, PurePosixPath("/").suffix
('', '')

>>> PurePosixPath(".gitignore").name, PurePosixPath(".gitignore").suffix
('.gitignore', '')  # name 存在，但 suffix 不存在，此处的点代表隐藏文件

>>> PurePosixPath("lib/library.tar.gz.").name, PurePosixPath("lib/library.tar.gz.").suffix
('library.tar.gz.', '')  # name 存在，但点位于最后一位，同样说明无 suffix
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

is 判断方法会返回一个 [[布尔]] 变量。

### is_absolute()

判断是否是绝对路径。

### is_relative_to(other)

判断是否是相对于 *other* 的相对路径。

允许传入多个参数，参数会先按照初始化规则组成一个路径对象，然后再判断。

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
传入的参数需要标识后缀的点，否则不会被视作一个后缀，进而抛出 `ValueError`。

```Python
>>> PurePath("library.tar.gz").with_suffix(".bz2")
PurePosixPath('library.tar.bz2')
```

传入参数为空字符串时，该方法会删除原有的后缀。

```Python
>>> PurePath("library.tar.gz").with_suffix("")
PurePosixPath('library.tar')
```

原本的路径对象可以没有 `suffix`，该方法会添加新的后缀。

```Python
>>> PurePath("library").with_suffix(".tar")
PurePosixPath('library.tar')
```

## 其他方法
    
### joinpath(other)

返回一个新路径对象，将传入的多个参数和原路径对象拼接在一起。

细节和斜杠运算符一致。

```Python
>>> PurePath("/a").joinpath("/b")
PurePosixPath('/b')
```

### relative_to(other)

返回一个新路径对象，计算原路径对象相对于 *other* 的相对路径版本。

*other* 允许是多个参数，参数会先按照初始化规则组成一个路径对象，然后才计算相对路径。

```Python
>>> PurePath("a/b/c").relative_to("a", "b")
PurePosixPath('c')
```

如果不可计算，抛出 `ValueError`。
绝对路径和相对路径之间必然是不可计算的，即使它们在真实文件系统上可能是可计算的。

### match(pattern) 匹配判断

返回一个布尔变量。

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
