---
typora-root-url: ../../../static
title: "Git的使用"
date: 2020-03-15T20:07:36+08:00
draft: false
categories: ["Other"]
tags: ["Git"]
---

## 初步使用
在windows下直接在官网下载安装程序，然后安装即可（记得勾选安装git bash）使用git。安装完毕后需要设置本机的git的用户名和邮箱，命令如下：

	git config --global user.name "Your name"
	git config --global user.email "email@example.com"

设置完毕后，即可创建本地git仓库，进入到自己的仓库目录，然后输入一下命令完成仓库初始化：

	git init

初始化完毕后，本地仓库是一个空的仓库，需要把文件添加到版本库，添加文件到本地仓库需要两步，命令如下：

	git add YourFile
	git commit -m "Your Tips"

***Notice:*** `git add` 之前可以先 `git status` 查看一下仓库的状态。输入 `git diff` 可以查看修改的文件内容。

## 版本管理
### 版本回退
1. 使用 `git log` 或者 `git log --pretty=oneline` 命令查看每次提交（commit）的日志。
2. 使用命令 `git reset --hard HEAD^` 来回退到前一个版本，如果是 `HEAD^^` 就是前前一个版本，依次类推。
3. 如果回退到以前的版本了，想再跳回到现在的版本，可以先输入 `git reflog` 查看日志（此时通过 `git log` 是查看不到现在的版本的，因为你回退到之前了，只能看到回退后到更早之前的）。
4. 然后输入 `git reset --hard xxxx` ，其中的 `xxxx` 是你需要跳到的版本的commit_id，不用输全，前几位就行了。

### 工作区和暂存区
- 使用 `git add FILE` 将工作区的文件添加到stage暂存区。
- 使用 `git commit -m "tips"` 将暂存区的所有内容提交到当前分支。

示意图如下：

![工作区和暂存区][p0]

### 其他
- 使用 `git diff HEAD FILENAME` 可以查看工作区和版本库里面最新版本的区别。
- 使用 `git checkout -- FILENAME` 可以将工作区的状态变成暂存区或者版本库的状态（丢弃工作区的修改）。
- 使用 `git reset HEAD FILENAME` 可以将暂存区的修改撤销，重新放到工作区。
- 使用 `git rm FILENAME` 可以将文件从版本库和工作区中删除。

## 远程仓库
### 配置SSH Key
1. 使用 `git config --list` 可以查看git的全局配置。如果发现用户名和邮箱不对，可以通过前面的“初步使用”里的命令设置。
2. 使用 `ssh-keygen -t rsa -C "youremail@example.com"` 可以在目录 `~/.ssh` 内生成id_rsa和id_rsa.pub两个文件。
3. 将id_rsa.pub文件内的内容添加到github内的SSH Keys。
4. 使用 `ssh -T git@github.com` 测试是否添加成功。

### 克隆及推送
- 使用 `git remote add origin git@server-name:path/repo-name.git` 可以关联一个远程仓库。
- 使用 `git push -u origin master` 可以第一次推送master分支的所有内容，同时把本地的master分支和远程的master分支关联起来。
- 使用 `git clone git@server-name:path/repo-name.git` 可以克隆远程仓库，git的克隆支持多种协议，包括https、ssh。使用ssh速度更快，也不必每次都输入密码。

### 分支管理
- 使用 `git branch` 查看所有分支。
- 使用 `git checkout -b dev` 或 `git branch -c dev` 创建并切换到dev分支。
- 使用 `git checkout dev` 或 `git switch dev` 切换到dev分支。
- 使用 `git merge dev` 将dev分支合并到当前分支，默认使用fast-forward模式合并。
- 使用 `git branch -d dev` 可以删除dev分支。
- 使用 `git log --graph --pretty=oneline --abbrev-commit` 可以查看提交日志，并以图像显示。
- 使用 `git merge --no-ff -m "tips" BRANCH_NAME` 可以以禁用fast-forward来合并分支。
- 使用 `git stash` 将为完成的工作“隐藏”起来，等以后回复现场后继续工作。
- 使用 `git stash list` 列出stash内的快照。
- 使用 `git stash apply stash@{x}` 将现场恢复到stash@{x}，但是stash内不会删除，使用 `git stash drop stash@{x}` 可以删除。
- 使用 `git stash pop` 可以自动回复现场并且删除stash内的快照。
- 使用 `git remote -v` 显示远程库的信息。
- 使用 `git push -u origin/dev dev` 指定本地dev分支默认推送到远程dev分支。
- 使用 `git push origin master` 向远程master库推送本地的master库。
- 使用 `git pull` 是 `git fetch` 和 `git merge` 的合并版本。
- 使用 `git rebase` 把分叉的提交历史“整理”成一条直线，看上去更直观。

### 标签管理
- 使用 `git tag <tagname>` 用于新建一个标签，默认为HEAD，也可以指定一个commitID。
- 使用 `git tag -a <tagename> -m "blablabla..."` 可以指定标签的信息。
- 使用 `git tag` 可以查看所有标签。
- 使用 `git show <tagname>` 可以查看指定的标签。
- 使用 `git push origin <tagname>` 推送一个本地标签。
- 使用 `git push origin --tags` 可以推送全部未推送过的本地标签。
- 使用 `git tag -d <tagname>` 可以删除一个本地标签。
- 使用 `git push orgin :ref/tags/<tagename>` 可以删除一个远程标签。






[p0]:/media/20200316-1.png