---
Title: pathlib (3) 路径类
Tags: [Note, Series, Python3]
---

# pathlib (3) 路径类

![[pathlib]]

## 继承自纯路径类

三种路径类分别继承自三种 [[pathlib (2) 纯路径类 | 纯路径类]]。
除了 [[父类]] 提供的 纯路径计算以外，路径类还封装了系统调用的 I/O 操作。

和纯路径一样，`Path` 会自动根据当前的系统路径风格，转化成 `PosixPath` 或 `WindowsPath`。
但是手动实例化的话，只能使用与当前系统风格相同的类。

## 类方法

### cwd()

返回一个表示 [[当前目录]] 的 [[绝对路径]] 的新对象。

```Python
>>> Path.cwd()
PosixPath('/home/anyone/pathlib')
```

### home()

返回一个表示当前 [[用户]] 家目录的绝对路径的新对象。

## 读取文件信息

### stat()

返回一个 `os.stat_result` 对象，其中包含此路径的相关信息。
结果不会存储，每次都会重新搜索。

```Python
>>> Path("setup.py").stat().st_size
956
```

### lstat()

和 [[#stat()]] 一样，但该方法在面对 [[软链接]] 时，返回的是软链接的信息，而不是软链接的目标的信息。

### owner()

返回一个字符串，表示拥有此文件的 [[用户名]]。
如果不能根据 [[UID]] 找到用户，那么会抛出 `KeyError`。

### group()

返回一个字符串，表示拥有此文件的 [[用户组]]。
如果不能根据 [[GID]] 找到用户组，那么会抛出 `KeyError`。

```Python
>>> Path("/etc/asl.conf").group()
'wheel'
```

## 修改路径对象信息

### rename(target)

### replace(target)

### chmod(mode)

改变文件的模式和权限，和 `os.chmod()` 一样。

### lchmod(mode)

如果路径指向软链接，那么修改的是软链接的模式和权限，而不是软链接的目标。

## 读写操作

### open(...)

原型：

```Python
Path.open(mode='r', buffering=1, encoding=None, errors=None, newline=None)
```

打开路径对象所指向的文件，功能和内置的 `open()` 函数所做的是一致的。
不过，`pathlib` 的 `open()` 是由对象发起的，可以由 [[上下文管理器]] 管理打开和关闭。

```Python
>>> p = Path("setup.py")
>>> with p.open() as f:
...     f.readline()
...
'#!/usr/bin/env python3\n'
```

### write_bytes(data)

将文件以二进制模式打开，写入 data 并关闭。

```Python
>>> Path("binary_file").write_bytes(b"Binary contents")
15
```

### read_bytes()

以字节的形式，返回文件的二进制内容。

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

### read_text()

原型：

```Python
Path.read_text(encoding=None, errors=None)
```

以字符串的形式，返回文件解码后的文本内容。

```Python
>>> Path("binary_file").read_text()
'Text contents'
```

## 目录操作

### expanduser()

展开波浪线指令 `~` 的构造，会返回新的路径对象。
使用上和 `os.path.expanduser()` 功能一致。

```Python
>>> PosixPath("~/Documents/Note").expanduser()
PosixPath('/home/snack/Documents/Note')
```

### mkdir(...)

原型：

```Python
Path.mkdir(mode=0o777, parents=False, exist_ok=False)
```

根据给定的路径，新建一个目录；如果给出了 `mode` 需要和 [[进程]] 的 `umask` 值合并来决定。

`parents` 设置为 `True` 后，从当前路径到目标路径中不存在的目录都将被创建；权限是默认的权限，而不跟随 `mode`；相当于 `mkdir -p` 命令。
否则，当父路径不存在时，抛出 `FileNotFoundError`。

- `exist_ok` 形参是在 3.5 版本才加入的，默认值为 `False`，若目标路径已存在目录的话，会抛出 `FileExistsError`；反之，忽略该异常。

### rmdir()

## 软链接操作

### readlink()

### resolve(strict=False)

## is 判断方法

is 判断方法会根据路径对象的指向，判断是否属于某种类型，然后返回 `True` 或 `False`。
如果路径对象指向的是软链接的话，那么需要判断软链接实际代表的目标再返回。

这类方法会传播它所遇到的，如权限错误 `PermissionError` 等错误。

- 如果是因为编解码的原因，导致文件路径不可访问，在 3.8 及之后的版本返回 `False`，早期的版本则会抛出 `ValueError`。

### is_dir()

判断路径是否指向一个 [[目录]]，或目录的软链接。

### is_file()

判断路径是否指向一个 [[正常文件]]，或正常文件的软链接。

### is_mount()

判断路径是否指向一个 [[挂载点]]，或挂载点的软链接。

判断时会根据路径对象的父目录判断是否是挂载的。

[[Windows]] 未实现该方法。

### is_symlink()

判断路径是否指向一个软链接。

### is_socket()

判断路径是否指向一个 [[Unix]] [[socket 文件]]，或它的软链接。

### is_fifo()

判断路径是否指向一个 [[先进先出]] 存储，或它的软链接。

### is_block_device()

判断路径是否指向一个 [[块设备]]，或块设备的软链接。

### is_char_device()

判断路径是否指向一个 [[字节设备]]，或字节设备的软链接

## 更新内容

- 3.5 新版功能
    - [[#home()]]
    - [[#expanduser()]]
    - [[#write_bytes(data)]]
    - [[#read_bytes]]
    - [[#write_text(data)]]
    - [[#read_text]]

- 3.7 新版功能
    - [[#is_mount()]]

- 3.5 跟新内容
    - [[#mkdir(...)]]

- 3.8 更新内容
    - [[#is 判断方法]]
    - [[#exists()]]

## 系列

- [[pathlib (1) 面向对象]]
- [[pathlib (2) 纯路径类]]
- [[pathlib (3) 路径类]] **<**
- [[pathlib (4) 源码阅读——功能]]