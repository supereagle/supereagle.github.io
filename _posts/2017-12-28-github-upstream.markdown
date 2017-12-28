---
layout:     post
title:      "如何高效管理Github Upstream和Fork项目"
subtitle:   "How to Effectively Manage Github Upstream and Fork Projects"
date:       2017-12-28
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Git
    - Github
    - Golang
---

参与Github开源项目的一般流程是先fork开源项目，然后基于自己fork的项目开发，最后通过提交PR将自己的代码merge到开源项目中。
这里会涉及到两个项目：开源项目（上游项目）和自己fork的项目，它们之间最终通过PR关联起来。
但是，由于Golang特定的依赖使用和管理模式，导致这种方式针对于Golang项目会有些问题。下面以一个Golang项目为例子，介绍如何高效地在上游项目和fork项目间进行协作。

## Background

参与Github开源项目的一般流程是：
1. 将开源项目fork自己的账户下；
2. Clone自己fork的项目到本地，并进行开发；
3. 将修改的代码push到自己fork的项目中；
4. 从fork的项目中为自己的代码创建PR；
5. 等这个PR被merge后，自己的代码才能进入开源项目。

举个栗子🌰，以才云的开源项目[Cyclone](https://github.com/caicloud/cyclone)为例，来演示整个过程。为描述清晰、方便，有如下约定：
* 上游项目为caicloud/cyclone
* fork项目为supereagle/cyclone

按照常规的Github开发流程，在本地clone的fork项目中直接`go build`，会存在如下问题：
* 提示项目内部的package不存在
```
# pwd
/Users/robin/gocode/src/github.com/supereagle/cyclone
# cd cmd/server
# go build .
options.go:20:2: cannot find package "github.com/caicloud/cyclone/api/server" in any of:
	/Users/robin/gocode/src/github.com/supereagle/cyclone/vendor/github.com/caicloud/cyclone/api/server (vendor tree)
	/usr/local/Cellar/go/1.9.2/libexec/src/github.com/caicloud/cyclone/api/server (from $GOROOT)
	/Users/robin/gocode/src/github.com/caicloud/cyclone/api/server (from $GOPATH)
options.go:21:2: cannot find package "github.com/caicloud/cyclone/cloud" in any of:
	/Users/robin/gocode/src/github.com/supereagle/cyclone/vendor/github.com/caicloud/cyclone/cloud (vendor tree)
	/usr/local/Cellar/go/1.9.2/libexec/src/github.com/caicloud/cyclone/cloud (from $GOROOT)
	/Users/robin/gocode/src/github.com/caicloud/cyclone/cloud (from $GOPATH)
server.go:28:2: cannot find package "github.com/caicloud/cyclone/pkg/scm/provider" in any of:
	/Users/robin/gocode/src/github.com/supereagle/cyclone/vendor/github.com/caicloud/cyclone/pkg/scm/provider (vendor tree)
	/usr/local/Cellar/go/1.9.2/libexec/src/github.com/caicloud/cyclone/pkg/scm/provider (from $GOROOT)
	/Users/robin/gocode/src/github.com/caicloud/cyclone/pkg/scm/provider (from $GOPATH)
```

这里可能很让人困惑，clone的是整个项目，怎么还提示项目中的package不存在了？仔细看一下错误提示，你就会明白原因。
Fork出来的项目在GOPATH中的路径发生了改变，但是代码中import的路径没有改变，所以没有使用fork项目中的package。

* 即使构建成功，但是真正使用也不是fork项目中修改的代码
有可能GOPATH中已经存在caicloud/cyclone，所以按照上面的方式构建，不会提示错误。但是如果在supereagle/cyclone中修改的代码，是无法生效的。
因为代码中引用的是caicloud/cyclone的package，不是supereagle/cyclone中修改过的package。

## Good Practices

由于Golang包管理的这种限制，所以不要基于fork出来的项目修改，而是直接修改上游的项目，然后将修改push到自己fork出来的项目中，从而避免上面两个构建的问题。
主要步骤：
1. 将origin修改为supereagle/cyclone
```
# pwd
/Users/robin/gocode/src/github.com/caicloud/cyclone
# cat .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = https://github.com/supereagle/cyclone          # 将origin修改为supereagle/cyclone
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

2. 将caicloud/cyclone添加为upstream
```
# git remote add upstream https://github.com/caicloud/cyclone
# cat .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = https://github.com/supereagle/cyclone
	fetch = +refs/heads/*:refs/remotes/origin/*
[remote "upstream"]                                      # 添加的upstream
	url = https://github.com/caicloud/cyclone
	fetch = +refs/heads/*:refs/remotes/upstream/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

3. Fetch upstream的所有branch和tag
```
# git fetch upstream
remote: Counting objects: 167, done.
remote: Total 167 (delta 104), reused 104 (delta 104), pack-reused 62
Receiving objects: 100% (167/167), 40.36 KiB | 0 bytes/s, done.
Resolving deltas: 100% (114/114), completed with 71 local objects.
From https://github.com/caicloud/cyclone
 * [new branch]      master        -> upstream/master
 * [new branch]      release-0.1.1 -> upstream/release-0.1.1
 * [new branch]      release-0.3   -> upstream/release-0.3
 * [new branch]      v1-preview    -> upstream/v1-preview
 * [new tag]         v0.1          -> v0.1
 * [new tag]         v0.2          -> v0.2
 * [new tag]         v0.2.0        -> v0.2.0
 * [new tag]         v0.3.0        -> v0.3.0
 * [new tag]         v0.3.3        -> v0.3.3
 * [new tag]         v0.3.1        -> v0.3.1
```

4. 在`.git/config`中将master branch的source设置为upstream
```
# cat .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = https://github.com/supereagle/cyclone
	fetch = +refs/heads/*:refs/remotes/origin/*
[remote "upstream"]
	url = https://github.com/caicloud/cyclone
	fetch = +refs/heads/*:refs/remotes/upstream/*
[branch "master"]
	remote = upstream               # 将remote由origin改为upstream
	merge = refs/heads/master
```

5. 基于master branch创建开发branch，并开发好了之后push到自己项目中
```
# git push origin fix-err:fix-err
```

## Summary

本文以参与Golang开源项目中遇到的依赖问题为背景，介绍了一套新的Github upstream和fork项目的管理方法，解决依赖查找的问题。
其实，这套方法不局限于Golang项目，同时适用于任何语言的开源项目，能够帮助高效、快捷地在upstream和fork项目中协作。
