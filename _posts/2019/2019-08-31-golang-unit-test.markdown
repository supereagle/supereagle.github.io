---
layout:     post
title:      "Golang 中如何做好单元测试"
subtitle:   ""
date:       2019-08-31
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
---

单元测试是保证软件质量的最基本的测试，也是各种测试中执行成本较低、粒度较小的一种。由开发者在开发软件功能的同时，添加单元测试代码来保证各函数的逻辑的正确性。
如果很多缺陷能够通过单元测试发现，被开发者在开发功能过程中解决，就能避免这些缺陷进入软件生命周期的后续阶段，极大地减少解决这些缺陷的成本。
因此，保证代码的单元测试覆盖率，是一个程序员职业素养的重要体现，也会软件质量的重要保证。本文主要介绍 Golang 中如何做好单元测试。


## Unit Test Coverage

Golang 不仅自带跑单元测试的命令，同时也提供单元测试覆盖率的统计和分析工具。下面以开源 Golang 项目 [Cyclone](https://github.com/caicloud/cyclone) 为例，详细介绍如何清晰地了解你代码的单元测试覆盖率。

提前下载 Cyclone 项目的源码到本地，并进入该项目的目录下

```shell
$ go get github.com/caicloud/cyclone
$ cd $GOPATH/src/github.com/caicloud/cyclone
```

### Unit Test

为了分析单元测试覆盖率，首先需要运行单元测试。在平时运行单元测试 `go test` 命令后面需要加入 `-coverprofile` 将各个 package 的单元测试覆盖率的概况信息输出到文件中。

```shell
# 获得需要分析单元测试覆盖率的 package，可以过滤掉 vendor 等特殊目录
$ PKGS=$(go list ./... | grep -v /vendor | grep -v /test)
# 运行单元测试，同时将测试覆盖率结果输出到文件
$ go test $PKGS -coverprofile=coverage.out
...
?       github.com/caicloud/cyclone/pkg/workflow/coordinator/cycloneserver      [no test files]
?       github.com/caicloud/cyclone/pkg/workflow/coordinator/k8sapi     [no test files]
ok      github.com/caicloud/cyclone/pkg/workflow/values/ref     0.034s  coverage: 88.0% of statements
ok      github.com/caicloud/cyclone/pkg/workflow/workflowrun    3.054s  coverage: 33.6% of statements
?       github.com/caicloud/cyclone/pkg/workflow/workload/delegation    [no test files]
ok      github.com/caicloud/cyclone/pkg/workflow/workload/pod   0.032s  coverage: 61.4% of statements
?       github.com/caicloud/cyclone/tools/generator/client-gen  [no test files]
ok      github.com/caicloud/cyclone/tools/generator/client-gen/args     0.012s  coverage: 40.3% of statements
?       github.com/caicloud/cyclone/tools/generator/client-gen/generators       [no test files]
?       github.com/caicloud/cyclone/tools/generator/client-gen/generators/fake  [no test files]
?       github.com/caicloud/cyclone/tools/generator/client-gen/generators/scheme        [no test files]
ok      github.com/caicloud/cyclone/tools/generator/client-gen/generators/util  0.010s  coverage: 78.0% of statements
?       github.com/caicloud/cyclone/tools/generator/client-gen/path     [no test files]
ok      github.com/caicloud/cyclone/tools/generator/client-gen/types    0.009s  coverage: 26.1% of statements
...
```

### Unit Test Analysis

上面产生的 `coverage.out` 是各 package 的单元测试覆盖率信息，接下来需要用 `go tool cover` 对 `coverage.out` 进行分析，获得整个项目的单元测试覆盖率。

```shell
$ go tool cover -func coverage.out | tail -n 1 | awk '{ print "Total coverage: " $3 }'
Total coverage: 42.6%
```

如果整体的单元测试覆盖率较低，需要知道每段具体代码的单元测试覆盖情况，便于有针对增加单元测试，可以通过下面命令展示更详细信息。

```shell
$ go tool cover -html coverage.out
```

上述命令会自动打开浏览器，明确指出每个文件中的每段代码的单元测试覆盖情况，如下图所示：

![golang-ut-coverage](/img/in-post/golang-ut/ut-coverage.png)

## Reference

- [Command go](https://golang.org/cmd/go/#hdr-Test_packages)
