---
type: Toolkit
sub-type: Python
---
```embed
title: "GitHub - pyenv/pyenv: Simple Python version management"
image: "https://opengraph.githubassets.com/134f354440c777117535bb4e65c170a3b6020bca6c6f6253f865cbb8a9ec7273/pyenv/pyenv"
description: "Simple Python version management. Contribute to pyenv/pyenv development by creating an account on GitHub."
url: "https://github.com/pyenv/pyenv"
favicon: ""
aspectRatio: "50"
```

## 1	安装 Pyenv

### 1.1	macOS

```bash
brew install pyenv
```

### 1.2	Windows

Pyenv does not officially support Windows and does not work in Windows outside the Windows Subsystem for Linux. Moreover, even there, the Pythons it installs are not native Windows versions but rather Linux versions running in a virtual machine -- so you won't get Windows-specific functionality.

If you're in Windows, we recommend using @kirankotari's [pyenv-win](https://github.com/pyenv-win/pyenv-win) fork -- which does install native Windows Python versions.


## 2	核心使用命令

这是你日常使用 Pyenv 最重要的四个命令：

| 动作 | 命令 | 作用 |
| :--- | :--- | :--- |
| 查看可安装版本 | `pyenv install --list` | 列出所有可以安装的 Python 版本（如 3.10.12, 3.11.5）.  |
| 安装指定版本 | `pyenv install 3.11.5` | 下载并编译安装 Python 3.11.5 版本.  |
| 查看已安装版本 | `pyenv versions` | 列出你本地通过 Pyenv 安装的所有版本, 当前激活的版本会有星号 `*` 标记.  |
| 全局切换版本 | `pyenv global 3.11.5` | 设置默认的 Python 版本.  |

## 3	项目级版本管理

在日常开发中, 我们通常为每个项目设置独立的 Python 版本, 以确保环境隔离. 
 
```bash
# 进入项目目录
cd /path/to/your/project

# 设置项目版本
pyenv local 3.10.12

# 验证当前版本
python --version
```

设置项目版本会在当前目录下创建一个 `.python-version` 文件, 当你进入这个目录后, pyenv 会帮你自动切换版本.