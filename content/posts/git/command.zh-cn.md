---
title: "Git 常用命令"
date: 2020-07-16
description: ""
menu:
  sidebar:
    name: "Git 常用命令"
    identifier: git-command
    parent: git
    weight: 100
tags: ["Git"]
categories: ["Git"]
---

根据 *[Pro Git (2nd Edition)](https://git-scm.com/book/zh/v2)*（中文版）整理。

<!--more-->

## Git 基础

### 初次运行 Git 前的配置

```bash
git config --global user.name "FreeFlyingSheep"
git config --global user.email "chris.chenfeiyang@outlook.com"
```

### 代理配置

#### 设置代理

```bash
git config --global https.proxy "http://127.0.0.1:xxxx"
git config --global https.proxy "https://127.0.0.1:xxxx"
git config --global http.proxy "socks5://127.0.0.1:xxxx"
git config --global https.proxy "socks5://127.0.0.1:xxxx"
```

#### 取消代理

```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

### 获取 Git 仓库

#### 在已存在目录中初始化仓库

```bash
cd <my_project>
git init
```

#### 克隆现有的仓库

```bash
git clone <url> [my_name]
```

### 查看文件状态

```bash
git status
```

### 查看修改内容

```bash
git diff
```

`git diff` 本身只显示尚未暂存的改动，而不是自上次提交以来所做的所有改动。

### 跟踪新文件

```bash
git add <file> ...
```

### 提交更新

```bash
git commit -m '<message>'
```

### 忽略文件

文件 `.gitignore` 的格式规范如下：

- 所有空行或者以 `#` 开头的行都会被 Git 忽略
- 可以使用标准的 glob 模式匹配，它会递归地应用在整个工作区中
- 匹配模式可以以（`/`）开头防止递归
- 匹配模式可以以（`/`）结尾指定目录
- 要忽略指定模式以外的文件或目录，可以在模式前加上叹号（`!`）取反

所谓的 glob 模式是指 shell 所使用的简化了的正则表达式：

- 星号（`*`）匹配零个或多个任意字符；`[abc]` 匹配任何一个列在方括号中的字符（这个例子要么匹配一个 `a`，要么匹配一个 `b`，要么匹配一个 `c`）
- 问号（`?`）只匹配一个任意字符
- 如果在方括号中使用短划线分隔两个字符，表示所有在这两个字符范围内的都可以匹配（比如 `[0-9]` 表示匹配所有 `0` 到 `9` 的数字）
- 使用两个星号（`**`）表示匹配任意中间目录，比如 `a/**/z` 可以匹配 `a/z`、`a/b/z` 或 `a/b/c/z` 等

GitHub 有一个十分详细的针对数十种项目及语言的 [`.gitignore` 文件列表](https://github.com/github/gitignore)。

在最简单的情况下，一个仓库可能只根目录下有一个 `.gitignore` 文件，它递归地应用到整个仓库中。然而，子目录下也可以有额外的 `.gitignore` 文件。子目录中的 `.gitignore` 文件中的规则只作用于它所在的目录中。

### 移除文件

```bash
git rm <file> ...
```

如果要删除之前修改过或已经放到暂存区的文件，则必须使用强制删除选项 `-f`。

另外一种情况是，我们想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录中。换句话说，你想让文件保留在磁盘，但是并不想让 Git 继续跟踪。当你忘记添加 `.gitignore` 文件，不小心把一个很大的日志文件或一堆 .a 这样的编译生成文件添加到暂存区时，这一做法尤其有用。为达到这一目的，使用 `--cached` 选项。

### 移动文件

```bash
git mv <file_from> <file_to>
```

### 查看提交历史

```bash
git log
```

### 撤消操作

#### 补上遗漏的文件

有时候我们提交完了才发现漏掉了几个文件没有添加，或者提交信息写错了，此时，可以运行带有 `--amend` 选项的提交命令来重新提交：

```bash
git commit -m 'initial commit'
git add forgotten_file
git commit --amend
```

#### 取消暂存的文件

```bash
git reset HEAD <file> ...
```

#### 撤消对文件的修改

```bash
git checkout -- <file>
```

请务必记得这是一个危险的命令，你对那个文件在本地的任何修改都会消失—— Git 会用最近提交的版本覆盖掉它。除非你确实清楚不想要对那个文件的本地修改了，否则请不要使用这个命令。

### 远程仓库的使用

#### 查看远程仓库

```bash
git remote -v
```

#### 添加远程仓库

```bash
git remote add <shortname> <url>
```

#### 从远程仓库中抓取

```bash
git fetch [<remote>]
```

这个命令会访问远程仓库，从中拉取所有你还没有的数据。执行完成后，你将会拥有那个远程仓库中所有分支的引用，可以随时合并或查看。

必须注意该命令只会将数据下载到你的本地仓库——它并不会自动合并或修改你当前的工作。当准备好时你必须手动将其合并入你的工作。

#### 从远程仓库中拉取

```bash
git pull [<remote>]
```

如果你的当前分支设置了跟踪远程分支，那么可以用 `git pull` 命令来自动抓取后合并该远程分支到当前分支。

默认情况下，`git clone` 命令会自动设置本地 `master` 分支跟踪克隆的远程仓库的 `master` 分支（或其它名字的默认分支）。运行 `git pull` 通常会从最初克隆的服务器上抓取数据并自动尝试合并到当前所在的分支。

#### 推送到远程仓库

```bash
git push <remote> <branch>
```

#### 查看某个远程仓库

```bash
git remote show <remote>
```

#### 远程仓库的重命名

```bash
git remote rename <old> <new>
```

#### 远程仓库的移除

```bash
git remote remove <remote>
```

### 标签

轻量标签很像一个不会改变的分支——它只是某个特定提交的引用。

附注标签是存储在 Git 数据库中的一个完整对象，它们是可以被校验的，其中包含打标签者的名字、电子邮件地址、日期时间，此外还有一个标签信息，并且可以使用 GNU Privacy Guard（GPG）签名并验证。通常会建议创建附注标签，这样你可以拥有以上所有信息。但是如果你只是想用一个临时的标签，或者因为某些原因不想要保存这些信息，那么也可以用轻量标签。

#### 列出标签

```bash
git tag
```

#### 创建标签

创建附注标签：

```bash
git tag -a <tagname> -m "<message>"
```

创建轻量标签：

```bash
git tag <tagname>
```

轻量标签本质上是将提交校验和存储到一个文件中——没有保存任何其他信息。创建轻量标签，不需要使用 -a、-s 或 -m 选项，只需要提供标签名字。

#### 显示标签信息

```bash
git show <tagname>
```

在附注标签上运行 `git show`，输出会显示打标签者的信息、打标签的日期时间、附注信息，然后显示具体的提交信息。

在轻量标签上运行 `git show`，你不会看到额外的标签信息，命令只会显示出提交信息。

#### 后期打标签

你也可以对过去的提交打标签，只需要在命令的末尾指定提交的校验和（或部分校验和）。

示例：

```bash
$ git log --pretty=oneline
15027957951b64cf874c3557a0f3547bd83b3ff6 Merge branch 'experiment'
a6b4c97498bd301d84096da251c98a07c7723e65 beginning write support
0d52aaab4479697da7686c15f77a3d64d9165190 one more thing
6d52a271eda8725415634dd79daabbc4d9b6008e Merge branch 'experiment'
0b7434d86859cc7b8c3d5e1dddfed66ff742fcbc added a commit function
4682c3261057305bdd616e23b64b0857d832627b added a todo file
166ae0c4d3f420721acbb115cc33848dfcc2121a started write support
9fceb02d0ae598e95dc970b74767f19372d61af8 updated rakefile
964f16d36dfccde844893cac5b347e7b3d44abbc commit the todo
8a5cbc430f1a9c3d00faaeffd07798508422908a updated readme
$ git tag -a v1.2 9fceb02
$ git tag
v0.1
v1.2
v1.3
v1.4
v1.4-lw
v1.5
$ git show v1.2
tag v1.2
Tagger: Scott Chacon <schacon@gee-mail.com>
Date: Mon Feb 9 15:32:16 2009 -0800
version 1.2
commit 9fceb02d0ae598e95dc970b74767f19372d61af8
Author: Magnus Chacon <mchacon@gee-mail.com>
Date: Sun Apr 27 20:43:35 2008 -0700
updated rakefile
...
```

#### 共享标签

默认情况下，git push 命令并不会传送标签到远程仓库服务器上。在创建完标签后你必须显式地推送标签到共享服务器上。这个过程就像共享远程分支一样——你可以运行 `git push origin <tagname>`。

如果想要一次性推送很多标签，也可以使用带有 `--tags` 选项的 `git push` 命令。这将会把所有不在远程仓库服务器上的标签全部传送到那里。

#### 删除标签

要删除掉你本地仓库上的标签，可以使用命令 `git tag -d <tagname>`。

一种直观的删除远程标签的方式是使用命令 `git push origin --delete <tagname>`。

#### 检出标签

```bash
git checkout <tagname>
```

### Git 别名

```bash
git config --global alias.<alias> <commond>
```

例如，为了解决取消暂存文件的易用性问题，可以向 Git 中添加你自己的取消暂存别名：

```bash
git config --global alias.unstage 'reset HEAD --'
```

这会使下面的两个命令等价：

```bash
git unstage fileA
git reset HEAD -- fileA
```

这样看起来更清楚一些。通常也会添加一个 last 命令，像这样：

```bash
git config --global alias.last 'log -1 HEAD'
```

这样，可以轻松地看到最后一次提交。

## Git 分支

### 创建分支

```bash
git branch <branch>
```

### 切换分支

```bash
git checkout <branch>
```

### 创建并切换分支

```bash
git checkout -b <branch>
```

### 合并分支

切换（`checkout`）到想要合并入的分支，如 `master`，然后执行：

```bash
git merge <branch>
```

如何解决合并冲突，举个例子，如果你对 `#53` 问题的修改和有关 `hotfix` 分支的修改都涉及到同一个文件的同一处，在合并它们的时候就会产生合并冲突：

```bash
$ git merge iss53
Auto-merging index.html
CONFLICT (content): Merge conflict in index.html
Automatic merge failed; fix conflicts and then commit the result.
```

你可以在合并冲突后的任意时刻使用 `git status` 命令来查看那些因包含合并冲突而处于未合并（`unmerged`）状态的文件：

```bash
$ git status
On branch master
You have unmerged paths.
(fix conflicts and run "git commit")
Unmerged paths:
(use "git add <file>..." to mark resolution)
both modified: index.html
no changes added to commit (use "git add" and/or "git commit -a")
```

Git 会在有冲突的文件中加入标准的冲突解决标记，这样你可以打开这些包含冲突的文件然后手动解决冲突。出现冲突的文件会包含一些特殊区段，看起来像下面这个样子：

```text
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
```

这表示 `HEAD` 所指示的版本（也就是你的 `master` 分支所在的位置，因为你在运行 `merge` 命令的时候已经检出到了这个分支）在这个区段的上半部分（`=======` 的上半部分），而 `iss53` 分支所指示的版本在 `=======` 的下半部分。为了解决冲突，你必须选择使用由 `=======` 分割的两部分中的一个，或者你也可以自行合并这些内容。例如，你可以通过把这段内容换成下面的样子来解决冲突：

```html
<div id="footer">
please contact us at email.support@github.com
</div>
```

上述的冲突解决方案仅保留了其中一个分支的修改，并且 `<<<<<<<`，`=======`，和 `>>>>>>>` 这些行被完全删除了。在你解决了所有文件里的冲突之后，对每个文件使用 `git add` 命令来将其标记为冲突已解决。一旦暂存这些原本有冲突的文件，Git 就会将它们标记为冲突已解决。

### 删除分支

```bash
git branch -d <branch>
```

若要删除包含了还未合并的工作的分支，`-d` 选项需要改为 `-D`。

### 查看分支历史

```bash
git log --oneline --decorate --graph --all
```

### 查看分支列表

```bash
git branch
```

加 `-v` 选项查看每一个分支的最后一次提交，`--merged` 与 `--no-merged` 选项可以过滤这个列表中已经合并或尚未合并到当前分支的分支。

### 远程分支

#### 跟踪远程分支

远程分支以 `<remote>/<branch>` 命名，跟踪远程分支可以用如下命令：

```bash
git checkout -b <branch> <remote>/<branch>
```

或者可以使用如下命令：

```bash
git checkout --track <remote>/<branch>
```

如果你尝试检出的分支不存在且刚好只有一个名字与之匹配的远程分支，那么 Git 就会自动为你创建一个跟踪分支：

```bash
git checkout <branch>
```

设置已有的本地分支跟踪一个刚刚拉取下来的远程分支，或者想要修改正在跟踪的上游分支，你可以在任意时间使用 `-u` 或 `--set-upstream-to` 选项运行 `git branch` 来显式地设置：

```bash
git branch -u <remote>/<branch>
```

#### 查看跟踪分支

```bash
git branch -vv
```

#### 拉取远程分支

见[从远程仓库中抓取](#从远程仓库中抓取)和[从远程仓库中拉取](#从远程仓库中拉取)。

不管设置好的跟踪分支是显式地设置还是通过 `clone` 或 `checkout` 命令为你创建的，`git pull` 都会查找当前分支所跟踪的服务器与分支，从服务器上抓取数据然后尝试合并入那个远程分支。

由于 `git pull` 的魔法经常令人困惑所以通常单独显式地使用 `fetch` 与 `merge` 命令会更好一些。

#### 推送远程分支

见[推送到远程仓库](#推送到远程仓库)。

#### 删除远程分支

```bash
git push <remote> --delete <branch>
```

### 变基

```bash
git rebase <branch>
```

变基的目的是为了确保在向远程分支推送时能保持提交历史的整洁——例如向某个其他人维护的项目贡献代码时。在这种情况下，你首先在自己的分支里进行开发，当开发完成时你需要先将你的代码进行变基，然后再向主模块提交修改。
这样的话，该项目的维护者就不再需要进行整合工作，只需要快进合并便可。

以一个例子说明合并（`merge`）和变基（`rebase`）的区别，假设现在分支如下：

![现在的分支](/images/git/branch.png)

把 `experiment` 合并到 `master`：

![把 `experiment` 合并到 `master`](/images/git/merge.png)

对应的操作如下：

```bash
git checkout master
git merge experiment
```

可以看到，历史记录**不是一条直线** 。

如果先把 `experiment` 变基到 `master`：

![把 `master` 变基到 `experiment`](/images/git/rebase.png)

然后再进行合并操作：

![把 `experiment` 合并到 `master`](/images/git/merge2.png)

对应的操作如下：

```bash
git checkout experiment
git rebase master
git checkout master
git merge experiment
```

这样历史记录就**是一条直线**了。

变基的准则：**如果提交存在于你的仓库之外，而别人可能基于这些提交进行开发，那么不要执行变基。**

## Github

### 生成 SSH 公钥

```bash
ssh-keygen -t rsa -C "chris.chenfeiyang@outlook.com"
```

### 添加 SSH 公钥

```bash
cat ~/.ssh/id_rsa.pub
```

将内容添加到 Github [相应页面](https://github.com/settings/ssh/new)添加公钥。

### 验证

```bash
ssh -T git@github.com
```

首次使用需要确认并添加主机到本机SSH可信列表。若返回 `Hi FreeFlyingSheep! You've successfully authenticated, but GitHub does not provide shell access.` 内容，则证明添加成功。

## 子模块

### 添加子模块

```bash
git submodule add <url> <path>
```

生成的 `.gitmodules` 该配置文件保存了项目 URL 与已经拉取的本地目录之间的映射。

### 克隆含有子模块的项目

```bash
git clone <url>
git submodule init
git submodule update
```

其中，后两行命令可以合并为如下命令：

```bash
git submodule update --init
```

或者，直接使用如下命令一步到位：

```bash
git clone --recurse-submodules <url>
```

### 从项目远端拉取上游更改

可以进入到目录中运行 `git fetch` 与 `git merge`，或者执行以下命令：

```bash
git submodule update --remote
```

Git 默认会尝试更新所有子模块，你也可以传递想要更新的子模块的名字。

为了安全起见，如果主模块提交了你刚拉取的新子模块，那么应该在 `git submodule update` 后面添加 `--init` 选项，如果子模块有嵌套的子模块，则应使用 --recursive 选项。
如果你想自动化此过程，那么可以为 `git pull` 命令添加 `--recurse-submodule`。

### 在子模块上工作

首先，进入每个子模块并检出其相应的工作分支。

接着，若你做了更改就需要告诉 Git 它该做什么，然后运行 `git submodule update --remote` 来从上游拉
取新工作。你可以选择将它们合并（`--merge`）到你的本地工作中，也可以尝试将你的工作变基（`--rebase`）到新的更改上。

如果你忘记 `--rebase` 或 `--merge`，Git 会将子模块更新为服务器上的状态。并且会将项目重置为一个游离的
`HEAD` 状态。即便这真的发生了也不要紧，你只需回到目录中再次检出你的分支然后手动地合并或变基任何一个你想要的远程分支就行了。

### 查看子模块修改内容

```bash
git diff --submodule
```

### 查看子模块提交历史

```bash
git log -p --submodule
```

### 提交子模块更新

```bash
git commit -am '<message>'
```

### 发布子模块改动

在推送主模块前，应该先推送所有子模块。

可以使用 `check` 选项，这样如果任何提交的子模块改动没有推送，`push` 操作会直接失败：

```bash
git push --recurse-submodules=check
```

或者在主模块推送时采用 `on-demand` 选项，让 Git 自己尝试这么做：

```bash
git push --recurse-submodules=on-demand
```

### 切换带子模块的项目的分支

```bash
git checkout --recurse-submodules <branch>
```

### 删除子模块

1. 执行 `git rm <submodule>` 删除子模块目录。
2. 删除 `.git/modules` 下子模块相关内容。
3. 编辑 `.git/config` 删除子模块相关内容。
4. 执行 `git commit` 提交更新。

## 贮藏

### 贮藏工作

```bash
git stash [push]
```

### 查看贮藏的工作

```bash
git stash list
```

### 应用贮藏

将刚刚贮藏的工作重新应用：

```bash
git stash apply
```

应用一个更旧的贮藏，可以通过名字指定它：

```bash
git stash apply <stashname>
```

比如：

```bash
git stash apply stash@{2}
```

应用选项只会尝试应用贮藏的工作，不会把贮藏从堆栈上移除。

### 移除贮藏

```bash
git stash drop <stashname>
```

### 应用并移除贮藏

```bash
git stash pop
```
