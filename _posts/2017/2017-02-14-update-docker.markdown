---
layout:     post
title:      "从Docker v1.10.3升级到v1.13.0"
subtitle:   "Keep with the development of Docker, and enjoy its advantages"
date:       2017-02-14
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Docker
---

从2015年3月份将Docker升级到v1.10.3，已经在生产环境中使用了近一年的时间。从整体上看，该版本还是比较稳定的，虽然没有出现过重大问题，但是，也发现了一些小的不足。随着Docker的快速发展，被其一些新的特性所吸引，所有决定将Docker进行一次大的升级，直接升级到最新版的v1.13.0。

## Issues or shortages in Docker v1.10.3

* [Very slow when push or pull multiple images at the same time](https://github.com/supereagle/experience/issues/1)
* Containers will stop when daemon shuts down

## Motivation to update Docker from v1.10.3 to v1.13.0

* Keep containers running when daemon shuts down
* Support for live reloading daemon configuration through systemd
* Configurable `maxUploadConcurrency` and `maxDownloadConcurrency` to speed up image pushing and pulling speed
* [Key features in each Docker release](https://github.com/supereagle/experience/issues/5)
* Performance is greatly improved ([Performance comparation between Docker v1.13.0 and v1.10.3](https://github.com/supereagle/experience/issues/10))

## Steps to update Docker

1. Get docker-engine-1.13.0-1.el7.centos.x86_64.rpm and docker-engine-selinux-1.13.0-1.el7.centos.noarch.rpm from [Docker Stable Repository](https://yum.dockerproject.org/repo/main/centos/7/Packages/).
2. Download [aliyun yum repo](http://mirrors.aliyun.com/repo/Centos-7.repo) and place it in folder /etc/yum.repo.d/
3. Create Docker Daemon config file: /etc/docker/daemon.json
```json
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
    "insecure-registries": ["https://registry.robin.com"],
    "live-restore": true
}
```

4. Run update script

```shell
#!/bin/bash
set -ex

# Stop docker & kubelet
systemctl stop docker
systemctl stop kubelet

# Wait docker & kubelet to stop
sleep 20

# Use aliyun yum repo for dependencies
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
mv CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo
yum makecache

# Uninstall the old Docker
rpm -e --nodeps docker-1.10.3-10.el7.centos.x86_64 docker-selinux-1.10.3-10.el7.centos.x86_64

# Install the new Docker 
yum install -y docker-engine-selinux-1.13.0-1.el7.centos.noarch.rpm docker-engine-1.13.0-1.el7.centos.x86_64.rpm

# Create the Daemon config file
cp daemon.json /etc/docker/

# Start docker
systemctl daemon-reload
systemctl start docker
```

## Troubleshootings

* **Lack of dependencies**

Use aliyun yum repo.
```shell
$ mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
$ yum makecache
```

* **Kubernetes-1.12 and kubernetes-node-1.12 are uninstalled while uninstalling old Docker 1.10.3**

Kubernetes 1.12 on CentOS 7.1 depends docker.x86_64. Kubernetes-1.12 and kubernetes-node-1.12 are uninstalled when uninstall docker 1.10.3 by the command `yum erase -y docker docker-selinux`. At the same time, the uninstalled kubernetes-1.12 and kubernetes-node-1.12 can not be reinstalled after `docker-engine-1.13.0-1.el7.centos.x86_64.rpm` is installed, as they depend on **docker.x86_64**, not **docker-engine.x86_64**.

Use command `rpm -e --nodeps  docker-1.10.3-10.el7.centos.x86_64` to uninstall Docker instead of `yum erase -y docker`.

* **Can not reload daemon configuration through systemd**

Use `daemon.json` instead of config files such as `docker`, `docker-storage` and `docker-network` under `/etc/sysconfig/` for Docker daemon options. daemon.json is the recommended configuration in Docker 1.13.

* **The default storage driver is changed from Device mapper to OverlayFS**

Overlay requires the kernel 3.18+, and has known limitations with inode exhaustion and commit performance. The overlay2 driver addresses this limitation, but requires the kernel 4.0+.

* **No effect on Docker daemon after change /usr/lib/systemd/system/docker.service**

Run `systemctl daemon-reload` to flush changes before start Docker.


## Reference

* [What’s New in Docker 1.13](https://blog.codeship.com/whats-new-docker-1-13/)
* [时隔半年，Docker 再发重大版本 1.13](http://www.dockerinfo.net/4184.html)