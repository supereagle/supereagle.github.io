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

## Evolution of Containerized PaaS

### First Generation

![jobs](/img/in-post/jenkins/jobs.png)

每个流水线支持4个stage：compile、 unit test、code scan以及package，每个stage都是对应一个job。其中compile和package是必选的，unit test和code scan是可选的，用户可以根据实际情况skip。每个job都是由server控制：上个stage的job执行完了之后，将执行结果反馈给server。如果成功的话，server再触发下个stage的job，后面的stage依次类推地往下执行。
由于接入的业务较少、项目类型也比较单一，构建slave采用传统的VM的方式，单Jenkins master控制所有的job。第一代PaaS主要是流程打通，以及与现有系统的集成，因此没有进行广泛推广。

### Second Generation

第一代PaaS经过一定的打磨之后，基本功能都齐全了，就可以进行大面积推广。随着业务的不断接入，这种架构设计也逐渐暴露出一些列问题：
* job数量暴增，很快达到Jenkins支持的1000个瓶颈，导致性能下降（页面响应非常慢）；
* 一次构建中的多个job的执行，都需要由server控制，加大系统复杂度；
* 一次构建中的多个job存在重复的操作，例如：code checkout，dependency download等，导致整个构建时间很长
* 一个流水线的多个job的重复内容会浪费Jenkins master资源，包括config、log以及archive
* 多个项目同时在一台VM slave构建时存在干扰，例如：gradle或者sbt构建时会对dependency加锁，导致其他项目等待超时

由于Jenkins 2.0的发布，支持在一个pipeline中同时定义多个stage，这样一个流水线就只需要一个job，而且流水线中增加stage，并不会额外增加job数量。流水线被触发之后，各stage的执行，是由pipeline自己控制，server不再干扰pipeline的执行。pipeline在执行过程中，还是会将各stage的状态实时地report给server。这种方式的具体实现，可以参考我的开源项目[goline](https://github.com/supereagle/goline)。

![pipeline](/img/in-post/jenkins/pipeline.png)

多个项目同时在一台VM slave构建相互干扰的问题，是借助于[Jenkins Kubernetes Plugin](https://github.com/jenkinsci/kubernetes-plugin)这个plugin。在开始使用这个plugin的时候，它还[Carlos Sanchez的个人项目](https://github.com/carlossg/jenkins-kubernetes-plugin)，当时的版本还是v0.1。现在已经被Jenkins官方fork，并进行快速迭代开发中。这个plugin能够在流水线被触发的时候，动态地调用Kubernetes API产生一个pod，通过JNLP协议连接到Jenkins master上，然后这个流水线就可以在这个pod中执行。等流水线执行完后，又调用Kubernetes API销毁该pod。

## Reference
- [Jenkins 2.0新时代：从CI到CD](http://dockone.io/article/1421)




