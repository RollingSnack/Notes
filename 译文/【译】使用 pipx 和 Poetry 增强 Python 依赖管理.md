---
Title: 【译】使用 pipx 和 Poetry 增强 Python 依赖管理
Tags: [Translation, Python, Development]
Original Title: Improving Python Dependency Management With pipx and Poetry
Original Link: https://cedaei.com/posts/python-poetry-pipx/
Original Author: Ceda Ei
---

# 【译】使用 pipx 和 Poetry 增强 Python 依赖管理

---

随着时间的推移，我用 Python 开发应用的方式发生了显著的变化。
我将该话题分为三个章节并看看他们最终如何紧密结合在一起。

- 开发
- 打包
- 使用

## 开发

在开发中，我关注的问题是：

- 依赖管理
- 虚拟环境（virtualenvs）及其管理

过去是通过 `requirements.txt` 管理依赖。
我发现用 `requirements.txt` 管理得很费力。
在这个体系下，添加依赖并安装它是两个步骤：

- 将 `bar` 包添加到 `requirements.txt` 中
- 执行 `pip install bar` 或 `pip install -r requirements.txt`

不过在投入于开发时，我时常会遗忘这一两步。
此外，缺少锁文件（lock file）对我来说也是个小缺点（可能在他人那会是个大问题）。
`pip` 和 `requirements.txt` 之间的割裂很容易导致你意外地依赖一些包，它们被安装在系统或虚拟环境中，但没有被指定在 `requirements.txt` 中。

管理虚拟环境也很困难。
因为一个虚拟环境和一个项目是无关的，为此你需要一个目录结构；否则，你不能分辨哪个虚拟环境被用于哪个项目。
你可以为多个项目使用同一个虚拟环境，但这一定程度上违背了虚拟环境的目的，并使得 `requirements.txt` 更容易出错（忘记向其中添加包的可能性更高了）。
通常的用法是以下两种之一：

```ascii
foo/
├── foo_src/
└── foo_venv/
```

或

```ascii
foo_src/
└── venv/
```

我更喜欢第二种，因为第一种让源代码目录嵌套得更深了一层。

### 一个新标准 —— `pyproject.toml`

在 [PEP-518](https://www.python.org/dev/peps/pep-0518/) 中，Python 标准化了 `pyproject.toml`，它允许用户去选择备用的构建系统来生成包。

一个提供类似备用构建系统的项目是 [Poetry](https://python-poetry.org/)。
Poetry 一针见血地解决了我对传统工具的主要不满。

### Poetry 和虚拟环境

Poetry 自动管理虚拟环境并自动追踪哪个项目该使用哪个虚拟环境。
在一个使用 poetry 的现存项目上干活，就像这样简单：

```shell
$ git clone https://gitlab.com/ceda_ei/verlauf
$ poetry install
```

`poetry install` 命令会设置好虚拟环境，安装所有需要的依赖，并相应地备好命令（我很快会说到这部分）。
激活虚拟环境，只需执行：

```shell
. "$(poetry env info --path)/bin/activate"
```

我把它包装在一个小函数中，让我得以快速切换：

```shell
function poet() {
    POET_MANUAL=1
    if [[ -v VIRTUAL_ENV ]]; then
        deactivate
    else
        . "$(poetry env info --path)/bin/activate"
    fi
}
```

运行 `poet` 可以在虚拟环境未活动时激活它，也可以在虚拟环境活动时停用它。
为了让事情更简单些，我希望进入或离开项目目录时，虚拟环境可以被自动激活或停用。
为此，只需要把下面内容放入你的 `.bashrc` 即可：

```shell
function find_in_parent() {
    local path
    IFS="/" read -ra path <<<"$PWD"
    for ((i=${#path[@]}; i > 0; i--)); do
        local current_path=""
        for ((j=1; j<i; j++)); do
            current_path="$current_path/${path[j]}"
        done
        if [[ -e "${current_path}/$1" ]]; then
            echo "${current_path}/"
            return
        fi
    done
    return 1
}

function auto_poet() {
    ret="$?"
    if [[ -v POET_MANUAL ]]; then
        return $ret
    fi
    if find_in_parent pyproject.toml &> /dev/null; then
        if [[ ! -v VIRTUAL_ENV ]]; then
            if BASE="$(poetry env info --path)"; then
                . "$BASE/bin/activate"
                PS1=""
            else
                POET_MANUAL=1
            fi
        fi
    elif [[ -v VIRTUAL_ENV ]]; then
        deactivate
    fi
    return $ret
}

PROMPT_COMMAND="auto_poet;$PROMPT_COMMAND"
```

这个和 `poet` 函数紧密相连；你在 bash 会话中的任意时刻使用 `poet`，激活功能会从自动变为手动，而且变更目录时也不再自动切换虚拟环境。

![[【译】使用 pipx 和 Poetry 增强 Python 依赖管理【图1】.png]]

### Poetry 和依赖管理

作为 `requirements.txt` 的替代，poetry 在 `pyproject.toml` 中记录依赖。
poetry 在处理版本问题上，比 `pip` 要更严格。
依赖和开发依赖分别记录在 `tool.poetry.dependencies` 和 `tool.poetry.dev-dependencies` 中。
这是一个我正在做的项目的 `pyproject.toml` 示例。

```toml
[tool.poetry]
name = "bells"
version = "0.3.0"
description = "Bells is a program for keeping track of sound recordings."
authors = ["Ceda EI <ceda_ei@webionite.com>"]
license = "GPL-3.0"
readme = "README.md"
homepage = "https://gitlab.com/ceda_ei/bells.git"
repository = "https://gitlab.com/ceda_ei/bells.git"

[tool.poetry.dependencies]
python = ">=3.7,<3.11"
click = "^8.0.1"
questionary = "^1.10.0"
sounddevice = "^0.4.2"
SoundFile = "^0.10.3"
numpy = "^1.21.2"

[tool.poetry.dev-dependencies]

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"

# I will talk about this section soon
[tool.poetry.scripts]
bells = "bells.__main__:main"
```

Poetry 的一个优点在于你不需要亲自在 `pyproject.toml` 中管理依赖。
Poetry 增加了一个类似 `npm` 的接口用于添加和移除依赖项。
为了向你的项目中加一个依赖，简单地执行 `poetry add bar` 后它就会往你的 `pyproject.toml` 文件中添加内容，并在虚拟环境中安装好依赖。
为了移除一个依赖，只消执行 `poetry remove bar`。
对于开发时的依赖，仅需在命令中加上 `--dev` 标志。

## 打包

由于 poetry 取代了构建系统，我们现在可以用 poetry 搭配 `pyproject.toml` 配置构建。
在 `pyproject.toml` 中，`tool.poetry` 部分存储了所有构建所需信息；`tool.poetry` 包含元数据，`tool.poetry.dependencies` 包含了依赖，`tool.poetry.source` 包含了私有仓库的细节（如果你不想用 PyPi 的话）。

一个可选项是 `tool.poetry.scripts`。
它包含了项目公开的脚本。
这取代了 `setuptools` 中 `entry_points` 的 `console_scripts`。

举例来说，

```toml
[tool.poetry.scripts]
foobar = "foo.bar:main"
```

这会添加一个名为 `foobar` 的脚本到你的 `PATH` 环境下。
执行它等价于执行以下的脚本：

```Python
from foo.bar import main

if __name__ == "__main__":
    main()
```

有关更多的详细信息，请查阅 [参考资料](https://python-poetry.org/docs/pyproject/)。

Poetry 还移除了手动选择可编辑安装的需求（pip install -e）。
当你执行 `poetry install` 时，安装包会自动以可编辑模式安装。
当激活了 `venv` 时，会自动令 `tool.poetry.scripts` 中的脚本可工作于 `PATH` 中。

> 这也让应用的开发和生产版本之间切换良好。
> 本质上，当虚拟环境激活时，你使用的是开发脚本；当虚拟环境停用后，你使用全局（如生产环境）的版本。

为了构建包，简单地运行 `poetry build` 即可。
这个会在 dist 文件夹中生成一个 wheel 和 tarball。

若要向 PyPi（或其他仓库）发布这个包，只需运行 `poetry publish`。
你可以整合构建与发布到一个命令：`poetry publish --build`。

![[【译】使用 pipx 和 Poetry 增强 Python 依赖管理【图2】.png]]

## 使用

这部分更面向用户而不是开发者。
如果你想在全局范围内向用户公开一些脚本，（例如 `awscli`、`youtube-dl` 等等）通常的方法是这样的：执行类似 `pip install --user youtube-dl` 的命令。
这会在用户级别安装软件包，并通过 `~/.local/bin/youtube-dl` 暴露脚本。
然而，这会在同一用户级别安装所有的软件包。
假设你有 `foo` 和 `bar` 两个包，它们有着冲突的依赖关系，这会导致一个问题。
如果你执行，

```Python
$ pip install foo
$ pip install bar
$ bar # 可行
$ foo # 由于依赖不匹配而中断
```

安装 `bar` 时，`pip` 会警告你这将会破坏 `foo`，之后安装上 `bar` 的依赖。

> 准确的说，它会警告你这已经破坏了 `foo`，但它仍然会继续安装。

为了解决这一问题，有了 [`pipx`](https://github.com/pypa/pipx)。
Pipx 将每个包安装在一个单独的虚拟环境中，而不要求用户在使用前激活虚拟环境。

> 为了开发，poetry 还提供了 `poetry run`，它无需激活虚拟环境就可以运行某一文件。

和之前相同的例子，如下执行却能正常工作了。

```Python
$ pipx install foo
$ pipx install bar
$ bar # 可行
$ foo # 同样可行
```

在这一情况下，`bar` 和 `foo` 都被安装在了单独的虚拟环境中，所以它们冲突的依赖并不影响。

## 我在 bashrc 中额外的东西

```shell
function wrapper_no_poet() {
    local last_env
    if [[ -v VIRTUAL_ENV ]]; then
        last_env="$VIRTUAL_ENV"
        deactivate
    fi
    "$@"
    ret=$?
    if [[ -v last_env ]]; then
        . "$last_env/bin/activate"
    fi
    return $ret
}

alias wnp='wrapper_no_poet'
alias pm='POET_MANUAL=1'
```

就算虚拟环境处于激活状态，加上 `wnp` 前缀的任何命令也可以在虚拟环境外运行。
运行 `pm` 会关闭自动激活虚拟环境的功能。