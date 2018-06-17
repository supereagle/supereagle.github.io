---
layout:     post
title:      "Golang 项目多版本依赖的引入"
subtitle:   "Import of Multiple Versions in Golang"
date:       2018-06-17
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
    - Godep
---

Golang 是通过 vendor 来管理依赖的，它将所有的依赖，都放在该目录下。这样会导致一个问题：如何同时引用一个依赖的多个版本？
很明显，Golang 的这种依赖管理模式，是解决不了这个问题。下面介绍一种解决该问题的比较有意思的方法。

## 问题描述

假设当前开发的程序为 A，它引用了依赖 B，B 又依赖 `v1.0.0` 版本的 C。同时，A 又直接依赖 `v2.0.0` 版本的 C。

这种场景下，使用 dep 包管理工具的话， 就会出现版本冲突的问题。

## 解决方法

### Fork 依赖

将依赖 C 的 `v2.0.0` 版本 fork 到自己的，然后引用 fork 出来的依赖。

### gopkg.in

[gopkg.in](http://labix.org/gopkg.in) 支持 URL 中带 version，然后重定向到 Github 中相应的 version。

gopkg.in 的优势是 URL 简洁，轻量级，自身并不托管代码，而是重定向到 Github。

#### Version number

gopkg.in 虽然提供了引用指定版本依赖的灵活性，但是对依赖的版本规范提出了严格的要求。
依赖的版本必须遵守 [Semantic Versioning 2.0.0](https://semver.org/)，在出现不向下兼容的时候，需要升级版本号。

## Reference

- [gopkg.in](http://labix.org/gopkg.in)
- [Go and Package Versioning](http://zduck.com/2014/go-and-package-versioning/)
- [Diamond dependency problem in go dep](https://mycodesmells.com/post/diamond-dependency-problem-in-go-dep)
