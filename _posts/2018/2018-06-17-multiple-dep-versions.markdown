---
layout:     post
title:      "Golang 项目多版本依赖的引入"
subtitle:   "Import Multiple Versions of Deps in Golang"
date:       2018-06-17
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
    - Godep
---

Golang 是通过 vendor 来管理依赖的，它将所有的依赖，根据引用路径按照一定目录结构放在 vendor 目录下。这样会导致一个问题：如何同时引用一个依赖的多个版本？
很明显，Golang 的这种依赖管理模式，是解决不了这个问题。下面介绍一种解决该问题的比较有意思的方法。

## 问题描述

**场景1**

假设当前开发的程序为 A，引用了依赖 B，B 又依赖 `v1.0` 版本的 C。同时，A 为了使用 C 的一些新特性，又直接依赖 `v2.0` 版本的 C。

```
A ------> B
 \        |
  \       |
   \      |
    \     |
     \    |
 v2.0 \   |v1.0
       \  |
        \ |
         C
```

**场景2**

假设程序 A 需要同时支持 Gitlab v3 和 v4 的 API，但是其依赖的第三方库缺不能同时支持这两种 API。因此，程序 A 需要引入第三方依赖的两个版本，才能满足需求。

```
           A
          /  \
         /    \
        /      \
       /        \
      /          \
 gitlab v3    gitlab v4  
 ```

在上述两种场景下，使用 dep 包管理工具的话，就会出现版本冲突的问题。

## 解决方法

### Fork 依赖

如果是 B 没有及时更新自己的依赖 C，我们可以自己对 B 进行升级。将依赖 B fork 到自己的项目中，然后将 B 依赖的 C 从 `v1.0` 升级到 `v2.0`。
程序 A 同时引用 `v2.0` 的 C 和自己 fork 出来的 B，这样 C 的版本就统一成 `v2.0`，从而避免版本冲突的问题。

引用 fork 出来的 B，那是不是所有路径都得发生变化呢？可以通过 dep 提供的参数 source，将原来的引用路径指向自己 fork 的地址。dep 的更多用法，可以参考之前的博客[《Golang依赖管理工具：Dep》](https://supereagle.github.io/2017/10/05/golang-dep/)。

例如：

```
[[constraint]]
  name = "github.com/test/B"
  # source 指定依赖的来源。
  source = "github.com/my/B"

[[constraint]]
  name = "github.com/test/C"
  version = "2.0.0"
```

### gopkg.in

采用 fork 的方式，虽然能够解决问题，但是更像是一种 workaround。这种方式会产生两份代码，很容易导致代码不一致，增加维护成本。

[gopkg.in](http://labix.org/gopkg.in) 支持 URL 中带 version，然后重定向到 Github 中相应的 version。
gopkg.in 的优势是 URL 简洁，轻量级，自身并不托管代码，而是重定向到 Github。

[multiple-dep-versions](https://github.com/supereagle/go-example/tree/master/multiple-dep-versions) 是一个简单的示例，通过直接引用 [go-gitlab v0.4.1](https://github.com/xanzy/go-gitlab/tree/v0.4.1) 来支持 Gitlab v3 API，通过 `gopkg.in/xanzy/go-gitlab.v0` 间接引用最新版本 [go-gitlab v0.10.6](https://github.com/xanzy/go-gitlab/tree/v0.10.6) 来支持 Gitlab v4 API。

**Version number**

gopkg.in 虽然提供了引用指定版本依赖的灵活性，但是对依赖的版本规范提出了严格的要求。
依赖的版本必须遵守 [Semantic Versioning 2.0.0](https://semver.org/)，在出现不向下兼容的时候，需要升级版本号。
而且 gopkg.in 在 URL 只支持 major version。如果没有版本的 tag 或者 branch，URL 中可以使用 `v0` 来指向 master 分支。
`v0` 是为没有稳定 API 的情况预留的，不建议长时间使用。

## Reference

- [gopkg.in](http://labix.org/gopkg.in)
- [Go and Package Versioning](http://zduck.com/2014/go-and-package-versioning/)
- [Diamond dependency problem in go dep](https://mycodesmells.com/post/diamond-dependency-problem-in-go-dep)
