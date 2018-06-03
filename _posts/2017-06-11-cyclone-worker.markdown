---
layout:     post
title:      "Cyclone源码分析: Worker"
subtitle:   "Cyclone Source Code Reading: Worker"
date:       2017-06-11
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
    - 源码分析
---

> The version of Cyclone source code is **`v0.2`**.

[Cyclone](https://github.com/caicloud/cyclone)是[才云科技](https://caicloud.io/)于2016年11月
开源的一款容器化CI/CD平台。它完全在容器中完成应用的构建、集成和部署，能够定义应用间的依赖关系。
Cyclone采用Master/Slave的架构，master是Cyclone-Server，主要负责应用的管理和发布任务的调度；slave是Cyclone-Worker，主要负责执行具体的发布任务。本文主要从源码角度分析Cyclone-Worker的实现。

## Overview

### [Workflow](https://github.com/caicloud/cyclone/blob/master/docs/developer-guide_zh-CN.md#工作流)

![workflow](/img/in-post/cyclone/flow.png)

Cyclone从版本控制服务器（例如：gitlab，github等）上获取源码，然后经过软件生命周期中的一系列过程，包括构建、
集成、部署等，最终发布到Kubernetes集群中。


### [Architecture](https://github.com/caicloud/cyclone/blob/master/docs/developer-guide_zh-CN.md#软件架构)

![architecture](/img/in-post/cyclone/architecture.png)

Cyclone-Server主要功能：
* 负责统一对外提供Restful API服务
* 作为Log server，接受来自Cyclone-Worker的log，并从Kafka中pull log返回给用户
* 管理所有项目信息和执行结果，并持久化到MongoDB中
* WorkerManager根据待处理事件动态地启动Cyclone-Worker

Cyclone-Worker主要功能：
* 拉取并解析待处理事件
* 为每个阶段启动新的容器来执行该阶段任务，并反馈执行结果
* 收集各阶段的执行日志，并发送到Cyclone-Server

Cyclone-Server会将处理时间较长的待处理任务写入ETCD，EventManager会watch ETCD中待处理任务的变化，
并将新增的待处理任务发送给WorkerManager。WorkerManager会调用Docker API，以容器的方式启动Cyclone-Worker，并将需要的相关参数通过环境变量的方式传入。

### Worker工作原理

![cyclone-worker](/img/in-post/cyclone/workflow-of-cyclone-worker.png)

caicloud.yml中的5个阶段：
- **Pre Build**：构建源码。指定构建环境image，将代码挂载到容器中，执行build命令。构建成功后，将构建outputs从构建容器中copy出来，供后面构建镜像使用。
- **Build**：构建镜像。根据context和Dockerfile构建镜像。
- **Integration**：根据服务间的依赖关系，启动依赖的服务。
- **Post Build**：构建结束后的hook，能够触发其他操作。
- **Deploy**：将新产生的最新镜像，deploy到指定的cluster中。

Cyclone-Worker启动后会立即根据Event Id，通过`getEvent()`向Cyclone-Server获得待处理任务的完整信息。
`Parse()`从Event中解析出任务步骤的节点树（Node Tree）。Node Tree主要包含3种类型的节点：
- **ListNode**: Holds a sequence of nodes.
- **DockerNode**: Build process should run in Docker container.
- **DeployNode**: For deploy section in yaml.

从Node Tree的根节点，递归地处理树上所有node。大部分构建步骤都是`DockerNode`类型，下面重点对这种类型的构建步骤的实现进行分析。
大致步骤一般如下：
1. 记录stepLog的开始；
2. 根据执行步骤生成构建容器的参数和启动命令；
3. 创建容器，并执行该步骤的脚本；
4. 记录stepLog的结束。

[worker/ci/runner/runner.go](https://github.com/caicloud/cyclone/blob/v0.1/worker/ci/runner/runner.go#L69-L191)
```Golang
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
			...
		case parser.NodeBuild:
			...
		case parser.NodePreBuild:
			...
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

Docker Client的`start()`只会create容器，不会stop和remove容器。它首先判断image是否存在，不存在先pull image，然后根据容器参数创建容器。

[worker/ci/runner/docker.go](https://github.com/caicloud/cyclone/blob/v0.1/worker/ci/runner/docker.go#L236-L270)
```Golang
// start a container with the given CreateContainerOptions.
func start(b *Build, cco *docker_client.CreateContainerOptions) (*docker_client.Container, error) {
	log.InfoWithFields("About to inspect the image.", log.Fields{"image": cco.Config.Image})
	result, err := b.dockerManager.IsImagePresent(cco.Config.Image)
	if err != nil {
		return nil, err
	}
	if result == false {
		log.InfoWithFields("About to pull the image.", log.Fields{"image": cco.Config.Image})
		err := b.dockerManager.PullImage(cco.Config.Image)
		if err != nil {
			return nil, err
		}
		log.InfoWithFields("Successfully pull the image.", log.Fields{"image": cco.Config.Image})
	}

	log.InfoWithFields("About to create the container.", log.Fields{"config": *cco})
	client := b.dockerManager.Client
	container, err := client.CreateContainer(*cco)
	if err != nil {
		return nil, err
	}
	err = client.StartContainer(container.ID, cco.HostConfig)
	if err != nil {
		// TODO: Check the error.
		client.RemoveContainer(docker_client.RemoveContainerOptions{
			ID: container.ID,
		})
		return nil, err
	}
	log.InfoWithFields("Successfully create the container.", log.Fields{"config": *cco})
	// Notice that the container wouldn't be removed before the return. So it should
	// be done at the runner.Build.TearDown().
	return container, nil
}
```

Docker Client的`run()`首先调用`start()`，然后单独起goroutine收集执行日志，执行成功后copy出outputs。无论执行是否成功，构建容器最终都会被stop并remove。

[worker/ci/runner/docker.go](https://github.com/caicloud/cyclone/blob/v0.1/worker/ci/runner/docker.go#L272-L362)
```Golang
// run a container with the given CreateContainerOptions, currently it
// involves: start the container, wait it to stop and record the log
// into output.
func run(b *Build, cco *docker_client.CreateContainerOptions,
	outPutFiles []string, outPutPath string, nodetype parser.NodeType, output filebuffer.FileBuffer) (*docker_client.Container, error) {
	// Fetches the container information.
	client := b.dockerManager.Client
	container, err := start(b, cco)
	if err != nil {
		return nil, err
	}
	// Ensures the container is always stopped
	// and ready to be removed.
	defer func() {
		if nodetype == parser.NodeIntegration {
			for _, ID := range b.ciServiceContainers {
				b.dockerManager.StopAndRemoveContainer(ID)
			}

			//number := len(b.ciServiceContainers)
		}
		client.StopContainer(container.ID, 5)
		client.RemoveContainer(docker_client.RemoveContainerOptions{
			ID: container.ID,
		})
	}()

	// channel listening for errors while the
	// container is running async.
	errc := make(chan error, 1)
	containerc := make(chan *docker_client.Container, 1)
	go func() {
		// Options to fetch the stdout and stderr logs
		// by tailing the output.
		logOptsTail := &docker_client.LogsOptions{
			Follow:       true,
			Stdout:       true,
			Stderr:       true,
			Container:    container.ID,
			OutputStream: output,
			ErrorStream:  output,
		}

		// It's possible that the docker logs endpoint returns before the container
		// is done, we'll naively resume up to 5 times if when the logs unblocks
		// the container is still reported to be running.
		for attempts := 0; attempts < 5; attempts++ {
			if attempts > 0 {
				// When resuming the stream, only grab the last line when starting
				// the tailing.
				logOptsTail.Tail = "1"
			}

			// Blocks and waits for the container to finish
			// by streaming the logs (to /dev/null). Ideally
			// we could use the `wait` function instead
			err := client.Logs(*logOptsTail)
			if err != nil {
				log.Errorf("Error tailing %s. %s\n", cco.Config.Image, err)
				errc <- err
				return
			}

			info, err := client.InspectContainer(container.ID)
			if err != nil {
				log.Errorf("Error getting exit code for %s. %s\n", cco.Config.Image, err)
				errc <- err
				return
			}

			if info.State.Running != true {
				containerc <- info
				return
			}
		}

		errc <- errors.New("Maximum number of attempts made while tailing logs.")
	}()

	select {
	case info := <-containerc:
		err = CopyOutPutFiles(b.dockerManager, container.ID, outPutFiles, outPutPath)
		if nil != err {
			return container, err
		}
		return info, nil
	case err := <-errc:
		log.InfoWithFields("Run the container failed.", log.Fields{"config": cco})
		return container, err
	}
}
```

# 评价

Cyclone基于容器实现了CI/CD，基本支持了持续构建、持续集成和持续部署。采用Master/Slave的架构，具有很好的可扩展性；
将长时间任务缓存到ETCD，提供了整个系统的稳定性和可靠性；通过Kafka收集和存储运行过程中的日志，也是一种较好的日志实时展示的解决方案。
Cyclone-Worker的实现思想与[Openshift Source-To-Image (S2I)](https://github.com/openshift/source-to-image)类似，可能在一定程度上有参考和借鉴。

个人认为，Cyclone在某些地方仍有改进的空间。系统比较复杂，引入Kafka、Zookeeper等比较重的组件，而它们仅仅只是为了解决日志实时展示的问题。自动化测试是CI/CD中不可或缺的环节，包括构建时的单元测试，部署后的集成测试以及性能测试，测试结果的持久化和展示也是一个非常重要的问题，而Cyclone这方面支持的比较少。每个步骤都重启启动一个容器，甚至从容器中将构建的outputs copy处理之后来build image。这样的成本比较高，严重影响效率，为什么不能直接在一个容器中把整个过程都跑完呢？Cyclone给出的解释是为了避免Docker in Docker，同时为了避免构建出来的image包含不必要的构建依赖，减小image size。本人认为这两个问题都有解决方案可以避免，更可能的原因是为了单独收集每个阶段的日志。