---
title: git revert merge
date: 2019-06-29 20:22:49
tags:
- git
---

# git revert merge 

合并代码的时候出了一些问题，想要将这次合并revert调， 结果出错了

    error: commit 9002453bd778790d03022d00b271f71fdb146b76 is a merge but no -m option was given.
    fatal: revert failed
    
-m 是什么鬼，查看一下帮助文档

     -m parent-number, --mainline parent-number
            Usually you cannot revert a merge because you do not know which side of the merge should be considered the mainline. This option specifies the parent number (starting from 1) of the mainline and
           allows revert to reverse the change relative to the specified parent.

           Reverting a merge commit declares that you will never want the tree changes brought in by the merge. As a result, later merges will only bring in tree changes introduced by commits that are not
           ancestors of the previously reverted merge. This may or may not be what you want.

           See the revert-a-faulty-merge How-To[1] for more details.

原因是一次merge提交会有两个parent,我们需要指定一个parent作为主线,那么如何指定一条主线呢？如果你在A分支上合并B分支到A分支，那么1 就代表A分支，2 就代表B分支。在分支A上执行 git revert <commitID> -m 1 就将从B分支上和并过来的提交给删除了。

或者通过git log 查看一下，Merge后面第一个commit ID就是1，第二个commit ID就是2

    commit 9002453bd778790d03022d00b271f71fdb146b76 (HEAD -> master)
    Merge: 7df7c3c 4b6f9c2
    Author: yangjh <yangjh@ibaodashi.com>
    Date:   Sat Jun 15 16:21:00 2019 +0800
    
    
现在根据需求我只需要 git revert 900245 -m 1  就可以了。
