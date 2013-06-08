---
layout: post
title: "使用Git Submodule"
date: 2013-06-08 17:12
comments: true
categories: [Git, Tips]
---

最近clone的项目里面发现使用了Git的Submodule功能，所以在这儿记录一下基本的使用方法

### 添加submodule ###

    git submodule add git@github.com:xxx/submoule.git src/submodule

在项目中就会自动生成一个.gitmodules文件来保存submodule的关系，再提交到远程库就ok了

### clone项目 ###

在clone带有submodule的项目，直接使用`--recursive`就可以递归地将所有submodule，包括submodule中的submodule都一次性clone到本地

    git clone --recursive git@github.com:xxx/xxx.git

不然就得使用如下的命令组合来完成初次clone，这样还不能clone到submodule中的submodule

    git clone git@github.com:xxx/xxx.git
    git submodule init    # 将.gitmodules中的内容注册到.git/config中
    git submodule update  # clone submodule并且checkout指定commit

### 更新项目 ###

如果你不会到submodule中进行开发，那么每次submodule有了更新只需要再update一次就行了

    git pull origin master
    git submodule update --recursive # 必须在项目顶层目录执行此命令，也可以添加--init保证init执行

### 更新submodule ###

submodule其实也就是一个单独的Git repo，只是会在项目顶层目录的.gitmodules目录中记录下路径和submodule源的映射关系

    # .gitmodules的内容
    [submodule "src/cloud_controller_ng"]
        path = src/cloud_controller_ng
        url = https://github.com/cloudfoundry/cloud_controller_ng.git

但是这个文件中并没有指定submodule的commit id，git怎么知道该用submodule的哪个commit呢？

在顶层项目的`.git/modules`里会保存submodule的git配置，同时也就知道了使用的是submodule的哪个commit
而submodule的.git则只是记录了一下路径

    # submodule目录中的.git的内容
    gitdir: ../.git/modules/dea

所以在submodule中进行commit或者push就和普通repo完全一样，只是最后需要在外层repo中更新一下

现在假设已经在submodule目录或者单独的submodule项目目录中完成了`add`、`commit`和`push`操作，或者执行了`git pull`获取了新的源码
这时需要回到外层项目，再次更新一遍

    git status  # 会提示submodule有new commit
    git add .   # 添加更改
    git commit -m "update submoule"
    git push

在另一个项目repo中执行以下命令，就可以将submodule的commit同步了

    git pull
    git submodule update --recursive  # 可以加上--init

### 其他 ###

    git submoule foreach 'echo $path `git rev-parse HEAD`'  # foreach 可以对每个submodule执行shell命令

    git help submodule   # git帮助文档
