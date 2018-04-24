---
layout:     post
title:      "镜像仓库中镜像存储的原理解析"
subtitle:   "Principle of Image Storage in Docker Registry"
date:       2018-04-24
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Docker
---

[镜像仓库](https://docs.docker.com/registry/)主要是用来存储和分发镜像的，并对外提供一套 [HTTP API V2](https://docs.docker.com/registry/spec/api/)。镜像仓库中的所有镜像，都是以数据块 (Blob) 的方式存储在文件系统中。
支持多种[文件系统](https://docs.docker.com/registry/storage-drivers/#provided-drivers)，主要包括filesystem，S3，Swift，OSS等。
下面详细介绍一下，镜像的所有数据，是如何存储在镜像仓库的文件系统中的。



## 实验

实验目的：同一镜像，在不同镜像仓库中，存储的方式和内容完全一样。

### 实验环境

* 2 台镜像仓库
  * https://cargo-single-tenant-current.caicloudprivatetest.com (192.168.20.223)
  * https://cargo-multiple-tenant-current.caicloudprivatetest.com (192.168.20.225)
* 测试镜像：release/cyclone-server:v0.5.0-beta.1

### 实验步骤

**1. 通过 Registry API 获得两个镜像仓库中镜像release/cyclone-server:v0.5.0-beta.1的 manifest 信息**

```
digest: sha256:6d47a9873783f7bf23773f0cf60c67cef295d451f56b8b79fe3a1ea217a4bf98
payload: 
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 3688,
      "digest": "sha256:790ca1071242c78a2ced2984954322d94e9d2b94829e1839804af5340087d6c7"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1991747,
         "digest": "sha256:605ce1bd3f3164f2949a30501cc596f52a72de05da1306ab360055f0d7130c32"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1843790,
         "digest": "sha256:77a50f74e4304f93f39148b88dbe5f83400ab120d04856894db4be294f47bf7d"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 7738438,
         "digest": "sha256:7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1474,
         "digest": "sha256:7ee27521e3b23c3c2713acb1394e45a601ae894bbb5d4bf1761ebce8e97060a3"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 6329,
         "digest": "sha256:71b1ca055a6770e6e61e1573719dd450982998f9e483e02e27ab047d79929a2e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1054564,
         "digest": "sha256:e0deed02c7d20c528f44000b96ec2c3269307de184ece214a7ff3c8e44f3d16d"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 520,
         "digest": "sha256:6d140169e8a78b952428f4c20a9ab7344b6efadcf67af25a45c93106ecfa8e65"
      }
   ]
}
```

> 结论：经过对比，通过 Registry API 获得的两个镜像仓库中相同镜像的 manifest 信息完全相同。

**2. 查看两个镜像仓库中镜像数据的存储**

* Manifest 信息的存储

对比两个镜像仓库中相同镜像的 Manifest 信息的存储，比较它们的存储路径和存储内容是否相同。

223上的数据:

```
[root@c720v223 v2]# pwd
/var/lib/kubelet/cargo/release/data/harbor/registry/docker/registry/v2
# 查看镜像 Manifest 对应的blob id
[root@c720v223 v2]# cat repositories/release/cyclone-server/_manifests/tags/v0.5.0-beta.1/current/link
sha256:6d47a9873783f7bf23773f0cf60c67cef295d451f56b8b79fe3a1ea217a4bf98
# 查看镜像 Manifest 
[root@c720v223 v2]# cat blobs/sha256/6d/6d47a9873783f7bf23773f0cf60c67cef295d451f56b8b79fe3a1ea217a4bf98/data
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 3688,
      "digest": "sha256:790ca1071242c78a2ced2984954322d94e9d2b94829e1839804af5340087d6c7"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1991747,
         "digest": "sha256:605ce1bd3f3164f2949a30501cc596f52a72de05da1306ab360055f0d7130c32"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1843790,
         "digest": "sha256:77a50f74e4304f93f39148b88dbe5f83400ab120d04856894db4be294f47bf7d"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 7738438,
         "digest": "sha256:7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1474,
         "digest": "sha256:7ee27521e3b23c3c2713acb1394e45a601ae894bbb5d4bf1761ebce8e97060a3"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 6329,
         "digest": "sha256:71b1ca055a6770e6e61e1573719dd450982998f9e483e02e27ab047d79929a2e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1054564,
         "digest": "sha256:e0deed02c7d20c528f44000b96ec2c3269307de184ece214a7ff3c8e44f3d16d"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 520,
         "digest": "sha256:6d140169e8a78b952428f4c20a9ab7344b6efadcf67af25a45c93106ecfa8e65"
      }
   ]
}
```

225上的数据:

```
[root@c820v255 v2]# pwd
/var/lib/kubelet/cargo/release/data/harbor/registry/docker/registry/v2
# 查看镜像 Manifest 对应的blob id
[root@c820v255 v2]# cat repositories/release/cyclone-server/_manifests/tags/v0.5.0-beta.1/current/link
sha256:6d47a9873783f7bf23773f0cf60c67cef295d451f56b8b79fe3a1ea217a4bf98
# 查看镜像 Manifest 
[root@c820v255 v2]# cat blobs/sha256/6d/6d47a9873783f7bf23773f0cf60c67cef295d451f56b8b79fe3a1ea217a4bf98/data
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 3688,
      "digest": "sha256:790ca1071242c78a2ced2984954322d94e9d2b94829e1839804af5340087d6c7"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1991747,
         "digest": "sha256:605ce1bd3f3164f2949a30501cc596f52a72de05da1306ab360055f0d7130c32"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1843790,
         "digest": "sha256:77a50f74e4304f93f39148b88dbe5f83400ab120d04856894db4be294f47bf7d"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 7738438,
         "digest": "sha256:7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1474,
         "digest": "sha256:7ee27521e3b23c3c2713acb1394e45a601ae894bbb5d4bf1761ebce8e97060a3"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 6329,
         "digest": "sha256:71b1ca055a6770e6e61e1573719dd450982998f9e483e02e27ab047d79929a2e"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 1054564,
         "digest": "sha256:e0deed02c7d20c528f44000b96ec2c3269307de184ece214a7ff3c8e44f3d16d"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 520,
         "digest": "sha256:6d140169e8a78b952428f4c20a9ab7344b6efadcf67af25a45c93106ecfa8e65"
      }
   ]
}
```

> 结论：经过对比，两个镜像仓库中相同镜像的 manifest 信息的存储路径和内容完全相同。

* Blob 数据的存储

对比两个镜像仓库中相同镜像的 Blob 信息的存储，比较它们的存储路径和存储内容是否相同。以其中的 Blob sha256:7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e 为例，进行验证。

223上的数据:

```
[root@c720v223 v2]# pwd
/var/lib/kubelet/cargo/release/data/harbor/registry/docker/registry/v2
# 查看镜像 blob 的位置和大小
[root@c720v223 v2]# du -s blobs/sha256/71/7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e/data
7560	blobs/sha256/71/7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e/data
```

225上的数据:

```
[root@c820v255 v2]# pwd
/var/lib/kubelet/cargo/release/data/harbor/registry/docker/registry/v2
[root@c820v255 v2]# du -s blobs/sha256/71/7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e/data
7560	blobs/sha256/71/7118e0a5b59500ceeb4c9686c952cae5e8bfe3e64e5afaf956c7098854a2390e/data
```

> 结论：经过对比，两个镜像仓库中相同镜像的 blob 信息的存储路径和内容完全相同。

## 结论

* 通过 Registry API 获得的两个镜像仓库中相同镜像的 manifest 信息完全相同。
* 两个镜像仓库中相同镜像的 manifest 信息的存储路径和内容完全相同。
* 两个镜像仓库中相同镜像的 blob 信息的存储路径和内容完全相同。

