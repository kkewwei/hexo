---
title: git 基本命令学习心得-1
date: 2017-03-08 12:46:57
tags:
---
rebase, cherry-pick、merge等都有一个概念:提交commit, 就以下图为例, 把当一次提交D合并到另外一个提交E, 产生E', 这里的提交D指的当前D的所有全量代码, 去和E合并, 并不是由C commit到D时产生的增量代码去和E合并。
# rebase
git rebase是用来重新提交一连串的commit, 比如在master分支记性了A、B、C、D提交, 在dev分支进行了E、F、G提交, 此时为了保证D能够合并到master最新的D提交上, 那么就使用rebase。
<img src="http://pgagp8vnu.bkt.clouddn.com/git_rebase3.png" height="300" width="500"/>
`git:(dev) git rebase master`, 执行后开始像放电影一样, 会将E、F、G提交的每次变动内容逐渐与D合并。比如基于D与E合并后变成D', 基于D'与F合并变成F', 基于F'和G合并变成G'。箭头代表着基于哪些commit进行了merge
用法:1.git rebase master 2. 通过vim解决冲突, 3.使用git add .保存 4.git rebase --continue继续解决下一个冲突
注意: 并没有要求修改comment, commit不变

## rebase -i 参数
-i参数使rebase将于用户交互的形式完成merge, 根据这个参数, 用户可以完成多次提交顺序的置换、删除、编辑commit等一系列操作。
`git:(dev): git rebase -i master`之后, 将弹出这样的界面:
<img src="http://pgagp8vnu.bkt.clouddn.com/git_rebase5.png" height="350" width="450"/>
红框1说明dev分支从master分支产生之后, 进行了三次提交, 这三次提交这里在于master最新的提交一次次合并进去。 红框1里面每行分为三部分:
+ 操作action
+ commit hash值
+ commit commit
其中action介绍在红框2中, 主要分为这几种类型:
# p, pick = use commit: 提交这次commit, 分别修改每次提交的comment内容。
git rebase -i master这种和别的分支进行rebase, 所有提交的comment是可以修改成功的。
git rebase -i HEAD~3 这种和本身历史提交进行rebase, 也提示需要修改comment, 但是最终没有生效。

# r, reword = use commit, but edit the commit message: 提交这次commit, 同时修改这次comment的内容
git rebase -i HEAD~3 这种和本身历史提交进行rebase, 也提示需要修改comment, 同时修改这次comment的内容, 可以成功, 仅仅当前一次提交。(与说明相符)
git rebase -i master这种和别的分支进行rebase, 修改comment是可以修改成功的。 但是总是发生某些提交丢失的情况, 一般禁止使用。


# e, edit = use commit, but stop for amending
git rebase -i master: 和p参数没啥区别, 能修改所有提交的comment
git rebase -i HEAD~3 : 可以停下来让你修改当前comment, 直到你调用--continue才能继续

# s, squash = use commit, but meld into previous commit: 提交这次commit, 将本次提交和上次的提交合并(提交内容和提交comment都合并进去),使看起来是一次提交。
执行以下命令:git:(dev): git rebase -i HEAD~3, 并且修改action如下:
```
pick 159f932 d1
squash 4279e75 d2
pick 116755e d3
```
<img src="http://pgagp8vnu.bkt.clouddn.com/git_rebase6.png" height="350" width="450"/>
git log展示如上, 可以看出第一个和第二次的提交合并了, 所有的提交comment都修改了。
git rebase -i master也能和描述也是一致的。



# f, fixup = like "squash", but discard this commit's log message: 和squash效果一样, 和之前提交的commit会合并到一起。 本次提交的comment会直接丢弃
git rebase -i master:   可以一起修改前缀的comment(也可以将#去掉,同一个comment分两行显示两个comment), 后续的提交还可以修改comment
git rebase -i HEAD~3 :   这次提交的coment直接丢弃, 所有提交都不给改的机会。
# x, exec = run command (the rest of the line) using shell :执行shell命令, 可以忽略使用
# d, drop = remove commit 直接丢弃这次提交(包括提交内容与comment)

## 参数控制复制、删除、重置提交顺序等工作
rebase -i参数可以通过重复上面的提交记录操作这些操作, 比如:
```
pick 159f932 d1
squash 4279e75 d2
pick 116755e d3
pick 159f932 d1
```
该提交将重复提交第一次提交, 提交结果如下:
<img src="http://pgagp8vnu.bkt.clouddn.com/git_rebase7.png" height="350" width="450"/>
本地rebase之后再向远程推送, 可能会冲突, 这时确定没有人在基于那个分支分发的话, 可以通过git push --force origin mybranch分支。 master一般不允许直接这么弄

# cherry-pick
# revert

# merge

# reset

# checkout