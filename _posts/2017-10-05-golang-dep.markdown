---
layout:     post
title:      "Golang依赖管理工具：Dep"
subtitle:   "Golang Dependency Management Tool: Dep"
date:       2017-10-05
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
    - 依赖管理
---

对于任何编程语言，依赖管理都是其必须考虑的一个问题。尤其是在大规模协作的软件开发中，如何保证大家都使用同一份依赖，项目能够随时随地重复编译，
是考核编程语言成熟度的重要指标之一。一些成熟的编程语言，例如Java、Python在这方面已经做得比较好了，但是对于新秀Golang，还要比较长的一段路要走。

Golang团队一直秉持着"简约"的设计原则，甚至强调代码的简洁和清晰度胜过代码的复用。因此，他们对依赖管理的设计非常重视和谨慎，直到v1.5才开始逐步
引入依赖管理的设计。在v1.5中实验性地加了vendor目录来支持本地依赖管理，通过环境变量`GO15VENDOREXPERIMENT`来控制是否使用该特性，默认不启用。
如果要使用vendor特性，需要设置环境变量`GO15VENDOREXPERIMENT=1`。v1.6中会默认启用vendor特性，不需要再额外设置`GO15VENDOREXPERIMENT`，
v1.7中已经将vendor纳入标准特性中，并废弃`GO15VENDOREXPERIMENT`。

> "Through the design of the standard library, great effort was spent on controlling dependencies. It can be better to
> copy a little code than to pull in a big library for one function. Dependency hygiene trumps code reuse."
> --- Go at Google

Golang虽然已经提供vendor特性，但只是用来存储本地依赖。如何将这些依赖高效、简洁地管理起来，目前官方还没有给出明确的指导意见。只是在其官方的
Wiki [Package Management Tools](https://github.com/golang/go/wiki/PackageManagementTools)中列举和对比了各种依赖管理工具，其中
包括其官方的实验性的工具[Dep](https://github.com/golang/dep)。Dep项目开始于2016年3月，目前还处于开发和实验阶段，并没有纳入到Golang
官方的工具链中，这也是其最终目标。不过Dep只支持Go 1.8及以上版本，对于使用低于1.8版本的用户，还是需要借助Wiki中推荐的其他工具。
由于Dep是官方推出的依赖管理工具，因此备受大家的关注和期待，目前Github stars数量已经达到4800+个，下面对其进行重点介绍。

## Overview

Dep通过两个metadata文件来管理依赖：manifest文件`Gopkg.toml`和lock文件`Gopkg.lock`。`Gopkg.toml`可以灵活地描述用户的意图，包括依赖的
source、branch、version等。`Gopkg.lock`仅仅描述依赖的具体状态，例如各依赖的revision。`Gopkg.toml`可以通过命令生产，也可以被用户根据
需要手动修改，`Gopkg.lock`是自动生成的，不可以修改。

### Gopkg.toml语法

> 跟Shell脚本中一样，`#`表示注释一行。

**required**

`required`是一个packages(不是projects)的列表，主要针对满足如下3个特性的packages：
* 被项目需要
* 没有被直接或者传递import
* 不想被加入到GOPATH中，和/或者想lock它的版本

**ignored**

`ignored`是一个packages(不是projects)的列表，主要是避免将某些package加入到依赖中。

**constraint**

`constraint`指定直接依赖的相关信息。

```toml
[[constraint]]
  # Required: the root import path of the project being constrained.
  name = "github.com/user/project"
  # Recommended: the version constraint to enforce for the project.
  # Only one of "branch", "version" or "revision" can be specified.
  version = "1.0.0"
  branch = "master"
  revision = "abc123"

  # Optional: an alternate location (URL or import path) for the project's source.
  source = "https://github.com/myfork/package.git"
```

> name是代码中project的import path，必须指定；branch、version、revision三者中，只能选择其中一种方式指定version。

**override**

`override`跟`constraint`数据结构相同，但是用来指定传递依赖的相关信息。

**version**

`version`是`constraint`和`override`中的属性，用来指定依赖的版本。支持版本的比较操作，例如：`~`，`=`等。如果不指定操作，默认是`^`操作，
只是限制最左非零的版本号，例如，`^1.2.3`表示`1.2.3 <= X < 2.0.0`，`^0.2.3`表示`0.2.3 <= X < 0.3.0`。

### Characteristics

Dep的特性：

* 支持语义化版本和丰富的版本比较操作
* 支持从其他依赖管理工具转换
* 支持依赖树的可视化

Dep存在的不足：

* 目前速度非常慢，性能有待提升
* 快速开发迭代中，版本不稳定

## Installation

MacOS上通过Homebrew安装:

```sh
$ brew install dep
$ brew upgrade dep
```

直接通过`go get`安装:

```sh
go get -u github.com/golang/dep/cmd/dep
```

## Usage

### Start the Management

```sh
$ dep init
```

对于一个项目开始使用Dep进行管理，直接在项目根目录下运行`dep init`。`dep init`会进行如下操作：

1. Look for [existing dependency management files](https://github.com/golang/dep/blob/master/docs/FAQ.md#what-external-tools-are-supported) to convert
1. Check if your dependencies use dep
1. Identify your dependencies
1. Back up your existing `vendor/` directory (if you have one) to `_vendor-TIMESTAMP/`
1. Pick the highest compatible version for each dependency
1. Generate [`Gopkg.toml`](https://github.com/golang/dep/blob/master/docs/Gopkg.toml.md) ("manifest") and `Gopkg.lock` files
1. Install the dependencies in `vendor/`

### Add dependencies

```sh
$ dep ensure
```

`dep ensure` will ensure the dependencies already in `vendor/` to match the constraints from the manifest, and install
the latest version allowed by the manifest for the missing dependencies in `vendor/`.

### Add a dependency

```sh
$ dep ensure -add github.com/golang/glog
"github.com/golang/glog" is not imported by your project, and has been temporarily added to Gopkg.lock and vendor/.
If you run "dep ensure" again before actually importing it, it will disappear from Gopkg.lock and vendor/.
```

`dep ensure -add` will update `Gopkg.toml` and `Gopkg.lock`, and install the dependency in `vendor/`.
If the dependency has been imported in your code, just need to run `dep ensure`.

### Update a dependency

1. Manually edit `Gopkg.toml` like change the `version/branch/revision`
2. Ensure the dependency

```sh
$ dep ensure
```

> In order to forcibly update a dependency, use `dep ensure -v -update {package_name}`.

### Remove a dependency

```sh
$ dep ensure
```

`dep ensure` will clean up the installed dependencies but not actually imported in code.

### Check status of dependency

```sh
$ dep status
Lock inputs-digest mismatch due to the following packages missing from the lock:

PROJECT                   MISSING PACKAGES
github.com/docker/docker  [github.com/docker/docker/client]

This happens when a new import is added. Run `dep ensure` to install the missing packages.
```

After import new dependency in code or add directly add it into `Gopkg.toml`, `dep status` will find that they are
missing in `Gopkg.lock`, and suggest you to run `dep ensure` to update lock file and install missing dependencies.

## Troubleshooting

* Fail to checkout version

Error:

```
Unable to update checked out version: : command failed: [git checkout 9f8ebd171479bec0ada837d7ee641dec2f8c6dd1]: exit status 1
```

Solution:

```
# echo $GOPATH
/Users/robin/gocode
# cd $GOPATH/pkg/dep
# ls -l
drwxr-xr-x  10 robin  staff  340  3 23 16:08 darwin_amd64        # Some deps are cached here, also need to remove them.
drwxr-xr-x   5 robin  staff  170  3 23 16:08 dep                 # Cached deps
drwxr-xr-x   3 root   staff  102  9  1  2017 linux_amd64         # 
# rm -rf $GOPATH/pkg/dep
```

* 传递依赖版本不符

依赖的依赖（传递依赖）不满足版本要求，从依赖的版本管理信息中，获得传递依赖的版本信息。然后在 `Gopkg.toml` 中通过 `override` 明确指定传递依赖的版本。

例如，通过 `dep init` 之后，仍然存在如下依赖问题：
```
# github.com/caicloud/cyclone/vendor/github.com/docker/docker/oci
vendor/github.com/docker/docker/oci/defaults_linux.go:19:11: unknown field 'Platform' in struct literal of type specs.Spec
vendor/github.com/docker/docker/oci/defaults_linux.go:62:25: cannot use []string literal (type []string) as type *specs.LinuxCapabilities in assignment
vendor/github.com/docker/docker/oci/defaults_linux.go:96:17: undefined: specs.Namespace
vendor/github.com/docker/docker/oci/defaults_linux.go:107:14: undefined: specs.Device
vendor/github.com/docker/docker/oci/defaults_linux.go:108:15: undefined: specs.Resources
vendor/github.com/docker/docker/oci/devices_linux.go:15:32: undefined: specs.Device
vendor/github.com/docker/docker/oci/devices_linux.go:27:38: undefined: specs.DeviceCgroup
vendor/github.com/docker/docker/oci/devices_linux.go:39:85: undefined: specs.Device
vendor/github.com/docker/docker/oci/devices_linux.go:39:116: undefined: specs.DeviceCgroup
vendor/github.com/docker/docker/oci/namespaces.go:6:44: undefined: specs.NamespaceType
vendor/github.com/docker/docker/oci/defaults_linux.go:108:15: too many errors
github.com/caicloud/cyclone/vendor/github.com/zoumo/register
```

根据错误提示，找到 Docker 依赖的 github.com/opencontainers/runtime-spec 存在版本不一致的问题。
当前 Docker 使用的版本是 `v1.13.1`， 根据其依赖管理工具记录的依赖版本信息 [vendor.conf](https://github.com/moby/moby/blob/v1.13.1/vendor.conf#L64)，
获得 github.com/opencontainers/runtime-spec 的准确版本号应该是 `1c7c27d043c2a5e513a44084d2b10d77d1402b8c`。因此，在 `Gopkg.toml` 中添加如下内容：

```
[[override]]
  name = "github.com/opencontainers/runtime-spec"
  revision = "1c7c27d043c2a5e513a44084d2b10d77d1402b8c"
```

说明：

* 因为指定的是传递依赖的信息，所以使用的是 `override`
* 因为直接指定的是commit id，所以使用的是 `revision`

## Reference

- [Golang Package Management Tools](https://github.com/golang/go/wiki/PackageManagementTools)
