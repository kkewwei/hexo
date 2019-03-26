---
title: git 基本命令学习心得-1
date: 2017-03-08 12:46:57
tags: git, rebase, cherry-pick, reset, checkout
---
一般在公共分支上操作,不能修改分支的提交记录, 但是可以使用cherry-pick, revert这样的可以使用, 而rebase, reset这样的命令一般在私有分支上才可用。
# rebase
git rebase是用来更改提交的基, 通过重新在当前分支提交一连串的commit来实现的, 比如dev分支从master A提交产生的, 在master分支又进行了B、C、D提交, 在dev分支进行了E、F、G提交, 此时为了保证D能够合并到master最新的D提交上, 那么就使用rebase。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_rebase3.png" height="300" width="550"/>
`git:(dev) git rebase master`, 此时处于G(最新提交)操作。变基是以共同祖先节点开始变的, 执行后就像放电影一样, 会将E、F、G的所有内容顺序与D合并。比如基于D与E合并后变成D', 基于D'与F合并变成F', 基于F'和G合并变成G'。箭头代表着基于哪些commit进行了merge。 完成rebase操作后, 提交链路被就改了。
用法:1.git rebase master 2. 通过vim解决冲突, 3.使用git add .保存 4.git rebase --continue继续解决下一个冲突
注意: 并没有要求修改comment, commit不变
## 半路rebase
以上示例是我们在dev分支的最新提交H上进行rebase的, 假如我们不在最新G上操作, 而是在F上进行rebase操作, 会出现什么结果呢?
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_rebase2.png" height="300" width="550"/>
可以看到dev分支没有任何变化, 而是将EF提交分别与D提交合并, 使EF部分变基到master分支上。
## rebase -i 参数
-i参数使rebase将于用户交互的形式完成merge, 根据这个参数, 用户可以完成多次提交顺序的复制、删除、编辑commit、修改提交顺序等一系列操作。
`git:(dev): git rebase -i master`之后, 将弹出这样的界面:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_rebase5.png" height="350" width="450"/>

红框1说明dev分支从master分支产生之后, 进行了三次提交, 这三次提交会和master最新的提交分别顺序进行合并。 红框1里面每行分为三部分:
+ 操作action
+ commit hash值
+ commit commit
其中action介绍在红框2中, 主要分为这几种类型:
1) p pick = use commit: 提交这次commit, 分别修改每次提交的comment内容。
git rebase -i master这种和别的分支进行rebase, 所有提交的comment是可以修改成功的。
git rebase -i HEAD~3 这种和本身历史提交进行rebase, 也提示需要修改comment, 但是最终没有生效。
2) r, reword = use commit, but edit the commit message: 提交这次commit, 同时修改这次comment的内容
git rebase -i HEAD~3 这种和本身历史提交进行rebase, 也提示需要修改comment, 同时修改这次comment的内容, 可以成功, 仅仅当前一次提交。(与说明相符)
git rebase -i master这种和别的分支进行rebase, 修改comment是可以修改成功的。 但是总是发生某些提交丢失的情况, 一般禁止使用。
3) e, edit = use commit, but stop for amending
git rebase -i master: 和p参数没啥区别, 能修改所有提交的comment
git rebase -i HEAD~3 : 可以停下来让你修改当前comment, 直到你调用--continue才能继续
4) s, squash = use commit, but meld into previous commit: 提交这次commit, 将本次提交和上次的提交合并(提交内容和提交comment都合并进去),使看起来是一次提交。
执行以下命令:`git:(dev): git rebase -i HEAD~3`, 并且修改action如下:
```
pick 159f932 d1
squash 4279e75 d2
pick 116755e d3
```
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_rebase6.png" height="350" width="450"/>
git log展示如上, 可以看出第一个和第二次的提交合并了, 所有的提交comment都修改了。
git rebase -i master也能和描述也是一致的。
5) f, fixup = like "squash", but discard this commit's log message: 和squash效果一样, 和之前提交的commit会合并到一起, 本次提交的comment会直接丢弃。
git rebase -i master: 可以一起修改前缀的comment(也可以将#去掉,同一个comment分两行显示两个comment), 后续的提交还可以修改comment。
git rebase -i HEAD~3 : 这次提交的coment直接丢弃, 所有提交都不给改的机会。
6) x, exec = run command (the rest of the line) using shell :执行shell命令, 可以忽略使用
7) d, drop = remove commit 直接丢弃这次提交(包括提交内容与comment)

## i参数控制复制、删除、重置提交顺序等
rebase -i参数可以通过重复上面的提交记录操作这些操作, 比如:
```
pick 159f932 d1
squash 4279e75 d2
pick 116755e d3
pick 159f932 d1
```
该提交将重复提交第一次提交, 提交结果如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_rebase7.png" height="350" width="450"/>
本地rebase之后再向远程推送, 可能会冲突, 这时确定没有人在基于那个分支分发的话, 可以通过git push --force origin mybranch分支。 master一般不允许直接这么弄

# cherry-pick
cherry-pick主要是将一个分支单独的提交变化合并到另外一个分支上, 以下图为例, 当前处于master的D提交上, 想让dev分支上的G提交改变合并到当前master分支上, 那么就执行:
`git:(master D提交): git cherry-pick G`, 提交完成后, G提交出的变化代码就会和master分支D处全部代码合并, 产生提交D'。注意,这里虽然合并了, 但是并没有改变分支提交记录, 图中用虚线表达着从提交历史上看, D'和G毫无关联。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_rebase10.png" height="350" width="450"/>
cherry-pick在提取G提交的变化时, 能将变化抽取出来的, 就将变化提取出来, 能合并就合并。变化合并到D时, 有冲突就解决冲突。
cherry-pick与rebase使用上的区别:
rebase: 修改提交历史, 改变的是整个分支的提交基, 将每次提交都与另外一个分支提交一一合并。
cherry-pick: 不会修改提交历史,仅仅产生一个新的提交。像挑选樱桃一样, 可以某个分支某次提交与另一个分支提交代码合并。

# revert
revert的含义是撤销(丢弃)某次提交, 下图为例, 比如想撤销G提交: `git:(master D提交): git revert G`, 实际就是丢弃G的提交, 具体实现是将G提交变化从当前提交D中去掉, 然后产生新的提交D'。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_revert1.png" height="350" width="450"/>
同cherry-pick一样, 并不改变历史提交记录, 仅仅将D和F(G的父提交)合并, 产生的新提交D'与G提交没有任何关系。
比如`git:(master D提交): git revert HEAD`, 撤销最近一次提交(也就提交D提交), 可能要产生冲突, 解决冲突后通过git add、git revert --continue来完成此次操作。

# checkout
checkout主要是从`对象库中(仓库)`拿出一个提交, 然后放在工作目录中, HEAD会指向当前提交(前提是工作区、暂存区、本地仓库一致, 否则会冲突); 附带功能是从`暂存区(索引)`中检出文件来重置工作区的文件, 使用示例如下:
+ git:(master D提交): git checkout G
将当前分支切换到G提交上面去, 此时工作区、暂存区、本地仓库代码将一致
+ git:(master D提交): git checkout -- file1
用`暂存区`的文件file1来重置`工作区`的file1, 而不是从本地代码仓库来恢复。
+ git:(master D提交): git checkout G -- file1
用G提交的代码`暂存区`的文件file1来重置`工作区`的file1。
checkout不会去修改提交记录, 仅仅是修改了HEAD。

# reset
reset主要是将仓库中的某次提交拿出来, 然后放回到暂存区、工作目录中, HEAD, master会指向当前提交, 提交历史会被改变。
+ 在切换到某一次提交时, 可使用 三个参数:
---soft: 仅仅是将当前master、HEAD指针指向commit3
---mixed: 在---soft的基础上, 用commit3的提交来重置暂存区。(默认)
---hard: 在---hard的集群上, 用commit3的提交来重置暂存区、工作区。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/git_reset.png" height="250" width="700"/>
+ git:(master D提交): git reset HEAD file1
用`本地代码仓库`的文件file1来重置`暂存区`(索引)的file1。
