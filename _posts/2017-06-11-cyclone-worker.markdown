---
layout:     post
title:      "Cyclone源码分析: Worker"
subtitle:   "Cyclone Source Code Reading: Worker"
date:       2017-06-11
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Golang
    - 源码分析
---

[Cyclone](https://github.com/caicloud/cyclone)是[才云科技](https://caicloud.io/)于2016年11月
开源的一款容器化CI/CD平台。它完全在容器中完成应用的构建、集成和部署，能够定义应用间的依赖关系。
Cyclone采用Master/Slave的架构，master是Cyclone-Server，主要负责应用的管理和发布任务的调度；slave是Cyclone-Worker，主要负责执行具体的发布任务。本文主要从源码角度分析Cyclone-Worker的实现。


## Overview

### [Workflow](https://github.com/caicloud/cyclone/blob/master/docs/developer-guide_zh-CN.md#工作流)

![workflow](https://github.com/caicloud/cyclone/blob/master/docs/flow.png)

Cyclone从版本控制服务器（例如：gitlab，github等）上获取源码，然后经过软件生命周期中的一系列过程，包括构建、
集成、部署等，最终发布到Kubernetes集群中。


### [Architecture](https://github.com/caicloud/cyclone/blob/master/docs/developer-guide_zh-CN.md#软件架构)

![architecture](https://github.com/caicloud/cyclone/blob/master/docs/architecture.png)

Cyclone-Server主要功能：
* 负责统一对外提供Restful API服务
* 作为Log server，从Kafka中pull log并返回给用户
* 管理所有项目信息，并持久化到MongoDB中
* WorkerManager根据待处理事件动态地启动Cyclone-Worker

Cyclone-Worker主要功能：
* 拉取并解析待处理事件
* 为每个阶段启动新的容器来执行该阶段任务，并反馈执行结果
* 收集各阶段的执行日志，并发送到Kafka

Cyclone-Server会将处理时间较长的待处理任务写入ETCD，EventManager会watch ETCD中待处理任务的变化，
并将新增的待处理任务发送给WorkerManager。WorkerManager会调用Docker API，以容器的方式启动Cyclone-Worker，并将需要的相关参数通过环境变量的方式传入。

### Worker工作原理

![cyclone-worker](/img/in-post/cyclone/workflow-of-cyclone-worker.png)

caicloud.yml中的5个阶段：
- **Pre Build**：构建源码。指定构建环境image，将代码挂载到容器中，执行build命令。构建成功后，将构建outputs从构建容器中copy出来，供后面构建镜像使用。
- **Build**：构建镜像。根据context和Dockerfile构建镜像。
- **Integration**：根据服务间的依赖关系，启动依赖的服务。
- **Post Build**：
- **Deploy**：将新产生的最新镜像，deploy到指定的cluster中。

Cyclone-Worker启动后会立即根据Event Id，通过`getEvent()`向Cyclone-Server获得待处理任务的完整信息。
`Parse()`从Event中解析出任务步骤的节点树（Node Tree）。Node Tree主要包含3种类型的节点：
- **ListNode**: Holds a sequence of nodes.
- **DockerNode**: Build process should run in Docker container.
- **DeployNode**: For deploy section in yml.

大部分构建步骤都是`DockerNode`类型，下面重点对这种类型的构建步骤的实现进行分析。

[worker/ci/runner/runner.go](https://github.com/caicloud/cyclone/blob/v0.1/worker/ci/runner/runner.go#L69-L191)
```go
// RunNode walks through the tree, run the build job.
func (b *Build) RunNode(flags parser.NodeType) error {
	b.flags = flags
	return b.walk(b.tree.Root)
}

// walk through the tree, recursively.
func (b *Build) walk(node parser.Node) (err error) {
	outPutPath := b.event.Data["context-dir"].(string) + "/"
	switch node := node.(type) {
	case *parser.ListNode:
		for _, node := range node.Nodes {
			err = b.walk(node)
			if err != nil {
				return err
			}
		}

	case *parser.DockerNode:
		if shouldSkip(b.flags, node.NodeType) {
			break
		}
		if isLackOfCriticalConfig(node) {
			break
		}

		switch node.Type() {
		case parser.NodeService:
			// Record image name
			createContainerOptions := toServiceContainerConfig(node, b)

			// Run the docker container.
			container, err := start(b, createContainerOptions)
			if err != nil {
				return err
			}
			// Set the container ID, to stop and remove the containers at
			// post hook function.
			b.ciServiceContainers = append(b.ciServiceContainers, container.ID)

		case parser.NodeIntegration:
			// Record image name
			createContainerOptions := toBuildContainerConfig(node, b, parser.NodeIntegration)
			// Encode the commands to one line script.
			Encode(createContainerOptions, node)

			// Run the docker container.
			container, err := run(b, createContainerOptions, node.Outputs, outPutPath, node.Type(), steplog.Output)
			if err != nil {
				return err
			}

			// Check the exitcode from container
			if container.State.ExitCode != 0 {
				return fmt.Errorf("container meets error")
			}

		case parser.NodeBuild:
			log.Info("Build with Dockerfile path: ", node.DockerfilePath,
				" ", node.DockerfileName)
			if err := b.dockerManager.BuildImageSpecifyDockerfile(b.event,
				node.DockerfilePath, node.DockerfileName, steplog.Output); err != nil {
				return err
			}

		case parser.NodePreBuild:
			// Record image name
			steplog.InsertStepLog(b.event, steplog.PreBuild, steplog.Start, nil)
			if "" != node.DockerfilePath || "" != node.DockerfileName {
				log.Info("Pre_build with Dockerfile path: ", node.DockerfilePath,
					" ", node.DockerfileName)
				errDockerfile := preBuildByDockerfile(steplog.Output, b.dockerManager,
					b.event, node.DockerfilePath, node.DockerfileName, node.Outputs,
					outPutPath)
				if nil != errDockerfile {
					steplog.InsertStepLog(b.event, steplog.PreBuild, steplog.Stop, errDockerfile)
					return errDockerfile
				}
			} else {
				createContainerOptions := toBuildContainerConfig(node, b, parser.NodePreBuild)
				// Encode the commands to one line script.
				Encode(createContainerOptions, node)

				// Run the docker container.
				container, err := run(b, createContainerOptions, node.Outputs, outPutPath, node.Type(), steplog.Output)
				if err != nil {
					steplog.InsertStepLog(b.event, steplog.PreBuild, steplog.Stop, err)
					return err
				}
				// Check the exitcode from container
				if container.State.ExitCode != 0 {
					errExit := fmt.Errorf("container meets error")
					steplog.InsertStepLog(b.event, steplog.PreBuild, steplog.Stop, errExit)
					return errExit
				}
			}
			steplog.InsertStepLog(b.event, steplog.PreBuild, steplog.Finish, nil)

		case parser.NodePostBuild:
			// Record image name
			steplog.InsertStepLog(b.event, steplog.PostBuild, steplog.Start, nil)
			// create the container with default network.
			createContainerOptions := toBuildContainerConfig(node, b, parser.NodePostBuild)
			// Encode the commands to one line script.
			Encode(createContainerOptions, node)

			// Run the docker container.
			container, err := run(b, createContainerOptions, node.Outputs, outPutPath, node.Type(), steplog.Output)
			if err != nil {
				steplog.InsertStepLog(b.event, steplog.PostBuild, steplog.Stop, err)
				return err
			}
			// Check the exitcode from container
			if container.State.ExitCode != 0 {
				errExit := fmt.Errorf("container meets error")
				steplog.InsertStepLog(b.event, steplog.PostBuild, steplog.Stop, errExit)
				return errExit
			}
			steplog.InsertStepLog(b.event, steplog.PostBuild, steplog.Finish, nil)
		}
	}
	return nil
}
```
