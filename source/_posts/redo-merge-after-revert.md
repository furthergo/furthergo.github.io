---
title: '如何再次合并已经被revert的合并？'
tags:
  - git
categories:
  - How&Why
date: 2020-10-25 00:29:34
---

# What Is The Problem
初始化git，创建master分支
1. master分支： commit `init"`
    ```
    master:  ---init---
    ```
2. 创建feature分支：提交commit `"feat 2"`和`"feat 3"`
    ```
    master:  ---init---
    feature: ---init---feat 2---feat 3---
    ```
3. 切到master分支合并feature分支
    ```
    master:  ---init---feat 2---feat 3---
    feature: ---init---feat 2---feat 3---
    ```
<!-- more -->
4. master分支发现问题，在master上revert这次merge的`"feat 2"`和`"feat 3`
    ```
    master:  ---init---feat 2---feat 3---revert `feat 2`---revert `feat 3`---
    feature: ---init---feat 2---feat 3---
    ```
5. 切到feature分支，修复问题提交commit`"fix 4"`
    ```
    master:  ---init---feat 2---feat 3---revert `feat 2`---revert `feat 3`---
    feature: ---init---feat 2---feat 3---fix 4---
    ```
6. 切到master分支再次合并feature分支
    ```
    master:  ---init---feat 2---feat 3---revert `feat 2`---revert `feat 3`---fix 4---
    feature: ---init---feat 2---feat 3---fix 4---
    ```

问题：**此时master分支实际只包含了commit`"init"`和`"fix 4"`的改动，而不包含`"feat 2"`和`"feat 3`的改动**

# Why
Linus的解释：
>Reverting a regular commit just effectively undoes what that commit did, and is fairly straightforward. But reverting a merge commit also undoes the _data_ that the commit changed, but it does absolutely nothing to the effects on _history_ that the merge had.

>So the merge will still exist, and it will still be seen as joining the two branches together, and future merges will see that merge as the last shared state - and the revert that reverted the merge brought in will not affect that at all.

>So a "revert" undoes the data changes, but it's very much _not_ an "undo" in the sense that it doesn't undo the effects of a commit on the repository history.

>So if you think of "revert" as "undo", then you're going to always miss this part of reverts. Yes, it undoes the data, but no, it doesn't undo history.



1. revert一个常规commit，仅仅是撤销这个commit做的改动
2. revert一个merged commit，撤销这个commit的改动，同时不会对master分支产生任何对这个merge状态的影响
3. 意味着在master分支上merge状态仍然存在
4. feature修复后再发起merge时，会看到master分支上存在merged状态
5. 因此只会带上merge状态之后的改动，即`"fix commit on feature"`，之前被revert的改动不会再被merge到master分支上

# How To Fix
1. 方案一：**在master上复原feature的改动**，即修复后，不依赖feature将原commit带到master分支
    * 在master分支上revert掉revert的commit，再把feature分支的修复合到master
2. 方案二：从feature将原commit的内容带到master，需要**产生和原merge不同的状态（不同的commit SHA ID）**
    * 在feature分支reset+commit，产生新的commits：1‘和2’，修复后再merge到master
    * 在feature使用`--no-ff`option rebase到feat commits之前的commit，产生1‘和2’commit: `git rebase --no-ff "init"(parent )`

关于`--no-ff`:
>Individually replay all rebased commits instead of fast-forwarding over the unchanged ones. This ensures that the entire history of the rebased branch is composed of new commits.

>You may find this helpful after reverting a topic branch merge, as this option recreates the topic branch with fresh commits so it can be remerged successfully without needing to "revert the reversion" (see the revert-a-faulty-merge How-To[1] for details).

# 参考资料

- [1] [how to revert-a-faulty-merge](https://github.com/git/git/blob/master/Documentation/howto/revert-a-faulty-merge.txt)