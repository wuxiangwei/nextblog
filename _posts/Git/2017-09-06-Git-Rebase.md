---
layout: post
title: Git Rebase
date: 2017-09-06
author: wuxiangwei
categories: Git
tags: Git
---

<br>
`git rebase <upstream> [--onto <newbase>]`命令。    

rebase，顾名思义，修改分支的base，然后在新base上应用已经修改的commits。这涉及两个方面，一个是新base，一个是reapply哪些commits。命令中的newbase用来指定新base。需要重新apply的commits通过upstream来计算，HEAD和upstream间的commits即是需要重新apply的提交。具体步骤如下：    

> All changes made by commits in the current branch but that are not in <upstream> are saved to a temporary area. This is the same set of commits that would be shown by git log <upstream>..HEAD (or git log HEAD, if --root is specified).            

将所有仅存在于当前分支，在upstream中没有的commit保存到一个临时区域。这些commits可以通过`git log upstream..HEAD`查看。

> The current branch is reset to <upstream>, or <newbase> if the --onto option was supplied. This has the exact same effect as git reset --hard <upstream> (or <newbase>). ORIG_HEAD is set to point at the tip of the branch before the reset.        

把当前分支reset到newbase，效果等同于执行`git reset --hard newbase`命令。

> The commits that were previously saved into the temporary area are then reapplied to the current branch, one by one, in order. Note that any commits in HEAD which introduce the same textual changes as a commit in HEAD..<upstream> are omitted (i.e., a patch already accepted upstream with a different commit message or timestamp will be skipped).        

逐个将保持在临时区域内的commit重新apply到当前分支。


应用场景1：       
两个分支a和b，在分支a中做了修改c，欲将c同步到b中。    

1. 基于分支a创建新分支aa。执行`git branch aa a`命令；
2. 将分支aa reset到分支b，将修改c应用到分支aa。执行`git rebase a --onto b`命令。
3. 将分支aa提交到refs/for/b中评审，评审结束后直接merge到分支b，这部分工作由gerrit来完成。


应用场景2：   
本地代码库有commit没提交到远程版本库，远程版本库也有新的commit。此时，先拉取远程版本库再rebase。

1. 执行`git fetch origin`命令拉取远程版本，但不自动合并；
2. 执行`git rebase origin/master`将本地的commit重新apply到origin/master；

也可以直接`git pull origin`，但这会自动新增一个merge提交。在对提交的注释比较严谨的地方，使用fetch配合rebase的方法得到的结果会比较干净。




