---
layout: post
title: "在Windows7的Git环境中使用WinMerge作为difftool"
date: 2012-12-11 14:55
comments: true
categories: [Git, WinMerge]
---

在Git使用WinMerge作为difftool，使用Git的环境不同需要不一样的配置，而且两种配置不通用。

### 使用CMD

#### 添加配置
	
	git config --global diff.tool winmerge
	git config --global difftool.winmerge.cmd "C:/git-difftool.bat \"$LOCAL\" \"$REMOTE\""
	git config --global difftool.prompt false

也可以手动在.gitconfig文件中添加如下配置

	[diff]
		tool = winmerge
	[difftool "winmerge"]
		cmd = C:/git-difftool.bat \"$LOCAL\" \"$REMOTE\"
	[difftool]
		prompt = false

#### 在指定位置创建git-difftool.bat

#### 在git-difftool.bat中添加一行
	
	"C:/Program Files/WinMerge/WinMergeU.exe" -e -ub -dl "Base" -dr "Mine" "$1" "$2"

### 使用Git Bash

步骤和之前的一样，只是需要注意**转义符**和**路径**的写法。

#### 添加配置
	
	git config --global diff.tool winmerge
	git config --global difftool.winmerge.cmd "/C/git-difftool.bat \"\$LOCAL\" \"\$REMOTE\" "
	git config --global difftool.prompt false

也可以手动在.gitconfig文件中添加如下配置

	[diff]
		tool = winmerge
	[difftool "winmerge"]
		cmd = /C/git-difftool.bat "$LOCAL" "$REMOTE"
	[difftool]
		prompt = false

#### 在指定位置创建git-difftool.bat

#### 在git-difftool.bat中添加一行

	"/C/Program Files/WinMerge/WinMergeU.exe" -e -ub -dl "Base" -dr "Mine" "$1" "$2"