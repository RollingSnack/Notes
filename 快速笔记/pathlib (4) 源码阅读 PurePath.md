---
Title: pathlib (4) 源码阅读 PurePath
Tags: [Note, Series, Python3]
---

# pathlib (4) 源码阅读 PurePath

![[Concepts/pathlib]]

[[Cpython]] 中的 pathlib [[模块]] 存放在 `Lib/pathlib.py` 文件内。

本篇笔记将首先从功能角度，分析其代码在实现各个功能上的一些细节。

## 两种路径风格

纯路径不访问文件系统，只参与路径计算；因此无论在什么系统上都可以实例化前 3 个类。

`PurePosixPath` 和 `PureWindowsPath` 分别表示两种路径风格，后者是以 [[Windows]] 系统为代表的路径风格。

当使用 `PurePath` 时，会根据当前系统的路径风格，实例化为具体的某一种。

```Python
>>> PurePath("setup.py")
PurePosixPath('setup.py')  # 在 Unix 系统上运行时实例化
```

源代码中，指定风格实例化的判断分支的代码写到了 `__new__` 中。

```Python
# CPython - Lib/pathlib.py
def __new__(cls, *args):
    """Construct a PurePath from one or several strings and or existing
    PurePath objects. The strings and path objects are combined so as
    to yield a canonicalized path, which is incorporated into the
    new PurePath object.
    """
    if cls is PurePath:
        cls = PureWindowsPath if os.name == 'nt' else PurePosixPath
    return cls._from_parts(args)
```

### 初始化参数

初始化时，可以传入任意个参数，它们是代表路径片段的字符串，或者是另一个路径对象。

```Python
>>> PurePath("foo", "some/path", "bar")
PurePosixPath('foo/some/path/bar')

>>> PurePath(Path("foo"), Path("bar"))
PurePosixPath('foo/bar')
```

路径片段参数可为空时，此时返回的是当前路径。

```Python
>>> PurePath("")
PurePosixPath('.')
```

路径片段中包含的假斜线(`//`)和当前路径(`.`)会直接被消除，同时路径末尾的斜杠也不会保留。
因此，不能直接使用 pathlib 模块像处理文件一样处理网址。
（形如 `http://` 等协议头的双斜线或许会无法被识别？）

```Python
>>> PurePath("foo//bar")
PurePosixPath('foo/bar')

>>> PurePath("foo/./bar")
PurePosixPath('foo/bar')

>>> PurePath("foo/bar/")
PurePosixPath('foo/bar')
```

与当前路径不同，双点(`..`)会被保留。
这是为了正确处理 [[软链接]] 的含义。

```shell
\-- Dir
    |-- LinkDir -> RealDirA/SubDir
    |-- RealDirA
    |   \-- SubDir
    \-- RealDirB
```

```Python
>>> PurePath("LinkDir/../SubDir")
PurePosixPath('LinkDir/../SubDir')  # 软链接，实际能否访问，纯路径对象不处理

>>> PurePath("RealDirA/../RealDirB")
PurePosixPath('RealDirA/../RealDirB')
```

传入若干个绝对路径参数时，只以最后一个绝对路径作为 [[锚点]]。

```Python
>>> PurePath("/etc", "/usr", "lib64")
PurePosixPath('/usr/lib64')
```

### 绝对路径

> All paths can have a drive and a root. For POSIX paths, the drive is always empty.

根据 [PEP 428](https://www.python.org/dev/peps/pep-0428/) 的设计，所有路径都有一个 [[盘符]] 和一个 [[根目录]]，只是 POSIX 路径的盘符总是为空。

> A POSIX path is absolute if it has a root. A Windows path is absolute if it has both a drive _and_ a root. A Windows UNC path (e.g. \\host\share\myfile.txt) always has a drive and a root (here, \\host\share and \, respectively).

POSIX 路径只要包含根目录就是绝对路径，而 [[Windows]] 系统中的路径必须同时包含盘符和根目录才算。
当执行如下代码时，由于第二个路径片段的盘符缺失，因此不是绝对路径，不满足前述锚点的情况；但是两个参数都表示根目录，所以仍然会取最后一个根目录返回。

```Python
>>> PureWindowsPath("c:/Windows", "/Windows.old")
PureWindowsPath('c:/Windows.old')  # 盘符保留，根目录变成 /Windows.old
```

## 纯路径计算

### 方法和运算符重载

#### 斜杠运算符(`/`)

可以直接使用斜杠运算符创建子路径，返回一个新的路径对象，表达效果更为直观。

```Python
>>> p = PurePath("local")
PurePosixPath('local')

>>> p / "bin"
PurePosixPath('local/bin')  # 字符串

>>> p / PurePath("lib")
PurePosixPath('local/lib')  # 路径对象

>>> "usr" / p
PurePosixPath('usr/local')  # 字符串在运算符左侧
```

斜杠运算符是 [[重载]] 了 `__truediv__` 和 `__rtruediv__` 两个方法。

前者是当路径对象出现在运算符左侧时 **首先** 执行。
上述例子中，`p1` 和 `p2` 走的是该逻辑。

```Python
# CPython - Lib/pathlib.py
def __truediv__(self, key):
    try:
        return self._make_child((key,))
    except TypeError:
        return NotImplemented
```

后者是当路径对象出现在运算符右侧时，且左侧的对象不支持斜杠运算符操作时执行。
上述例子中，`p3` 是该逻辑。
左侧的字符串不支持对路径对象使用斜杠运算符，然后才使用右侧的路径对象定义的方法。

```Python
# CPython - Lib/pathlib.py
def __rturediv__(self, key):
    try:
        return self._from_parts([key] + self._parts)
    except TypeError:
        return NotImplemented
```

#### os.PathLike 接口

python 3.6 版本新增的功能，用 `os.PathLike` 表示那些文件系统中的路径对象的抽象基类。
如库自带的 `open`，传入的参数就是这个抽象基类的各种实现。

事实上，这个接口只返回 `str` 字符串或者 `bytes` 字节串，故而它也能兼容原本的字符串型路径。
实现了这个接口的地方，`PurePath` 对象也可被接受，因为重载了 `__fspath__` 方法。

```Python
# CPython - Lib/pathlib.py
def __fspath__(self):
    return str(self)
```

#### 字符串表示

路径的字符串表示会按照原始的文件系统路径风格显示，如 Windows 系统使用反斜杠(`\`)。

```Python
>>> p = PurePosixPath("/etc")
>>> str(p)
'/etc'

>>> q = PureWindowsPath("c:/Program Files")
>>> str(q)
'c:\\Program Files'
```

这也是通过重载了 `__str__` 实现的。

```Python
# CPython - Lib/pathlib.py
def __str__(self):
    """Return the string representation of the path, suitable for
    passing to system calls."""
    try:
        return self._str
    except AttributeError:
        self._str = self._format_parsed_parts(self._drv, self._root,
                                              self._parts) or '.'
        return self._str
```

### attribute 属性

从类变量 `__slots__` 可以看出，路径对象只存储了如下 7 个属性，不会再动态增加。
其中：
- `_drv`、`_root` 和 `_parts` 是在初始化阶段生成的盘符、根目录和完整的分部件；
- `_hash` 和 `_cached_cparts` 涉及到哈希和排序比较，这里不做深入；
- `_str` 和 `_pparts` 分别会随着 `__str__` 和 `parts` 的第一次调用而生成。

```Python
# CPython - Lib/pathlib.py
__slots__ = (
    '_drv', '_root', '_parts',
    '_str', '_hash', '_pparts', '_cached_cparts',
)
```

这里，`_parts` 和 `_pparts` 的内容是一样的；但是前者是列表，正常情况下只在内部使用；后者是元祖，接口会暴露给外部。

```Python
# python 的单下划线只是个约定俗成的私有变量，解释器不会阻止程序从外部访问单下划线变量
# 此处只是为了举例说明，实际不应该出现这种 “危险” 代码
>>> p = PurePath("/etc")
>>> p._parts
['/', 'etc']

>>> p.parts  # 必须先访问一次 parts 特征属性，路径对象才会创建 _pparts 属性
>>> p._pparts
('/', 'etc')
```

元祖是不允许修改的，因此即便暴露给外部也可以避免出现不一致的情况。
否则的话可以参考如下例子。

```Python
>>> p._pparts[1] = 'lib'  # 尝试修改 _pparts，元祖，报错
TypeError: 'tuple' object does not support item assignment

>>> p._parts[1] = 'lib'  # 尝试修改 _parts，列表，成功，但产生了不一致
>>> p._parts
['/', 'lib']


>>> p.parts
('/', 'etc')  # 用得还是上一次生成的 _pparts

>>> p
PurePosixPath('/etc')  # 还是原来的 /etc
```

### property 属性

路径对象许多方便使用的属性都不是直接存储的属性，而是由 `@property` [[装饰器]] 修饰的方法而得到的属性。

#### parts 各个部分

返回的是各个部分组合而成的元祖。

```Python
>>> PurePosixPath("/usr/bin/python3").parts
('/', 'usr', 'bin', 'python3')

>>> PureWindowsPath("c:/Windows/lib").parts
('c:\\', 'Windows', 'lib')
```

正如前文所说，`parts` 在初次调用时会生成 `_pparts`，之后的调用直接返回 `_pparts`。

```Python
# CPython - Lib/pathlib.py
@property
def parts(self):
    """An object providing sequence-like access to the
    components in the filesystem path."""
    # We cache the tuple to avoid building a new one each time .parts
    # is accessed. XXX is this necessary?
    try:
        return self._pparts
    except AttributeError:
        self._pparts = tuple(self._parts)
        return self._pparts
```

#### drive、root、anchor

这三个分别是 盘符、根目录、锚点。
他们返回的是字符串，当相应属性不存在时，返回空字符串。

```Python
>>> PureWindowsPath("c:/Program Files").drive
'c:'
```

#### parent 逻辑父路径

`parent` 返回的也是个路径对象。

```Python
>>> PurePosixPath("a/b/c/d").parent
PurePosixPath('a/b/c')  # 返回的也是路径对象

>>> PurePosixPath("/").parent
PurePosixPath('/')  # 锚点的父目录还是锚点

>>> PurePosixPath("d").parent
PurePosixPath('.')  # 无子目录的相对路径，其父目录是当前目录
```

这个属性只涉及词法操作，不涉及文件系统。
库的源代码是直接使用 `parts[:-1]` 的切片操作；因此如果有软链接，它是不会正确解析的。

```Python
# CPython - Lib/pathlib.py
@property
def parent(self):
    """The logical parent of the path."""
    drv = self._drv
    root = self._root
    parts = self._parts
    if len(parts) == 1 and (drv or root):
        return self
    return self._from_parsed_parts(drv, root, parts[:-1])
```

#### parents 父路径序列

`parents` 返回了一个 `_PathParents` 类。
从命名可以看出，它不向外暴露，以类似于序列的方式访问若干层级的父路径。

```Python
>>> p = PureWindowsPath('/usr/lib/python3')
>>> p.parents[0]
PureWindowsPath('/usr/lib')

>>> p.parents[2]
PureWindowsPath('/')
```

首先关注类的定义，继承了 `collection.abc.Sequence` 类，那么各种序列的操作，它都是可以用的，如 `for-in` 语法循环迭代。

其次关注 `__init__`，`path` 参数是一个路径对象。
这里的注释特别强调了，不存储路径对象，是为了避免循环引用（如：PurePath -> PathParents -> PurePath）。
为了使用 `PurePath` 中的 `_from_parsed_parts`，则声明了一个 `type(path)` 的实例属性。

最后再关注重载的 `__getitem__`，每个父路径对象都是在需要的时候才创建的，而不是完整创建再按下表索引。
此外，也无法使用负数下标索引。

```Python
# CPython - Lib/pathlib.py
class _PathParents(Sequence):
    """This object provides sequence-like access to the logical ancestors
    of a path. Don't try to construct it yourself."""
    __slots__ = ('_pathcls', '_drv', '_root', '_parts')

    def __init__(self, path):
        # We don't store the instance to avoid reference cycles
        self._pathcls = type(path)
        self._drv = path._drv
        self._root = path._root
        self._parts = path._parts

    def __len__(self):
        if self._drv or self._root:
            return len(self._parts) - 1
        else:
            return len(self._parts)

    def __getitem__(self, idx):
        if idx < 0 or idx >= len(self):
            raise IndexError(idx)
        return self._pathcls._from_parsed_parts(self._drv, self._root,
                                                self._parts[:-idx - 1])

    def __repr__(self):
        return "<{}.parents>".format(self._pathcls.__name__)
```

#### name、suffix(es)、stem

`name` 属性指向 `parts`（除盘符和根目录外）的最后一部分，即不包含任何子目录的那一部分。

```Python
>>> PurePosixPath("lib/setup.py").name
'setup.py'

>>> PurePosixPath("/").name
''  # 根目录不纳入 name 属性，尽管它的确是 parts 的最后一部分
```

`suffix` 属性指向 `name` 属性的最后一个后缀，不存在时返回空字符串。

```Python
>>> PurePosixPath("lib/library.tar.gz").suffix
'.gz'  # 需要包含代表后缀的那个点
```

由于 `suffix` 只关注最后一个后缀，因此源码里使用的是 `rfind` 实现的。

```Python
# CPython - Lib/pathlib.py
@property
def suffix(self):
    """
    The final component's last suffix, if any.

    This includes the leading period. For example: '.txt'
    """
    name = self.name
    i = name.rfind('.')
    if 0 < i < len(name) - 1:
        return name[i:]
    return ''
```

`suffixes`，当路径可能有多个后缀时，按顺序返回列表；不存在时返回空列表。

```Python
>>> PurePosixPath("lib/library.tar.gz").suffixed
['.tar', '.gz']  # 点
```

在处理所有的后缀时，需要注意诸如 `.gitignore` 等以点(`.`)开头的文件。
如果直接使用 `split` 的话会生成一个空字符串作为返回列表的第一个。
源码使用得失 `lstrip` 将第一个点(`.`)删去。

```Python
# CPython - Lib/pathlib.py
@property
def suffixes(self):
    """
    A list of the final component's suffixes, if any.

    These include the leading periods. For example: ['.tar', '.gz']
    """
    name = self.name
    if name.endswith('.'):
        return []
    name = name.lstrip('.')
    return ['.' + suffix for suffix in name.split('.')[1:]]
```

`stem` 是无最后一个后缀的文件名，相当于 `name - suffix`。

```Python
>>> PurePosixPath("lib/library.tar.gz").stem
'library.tar'
```

### is 判断方法

## 系列

- [[pathlib (1) 面向对象]]
- [[pathlib (2) 纯路径类]]
- [[pathlib (3) 路径类]]
- [[pathlib (4) 源码阅读 PurePath]] **<**