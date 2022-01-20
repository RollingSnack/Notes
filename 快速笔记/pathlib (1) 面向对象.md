---
Title: pathlib (1) 面向对象
Tags: [类型/笔记, 类型/系列, 编程语言/Python]
---

# pathlib (1) 面向对象

![[Concepts/pathlib]]

## 面向对象

基于 [[面向对象]] 的思维，pathlib 把路径视作一个 [[对象]]，将 [[os]]、[[sys]] 等 [[模块]] 的操作封装进到了对象的方法中。
编写应用层代码时，合适地使用 pathlib，或许能获得更高的抽象层次和更佳的可读性。

模块向外暴露了 6 种可供使用的 [[类]]：`PurePath`、`PurePosixPath`、`PureWindowsPath`、`Path`、`PosixPath`、`WindowsPath`。
它们 [[继承]] 关系如图所示。

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

上面的 3 种是纯路径，只提供路径计算，而不提供 [[I/O]] 操作；下面的 3 种分别继承前三种，既提供路径计算，也提供 I/O 操作。

## os.PathLike

路径对象可以用于任何可以接受 `os.PathLike` 接口的地方。

```Python
>>> os.fspath(PurePath("/etc"))
'/etc'
```

fspath 相关概念的接口是 3.6 以上版本的新增功能，因此建议 pathlib 也尽可能工作在 3.6 以上的版本中。

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
