---
layout: post
title: hexo和主题多设备同步
categories:
  - hexo
tags:
  - linux
  - hexo
  - git
  - subtree
photo: /img/hexo-themes-subtree/abstract.png
abbrlink: d00638d8
date: 2019-04-12 19:39:27
---

使用hexo可以生成静态网页部署到GitHub和VPS上搭建个人博客，但是hexo的部署都是在本地，如果换了一套环境如何也能够编辑发布自己的博客网站呢？
<!-- more --> 
## 同步方案

由于部署博客已经使用了github仓库托管网页代码，我们可以考虑使用这个来做hexo部署发布管理的版本控制，由于部署的网站默认使用了master分支，因此我们可以使用一个新的分支hexo或者新建一个仓库来管理。

下面步骤默认已经安装好了hexo并且已经成功部署网站，首先切换到hexo主目录，git init进行初始化，如果已经纳入git管理并且关联了远程仓库，可能需要删除重新关联。

```shell
# git 初始化
$ git init
# 查看关联的远程仓库
$ git remote
# 删除
$ git remote rm origin
# 设置新的远程仓库
$ git remote add origin git@github.com:Elietio/Elietio.github.io.git
# master分支会作为网站部署分支，因此我们切换一个新的分支hexo
$ git checkout -b hexo
# 推送远程
$ git add .
$ git commit -m ""
$ git push -u origin hexo
```

到这一步似乎已经大功告成了，然而，如果你使用了第三方主题，并且是直接git clone，你会发现一个问题，上传的themes/下面主题是空目录，因为git无法直接管理这样的嵌套模块，那么该怎么做呢？最暴力，直接取消主题模块的git管理，但是这样后续主题模块的更新就是一个问题了，所以不推荐，好在我们可以通过git的submodule或subtree来实现，对于git clone安装的其它主题和插件都可以按照这个思路。

## git submodule vs git subtree

简单来说，submodule 和 subtree 最大的区别是，submodule 保存的是子仓库的 link，而 subtree 保存的是子仓库的 copy。

### git submodule

child 目录被当做一个独立的 Git 仓库，所有的 Git 命令都可以在 child 目录以及上层项目下独立工作。尽管 child 是子目录，当你不在 child 目录时并不记录它的内容。而当你在那个子目录里修改并提交时，子项目会通知那里的 HEAD 已经发生变更并记录你当前正在工作的那个提交。而此时上层项目会显示 child 目录下的改动，将它记录成来自那个仓库的一个特殊的提交。

若他人要克隆该项目，会发现 child 目录为空。这时需要执行 git submodule init 来初始化你的本地配置文件，以及 git submodule update 拉取数据并切换到合适的提交。而后每次从主项目拉取子模块的变更时，由于主项目只更新了子模块提交的引用而没有更新子模块目录下的代码，必须执行 git submodule update 来更新子模块代码。

### git subtree

不同于 git submodule，此时的 child 仅仅是含有相关代码的普通目录，而不是一个独立的 Git 仓库。因此当在 child 进行修改时，上层项目会立刻记录其改动，而不是像之前那样先在子项目中提交才能进行记录。克隆上层仓库时 child 目录也不再为空。但同时，child 也不能再执行独立的 Git 命令，只有 git subtree 相关的操作。

进行操作前请先备份已有的next主题目录，根据不同情况操作中可能需要删除并且重新clone下来

```shell
首先在自己的github上fork一份next源码
git@github.com:Elietio/hexo-theme-next.git     

# 为hexo添加远程仓库 
# git remote add -f <子仓库名> <子仓库地址>
$ git remote add -f next git@github.com:Elietio/hexo-theme-next.git
# 添加subtree
# git subtree add --prefix=<子目录名> <子仓库名> <分支> squash意思是把subtree的改动合并成一次commit
$ git subtree add --prefix=themes/next next master --squash
# 更新子项目
$ git fetch next master
$ git subtree pull --prefix=themes/next next master --squash
整个项目的pull、push同样会对子项目起作用

而对next子项目进行pull、push操作需要使用subtree
# git subtree push --prefix=<子目录名> <远程分支名> 分支
$ git subtree push --prefix=themes/next next master  

# git subtree pull --prefix=<子目录名> <远程分支名> 分支
$ git subtree pull --prefix=themes/yilia yilia master --squash
```

## 其它设备同步

上面操作已经把hexo的源目录同步到Git，因此我们只需要clone，并且安装node.js和hexo环境

```shell
$ git clone -b hexo git@github.com:Elietio/Elietio.github.io.git
# 安装 hexo
$ npm install hexo(npm install hexo-cli -g)
# 安装依赖库
$ npm install 
# 安装部署相关配置
$ npm install hexo-deployer-git
```
