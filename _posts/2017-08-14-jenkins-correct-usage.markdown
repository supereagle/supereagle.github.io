---
layout:     post
title:      "Jenkins的正确打开方式"
subtitle:   "The Correct Way to Open Jenkins"
date:       2017-08-14
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Jenkins
---

[Jenkins](https://jenkins.io/)是大家所熟知而且应用最广泛的持续集成工具。其前身是[Hudson](https://zh.wikipedia.org/wiki/Hudson_(%E8%BD%AF%E4%BB%B6))，是由当时还在Sun工作的日本人[Kohsuke Kawaguchi](http://kohsuke.org/)于2004年开始的，没想到13年过去了还能统治着CI领域，不得不感叹其强大的生命力。Sun被Oracle收购之后，Oracle于2010年12月将Hudson申请为注册商标。社区对此不买账，于是在2011年将Hudson更名为Jenkins。

## Background

Jenkins是用Java开发的，包括其上千个plugin也是Java开发的。其最开始的定位是持续集成工具，不过随着DevOps的兴趣，用户对持续部署也提出了更高的要求，所有Jenkins创始人Kohsuke Kawaguchi于2015年9月提出[Jenkins 2.0](https://wiki.jenkins.io/display/JENKINS/Jenkins+2.0)，迅速得到广泛的响应，并于2016年4月正式发布Jenkins 2.0。
Jenkins 2.0最大的卖点是[Pipeline Plugin](https://wiki.jenkins.io/display/JENKINS/Pipeline+Plugin)。其实这个plugin并不是新创的，而是由已经存在很多年的[Build Flow Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Flow+Plugin)以及后来的[Workflow Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Workflow+Plugin)一步步演化来的。不过，以Pipeline Plugin作为Jenkins 2.0最大的卖点，最主要是表明其希望往CD发力的战略。Pipeline Plugin的精髓是Pipeline as Code：将所有其他plugin的config，都以一套Groovy DSL描述。因此，Pipeline Plugin只是以Groovy脚本的方式组织各plugin，真正执行功能的还是各plugin。自从Jenkins 2.0发布之后，其他的plugin都开始一步步支持Pipeline Plugin。

一毕业刚工作就开始使用Jenkins，从最初的使用它来做持续测试，后来使用它做持续构建，不过这些都是针对一个项目的，到最后基于Jenkins搭建容器云PaaS。这个过程中，不仅充分体会到Jenkins的强大之处，同时也感受到其在大型PaaS系统中的局限性。针对简单的使用Jenkins做一些单个或者几个项目的CI/CD就没必要介绍了，下面主要介绍一下Jenkins在PaaS平台中的一些使用经验。







