# Foundation

git 其实就是一个键值对存储。

```js
{
    key: value
    // value is the data
    // key is hash of the data
}
```

## key: SHA1

- 加密函数
- 输入时一段数据，输出是一个40位的16进制数据 （git log hash）
- 相同的输入总能得到相同的输出

## value: blob

- 标识符（string）：`blob`
- 内容的长度
- 结束标志符（相当于C语言的`\0`）
- 内容

![](http://ov532c17r.bkt.clouddn.com/ewtzrgrqn6p.png)

## 手动获取SHA1的结果

```sh
echo 'Hello World' | git hash-object --stdin
557db03de997c86a4a028e1ebd3a1ceb225be238
```

```sh
echo 'blob 12\0Hello World' | openssl sha1
557db03de997c86a4a028e1ebd3a1ceb225be238
```

可以看到git就是把按照上述blob格式封装过的数据用sha1计算了一下

## 这些数据存哪儿了

这些数据被存储在`git object`中。`git object`有三种类型——blob， tree， commit。

```hs
echo 'Hello World' | git hash-object -w --stdin
```

👆这条命令把sha1的结果写到文件中存储起来。

查看`.git`下的目录结构：

![](http://ov532c17r.bkt.clouddn.com/jt97m3cyizj.png)

可以看到data的存储位置。

存储内容的git-object是blob。

## 文件名和目录结构

objects中存储的是文件内容（blob），而文件名和目录结构是通过tree来存储的

### tree

指针

- 指向blob
- 指向其他tree

metadata

- 指针类型（`blob` or `tree` ）
- 文件名或者目录名
- 模式（可执行文件，链接) 

![](http://ov532c17r.bkt.clouddn.com/j12yfyrc4.png)

在图中所示的例子中，`hello.txt`和`hello-copy.txt`内容是一样的。sha对于相同的输入总是得到相同的输出（blob），通过看它们的hash可以看出来。为了节省空间，而**在git中所有相同的blob只会存储一份**。所以在`copies`目录指向的tree中，其指向的blob其实跟根目录所在tree指向的blob是同一个。

## commit

一个commit包含如下信息

- 指向一个tree
- 包含一些metadata
  - 作者
  - 日期
  - message
  - parent commit (one or more)

一个commit的SHA1是以上所有信息的总和

![](http://ov532c17r.bkt.clouddn.com/9ebn4lmgg9p.png)

![](http://ov532c17r.bkt.clouddn.com/wfvns79ukia.png)

通过`git cat-file -t <sha1>`可以得到一个git object的类型，通过`git cat-file -p <sha1>`可以得到一个git object的内容。

### 先提交一个commit

- 增加一个文件hello.txt， 内容是`Hello World`

- ```sh
  git add .
  git commit -m "initial commit"
  tree .git
  ```

![](http://ov532c17r.bkt.clouddn.com/kbigu79gy9.png)

```sh
git cat-file -t ae4b5c1e8c3dcd8de5f7dad993c66c7648888ec9

commit
```

```sh
git cat-file -p ae4b5c1e8c3dcd8de5f7dad993c66c7648888ec9

tree 97b49d4c943e3715fe30f141cc6f27a8548cee0e
author Demon Monk <antdude@126.com> 1534586551 +0800
committer Demon Monk <antdude@126.com> 1534586551 +0800

initial commit
```

```sh
git cat-file -t 97b49d4c943e3715fe30f141cc6f27a8548cee0e

tree
```

```sh
git cat-file -p 97b49d4c943e3715fe30f141cc6f27a8548cee0e

100644 blob 557db03de997c86a4a028e1ebd3a1ceb225be238	hello.txt
```

```sh
git cat-file -t 557db03de997c86a4a028e1ebd3a1ceb225be238

blob
```

```sh
git cat-file -p 557db03de997c86a4a028e1ebd3a1ceb225be238

Hello World
```

# Areas

- Working: untracked by git
- Staging: tracked by git but not in the repo
- Repository: already in the git

## staging area

git用来指导当前commit和上个commit有啥区别。“clean” staging area并不是空的，而是上次commit的拷贝。

查看staging area的内容

```sh
git ls-files -s
```

### git add -p

这个命令用于将你工作区内的部分代码提交到staging区域。可以作为分隔commit的主要方法。

执行这个命令会提示你是否要将当前hunk提交到staging。如果你不确定应该接下来用哪个命令，就输入？，会有提示的。

![](http://ov532c17r.bkt.clouddn.com/vrozfqm5hrr.png)

### git stash

```sh
git stash 
# save stash with a recognized name
git stash save "msg"
# like git add -p to stash partial working files
git stash -p
```

# Reference

## Reference：指向commit （Or reference（HEAD））的指针 

- tag
- branch:
- HEAD: 指向当前branch（这是一个reference）（通常情况）或者当前commit（见Head-less / fDetached Head）

> 为啥git在切换分支的时候辣么快？因为只是把头指针更改到了对应的branch而已。

当有新的commit时，branch和HEAD自动指向新的commit

### reference存储位置

![](http://ov532c17r.bkt.clouddn.com/sd6lcxasknb.png)

![](http://ov532c17r.bkt.clouddn.com/jjq2lauk6s.png)

![](http://ov532c17r.bkt.clouddn.com/n7ybpkivos.png)

## tag

- tag也是一个指向commit的指针
- 当你在创建tag时，没有参数，自动使用HEAD中的值。即当你以后要回到这个tag时，你看到的代码应该跟现在HEAD指向的snapshot是一样的

```sh
# 列出当前仓库所有的tag
git tag
# 列出所有tag，并显示出其对应的commit
git show-ref --tags
# 列出所有指向某个commit的tag
git tag --points-at <commit>
```

![mage-20180819001218](/var/folders/kb/_ptsg_tj3_9gqyz8fv0mzlzw0000gn/T/abnerworks.Typora/image-201808190012188.png)

### annotated tag

```sh
# 添加一个tag，msg不是必须要的
git tag -a <tagName> -m "msg about this tag"
# 显示某个tag的所有信息
git show <tagName>
```

## tag和branch的区别

- 创建一个新的branch后，当有新的commit时，branch会自动指向新的commit


- 创建一个新的tag，tag的指向就不动了。他是一个仓库的snapshot。（感觉跟commit的作用是一样的，只不过是它指向了一个固定的commit，而且可以附加一些commit没有的信息，姑且算是一个大的commit，milestone之类的效果）

## Head-less / Detached Head

- **发生**：当你checkout一个特定的commit或者tag时，git就会把HEAD指向这个特定的commit（与上面所述的通常情况作区分）

  ![](http://ov532c17r.bkt.clouddn.com/79vhgfbxf37.png)

- **后果**：当你在这种情况下提交新的commit，没有branch指向这个新的commit。也就是说当你切到其他分支再回来时，这个commit就丢失了。

  ![](http://ov532c17r.bkt.clouddn.com/qh69vf7fnvm.png)

- **怎么办**：通过创建新的分支，并使新的分支指向最后提交的commit

  ```sh
  git branch <new-branch-name> <last-commit>
  ```



# Merge & Rebase

## merge commit

拥有两个或以上的parent commits的commit。

![](http://ov532c17r.bkt.clouddn.com/j12xjqhmnx.png)

### Fast Forward

在当前分支合并目标分支时，当前分支并没有做另外的改变。即目标分支完全是基于当前分支已有的代码的基础上进行commit的。这样在合并的时候，直接把当前分支指向目标分支指向的commit就行了，合并的时候并不会产生merge commit。这种行为被称为fast-forward。

![](http://ov532c17r.bkt.clouddn.com/5ozc47b7g3i.png)

![](http://ov532c17r.bkt.clouddn.com/bsud4vnre0n.png)

这种行为可能导致git历史所述的分支不清楚。以至于搞不清除bug是由哪个分支引入的。这时候我们需要禁掉这种特性：

```sh
git merge --no-ff
```

> 可以使用`git log --graph`查看git历史，更清楚地看到FF和no-ff的区别

## merge conflict

### git rerere

`rerere`是git的一个隐藏功能。它打开后，会记录解决冲突的过程。当你下次再遇到这个冲突时，会自动按照这次的方法解决冲突。 

#### 打开

```sh
# use --global flag to enanle for all projects
git config rerere.enabled true
```

# git history

## git log

```sh
# 通过正则匹配commit msg的形式过滤git log
git log -grep <regexp>
# mix with other filter flags
git log --grep=mail --author=nina --since=2.weeks
```

## hat & tilde

### ^：用于指向某个父级commit

- ^: 没有参数，代表第一个父级commit
- ^n: 代表指向nth 父级commit

### ~: 用于表示回溯多少个父级commit，只沿第一个父级commit回溯

- ~：没有参数代表回溯一级
- ~n：代表回溯n级

![](http://ov532c17r.bkt.clouddn.com/5lrxf1u6ygr.png)

## git show

```Sh
# show commit and contents
git show <commit>
```

![mage-20180819124925](/var/folders/kb/_ptsg_tj3_9gqyz8fv0mzlzw0000gn/T/abnerworks.Typora/image-201808191249255.png)

```sh
# show files changed in commit
git show <commit> --stat
```

![mage-20180819125013](/var/folders/kb/_ptsg_tj3_9gqyz8fv0mzlzw0000gn/T/abnerworks.Typora/image-201808191250131.png)

```sh
# look at a file from another commit
git show <commit>:<file>
```

![](http://ov532c17r.bkt.clouddn.com/6g7xp9l4lsl.png)

## git diff

```sh
# unstaged changes
git diff
# staged changes
git diff --staged
# 相对branch A来说， branch B有哪些改变
git diff <branchA> <branchB>
# or
git diff <branchA>..<branchB>
```

## diff branches

```sh
# 列出已经合进master的分支，可以被删除
git branch --merged master
# otherwise
git branch --no-merged master
```

# Fixing Mistakes

## git checkout

### checkout a branch

- HEAD指向新的branch
- 从新branch的最新commit中取出文件内容拷贝到staging area
- 从staging area拷贝到working area

![](http://ov532c17r.bkt.clouddn.com/pboims74lwr.png)

### checkout — file/path 

- 从staging area拷贝到working area中

![](http://ov532c17r.bkt.clouddn.com/3urtpnz4cwa.png)

> `—`是为了防止文件和分支重名

### checkout <commit> — file/path

- 从repo中取出文件内容拷贝到staging area中
- 从staging area中取出文件内容拷贝到working area中

![](http://ov532c17r.bkt.clouddn.com/b0jo36gy2wv.png)

## git clean

deleting untracked files

```sh
# see what files would be deleted
git clean --dry-run
# see what files and directories would be deleted
git clean -d --dry-run
```

![](http://ov532c17r.bkt.clouddn.com/y959b5e8f6.png)

```sh
# delete untracked files
git clean -f
```

![](http://ov532c17r.bkt.clouddn.com/9ws2zziv4g.png)

```sh
#delete untracked directories and files
git clean -fd
```

![mage-20180819141824](/var/folders/kb/_ptsg_tj3_9gqyz8fv0mzlzw0000gn/T/abnerworks.Typora/image-201808191418248.png)

## git reset

### git reset —soft 

仅仅将HEAD指向参数指定的commit

![](http://ov532c17r.bkt.clouddn.com/o1bav5vrin.png)

### git reset —mixed <default>

- 将HEAD指向参数指定的commit
- 并将对应commit的内容拷贝到staging area

![](http://ov532c17r.bkt.clouddn.com/0rwfzxfj1zho.png)

### git reset —hard

- 将HEAD指向参数指定的commit
- 将对应commit的内容拷贝到staging area
- 将staging area的内容拷贝到working area

![](http://ov532c17r.bkt.clouddn.com/91r6geo67t.png)

### git reset <commit>

1. 将HEAD和当前分支指向这个commit
2. 将对应内容拷贝到staging area
3. 将对应内容拷贝到working area

```sh
--soft (1)
--mixed (1) & (2)
--hard (1) & (2) & (3)
```

### git reset 会修改git history

通过git reset使HEAD从A到B，然后再进行新的commit，那么git history就变了。公共仓库已经有的history不能被改变。

### 恢复

git会保留HEAD的上一个版本叫`ORIG_HEAD`，所以`git reset`后，可以通过如下命令恢复

```sh
git reset ORIG_HEAD
```

### git reset <commit> —file

只修改staging area中的内容，不移动HEAD，不带`--soft --mixed --hard `

![](http://ov532c17r.bkt.clouddn.com/4uo857jegyj.png)

## git revert

创建一个新的commit，这个commit是和指定commit相反的操作。原来的commit依旧还在，所以不会修改git history。`git revert`用来撤销已经推入公共仓库的commit。

# 修改已有的commit

## Amend

commit之后发现有些东西没有提交上去，可以用

```sh
git commit --amend
```



![](http://ov532c17r.bkt.clouddn.com/vmskyi1wsff.png)

![](http://ov532c17r.bkt.clouddn.com/3mvnj6ypm0k.png)

  ## rebase：give a commit a new parent

当我们当前分支和master分支diverged（合并时不能fast forward）。我们希望在合并的时候不要产生看起来很混乱的merge节点。这时候正确的做法是

- 从master上获取最新的内容
- 使当前的分支以master的最新commit作为父节点

![](http://ov532c17r.bkt.clouddn.com/mcdfqo2605.png)

![](http://ov532c17r.bkt.clouddn.com/m4y1y1l0isq.png)

>git rebase只能对于还没有shared的分支使用（没有人基于你分支上独有的commit进行开发）。

### [git rebase -i (interaction)](https://robots.thoughtbot.com/git-interactive-rebase-squash-amend-rewriting-history)

![ ](http://ov532c17r.bkt.clouddn.com/fad0b1dlt6i.png)

#### split commits

![](http://ov532c17r.bkt.clouddn.com/8qyfhd17rmd.png)

 ### git rebase —abort  

在rebase中间的任何阶段都可以通过abort放弃这次rebase

### 当rebase用得不6时，rebase之前先切一个新的分支作备份

