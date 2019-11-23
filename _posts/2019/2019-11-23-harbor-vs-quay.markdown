---
layout:     post
title:      "私有镜像仓库选型：Harbor VS Quay"
subtitle:   "Private Docker Registry Selection: Harbor VS Quay"
date:       2019-11-23
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - CNCF
    - Docker
---

Docker 虽然提供公有镜像仓库 [Docker hub](https://hub.docker.com/)，但是满足不了绝大部分企业对镜像仓库私有化部署的需求。
私有镜像仓库解决方案中，作为 CNCF 项目的 [Harbor](https://github.com/goharbor/harbor) 长期处于统治地位，得到非常广泛的使用。
近期 Red Hat 开源镜像仓库项目 [Quay](https://github.com/quay/quay)，为私有镜像仓库增加了一种选择，因此引起了大家的极大兴趣。
下面将全面地对比 Harbor 和 Quay，为它们之间的选型提供指导。


## Overview

Harbor 是 VMware 中国研发团队在 2016 年开源的镜像仓库项目。当时一般使用 Docker 官方提供的 Docker Registry 部署私有镜像仓库，
但是它只能提供 API 和镜像的操作，缺少 UI 和其他辅助功能，对普通用户很不友好。
Harbor 就是抓住这点，在原生 Docker Registry 基础上打造企业级镜像仓库，满足企业对镜像仓库的一些额外需求，因此能够快速地推广开来。

Quay 其实是非常有名的开源公司 [CoreOS](https://coreos.com/) 的产品，CoreOS 在 2018 被 Red Hat 收购。
CoreOS 是一家非常有创新力的公司，开发出很多明星级的开源项目，例如：分布式 KV 存储 [etcd](https://github.com/etcd-io/etcd)，镜像安全扫描 [clair](https://github.com/quay/clair) 等。
Quay 在开源之前，一直以公有镜像仓库提供服务 [quay.io](https://quay.io/)，类似于 Docker hub。这次开源后，能够提供私有化部署。

Harbor 和 Quay 不仅功能上非常类似，而且名字上也非常接近。Harbor 是港湾、港口的意思，而 Quay 是指码头。

## Harbor vs Quay 

Harbor 和 Quay 的定位都是镜像仓库解决方案，它们之间的各方面对比如下表所示：

| 对比项 | Harbor | Quay |
|--|--------|------|
| 语言 | Golang | Python |
| 部署方式 | 私 | 公 + 私 |
| 用户管理 | 支持 DB、LDAP 和 OIDC | 支持 LDAP、Keystone、OIDC、Google 和 GitHub |
| 机器人账号 | 支持 | 支持 |
| 权限管理 | 支持 | 支持 |
| 图形用户界面 | 支持 | 支持 |
| 国际化 | 支持中英等 5 种语言 | 仅支持英文 |
| 使用文档 | 非常全面 | 较少 |
| 镜像安全扫描 | 支持 | 支持 |
| 镜像可信 | 支持 | 支持 |
| 镜像清理 | 支持 | 支持 |
| 审计日志 | 支持 | 支持 |
| Helm 管理 | 支持 | 支持 |
| 多仓库管理 | 支持 | 不支持 |
| 镜像同步 | 支持 | 不支持 |
| 代码仓库集成 | 不支持 | 支持 |
| 镜像构建 | 不支持 | 支持 |
| 镜像下载 | 不支持 | 支持 |
| 通知 | 支持 Webhook | 支持站内信、Webhook、Email、Slack 等 |

从多方面的详细对比来看，Harbor 和 Quay 的绝大部分功能都是重叠的。但是，Harbor 特有的多仓库管理以及镜像同步功能，一直作为 Harbor 非常大的亮点，并且有很大的实用价值。
Harbor 支持中文，对于广大中国用户的来说，也是一个非常大的优势。至于 Quay 有而 Harbor 没有的 SCM 集成的能力，并不能为 Quay 形成多大优势。
因为绝大部分企业已经有基本的 CI，而 Quay 提供的构建能力太简单，不足以替换现有的 CI。

## Conclusion

Quay 相对于 Harbor 并没有明显优势，但是存在作为新开源项目两个不可避免的劣势：用户积累，开源社区。
Harbor 已经积累了广泛的用户，各方面都得到充分地检验，加上 CNCF 的背书，会使其用户群体进一步快速扩大。在这种情况下，Quay 是很难跟 Harbor 抢用户的。
Harbor 的核心贡献者虽然仍是 VMWare，但是经过这么多年的积累，已经形成了相对完善的社区，这对一个开源项目的快速发展也是至关重要的。
因此，Quay 的开源并不能撼动 Harbor 的地位，Harbor 仍将是私有镜像仓库领域的王者。
