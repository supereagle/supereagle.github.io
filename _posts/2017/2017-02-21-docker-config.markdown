---
layout:     post
title:      "Docker Daemon Configuration"
subtitle:   "A better way to config Docker Daemon"
date:       2017-02-21
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Docker
---

## Installation

### Check Prerequisites

Docker对操作系统的版本有一定要求。目前，已经支持大多数Linux发行版本，macOS以及Windows，甚至支持一些公有云平台，例如：AWS，Azure等。
具体支持的平台，以及相应的要求，参考官网[Install Docker Engine](https://docs.docker.com/engine/installation/)。

### Install from Docker’s Repositories

CentOS官方提供docker RPM安装包，不过这个安装包是经过定制化的。与Docker官方提供的RPM包有些差异：  
* CentOS提供的RPM包的名字是`docker`，而Docker官方提供的RPM包的名字是`docker-engine`
* 提供docker-storage-setup，能够自动为Docker创建direct LVM的Device mapper

因此，已经在CentOS上安装了旧的Docker，需要先将旧的Docker卸载：`yum -y remove docker docker-selinux`，然后才能安装Docker官方提供的最新RPM包。

### Install from Docker Binaries

从官方[Release Notes](https://github.com/docker/docker/releases)下载平台对应的二进制文件压缩包，格式为`tar.gz`或者`zip`。将压缩包解压，并将解压后的二进制文件复制到/usr/bin/目录下。然后就可以后台启动Docker Daemon: `sudo dockerd &`。

一般情况下，不建议通过二进制文件安装。因为，这种方式安装的Docker不会被systemd管理，不方便Docker运行的管理和Debug。但是，这种方式非常适合Docker升级，不用卸载老的Docker，只需替换二进制文件，还是可以按以前的方式执行Docker。

## Configuration

由于一直使用的操作系统是CentOS 7.1，因此下面介绍的Docker Configuration都是基于该平台的。Docker同时支持`Command Options`和`--config-file`两种配置Docker Daemon的方式。之前在Docker v1.10.3中，使用`Command Options`的方式，后来升级到Docker v1.13.0之后，开始使用`--config-file`的方式。选择使用`--config-file`的主要原因：  
* JSON格式的配置文件，简单、清晰和集中；不像`Command Options`配置分散在多个文件和变量中
* docker.service更加简单，不用`EnvironmentFile`导入环境变量，`ExecStart`后面也不用跟各种参数
* 支持通过systemd动态加载配置，不用重启Docker（Docker v1.12.0开始引入）
* 这也是Docker官方建议的配置方式。

`Command Options`和`--config-file`两种配置方式可以一起使用，不过需要注意的是，它们之间不能有冲突。这两种配置中不能存在相同的option，无论它们的值是否相同，否则Docker Daemon将无法启动。

### Useful Options

|                |   Description  | 	Default    |
|----------------|----------------|----------------|
| config-file  | Daemon配置文件         | /etc/docker/daemon.json |
| disable-legacy-registry   |  禁用Docker Registry V1   | 	    |
| max-concurrent-downloads | pull镜像的最大并行数    | 	 3   |
| max-concurrent-uploads |  push镜像的最大并行数   | 	5    |
|  insecure-registries | 无需TLS检查的Registry  | 	    |
|    live-restore  | Daemon down时container继续run  | 	    |
|       debug         |  开启Debug模式   | 	    |
|     log-level       |  log级别   | 	info    |
|     log-opt           | log driver的选项    | 	    |
|     storage-driver           |  storage driver   | 	    |
|     storage-opts           |  storage driver选项   | 	    |


### Configuration for Docker v1.10.3

service文件中需要指定配置文件来导入环境变量，而且环境变量需要加到`ExecStart`才能生效。

```shell
$ cat /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target
Wants=docker-storage-setup.service

[Service]
LimitMEMLOCK=1288490188800
LimitSTACK=infinity
LimitNPROC=infinity
LimitNOFILE=1310720
LimitCORE=infinity
Type=notify
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
Environment=GOTRACEBACK=crash
ExecStart=/usr/bin/docker daemon $OPTIONS \
          $DOCKER_CERT \
          $DEFAULT_ULIMIT \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY
MountFlags=slave
TimeoutStartSec=1min
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

多个配置文件

/etc/sysconfig/docker
```shell
$ cat /etc/sysconfig/docker
# /etc/sysconfig/docker

# Modify these options if you want to change the way the docker daemon runs
OPTIONS='--disable-legacy-registry --log-opt max-size=2m --log-opt max-file=5 --log-level=warn'
DEFAULT_ULIMIT='--default-ulimit nofile=131072 --default-ulimit memlock=131941395333120 --default-ulimit core=-1 --default-ulimit nproc=-1 --default-ulimit stack=-1'
INSECURE_REGISTRY='--insecure-registry https://registry.access.redhat.com'

DOCKER_CERT_PATH=/etc/docker

# If you want to add your own registry to be used for docker search and docker
# pull use the ADD_REGISTRY option to list a set of registries, each prepended
# with --add-registry flag. The first registry added will be the first registry
# searched.
#ADD_REGISTRY='--add-registry registry.access.redhat.com'

# If you want to block registries from being used, uncomment the BLOCK_REGISTRY
# option and give it a set of registries, each prepended with --block-registry
# flag. For example adding docker.io will stop users from downloading images
# from docker.io
# BLOCK_REGISTRY='--block-registry'

# If you have a registry secured with https but do not have proper certs
# distributed, you can tell docker to not look for full authorization by
# adding the registry to the INSECURE_REGISTRY line and uncommenting it.
# INSECURE_REGISTRY='--insecure-registry'

# On an SELinux system, if you remove the --selinux-enabled option, you
# also need to turn on the docker_transition_unconfined boolean.
# setsebool -P docker_transition_unconfined 1

# Location used for temporary files, such as those created by
# docker load and build operations. Default is /var/lib/docker/tmp
# Can be overriden by setting the following environment variable.
# DOCKER_TMPDIR=/var/tmp

# Controls the /etc/cron.daily/docker-logrotate cron job status.
# To disable, uncomment the line below.
# LOGROTATE=false
```

/etc/sysconfig/docker-storage
```shell
$ cat /etc/sysconfig/docker-storage
DOCKER_STORAGE_OPTIONS=--storage-driver devicemapper --storage-opt dm.fs=xfs --storage-opt dm.thinpooldev=/dev/mapper/docker--vg-docker--pool --storage-opt dm.use_deferred_removal=true --storage-opt dm.use_deferred_deletion=true
```

/etc/sysconfig/docker-network
```shell
$ cat /etc/sysconfig/docker-network
# /etc/sysconfig/docker-network
DOCKER_NETWORK_OPTIONS=
```

### Configuration for Docker v1.13.0

service文件采用默认的即可，不需要额外配置。

```shell
$ cat /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target firewalld.service

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```

`daemon.json`配置文件集中配置，结构清晰。

```shell
$ cat /etc/docker/daemon.json
{
    "disable-legacy-registry": true,
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 20,
    "log-level": "warn",
    "log-opts": {
        "max-size": "2m",
        "max-file": "5"
    },
    "storage-driver": "devicemapper",
    "storage-opts": [
        "dm.fs=xfs",
        "dm.thinpooldev=/dev/mapper/docker--vg-docker--pool",
        "dm.use_deferred_removal=true",
        "dm.use_deferred_deletion=true"
    ],
    "insecure-registries": ["https://registry.access.redhat.com"],
    "live-restore": true
}
```

## Reference

* [Install Docker Engine](https://docs.docker.com/engine/installation/)
* [Configure and run Docker on various distributions](https://docs.docker.com/engine/admin/)
* [Docker Daemon Options](https://docs.docker.com/engine/reference/commandline/dockerd/)