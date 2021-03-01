---
title: git reset revert checkout的区别
date: 2019-08-18 16:58:40
tags:
- git
---

# git reset, revert, checkout


## [Git Reset](https://git-scm.com/docs/git-reset)

git reset [<mode>][<commit>]

将current branch head(HEAD) 指向 <commit>, git reset命令是用来将当前branch重置到另外一个commit的，而这个动作可能会将index以及work tree同样影响。

```
--soft  修改 HEAD, Index 和 Working tree 不变 
--mixed 修改 HEAD, 还原Index, Working tree 不变
--hard  修改 HEAD, 还原Index 和 Working Tree, <commit> 之后的修改都会被丢弃
```


* reset 回退之后后悔了怎么办？ 使用git reflog 查看reset 操作之前所在commit的id，使用reset --hard <commit_id> 即可撤销reset操作。

```
           HEAD (refers to branch 'master')
            |
            v
a---b---c  branch 'master' (refers to commit 'c')


$ git reset HEAD^

           HEAD (refers to branch 'master')
            |
            v
  branch 'master' (refers to commit 'c')
    |
    v
a---b---c

```


## [Git Revert](https://git-scm.com/docs/git-revert) 

撤销一次提交的修改，并创建一个新的提交来记录

* revert 是撤销一次提交，所以后面的commit id是你要撤销的commitID 不是撤销到该commit，比如Git revet HEAD 撤销最近的提交，Git reset HEAD 没有变化。
*  使用revert HEAD是撤销最近的一次提交，如果你最近一次提交是用revert命令产生的，那么你再执行一次，就相当于撤销了上次的撤销操作，换句话说，你连续执行两次revert HEAD命令，就跟没执行是一样的
*  使用revert HEAD~1 表示撤销最近第二2次提交，这个数字是从0开始的，如果你之前撤销过产生了commi id，那么也会计算在内的。


```
           HEAD (refers to branch 'master')
            |
            v
a---b---c  branch 'master' (refers to commit 'c')

$ git revert HEAD

                       HEAD (refers to commit 'd')
                        |
                        v
a---b---c---d  branch 'master' (refers to commit 'd')
```

## [Git Checkout]()

git checkout 

更新 Working tree 和 Index 或者 special tree 一致

git checkout <branch>  
切换分支 更新 HEAD Index, working tree, working tree 中的修改不变，可以提交到另一个分支。

提取某个提交中的某个文件： 
git checkout <commit> -- file

```
               HEAD (refers to branch 'master')
                |
                v
a---b---c---d  branch 'master' (refers to commit 'd')

$ git checkout master^^

   HEAD (refers to commit 'b')
    |
    v
a---b---c---d  branch 'master' (refers to commit 'd')

```

## 术语

### Reference
分支指向的commit 目录 .git/refs/heads 下的文件记录了分支指向的commi.
```
➜ git:(dev) ✗ cat .git/refs/heads/master
afb8dce836d3b04e0fa93dd11eb032c7b736db55
```

### HEAD
当前活跃分支的游标，你再哪里HEAD 就在哪里，HEAD 默认指向当前分支的最后一次提交，也通过checkout 命令可以指向一个commit。git reflog 显示的就是HEAD的历史。HEAD 的本质是一个指向reference 的指针
```
➜ git:(dev) ✗ cat .git/HEAD
ref: refs/heads/dev
```

### Index
也称为staging aera(暂存区) 是即将被下一次提交的文件集合。

### Working copy
当前工作和子目录中的文件集合
