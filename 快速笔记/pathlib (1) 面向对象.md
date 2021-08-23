---
Title: pathlib (1) 面向对象
Tags: [Note, Series, Python3]
---

# pathlib (1) 面向对象

![[Concepts/pathlib]]

## 面向对象

早期 [[python]] 的 [[路径]] 和 [[字符串]] 毫无差异，只是因为这个字符串恰好能代表一个路径，而被诸如 [[os]]、[[glob]] 等模块调用。

随着 pathlib 的加入，基于 [[面向对象]] 的思维，路径成为了与字符串完全不同的一个 [[对象]]。
使用 pathlib 编写的应用层代码，会具有更高的抽象层次和更好的可读性。
开发出来的应用层代码，也可以轻松地迁移到不同的 [[文件系统]] 上；甚至可以基于此，二次开发出一个自己定义的资源管理系统。

```ascii
                      +----------+
          +---------> + PurePath + <----------+
          |           +-----+----+            |
  +-------+-------+         ^        +--------+--------+
  | PurePosixPath |         |        | PureWindowsPath |
  +-------+-------+         |        +--------+--------+
          ^           +-----+----+            ^
          |   +-----> +   Path   + <------+   |
          | _/        +----------+         \_ |
          |/                                 \|
  +-------+-------+                  +--------+--------+
  |   PosixPath   |                  |   WindowsPath   |
  +---------------+                  +-----------------+
```

[[模块]] 向外暴露了 6 种可供使用的 [[类]]：`PurePath`、`PurePosixPath`、`PureWindowsPath`、`Path`、`PosixPath`、`WindowsPath`。
它们 [[继承]] 关系如图所示。

前 3 种是纯路径，只提供路径计算，而不提供 [[I/O]] 操作；后 3 种分别继承前三种，既提供路径计算，也提供 I/O 操作。

## os.PathLike

路径对象可以用于任何可以接受 `os.PathLike` 接口的地方。

```Python
>>> os.fspath(PurePath("/etc"))
'/etc'
```

该接口是 3.6 以上版本的新增功能，因此建议 pathlib 也尽可能工作在 3.6 以上的版本中。

## 基本用法

导入主类：

```Python
>>> from pathlib import Path
```

列出子目录：

```Python
>>> [x for x in Path('.').iterdir() if x.is_dir()]
[PosixPath('docs'), PosixPath('dist'), PosixPath('res')]
```

列出当前目录树下的所有 Python 文件：

```Python
>>> list(Path('.').glob('**/*.py'))
[PosixPath('setup.py'), PosixPath('docs/conf.py'), PosixPath('build/lib/pathlib.py')]
```

在目录树中移动：

```Python
>>> p = Path('/etc')
>>> q = p / 'init.d' / 'reboot'
>>> q
PosixPath('/etc/init.d/reboot')
```

路径查询：

```Python
>>> q.exists()
True

>>> q.is_dir()
False
```

打开一个文件：

```Python
>>> with q.open() as f: f.readline()
...
'#!/bin/bash\n'
```

## 系列笔记

- [[pathlib (1) 面向对象]] **<**
- [[pathlib (2) 纯路径类]]
- [[pathlib (3) 路径类]]
- [[pathlib (4) 源码阅读 PurePath]]