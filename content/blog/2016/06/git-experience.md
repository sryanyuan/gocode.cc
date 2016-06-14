+++
author = ""
categories = ["git"]
date = "2016-06-14T11:52:40+08:00"
description = ""
linktitle = ""
title = "Git使用心得"
type = "post"

+++

在刚工作的时候，肯定要接触到版本控制系统，版本控制系统对于大型项目的开发还是很重要的。

毕业工作后，第一个接触的版本控制系统就是svn，这个使用也的确简单，不过用的是图形化的软件，只是点点什么的，使用起来很方便。由于svn的限制，不能本地离线的commit。

后来移动项目逐步迁移到了git上，那时候接触git。从svn转到git还是非常不适应的，变化很大，而且感觉git很不好用，那时候的想法是比svn差多了。于是开始用sourcetree的图形化软件来操作git。那时候一个人负责一个项目，于是平时也就是add，commit，push之类的，使用起来也没碰到什么问题。

换了新环境后，还是用的git，而且多人负责一个项目，conflict的场景就出现了。于是那时候查了很多资料，觉得git是个好东西，只是入门有点儿痛苦，于是那时候干脆放弃sourcetree，开始全部用命令行版本来进行操作，现在想来是个明确的决定。

用熟悉了git后，感觉git真是个好东西，还能支持本地仓库的版本管理，于是自己开发的项目全部用这个来管理了，再也不用复制粘贴，每次更新了后怕出问题找不到问题。

下面我就按我熟悉命令的时间段来以我个人的观点来记录下常用的git命令吧，当然肯定不会非常正确，而且使用场景也只是简单的场景，复杂的参数我这儿也不涉及，只是涉及到常用的一些参数，只是我的想法而已，欢迎拍砖。

## 工作区 暂存区 本地仓库

工作区就是我们操作修改的文件，可见

暂存区就是将修改提交的某个临时区域

本地仓库就是我们最终commit到的地方，这个仓库可以和远程仓库进行提交或者拉取

## git clone

克隆一个远程的git仓库到本地，并且创建本地的仓库与远程仓库进行关联，简单的说就是clone后，你就可以在commit到本地的仓库并且push到远程仓库了。

## git branch

查看当前git仓库的分支，列一下常用的一些参数

* git branch -a 常看所有的分支，包括本地和远程的
* git branch -vv 可以查看本地分支是否关联到了远程分支
* git branch 无参数，查看本地分支
* git branch -d <branch-name> 删除某个分支

## git checkout

这是一个很常用的命令，对于不同的参数也有不同的处理方法，比较复杂，但是也非常的常用。

* git checkout <branch-name> 

    可以在不同的分支之间进行切换，参数跟的是分支名

* git checkout <file-name>

    可以恢复文件到工作区中。这个恢复是从暂存区中恢复的，也就是你提交了某个文件到了暂存区，然后又对这个文件进行了修改，然后进行checkout操作，那么你将会回滚到暂存区中的文件内容，而你没有存到暂存区中，checkout则直接恢复到本地仓库中的内容。当然你假如希望跳过暂存区直接恢复到原始内容，请使用git checkout HEAD <file-name>

* git checkout -b <new-branch-name> <remote-branch>
   
    创建一个新的分支，并且将新的分支和远程分支进行跟踪关联。当你忽略remote-branch的时候，则是从当前分支拷贝一个副本作为分支，不跟踪任何远程分支。
    
    这个使用也很多，开发某个版本的时候，创建一个本地分支来进行开发，然后将这个本地分支merge到当前的分支下，再进行提交，这个是常用的开发步骤。

## git rm

* git rm <file-name>

    删除一个版本跟踪的文件，并且提交到暂存区

## git reset

* git reset <file-name>

    丢弃暂存区中的某个文件，不会影响工作区的内容

## git status

* git status

    查看当前git仓库的状态，修改的文件，新增的文件，版本是否最新等等

## git commit

* git commit

    提交到本地分支上

## git push

* git push

    提交到远程git仓库上，默认是当前分支跟踪的远程分支，可以指定远程分支

## git pull

* git pull

    拉取最新远程仓库的内容到本地仓库中，会影响本地仓库，工作区，并自动进行合并，有可能有冲突，无法自动解决冲突的话，必须手动解决并且提交。

## git merge

* git merge <branch-name>

    将branch-name的分支合并到当前的分支上

## git fetch

* git fetch <repository-name> <branch-name> : <local-branch-name>

    这是一个比较复杂的命令，这里有几个基本的用法。

    git fetch origin dev

    拉取origin dev的最新的内容到本地，应该是作为分支存在的，但是用branch命令是无法查看到的，但是可以通过origin/dev来访问。于是我们在执行了这个命令后，我们可以使用

    git diff origin/dev来查看哪些文件被修改了，确定好后，我们就可以使用 git mergy origin/dev来把最新的分支合并到当前分支上了

    git fetch origin dev:temp

    这个比较好理解，就是上面的origin/dev分支变成了temp分支，就是远程最新的内容了，我们就可以执行合并什么的操作，然后可以branch -d来把这个分支给删除了。

    总结一下就是不加local branch就是生成了一个系统的分支，隐形的（待考证），我们可以将这个当做分支来进行操作

    git fetch + git merge = git pull
    