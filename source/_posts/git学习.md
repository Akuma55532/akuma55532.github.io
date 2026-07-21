---
title: git学习
date: 2026-07-21 13:25:03
tags:
---

# Git 学习笔记

之前只是把git当成版本管理工具，github当成一个开源代码存储仓库，没有在多人协作场景使用过分支合并等操作。

## Git 是什么

Git 是分布式版本控制工具，用来记录文件变化、创建分支、协作开发和回退历史。

几个常见概念：

- **工作区（working tree）**：电脑上正在编辑的文件。
- **暂存区（staging area / index）**：准备放进下一次提交的文件集合。
- **本地仓库**：本机 `.git` 中的提交历史。
- **远程仓库**：GitHub、GitLab 等服务器上的仓库。
- **提交（commit）**：某一时刻项目状态的记录。
- **分支（branch）**：一条独立的提交线。

## Git 常用命令

### 初始化与查看状态

```bash
git init                    # 在当前目录初始化仓库
git clone <仓库地址>          # 克隆远程仓库
git status                  # 查看工作区、暂存区状态
git log --oneline --graph --all  # 图形化查看提交历史
git diff                    # 查看未暂存的修改
git diff --staged           # 查看已暂存、尚未提交的修改
```

### 提交代码

```bash
git add <文件名>              # 将指定文件加入暂存区
git add .                    # 暂存当前目录下的全部修改
git commit -m "说明本次修改"   # 创建提交
```

提交前可用 `git status` 确认哪些文件会进入提交。提交信息应描述“做了什么”，例如 `feat: add 功能 validation`。

### 查看与撤销

```bash
git restore <文件名>          # 丢弃工作区中该文件未暂存的修改
git restore --staged <文件名> # 从暂存区撤回，但保留工作区修改
git revert <提交ID>           # 用一个新提交撤销历史提交，适合共享分支
```

`git restore` 可能丢失未提交内容，执行前先用 `git diff` 确认。不要在共享分支上随意使用会改写历史的 `reset`。

## 远程仓库

### 添加与查看远程地址

```bash
git remote add origin git@github.com:用户名/仓库名.git
git remote -v
```

`origin` 只是远程仓库的本地别名，通常默认使用这个名字。SSH 地址要求本机 SSH 公钥已添加到 GitHub/GitLab。

### 推送、获取与拉取

```bash
git push -u origin main      # 首次推送 main，并建立上游跟踪关系
git push                     # 后续推送当前分支
git fetch origin             # 下载远端最新引用，不修改当前工作区
git pull --ff-only origin main  # 获取并快进更新本地 main
```

上游跟踪关系表示本地分支默认对应的远端分支。例如 `git push -u origin main` 后，本地 `main` 跟踪 `origin/main`，后续可直接使用 `git push`、`git pull`。用下面命令查看：

```bash
git branch -vv
```

注意：`git fetch origin` **只更新** `origin/main`、`origin/feature` 等远端跟踪分支，不会自动修改本地 `main` 或 `feature`。

## Git 分支

### 基本操作

```bash
git branch                   # 查看本地分支
git branch -a                # 查看本地和远端分支
git switch main              # 切换到 main
git switch -c feature        # 创建并切换到功能分支
git branch -d feature        # 删除已合并的本地分支
git branch -M main           # 强制将当前分支重命名为 main
```

`git branch -M main` 只重命名本地当前分支；它不会自动修改远端默认分支。`-M` 会在目标分支名已存在时强制覆盖，应谨慎使用。

### 主分支与功能分支

团队协作中，通常让本地 `main`（有些项目叫 `master`）**只负责跟踪远端主分支**，尽量不直接开发。开发前从最新主分支创建功能分支：

```bash
git switch main
git pull --ff-only origin main
git switch -c feature/功能
```

所有功能修改和提交都放在 `feature\功能`，而不是直接修改 `main`。

## 推荐协作流程：feature 合并到 main

![](https://cloudflare-imgbed.wx3515753265.workers.dev/file/blog/git学习/1784611364488_git-recommended-collaboration-flow.drawio.png)

具体命令：

```bash
# 1. 开发前同步主分支
git switch main
git fetch origin
git merge --ff-only origin/main

# 2. 创建或进入功能分支，完成开发
git switch -c feature/功能
git add .
git commit -m "feat: add 功能"

# 3. 合并前，再次同步本地 main 到远端最新版本
git fetch origin
git switch main
git merge --ff-only origin/main

# 4. 将已同步的 main 合入功能分支
git switch feature/功能
git merge main

# 5. 解决冲突、运行测试后推送功能分支
git push -u origin feature/功能
```

然后在 GitHub/GitLab 创建 PR/MR：

```text
base（目标分支）：main
compare（来源分支）：feature/功能
```

审查和 CI 通过后，在平台合并 PR。最后同步本地主分支：

```bash
git switch main
git pull --ff-only origin main
```

不要先在本地把 `feature` 合进 `main`，再创建 PR；规范流程中，PR 负责将远端 `feature` 合并到远端 `main`。

## 本地 main 有修改时如何同步

### 未提交的修改

先提交或暂存，再同步：

```bash
git switch main
git stash push -m "WIP: local main changes"
git fetch origin
git merge --ff-only origin/main
git stash pop
```

### 已提交但未推送的修改

此时本地 `main` 与 `origin/main` 已分叉，`--ff-only` 会拒绝更新。先检查两边各自的提交：

```bash
git log --oneline origin/main..main
git log --oneline main..origin/main
```

若本地提交确实应进入主分支，优先通过 PR 处理；不要随意在共享 `main` 上 rebase。若团队明确允许直接合并，可执行 `git merge origin/main`，解决冲突并测试后再推送。

## 合并与冲突

Git 冲突不是“只要修改、覆盖或删除就一定发生”，而是两个分支对同一份基础内容做了 Git 无法自动协调的不兼容修改。

常见冲突包括：

- 两边修改了同一文件的同一段内容，且结果不同。
- 一边修改文件，另一边删除该文件。
- 两边新增了同一路径、但内容不同的文件。
- 一边重命名/移动文件，另一边修改或删除原文件。
- 两边修改同一个二进制文件（图片、Word、Excel、PDF 等）。

通常**不会**冲突的情况：

- 一个分支修改，另一个分支没有修改对应内容。
- 两边修改同一文件但位置相距较远。
- 两边都删除同一行或同一文件。

发生文本冲突时，文件会出现标记：

```text
<<<<<<< HEAD
当前分支的内容
=======
正在合入分支的内容
>>>>>>> feature/功能
```

按业务需要编辑成最终内容并删除标记，然后：

```bash
git add <已解决的文件>
git commit                    # merge 时完成合并提交
# 或 git rebase --continue     # rebase 时继续
```

放弃未完成的操作：

```bash
git merge --abort
git rebase --abort
```

即使 Git 自动合并成功，也应运行测试：不同代码行的修改仍可能在业务逻辑或接口层面不兼容。

## 合并策略

- **Squash merge**：将功能分支的多个提交压成主分支上的一个提交，适合保持主分支历史简洁。
- **Merge commit**：保留分支完整历史，主分支多一个合并提交。
- **Rebase and merge**：保持线性历史，但会重写功能分支提交的哈希；不适合已被多人共同使用的分支。

团队应统一合并策略，以仓库规则为准。
