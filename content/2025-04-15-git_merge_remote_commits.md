---
title: 如何将远端git仓库的多个commit合并成一个
draft: false
tags:
  - git
---

由于最近尝试给远端仓库提交代码，然而由于多次修改导致多个`commit`，显然不是很整洁，因此在此学习一下如何通过命令行合并远端仓库的`commit`。

# 流程
1. 首先需要查看远端有哪些`commit`，具体命令为`git log --oneline`，这样能看到形如以下的远端`commit`记录
```bash
ca707db (HEAD -> add-binary-search-to-next, origin/add-binary-search-to-next) fix lint
0d23379 [sst iterator optimize] change sstIterator::seek to binary search
5f9c3c0 [sst iterator optimize] change sstIterator::seet to binary search and fix lint
c6ebb06 [sst iterator optimize] change sstIterator::seet to binary search
33faf60 [sst iterator optimize] change sstIterator::seet to binary search
4be314e [sst iterator optimize] change sstIterator::seet to binary search
```

这样可以发现当前已经提交的`commit`记录，考虑如何合并。

2. 使用`git rebase -i 4be314e`，意思是对于某一个`commit(4be314e)`及之后的`commit`，开始进行交互式`rebase`，大概会出现形如以下的记录，假设提交记录如下所示
```bash
A - B - C - D (HEAD)
```
那么执行完命令会出现如下
```bash
pick abc1234 Commit B
pick def5678 Commit C
pick ghi9012 Commit D
```
如果想要将`BCD`都合并，那么改成如下
```bash
pick abc1234 Commit B
squash def5678 Commit C
squash ghi9012 Commit D
```
其中`pick`代表了保留，`squash`代表了和上一个`commit`合并。

3. 合并完之后，考虑提交到远端，这里用一个比较暴力的方式`git push -f origin remote-branch`，直接强制推送到远端，可以发现远端的提交被压缩了。