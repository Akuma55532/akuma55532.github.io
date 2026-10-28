---
title: python环境管理
tags: python
date: 2025-09-08 03:28:49
---


## 前言

python的环境管理一直是我比较注重的点，刚开始学习python的时候被他的依赖折磨麻了，所有package都放在一个环境下，整个一个依赖地狱，这篇文章就来整理一下，使用过的环境管理工具，从一开始的pip到conda再到现在的uv。

## pip

作为官网的python包管理工具，pip自带一个环境管理工具venv虚拟环境，

创建虚拟环境

~~~shell
python -m venv .venv
~~~

激活虚拟环境

~~~shell
source ./venv/bin/activate
~~~

## conda

在计算机科学计算领域中广受青睐，一般安装大量python中科学计算库，比如numpy，scipy等等。但是另一个组织自行支持的，就连python解释器也是自行编译的。

conda命令使用有一个csdn文章推荐：[conda基础命令](https://blog.csdn.net/weixin_52397563/article/details/126684139?spm=1001.2014.3001.5506)

## uv

是我现在比较喜欢的一个python环境管理和包管理工具，本身使用rust语言编写，下载package速度也快很多。

uv的一些常见命令

~~~
uv python list — 列出uv支持的python版本
uv python install cpython3.12 — 安装某个python版本 (3.12)
uv run -p 3.12 xxx.py — 使用特定版本python运行xxx.py
uv run -p 3.12 python — 运行python交互界面
uv run xxx.py — 使用系统python或当前工程的虚拟环境运行xxx.py
uv init — 创建工程
uv add pydantic_ai — 添加依赖 (pydantic_ai)
uv tree — 打印依赖树
uv remove pydantic_ai — 删除依赖
uv build — 编译工程
~~~


## pyproject.toml

元数据配置

~~~toml
[project]
name = "your-package-name"  # 包名，只能包含字母、数字、_和-
version = "0.1.0"           # 版本号，推荐遵循语义化版本规范
description = "..."         # 简短描述
readme = "README.md"        # 项目README文件
requires-python = ">=3.8"   # Python版本要求
license = {text = "MIT"}    # 许可证
keywords = ["key", "word"]  # 关键词列表
~~~

依赖管理

~~~
[project]
dependencies = [
    "requests>=2.25.0",
    "numpy>=1.20.0",
]

[project.optional-dependencies]
test = [
    "pytest>=6.0.0",
    "pytest-cov>=2.0.0",
]
dev = [
    "black>=21.0",
    "flake8>=3.9.0",
]
~~~

入口点和脚本

~~~
[project.scripts]
my-cli-command = "mypackage.module:main_func"

[project.gui-scripts]
my-gui-app = "mypackage.gui:main"
~~~