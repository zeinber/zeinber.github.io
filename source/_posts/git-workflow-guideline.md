---
title: Git 工作流指南
date: 2017-04-18 23:41:08
tags:
    - workflow
categories: Git
---

[Git 工作流（Git Workflows）](https://www.atlassian.com/git/tutorials/comparing-workflows)的本质问题是有效的项目流程管理和高效的开发协同约定。

<!--more-->

### 任务管理

在 GitLab 中有以下两种任务管理的方式：

#### milestone

- 一个迭代版本对应一个milestone
- 一组可被归类的任务可以是一个milestone
- 可以设置 dead line

#### issue

- 所有工作可以通过 issue 体现，例如： feature 开发、设计方案制定、问题讨论等
- 可以将这些 issue 继续拆分成针对开发过程的 issue
- 一个 MR 对应一个 issue
- Issue 需要在 MR + 3 合并后方可 merge

### Merge Request

- 创建

  - 创建 MR 时如果有代码冲突，需手动解决后再让他人 review
  - 谁创建的 MR 谁负责 Accept，创建者应该 Assign 给自己

- 约定

  - +1 表示同意合并，-1表示不允许合并
  - 加入被 review 过的代码出现线上故障，+1 的人同等责任
  - MR 合并后才允许提测，并关闭 jira 中的 Bug
  - 紧急发布无人 review 时，向上级报告后允许事后 MR
  - 一次 MR 的代码数量控制在 200 行以内

### 分支管理

正确分支包含基线分支和 issue 分支，MR 就是把 issue 分支合并到基线分支。

以客户端 Branch 为例

- 日常开发
  - 通常在 master 分支进行迭代开发
  - 所有 issue 通过创建临时分支进行开发，MR 后回 master
  - 发布后需要在最后一个 commit 上打 tag
  - 一次送测也需要一个 tag，命名上要和发布 tag 加以区分，例如 1.0.1A

- 并行开发
  - 从上个发布 tag 创建一个用于并行开发的基线分支，在该分支为 issue 创建临时分支进行开发， MR 后回 master
  - 如在基线分支未完成前产生了新的发布 tag，需要将该 tag 的代码合并到基线分支
  - 发布后合并回 master

- 热修复
  - 从对应的 tag 拉一个分支来进行 hotfix commit
  - 发布后需打上 tag，合并后回 master

- 维护分支
  - 确保一个仓库下仅有一个维护分支，清理及合并历史分支
  - 如果无法合并历史分支，可以采用“空合并”（即产生一个 merge 但不做任何修改）
  - 适当使用 rebase，绝不在公共分支上使用它

### [Git 常用指令](https://github.com/geeeeeeeeek/git-recipes/wiki)

#### 创建仓库

创建新仓库：

```
git init                    # 将当前的目录转换成一个 Git 仓库
git init <directory>        # 创建一个名为 directory，只包含 .git 子目录的空目录。
git init --bare <directory> # 创建一个名为 directory 的共享仓库
```

创建一个本地仓库的克隆版本：

```
git clone /path/to/repository
```

拉取远端服务器上的仓库：

```
git clone username@host:/path/to/repository # 通过 SSH
git clone https:/path/to/repository.git     # 通过 https
```

#### 查看仓库的状态

以下的每一步，都可以通过下面这个指令来查看工作目录和缓存区：

```
git status  # 这一命令会列出已缓存、未缓存、未追踪的文件
```

而下面这类命令用于查看项目历史的基本工具：

```
git log                        # 显示完整的项目历史，如果输出超过一屏，用空格键来滚动，按 q 退出
git log -n <limit>             # 只会显示 <limit> 个提交
git log --oneline              # 将每个提交压缩到一行
git log --stat                 # 除了项目历史信息之外，包含哪些文件被更改了，以及每个文件相对的增删行数
git log -p                     # 显示每个提交全部的差异（diff）
git log --author="<pattern>"   # 搜索特定作者的提交，<pattern> 可以是字符串或正则表达式
git log <since>..<until>       # 只显示发生在 <since> 和 <until> 之间的提交。<>参数可以是任何一种引用
git log <file>                 # 只显示包含特定文件的提交
git log --graph --decorate --oneline   # --graph 标记会绘制一幅字符组成的图形，左边是提交，右边是提交信息。--decorate 标记会加上提交所在的分支名称和标签。--oneline 标记将提交信息显示在同一行。
```

 #### 添加与提交

`git add` 命令主要用于把我们要提交的文件的信息添加到索引库中。

```
git add <path>         # 把 <path> 添加到索引库中，<path> 可以是文件也可以是目录
git add .              # 将所有修改添加到暂存区
git add *              # Ant风格添加修改
git add *filetype      # 将所有以 filetype 结尾的文件的所有修改添加到暂存区
git add filename*.     # 将所有以 filename 开头的文件的修改添加到暂存区。如:filename.h,filename.m...
git add filename?      # 将所有以 filename 开头，且后面只有一位的文件的修改提交到暂存区
git add -A             # 将所有跟踪文件中被修改过或已删除文件和所有未跟踪的文件信息添加到索引库
git add -i             # 进入交互式，查看所有修改过或已删除文件但没有提交的文件
git add -p             # 进入补丁模式，可以选择文件的一部分加入到下次提交缓存
git add -u             # 查看所有 tracked 文件中被修改过或已删除文件的信息添加到索引库
```

这是 git 基本工作流程的第一步。使用如下命令以实际提交改动：

```
git commit -m "commit message"
```

#### 推送改动

commit 后文件的改动已经在本地仓库的 HEAD 中了。执行如下命令以将这些改动提交到远端仓库：

```
git push origin master
```

你可以把 master 换成你想要推送的任何分支。

如果你还没有克隆现有仓库，并欲将你的仓库连接到某个远程服务器，可以使用如下命令添加：

```
git remote add origin <server>
```

#### 检出之前的提交

`git checkout`  这个命令有三个不同的作用：检出文件、检出提交和检出分支。常规用法有：

```
git checkout master            # 回到 master 分支
git checkout <commit> <file>   # 将工作目录中的 <file> 文件变成 <commit> 中文件的拷贝，并将它加入缓存区
git checkout <commit>          # 更新工作目录中的所有文件，使得和某个特定提交中的文件一致
```

#### 回滚错误的修改

回滚有 `revert` 和 `reset` 两种做法，注意这两类的区别：

##### git revert

一种*安全*的方式。用来撤销一个已经提交的快照。它是通过搞清楚如何撤销这次提交引入的更改，并在最后加上一个撤销了更改的新提交，而不是从项目历史中移除这次提交。

用法：

```
git revert <commit>
```

##### git reset

一种*危险*的方式。使用这个命令来重设更改时，我们无法获得原来的样子——这个撤销是不可恢复的。

用法：

```
git reset <file>           # 从缓存区移除特定文件，但不改变工作目录
git reset                  # 重设缓冲区，匹配最近的一次提交，但工作目录不变
git reset --hard           # 重设缓冲区和工作目录，匹配最近的一次提交
git reset <commit>         # 将当前分支的末端移到 <commit>，将缓存区重设到这次提交，但不改变工作目录
git reset --hard <commit>  # 将当前分支的末端移到 <commit>，将缓存区和工作目录都重设到这次提交
```

##### git clean

将未跟踪的文件从你的工作目录中移除，这一命令是无法撤消的。

`git clean` 命令经常和 `git reset —hard` 一起使用。`reset` 只影响被跟踪的文件，所以还需要一个单独的命令来清理未被跟踪的文件。这个两个命令相结合，你就可以将工作目录回到之前特定提交时的状态。

```
git clean -n         # 执行一次 clean 演习。它会告诉你哪些文件在命令执行后会被移除，并非真的删除它
git clean -f         # 移除当前目录下未被跟踪的文件
git clean -f <path>  # 移除未跟踪的文件，但限制在某个路径下
git clean -df        # 移除未跟踪的文件，以及目录
git clean -xf        # 移除当前目录下未跟踪的文件，以及 Git 一般忽略的文件
```

#### 保持同步

##### git remote

`git remote` 命令允许你创建、查看和删除和其它仓库之间的连接。远程连接更像是书签，而不是直接跳转到其他仓库的链接。用法如下：

```
git remote                   # 列出和其他仓库之间的远程连接
git remote -v                # 列出和其他仓库之间的远程连接，并同时显示每个连接的 URL
git remote add <name> <url>  # 创建一个新的远程仓库连接
git remote rm <name>         # 移除指定 name 的远程仓库的连接
git remote rename <old_name> <new_name>     # 将远程连接重命名
```

##### git fetch

`git fetch` 命令将提交从远程仓库导入到你的本地仓库。拉取下来的提交储存为远程分支，而不是我们一直使用的普通的本地分支。用法：

```
git fetch <remote>           # 拉取仓库中所有的分支，同时会从另一个仓库中下载所有需要的提交和文件
git fetch <remote> <branch>  # 拉取指定的分支
```

##### git pull

`git pull ` 命令用于从另一个存储库或本地分支获取并集成。它的效果和 `git fetch` 后接 `git merge origin/.`  一致。用法：

```
git pull <remote>            # 拉取当前分支对应的远程副本中的更改，并立即并入本地副本
git pull --rebase <remote>   # 拉取远程分支并与本地分支合并
```

#### 分支

##### 查看分支

```
git branch                # 查看本地分支
git branch -r             # 查看远程分支
git branch -v             # 查看各个分支最后提交的信息
git branch --merged       # 查看已经被合并到当前分支的分支
git branch --no-merged    # 查看尚未被合并到当前分支的分支
```

##### 添加分支

```
git checkout <branchname>              # 本地创建并切换到某个分支
git checkout -b <branchname>           # 创建新的分支，并且切换过去
git checkout -b <new_br> <branch>      # 基于 branch 创建新分支
git checkout <commit>         # 把某次提交记录 checkout 出来，但无分支信息，切换到其他分支会自动删除
git checkout <commit> -b <new_br>      # 把某次历史提交记录checkout出来，创建成一个分支
```

##### 合并分支

```
git merge <branchname>              # 将某分支合并到当前分支
git merge origin/master --no-ff     # 不要 Fast-Foward 合并，这样可以生成 merge 提交
git rebase master <branch>          # 将 master rebase 到branch，相当于：
git checkout <branchname> && git rebase master && git checkout master && git merge <branchname>
```

##### 发布分支

```
git push <repository> <local_branch>                    # 创建远程分支，repository 是仓库名
git push <repository> <local_branch>:<remote_branch>    # 创建远程分支
git push <repository> :<remote_branch>                  # 先删除本地分支，再 push 删除远程分支
```

##### 删除分支

```
git branch -d <branchname>   # 删除本地某个分支
git branch -D <branchname>   # 强制删除某个本地分支 (未被合并的分支被删除的时候需要强制)
```

#### 标签

Git 可以在某个时间节点的版本上打上标签，并通常在某版本的最近一次 commit 时标记。

##### 查看标签

```
git tag                       # 列出所有可用标签，显示的标签按字母顺序排列
git tag -l 'tag condition'    # 根据特定的 tag 特征列出符合条件的标签，如 git tag -l 'v1.0.*'
```

##### 添加标签

```
git tag <tagname>                       # 创建轻量标签
git tag -a <tagname> -m "tag message"   # 创建附注标签
git tag -a <tagname> <commit>           # 给指定的 commit 补打标签
```

##### 切换标签

```
git checkout <tagname>
```

##### 查看标签信息

```
git show <tagname>
```

##### 发布标签

```
git push origin <tagname>    # 将标签提交到 Git 服务器
git push origin –tags        # 将本地所有标签一次性提交到 Git 服务器
```

##### 删除标签

```
git tag -d <tagname>                   # 删除本地仓库标签
git push origin :refs/tags/<tagname>   # 删除远程仓库标签
```
