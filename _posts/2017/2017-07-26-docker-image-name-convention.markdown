---
layout:     post
title:      "企业级镜像仓库中Docker Image命名规范"
subtitle:   "Docker Image Name Convention for Enterprise Docker Registry"
date:       2017-07-26
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Docker
---

## Target

* 方便使用：统一规范的命令规则，使image name能够清晰的描述该image的环境信息和用途。
* 方便维护：能够有效地对所有image进行展示和查询，定期对无用image进行清理，释放存储空间。
* 方便管理：只有Image name满足一定规范，才能精确地对所有image进行配额管理和权限控制。

## Background

### Image Types

* 基础镜像：不包含具体业务的镜像。主要是为业务提供运行环境的，或者是一些开源项目的官方镜像。
* 业务镜像：基于基础镜像构建出来的包含具体业务的镜像，能够在测试或生产环境中部署和运行。

### Image Sources

* Maintainer：负责提供和维护基础镜像。
* CI流水线：项目持续集成生产的镜像。
* 个人用户：非maintainer手动docker push上传的测试image。

## Conventions

### Format of Image Name

**Format**：DOCKER_REGISTRY/repo/name：tag
* DOCKER_REGISTRY：公司统一的Docker Registry地址。
* repo：镜像仓库，用来管理一类镜像。
* name：具体某镜像的名称。
* tag：具体某镜像的标签。

**`注意：通常所说的image name应该是完整的repo/name:tag，不能只是其中的某一部分。`**

### Convention of Image Name

#### 基础镜像
* repo：统一用public仓库来进行管理。
* name：描述该image中所提供的软件，各软件间通过“-”连接。
* tag：依次顺序描述该image中所提供的软件的版本，各版本间通过“-”连接。


**`注意：`**
* 所有Base image除了尽量通过name和tag描述该image中所有的软件及其版本信息，还需要通过添加description的label，更加详细地描述image内容。
* 对于非软件版本的更新（例如：更新安全漏洞），Base image的tag不会更新。为了追踪Base image的版本信息，需要在image中加入构建该image的Dockerfile的commit id。

For example：

|      Image Name      |      Softwares      |
|----------------------|---------------------|
| public/tomcat:7.0.78 |    Tomcat 7.0.78    |
| public/tomcat:8.5.15 |    Tomcat 8.5.15    |
| public/java-tomcat:1.7.65-8.5.15 |Java 1.7.65, Tomcat 8.5.15|
| public/nginx-php:1.13.1-7.1.5 |Nginx 1.13.1, Php 7.1.5|

#### 业务镜像
* repo：用项目名作为仓库，来管理该项目下的所有镜像。
* name：描述该image中所包含的业务。
* tag：commit id（前7位）和timestamp（12位，yymmddHHMMSS）组合成唯一标识，中间通过“-”连接。

For example：

|             Image Name             |                          Softwares                          |
|------------------------------------|-------------------------------------------------------------|
| pay/frontend:7654321-170401040120  |pay项目中的frontend组件，基于7654321 commit ID于2017/04/01 04:01:20构建|
|  pay/backend:1234567-170602060238  |pay项目中的backend组件，基于1234567 commit ID于2017/06/02 06:02:38构建|
|  risk/control:132453-170102132415	 |风控项目中的control组件，基于132453 commit ID于2017/01/02 13:24:15构建|
|  risk/manager:431423-170205010408	 |风控项目中的manager组件，基于431423 commit ID于2017/02/05 01:04:08构建|

## Authority Management

### 基础镜像

对所有人可见，而且他们都能pull，但是只有maintainer才有push和delete的权限。

### 业务镜像

与项目相关的人员才可以看见和pull该项目的所有镜像，与项目无关人员无权限看见和pull。

|  Roles  |  View  |  Pull  |  Push  | Delete |
|---------|--------|--------|--------|--------|
|  Master |   ✓	   |   ✓	|   ✓	 |   ✓	  |
|Developer|   ✓	   |   ✓	|   ✓	 |   x	  |
|  Guest  |   x	   |   x	|   x	 |   x	  |

## Quota Management

每个image最多保留N（可配置）个tag。对于N个的话，按时间排序，优先将老的tag删除。