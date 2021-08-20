---
Title: pathlib (3) 路径类
Tags: [Note, Series, Python3]
---

# pathlib (3) 路径类

![[pathlib]]

## 继承自纯路径类

三种路径 [[类]] 分别 [[继承]] 自三种纯路径类。
除了 [[父类]] 提供的纯路径计算以外，路径类还封装了系统调用的 [[I/O]] 操作。

和纯路径类一样，`Path` 类会自动根据当前的 [[文件系统]] 路径风格，转化成 `PosixPath` 或 `WindowsPath`。
但是手动实例化的话，只能使用与当前系统风格相同的类，错误使用的话会抛出 `NotImplementedError` [[异常]]。

## 类方法

### cwd()

返回一个表示当前目录的 [[绝对路径]] 的 [[对象]]。

```Python
>>> Path.cwd()
PosixPath('/home/anyone/pathlib')
```

### home()

返回一个表示当前 [[用户]] [[家目录]] 的绝对路径的新对象。

## 文件操作

### mkdir(...)

原型：

```Python
Path.mkdir(mode=0o777, parents=False, exist_ok=False)
```

根据给定的路径，新建一个目录；如果给出了 *mode* 需要与 [[进程]] 的 *umask* 值合并来决定最终的模式和权限。

*parents* 设置为 `True` 后，从当前路径到目标路径中不存在的目录都将被创建；这些父路径的权限是默认的权限，不跟随 *mode* 改变；相当于 `mkdir -p` 命令。
设置为 `False` 且父路径不存在时，抛出 `FileNotFoundError`。

- 3.5 版本加入了 `exist_ok` [[形参]]，默认值为 `False`，若目标路径已存在目录的话，会抛出 `FileExistsError`；反之，忽略该异常。

### rmdir()

移除该目录，要求是该目录必须为空。

### touch(...)

原型：

```Python
touch(mode=0o666, exist_ok=True)
```

在原对象的路径下创建文件，如果给出 *mode* [[参数]]，则和当前进程的 *umask* 值合并确定模式和权限。

如果文件已存在，且 *exist_ok* 设置为 `True`，函数会直接返回；否则，抛出 `FileExistsError`。

### unlink(missing_ok=False)

如果路径是个文件，或者路径是个 [[符号链接]]，则会直接删除；如果路径对象是个目录，则会调用 [[#rmdir()]]。

- 3.8 新增形参 *missing_ok*，其为真时，方法会忽略 `FileNotFoundError`（和 `rm -f` 命令相同）。

### glob(pattern)

返回一个路径对象 [[列表]]。
解析相对于此路径的 [[通配符]] *pattern*，产生所有匹配的文件。
使用 `**` 会 [[递归]] 地匹配当前目录及所有子目录，有时可能会消耗非常多的时间。

```Python
>>> sorted(Path(".").glob("**/*.py"))
[PosixPath('build/lib/pathlib.py'),
 PosixPath('docs/conf.py'),
 PosixPath('pathlib.py')]
```

该操作会引发一个 [[审计事件]]，附带参数 `self` 和 `pattern`。

### rglob(pattern)

返回一个路径对象列表。
[[#glob(pattern)]] 主动触发子目录递归的版本，相当于在 *pattern* 参数的最开始添加一个 `**/`。

```Python
>>> sorted(Path(".").rglob("*.py"))
[PosixPath('build/lib/pathlib.py'),
 PosixPath('docs/conf.py'),
 PosixPath('pathlib.py')]
```

该操作也会引发一个审计事件，附带参数和 [[#glob(pattern)]] 一样。

### iterdir()

返回 [[迭代器]]，迭代产生路径下的所有路径对象，顺序是随机的，不包括 `.` 和 `..`。

```Python
>>> for child in Path("docs").iterdir(): child
...
PosixPath('docs/conf.py')
PosixPath('docs/index.rst')
```

[[模块]] 没有规定，在迭代器创建之后又有目录被移除或添加的情形，因此可能在动态环境下，结果不会如预期所想地展示。

### symlink_to(...)

原型：

```Python
Path.symlink_to(target, target_is_directory=False)
```

基于原路径对象，创建一个指向 `target` 的符号链接，即原路径对象成为符号链接。

[[Windows]] 下，符号链接指向目录的话，`target_is_directory` 必须为 `True`；[[POSIX]] 忽略该参数。

```ascii
                                             +-------------+
    symlink_to              +···· target ····+ Path Object |
                            :                +------+------+
                            v                       v
  +-------------+     +-----+-----+           +-----+-----+
  | return Path +<····| real_file +<==========+ link_file |
  +-------------+     +-----------+           +-----------+
```

### link_to(target)

创建一个路径为 *target* 的新路径对象，作为指向原路径的 [[硬链接]]，原路径对象不变，仍然指向真实文件。

```ascii
                      +-----------+           +-----------+     +-------------+
                      | real_file +<==========+ link_file +····>+ return Path |
                      +-----+-----+           +-----+-----+     +-------------+
                            ^                       ^
                     +------+------+                :
                     | Path Object +···· target ····+               link_to
                     +-------------+
```

### readlink()

返回一个新路径，代表符号链接指向的真实路径。

```Python
>>> Path("mylink").symlink_to("setup.py").readlink()
PosixPath('setup.py')
```

### resolve(strict=False)

将路径变为绝对路径，其中的符号链接和 `..` 符号也将被正确解析，返回新的路径对象。

如果解析时发生无限循环，则抛出 `RuntimeError`。

- 3.6 新增形参 *strict*，默认值 `False`；3.6 之前相当于 `strict=True`。
当路径不存在且 *strict* 为 `True` 时，抛出 `FileNotFoundError`。

### expanduser()

展开 [[波浪线]] 指令 `~` 的构造，会返回新的路径对象。
使用上和 `os.path.expanduser()` 功能一致。

```Python
>>> PosixPath("~/Documents/Note").expanduser()
PosixPath('/home/snack/Documents/Note')
```

## 读写操作

### open(...)

原型：

```Python
Path.open(mode='r', buffering=1, encoding=None, errors=None, newline=None)
```

打开路径对象所指向的文件，功能和内置的 `open()` 函数所做的是一致的。
不过，此处的 `open()` 是由对象发起的，可以由 [[上下文管理器]] 管理打开和关闭。

```Python
>>> p = Path("setup.py")
>>> with p.open() as f:
...     f.readline()
...
'#!/usr/bin/env python3\n'
```

### write_bytes(data)

将文件以 [[字节]] 模式打开，写入 data 并关闭。

```Python
>>> Path("binary_file").write_bytes(b"Binary contents")
15
```

如果有同名文件，将会直接被覆盖。

### read_bytes()

以字节对象的形式，返回文件内容。

```Python
>>> Path("binary_file").read_bytes()
b'Binary contents'
```

### write_text(...)

原型：

```Python
Path.write_text(data, encoding=None, errors=None)
```

将文件以文本模式打开，写入 data 并关闭。
可选参数的用法与含义和 `open()` 的相同。

```Python
>>> Path("text_file").write_text("Text contents")
13
```

如果有同名文件，将会直接覆盖。

### read_text()

原型：

```Python
Path.read_text(encoding=None, errors=None)
```

以 [[字符串]] 的形式，返回文件解码后的文本内容。

```Python
>>> Path("binary_file").read_text()
'Text contents'
```

## 读取元信息

### stat()

返回一个 `os.stat_result` 对象，其中包含此路径的相关信息。
返回的结果不会存储，每次调用该方法都会重新搜索并创建相关信息。

```Python
>>> Path("setup.py").stat().st_size
956
```

### lstat()

和 [[#stat()]] 一样，但该方法在面对符号链接时，返回的是符号链接的信息，而不是符号链接的目标的信息。

### owner()

返回一个字符串，表示拥有此文件的用户名。
如果不能根据 [[UID]] 找到用户，那么会抛出 `KeyError`。

### group()

返回一个字符串，表示拥有此文件的 [[用户组]]。
如果不能根据 [[GID]] 找到用户组，那么会抛出 `KeyError`。

```Python
>>> Path("/etc/asl.conf").group()
'wheel'
```

## 修改元信息

### rename(target)

将原路径对象的内容 [[重定向]] 到 *target*，并返回一个指向 *target* 的新路径对象。
如果原对象不存在会抛出 `FileNotFoundError`；如果 *target* 指向的文件确实已经存在于文件系统上，那么只有当用户有足够的权限才会静默替换。

参数 *target* 可以是一个字符串，或者是一个路径对象；既可以是 [[相对路径]]，也可以是绝对路径。

注意：*target* 的相对路径是相对于工作目录，而不是相对于原路径的目录。

```Python
>>> p = Path("foo")
>>> q = Path("bar")
>>> p.rename(q)  # p 和 q 两个对象都存在，但系统中只有 bar 文件，不存在 foo 文件
PosixPath('bar')
```

- 3.8 之后，添加了返回值。

### replace(target)

和 [[#rename(target)]] 一样，会重定向到 *target* 并返回一个指向 *target* 的路径对象。
区别是，使用该方法，如果 *target* 指向存在的文件时，会无条件替换。

- 3.8 之后，添加了返回值。

### chmod(mode)

改变文件的模式和权限，与 `os.chmod()` 一样。

```Python
>>> Path("setup.py").chmod(0o444)
```

### lchmod(mode)

如果路径指向符号链接，那么修改的是符号链接的模式和权限，而不是符号链接所指向的真实文件。

## is 判断方法

is 判断方法会根据路径对象的指向，判断是否属于某种类型，然后返回 `True` 或 `False`。
如果路径对象指向的是符号链接的话，那么需要判断符号链接实际的指向再返回。

这类方法会传播它所遇到的，如权限错误 `PermissionError` 等错误。

- 3.8 之后，如果是因为编解码的原因，导致文件路径不可访问，会返回 `False`；早期的版本则会抛出 `ValueError`。

### is_dir()

判断路径是否指向一个目录，或目录的符号链接。

### is_file()

判断路径是否指向一个正常文件，或正常文件的符号链接。

### is_mount()

判断路径是否指向一个 [[挂载点]]，或挂载点的符号链接。

判断时会根据路径对象的父目录判断是否是挂载的。

Windows 未实现该方法。

### is_symlink()

判断路径是否指向一个符号链接。

### is_socket()

判断路径是否指向一个 [[Unix]] [[socket 文件]]，或它的符号链接。

### is_fifo()

判断路径是否指向一个 [[先进先出]] 存储，或它的符号链接。

### is_block_device()

判断路径是否指向一个 [[块设备]]，或块设备的符号链接。

### is_char_device()

判断路径是否指向一个 [[字节设备]]，或字节设备的符号链接。

## 其他方法

### exists() 存在判断

判断路径是否指向一个已存在的文件或目录。
如果是符号链接，则判断的是符号链接指向的文件或目录。

- 3.8 之后，如果是因为编解码的原因，导致文件路径不可访问，会返回 `False`；早期的版本则会抛出 `ValueError`。

### samefile(other) 相同判断

判断此路径和 *other* 是否指向同一个文件或目录。
*other* 可以是字符串或另一个路径对象。
语义类似于 `os.path.samefile()` 和 `os.path.samestat()`。

如果两个都因为同一个原因不可访问，抛出 `OSError`。

```Python
>>> Path("foo').samefile("foo")
True
```

---

## 版本差异

- 3.5 新增方法
    - [[#home()]]
    - [[#write_bytes(data)]]
    - [[#read_bytes()]]
    - [[#write_text(...)]]
    - [[#read_text()]]
    - [[#expanduser()]]
    - [[#samefile(other) 相同判断]]
- 3.7 新增方法
    - [[#is_mount()]]
- 3.8 新增方法
    - [[#link_to(target)]]
- 3.9 新增方法
    - [[#readlink()]]
- 3.5 更改
    - [[#mkdir(...)]]
- 3.6 更改
    - [[#resolve(strict=False)]]
- 3.8 更改
    - [[#unlink(missing_ok=False)]]
    - [[#rename(target)]]
    - [[#replace(target)]]
    - [[#is 判断方法]]
    - [[#exists() 存在判断]]

## 系列笔记

- [[pathlib (1) 面向对象]]
- [[pathlib (2) 纯路径类]]
- [[pathlib (3) 路径类]] **<**
- [[pathlib (4) 源码阅读 PurePath]]