---
title: Git学习
date: 2024-4-21
tags: 
  - Git
---

## 关联本地仓库和远程仓库并进行push

~~~shell
git remote add <reposiotry-name> <url>
# <reposiotry-name>是自定义的一个远程仓库nickname
~~~

~~~shell
git push <remote-repository> <local-branch>:<remote:branch>
# <remote-repository>用名字或者url都行
~~~

## 克隆刚刚push的仓库修改后再push

~~~shell
git clone <remote-repository-url> <target-directory>
~~~

~~~
修改其中的sample.txt文件
~~~

~~~shell
git push
# 因为是clone的,在.git中保留了远程仓库的相关信息
~~~



`git diff`，比较工作区和暂存区的文件之间的差异。

以暂存区的文件作为参照物来描述工作区的文件的差异。

同理`git diff --staged`，比较暂存区和存储区的差异，以存储区作为参照物



在`Git`中，代码的管理围绕**提交记录**进行

提交记录就是一个**提交对象**，提交对象包含了很多信息，比如`提交时的注释`、`指向上一个提交对象的指针`、`指向树对象的指针`。

**项目结构树对象**，其上的根节点和分支节点其实就是目录或者说文件夹。而其叶子节点指向实际的文件对象，文件对象实际上就是`blob`对象

**blob对象**，就是实际的文件对象

项目结构树对象和若干个blob对象就组成了一次提交时产生的暂存内容的**快照**，提交对象虽然说实际上指向的是树对象，但也可以说指向的是这次提交产生的快照

**分支**，就是一个指针，这个指针指向提交对象，比如说`master`，本质就是一个指针，通常情况下指向最新的一次提交对象。因为分支实际是指针，所以在`Git`中，称分支为`ref`，引用

**HEAD**，也是一个指针，这个指针可以指向分支，也可以指向提交对象

特别需要说明的是，比如说`HEAD^`，通常说是上一个节点，但是`HEAD`哪来的上一个节点？假设这时的情况是`HEAD->master->commit1`，`commit1`是有上一个节点的，所以这个过程是从`HEAD`递归寻找到其最终指向的一个提交对象，然后然后得出该提交对象的上一个提交对象，即`commit0`。这有点像运算符重载。同理`master^`也是一样。`ref^`与`ref~n`运算的结果都是提交对象

---

`git checkout <ref>/<commit>`命令，用来修改`HEAD`的指向，后面跟一个分支名或者一个提交对象。

~~~shell
git checkout main # 让HEAD指向分支main
git checkout <commit> # 让HEAD指向某次提交
~~~

`git branch -f <ref1> <commit>/<ref2>/HEAD`命令，改变分支`ref1`的指向，后面跟一个提交对象或者另一个分支。注意分支最终指向的一定是提交对象，所以这里第二个参数为分支或`HEAD`也有点递归的意思。

~~~shell
git branch -f main <commit> # 让分支main指向某次提交
git branch -f main master # 让分支main指向分支master指向的提交
git branch -f main HEAD # 让分支main指向HEAD最终指向的提交
~~~

`git reset <commit>/<ref>`命令，这个命令是修改当前`HEAD`所指向的分支的指向，但是`HEAD`的指向仍然是刚才的分支不会变。比如此时`HEAD->main->c1`，用了这个命令后`HEAD->main->c0`，也有点递归的意思

~~~shell
git reset master # 假如此前HEAD->main->c1，master->c0; 此后HEAD->main->c0
~~~



`git cherry-pick <commit>/<ref>`命令，复制其它分支上的提交对象，commit 到当前`HEAD`所指向的分支，比如此时`HEAD->main->c1`,`master->c2`，使用命令后`HEAD->main->c2->c1'`

~~~shell
# HEAD->main-><commit>
git cherry-pick <commit>
git cherry-pick master
git cherry-pick master^
~~~



`git rebase <ref>`命令，修改当前`HEAD`所指向的分支的基。`Git`会找到当前分支与参数中`ref`分支的第一个公共祖先节点，然后把当前分支指向的节点之前的提交对象做一个拷贝，把拷贝的对象都移到`ref`指向的提交对象的后面，最后修改当前分支的指向到最后一个拷贝对象上，注意`HEAD`的指向的还是当前分支不变，由此实现变基。加了`-i或者--iteractive`还可以打开交互式GUI界面，改变当前分支指向的节点到公共祖先节点之间的节点的顺序，或者删除节点

~~~shell
# HEAD->main->c4->c3->c1，master->c2->c1
git rebase master # 此后HEAD->main->c4'->c3'->c2->c1, master->c2->c1, c4->c3->c1
~~~

`git rebase <ref1> <ref2>`

~~~shell
git rebase <ref1> <ref2> # 把<ref2>分支变基到<ref1>分支上
~~~



`git commit --amend`命令，只会再次提交一次当前`HEAD`指向的提交对象，然后修改分支指向，不会影响后面子提交的父指向

~~~shell
# c3->c2->c1. HEAD->main->c2->c1
git commit --amend # c3->c2->c1, HEAD->main->c2'->c1
~~~



`git describe <commit>/<ref>/HEAD`，描述离指定节点最近的一次有`tag`的节点

`git checkout main; git meerge master`，合并的新节点属于`main`分支，也就是`HEAD->main-><new-commit>`

---

### git clone

既然你已经看过 `git clone` 命令了，咱们深入地看一下发生了什么。

你可能注意到的第一个事就是在我们的本地仓库多了一个名为 `o/main` 的分支, 这种类型的分支就叫**远程**分支。由于远程分支的特性导致其拥有一些特殊属性。

远程分支反映了远程仓库(在你上次和它通信时)的**状态**。这会有助于你理解本地的工作与公共工作的差别 —— 这是你与别人分享工作成果前至关重要的一步.

远程分支有一个特别的属性，在你切换到远程分支时，自动进入分离 HEAD 状态。Git 这么做是出于不能直接在这些分支上进行操作的原因, 你必须在别的地方完成你的工作, （更新了远程分支之后）再用远程分享你的工作成果。

> ### 为什么有 `o/`？
>
> 你可能想问这些远程分支的前面的 `o/` 是什么意思呢？好吧, 远程分支有一个命名规范 —— 它们的格式是:
>
> - `<remote name>/<branch name>`
>
> 因此，如果你看到一个名为 `o/main` 的分支，那么这个分支就叫 `main`，远程仓库的名称就是 `o`。
>
> 大多数的开发人员会将它们主要的远程仓库命名为 `origin`，并不是 `o`。这是因为当你用 `git clone` 某个仓库时，Git 已经帮你把远程仓库的名称设置为 `origin` 了
>
> 不过 `origin` 对于我们的 UI 来说太长了，因此不得不使用简写 `o` :) 但是要记住, 当你使用真正的 Git 时, 你的远程仓库默认为 `origin`!

`git clone`命令会创建一个新的分支`o/main`与一般分支不同的是，它是一个**远程分支**。远程分支有其特殊的属性，比较突出的是：

* 当`git checkout o/main`时，`HEAD`并不会指在远程分支`o/main`上，相应的`HEAD`指向到提交对象上。呈现出`HEAD`游离状态。

* 在远程分支的提交对象上`commit`新的结点后，一般的的分支会立即指向新的结点，但是远程分支不会，远程分支仍然指向先前的结点，取而代之的是游离态的`HEAD`会直接指向新的提交对象。



---

### git fetch 做了些什么

`git fetch` 完成了仅有的但是很重要的两步:

- 从远程仓库下载本地仓库中缺失的**提交记录**（不会改变本地文件）
- 更新远程分支指针（如 `o/main`）

`git fetch` 实际上将本地仓库中的远程分支更新成了远程仓库相应分支最新的状态。

如果你还记得上一节课程中我们说过的，远程分支反映了远程仓库在你**最后一次与它通信时**的状态，这里 `git fetch` 就是你与远程仓库通信的方式！希望我说的够明白了，你已经了解 `git fetch` 与远程分支之间的关系了吧？

`git fetch` 通常通过互联网（使用 `http://` 或 `git://` 协议）与远程仓库通信。

### git fetch 不会做的事

`git fetch` 并不会改变你本地仓库的状态。它不会更新你的 `main` 分支，也不会修改你磁盘上的文件。

理解这一点很重要，因为许多开发人员误以为执行了 `git fetch` 以后，他们本地仓库就与远程仓库同步了。它可能已经将进行这一操作所需的所有数据都下载了下来，但是**并没有**修改你本地的文件。我们在后面的课程中将会讲解能完成该操作的命令 :D

所以, 你可以将 `git fetch` 的理解为单纯的下载操作。

## 先抓取更新再合并到本地分支

这个流程指的是先`git fetch`，

再执行以下命令：

- `git cherry-pick orign/main`
- `git rebase orign/main`
- `git merge orign/main`
- 等等

但是，`git pull`合并了这两步操作

`git pull` 就是 `git fetch` 和 `git merge `的缩写！

`git pull --rebase` 就是 `git fetch` 和 `git rebase` 的简写！

---

## Git Push

OK，我们已经学过了如何从远程仓库获取更新并合并到本地的分支当中。这非常棒……但是我如何与大家分享**我的**成果呢？

嗯，上传自己分享内容与下载他人的分享刚好相反，那与 `git pull` 相反的命令是什么呢？`git push`！

`git push` 负责将**你的**变更上传到指定的远程仓库，并在远程仓库上合并你的新提交记录。一旦 `git push` 完成, 你的朋友们就可以从这个远程仓库下载你分享的成果了！

你可以将 `git push` 想象成发布你成果的命令。它有许多应用技巧，稍后我们会了解到，但是咱们还是先从基础的开始吧……

*注意 —— `git push` 不带任何参数时的行为与 Git 的一个名为 `push.default` 的配置有关。它的默认值取决于你正使用的 Git 的版本，但是在教程中我们使用的是 `upstream`。 这没什么太大的影响，但是在你的项目中进行推送之前，最好检查一下这个配置。*

`git push` 负责将**你的**变更上传到指定的远程仓库，并在远程仓库上**合并**你的新提交记录。

`git push`默认会把当前`HEAD`指向的分支上的所有节点提交到远程仓库，在远程仓库修改或创建同名分支，在本地仓库修改或创建`origin/<ref>`远程分支以同步远程仓库



- pull 操作时, 提交记录会被先下载到 o/main 上，之后再合并到本地的 main 分支。隐含的合并目标由这个关联确定的。
- push 操作时, 我们把工作从 `main` 推到远程仓库中的 `main` 分支(同时会更新远程分支 `o/main`) 。这个推送的目的地也是由这种关联确定的！

直接了当地讲，`main` 和 `o/main` 的关联关系就是由分支的“remote tracking”属性决定的。`main` 被设定为跟踪 `o/main` —— 这意味着为 `main` 分支指定了推送的目的地以及拉取后合并的目标。

当你克隆时, Git 会为远程仓库中的每个分支在本地仓库中创建一个远程分支（比如 `o/main`）。然后再创建一个跟踪远程仓库中活动分支的本地分支，默认情况下这个本地分支会被命名为 `main`。

这也解释了为什么会在克隆的时候会看到下面的输出：

```
local branch "main" set to track remote branch "o/main"
```

可以让任意分支跟踪 `o/main`, 然后该分支会像 `main` 分支一样得到隐含的 push 目的地以及 merge 的目标。 这意味着你可以在分支 `totallyNotMain` 上执行 `git push`，将工作推送到远程仓库的 `main` 分支上。

有两种方法设置这个属性，第一种就是通过远程分支切换到一个新的分支，执行:

```
git checkout -b totallyNotMain o/main
```

就可以创建一个名为 `totallyNotMain` 的分支，它跟踪远程分支 `o/main`。

另一种设置远程追踪分支的方法就是使用：`git branch -u` 命令，执行：

```
git branch -u o/main foo
```

这样 `foo` 就会跟踪 `o/main` 了。如果当前就在 foo 分支上, 还可以省略 foo：

```
git branch -u o/main
```

---

## 远程服务器拒绝!(Remote Rejected)

如果你是在一个大的合作团队中工作, 很可能是main被锁定了, 需要一些Pull Request流程来合并修改。如果你直接提交(commit)到本地main, 然后试图推送(push)修改, 你将会收到这样类似的信息:

```
! [远程服务器拒绝] main -> main (TF402455: 不允许推送(push)这个分支; 你必须使用pull request来更新这个分支.)
```

## 为什么会被拒绝?

远程服务器拒绝直接推送(push)提交到main, 因为策略配置要求 pull requests 来提交更新.

你应该按照流程,新建一个分支, 推送(push)这个分支并申请pull request,但是你忘记并直接提交给了main.现在你卡住并且无法推送你的更新.
