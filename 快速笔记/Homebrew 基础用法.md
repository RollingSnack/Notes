---
Title: Homebrew 基础用法
Tags: [Note, packageManager]
---

# Homebrew 基础用法

![[Homebrew]]

## 开始

### 环境要求

1. 64 位 Intel CPU 或者是 Apple Silicon CPU
2. macOS Mojave (10.14) 以上版本
    - 需要参考 [[Tigerbrew]]
3. 提供编译功能的命令行工具
    ```shell
    xcode-select install
    ```
4. [[Bourne shell]]（如：[[bash]] / [[zsh]]）
5. [[Ruby]] 以及 [[Git]]

### 安装

从远端获取安装用的 [[shell]] 脚本 `install.sh` 到本地，然后执行该脚本安装。

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

使用 `brew` 命令查看 Homebrew 版本，以确认正确安装成功。

```shell
brew --version
# OR
brew -v
```

## 安装与卸载

```shell
brew install <package>  # 安装 <package>
brew uninstall <package>  # 卸载 <package>
```

如果遗忘了安装包的名字，可以尝试使用 `search` 命令进行关键字搜索；然后使用 `info` 命令查看信息，确认是否是自己需要的软件。

```shell
brew search <keyword>  # 搜索 <keyword>
brew info <package>  # 查看软件包信息
```

### Cask 和 Formuale

Homebrew 既可以安装命令行工具，也可以对 [[macOS]] 原生应用的进行管理；后者通常来说，就是带有 [[GUI]] 的界面。

可以使用 `--formula` 选项和 `--cask` 分别操作；前者用于命令行软件，后者用于 GUI 软件。

```shell
brew install --formula <package>  # 安装 formulae 类型的 <package>
brew info --cask <package>  # 查看 cask 类型的 <package>
```

Formuale 软件的默认安装目录在 `/usr/local/Cellar` 文件夹下，而 Casks 软件的默认安装目录在 `/usr/local/Caskroom` 文件夹下；两个目录避免了权限问题，因此不需要 `sudo` 也能执行安装。

## 更新

### 查找新版本

`update` 命令既用于查找 Homebrew 自身的最新版本，也用于从 [[GitHub]] 处获取所有 **Formulae** 的最新版本。

```shell
brew update
```

### 安装新版本

`upgrade` 命令后面可以不接任何的参数。与 `update` 不同的是，该命令会对包括 Cask 和 Formulae 在内的软件全局更新。

```shell
brew upgrade  # 更新所有软件
```

也可以附加选项对 Cask 和 Formulae，或者任何一个指定了名字的软件更新。

```shell
brew upgrade --cask  # 更新所有的 cask 类型的软件
brew upgrade --formulae  # 更新所有的 formulae 类型的软件
brew upgrade <package>  # 更新指定的 <package>
```

### 忽视 Casks 的内部更新

部分 GUI 软件会在软件内部设置自动更新，或者保持版本的自我检查。这样的软件在 Homebrew 中会被标记为 `auto_update true` 或者保持 `version :latest` 状态，从而脱离 Homebrew 的版本管理，不会随着 `upgrade` 命令而一并更新。

如果要求 Homebrew 对这类软件进行版本管理的话，需要在更新时加上 `--greedy` 参数。该参数会把包含有上述标记的 Cask 软件一并纳入。

```shell
brew upgrade --greedy
```

## 帮助

任何时候都可以使用 `--help|-h` 参数查看文档。

```shell
brew --help  # 查看 Homebrew 的帮助
brew install --help  # 查看 Homebrew install 子命令的帮助
```

`commands` 子命令还可以帮助查看 Homebrew 的所有子命令，包括没在 `--help` 中显示的。

```shell
brew commands
```

永远不要忘记在需要帮助时，查看官方的帮助手册。

```shell
man brew
```

[官方文档](https://docs.brew.sh/) 和 [论坛](https://github.com/Homebrew/discussions/discussions) 有时也能提供必要的帮助。