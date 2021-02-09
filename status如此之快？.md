---
layout: 为什么git
title: status如此之快？
date: 2021-02-08 19:06:08
tags:
---

## 为什么 git  status 如此之快？

### 先扯一下 git checkout ， reset 为什么如此之快
最近发现了一篇文章
[图解git原理的几个关键概念](https://tonybai.com/2020/04/07/illustrated-tale-of-git-internal-key-concepts/) 这篇文章将git的核心原理讲解得非常清楚

之前经常想在为什么在一个拥有上万个文件和上万个commit的项目中git的checkout速度还能如此快， 原因就是使用了默克尔树， 每一次commit 都会创建一个新的root节点,即每个commit都有一颗完整的默克尔树将所有文件组织起来（相当于一个**快照**）， 而这些默克尔树的大部分节点都是公用的，所以大大节省了空间。

所以每次checkout 操作只需要用对应的默克尔树将**快照**取出来，
所以branch和tag 只需要是一个指向commit 的指针。


#### 回到标题， 为什么一个拥有上万文件的项目执行git status 时Git能快速地将发生修改的文件找出来？

一开始我的猜测是遍历项目下的所有文件并计算哈希然后和最近的快照中保存的哈希进行比较。 但感觉如果计算哈希速度不可能如此快速。
通过一通搜索发现了这篇文章 
[Use of index and Racy Git problem
](https://mirrors.edge.kernel.org/pub/software/scm/git/docs/technical/racy-git.txt)

下面说一下我的大概理解

#### 先介绍一下git index (.git/index)
Git索引是一个在你的工作目录和项目仓库间的暂存区(staging area). 有了它, 你可以把许多内容的修改一起提交(commit). 如果你创建了一个提交(commit), 那么提交的是当前索引(index)里的内容, 而不是工作目录中的内容.

因此只要执行了git add，就算工作目录中的文件被误删除，也不会引起文件的丢失，因为文件已经被存入Git对象，并使用索引进行定位。

我们可以通过git ls-files --stage命令看到仓库中每一个文件及其所对应的文件对象，也可以直接通过查看二进制索引文件的方式来了解更多信息


Index 文件是用二进制存储的，包含有 ctime 和 mtime 时间信息，文件存储的设备信息，磁盘的inode信息，文件的 mode信息，UID，GID，文件大小，文件的SHA-1码，flag，文件的file path等信息。

#### 简要过程

当执行git status 时Git会启动多个线程，对工作区中所有的文件执行stat命令，得到文件[节点信息](https://www.cnblogs.com/cherishry/p/5885107.html)中的修改时间，文件大小等信息然后和index中保存的信息进行比较，如果一样则说明该文件没有发生修改。

这里又出现了一个问题，stat命令得到的时间戳仅仅精确到秒，如果文件修改的很快并且大小又不变岂不是会漏掉更新。为了解决这个问题需要增加以下逻辑：

如果该文件的修改时间小于index文件的的修改时间，则可以肯定改文件没有发生修改， 如果修改时间大于或者等于index文件的修改时间则需要找到object中保存的原始文件进行比较判断是否发生修改。

### 参考
[How does git detect that a file has been modified?
](https://stackoverflow.com/questions/1778862/how-does-git-detect-that-a-file-has-been-modified)

[Use of index and Racy Git problem
](https://mirrors.edge.kernel.org/pub/software/scm/git/docs/technical/racy-git.txt)

[How is it possible that 'git status' is so fast?](https://www.quora.com/How-is-it-possible-that-git-status-is-so-fast)
