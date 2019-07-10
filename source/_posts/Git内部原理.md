---
title: Git内部原理
date: 2019-02-24 15:29:54
tags:
    - 基础知识
categories:
    - 基础知识
---

# git基本原理
常见的git工作流如下, 来分解下看常见的工作流背后的逻辑.

1. `git hash-object -w ${要保存的文件}` => 生成要保存的文件(Git对象)的HASH值
2. `git update-index --add --cacheinfo 100644 ${Git对象的HASH值} ${要保存的文件}` => 向暂存区保存文件
3. `git write-tree` => 生成目录对象(Tree对象)的HASH值
4. `echo "first commit" | git commit-tree ${目录对象(Tree对象)的HASH值}` => 包含元数据和目录树的对象HASH值(最终的commit对象)
5. `echo ${commit对象的HASH值} > .git/refs/heads/${要commit的分支名称}` 表示指定分支指针指向该快照.

## git init
在项目根目录下创建一个.git子目录, 用来保存版本信息. objects 目录存储所有数据内容，refs 目录存储指向数据 (分支) 的提交对象的指针，HEAD 文件指向当前分支，index 文件保存了暂存区域信息

## git add .
该命令实际可以拆分为`保存对象`和`更新暂存区`两部分操作.

### 保存文件为二进制文件(Git对象)
1. `git hash-object -w test.txt`命令实际分为两步: 第一步把test.txt文件压缩为二进制文件(Git对象), 保存在.git/objects目录下. 第二步计算test.txt的HASH值, 作为二进制文件的名称. 最终输出 **Git对象的HASH值**

### 二进制文件存放到"暂存区"
文件保存成二进制对象以后, 需要通知 Git 哪些文件发生了变动. 所有变动的文件, Git 都记录在"暂存区".
1. `git update-index --add --cacheinfo 100644 ${Git对象的HASH值} test.txt`. 向暂存区写入文件名test.txt, 二进制对象名（哈希值）和[https://stackoverflow.com/questions/737673/how-to-read-the-mode-field-of-git-ls-trees-output](文件权限)(100代表regular file，644代表文件权限.).
2. `git ls-files --stage`. 用来查看暂存区的文件.

## git status
`git status`实际就是检查暂存区的内容.

## git commit
仅仅记录单个文件的变化是不够的, 我们还需要记录文件之间的目录关系.
1. `git write-tree`也是分两步. 第一步从当前索引创建一个树形对象, 保存为二进制文件(Git对象), 保存在.git/objects目录下. 第二步计算HASH值, 作为二进制文件的名称. 最终输出**Tree对象的HASH值**
2. `echo "commit说明" | git commit-tree ${Tree对象的HASH值}`. 该命令实际也是分两步. 第一步给出commit的备注. 第二部将元数据和目录树一起生成一个Git对象. 最终输出**Commit对象的HASH值**
3. 此时使用`git log`是看不到内容的, 因为没有branch信息.
4. `echo ${Commit对象的HASH值} > .git/refs/heads/master` 表示master指针指向该快照.

## branch
分支（branch）就是指向某个快照的指针, 分支名就是指针名, 但是HASH值不便于记忆, 所以我们概念中的branch就是分支名的别名. 分支还会自动更新, 如果当前分支有新的快照, 指针就会自动指向它. 比如, master 分支就是有一个叫做 master 指针, 它指向的快照就是 master 分支的当前快照.
Git 有一个特殊指针HEAD, 总是指向当前分支的最近一次快照. 另外, Git 还提供简写方式, `HEAD^`指向 HEAD的前一个快照(父节点). `HEAD~6`则是HEAD之前的第6个快照.
每一个分支指针都是一个文本文件, 保存在.git/refs/heads/目录, 该文件的内容就是它所指向的快照的二进制对象名(哈希值).

## tag
**Tag对象**是一种对象, Tag 对象指向一个 commit 而不是一个 tree. 它就像是一个分支引用, 但是不会变化——永远指向同一个 commit, 仅仅是提供一个更加友好的名字.

## 注意
1. `git add -all(git add .)`相当于对目录下的所有文件遍历并且做了上面的操作.
2. `git cat-file -p ${Git对象的HASH值}`用来查看二进制文件的原始内容.
3. 每次生成Git对象, 实际是对有变动的文件做的全量的快照(当前的目录结构, 以及每个文件对应的二进制对象).
4. 通过`git config user.name ${用户名}`, `git config user.email ${Email 地址}`来设置是谁提交的记录.
5. `git log --stat ${Git对象的HASH值}`, 也可以查看快照在当前分支上的变动信息.
6. `git checkout ${Git对象的HASH值}`, 用于切换到某个快照.
7. `git show ${Git对象的HASH值}`, 用于展示某个快照的所有代码变动.
8. `git commit-tree ${目录结构Git对象的HASH} -p ${基于本次结构Git对象的HASH值}`. -p参数用来指定父节点, 也就是本次快照所基于的快照.
9. 100  644, 表明这是一个普通文件. 其他可用的模式有：100 755 表示可执行文件, 120 000 表示符号链接. 文件模式是从常规的 UNIX 文件模式中参考来的.
10. Git的对象存储大致如下, 是通过Ruby脚本来实现的
    ```
    content = "what is up, doc?"
    header = "blob(数据对象) #{content.length}\0"
    
    store = header + content
    sha1 = Digest::SHA1.hexdigest(store)
    zlib_content = Zlib::Deflate.deflate(store)
    将zlib_content写入磁盘, 命名为sha1
    ```

------
参考资料:

[http://wiki.jikexueyuan.com/project/pro-git/git-internals.html](1)
[http://www.ruanyifeng.com/blog/2018/10/git-internals.html](2)
[https://medium.com/@shalithasuranga/how-does-git-work-internally-7c36dcb1f2cf](3)



