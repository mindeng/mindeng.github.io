
---
title: "Git Tips"
date: 2022-12-03T10:18:00.000Z
lastmod: 2022-12-04T13:59:00.000Z
tags: ['tools', 'git']
draft: false
---



## 安全的强推 push —froce-with-lease

推荐使用更安全的 force push 命令：

```bash
git push --force-with-lease
```

该版本可以确保，不会覆盖其他人的提交。如果有尚未 fetch 的远程提交，该命令会提示并中止执行。


## 定位某个 commit 合入的版本

``git name-rev <commit-id>``


## 查看本地尚未 push 的提交

``git cherry -v``

指定要比较的远程分支：

``git cherry -v origin/somebranch``


## merge & rebase  
  
-   执行 ``git checkout --ours path`` （以及 ``git checkout --theirs path`` ）命令时，要格外注意，这里的 *ours* 和 *theirs* 容易搞混：  
      
    -   在做 *merge* 时：  
              
            -    *ours* 指当前的分支          
            -   *theirs* 指要合入的目标分支      
    -   在做 *rebase* 时：  
              
            -    *ours* 指 rebase 参数所指定的分支          
            -   *theirs* 指当前在做 rebase 的分支    
    
    原因解释：因为 *rebase* 是通过一系列的 *cherry-pick* 来实现的，是**把当前分支的 commit *****cherry-pick***** 到指定分支**。因此，在 *cherry-pick* 的过程中，指定的分支被视为 *ours*，而被 rebase 的当前分支被视为 *theirs* 。  
-   [``HEAD~``](https://stackoverflow.com/a/2222920/1066512)[ 和 ](https://stackoverflow.com/a/2222920/1066512)[``HEAD^``](https://stackoverflow.com/a/2222920/1066512)[ 的区别](https://stackoverflow.com/a/2222920/1066512)


## Tags

push tags: ``git push --tags``

rename tag:

```bash
git tag new old
git tag -d old

# old 前面的 `:` 会从远程仓库中删除该 tag
git push origin new :old
```


## Git 代理配置

 ``.gitconfig`` 配置：

```plain text
[user]
    email = username@gmail.com
[http]
    proxy = localhost:7777
```

``.gitconfig`` 文件可放在家目录下（影响当前用户），亦可放在单个 git 仓库下（仅影响当前仓库）。


## Git through ssh

```python
# git through ssh
git remote add origin ssh://user@my.git.svr/path/to/repo
```


## git filter-branch

// TODO


## Git branching model

![](/uploads/images/8cb29db0-8c81-41c8-a5ac-0e85e3cddca0/Untitled.png)

参考：[https://nvie.com/posts/a-successful-git-branching-model/](https://nvie.com/posts/a-successful-git-branching-model/) 