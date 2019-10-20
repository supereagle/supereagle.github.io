---
layout:     post
title:      "Cyclone源码分析: Server"
subtitle:   "Cyclone Source Code Reading: Server"
date:       2017-08-19
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
Cyclone采用Master/Slave的架构，master是Cyclone-Server，主要负责应用的管理和发布任务的调度；slave是Cyclone-Worker，主要负责执行具体的发布任务。之前的一篇博客[Cyclone源码分析: Worker](/2017/06/11/cyclone-worker/)已经从源码角度分析Cyclone-Worker的实现。本文主要从源码角度分析Cyclone-Server的实现。

## Overview

### [Architecture](https://github.com/caicloud/cyclone/blob/master/docs/developer-guide_zh-CN.md#软件架构)

![architecture](/img/in-post/cyclone/architecture.png)

**`注意：架构中有处细小的错误，Cyclone-Worker不会直接将log push到Kafka中，而是先push到Cyclone-Server中的Log-server组件，然后由Log-server再push到Kafka。`**

Cyclone-Server主要功能：
* 负责统一对外提供Restful API服务
* 作为Log server，从Kafka中pull log并返回给用户
* 管理所有项目信息，并持久化到MongoDB中
* WorkerManager根据待处理事件动态地启动Cyclone-Worker

Cyclone-Worker主要功能：
* 拉取并解析待处理事件
* 为每个阶段启动新的容器来执行该阶段任务，并反馈执行结果
* 收集各阶段的执行日志，并发送到Cyclone-Server

### 源码目录结构

```shell
# tree -d -I vendor
.
├── api
│   ├── rest            # 注册REST API
│   ├── server          # 初始化并启动server
│   └── templates       # 提供Dockerfile和caicloud.yaml的模板
│       ├── dockerfile
│       └── yaml
├── cloud               # 支持k8s和docker两种cloud来提供worker
├── cmd                 # 程序入口
│   ├── server
│   └── worker
├── docker              # 封装Docker client，提供container和image的各种基本操作
├── docs                # 相关docs
├── etcd                # 封装ETCD client
├── event               # 管理event
├── http                # 启动log server，仅供获取log（这个会用到吗？）
│   └── web
├── kafka               # 封装Kafka client
├── node_modules
│   └── swagger-ui      # Swagger UI的Web内容
├── notify              # 通知构建结果
│   └── provider
├── pkg                 # 一些基本的utils
│   ├── executil
│   ├── filebuffer
│   ├── kubefaker
│   ├── log
│   ├── osutil
│   ├── pathutil
│   ├── register
│   └── wait
├── remote              # SVC相关操作的接口（主要是auth、webhook相关）
│   └── provider        # SVC相关操作的具体实现，支持Github、Gitlab
├── scripts             # 一些自动化脚本
│   ├── buildin
│   ├── clair_config
│   ├── haproxy
│   ├── k8s
│   ├── lib
│   ├── registry
│   │   └── ssl
│   ├── tools
│   └── worker_image
├── store               # 各resource的MongoDB操作
├── tests               # e2e Test
│   ├── common
│   ├── fake_repo_for_test
│   │   ├── deploy
│   │   └── Integration
│   ├── project
│   ├── service
│   ├── version
│   └── yaml
├── utils               # 一些deploy相关的utils
├── websocket           # Websocket for log server
└── worker              # Worker的具体实现
    ├── ci              # caicloud.yml中各阶段的具体实现
    │   ├── parser      # caicloud.yml解析器，将其解析为一个tree
    │   ├── runner      # 在容器中执行各阶段
    │   └── yaml        # caicloud.yml配置对应的数据结构
    ├── clair           # 通过Klar集成Clair，对镜像进行安全扫描
    ├── handler         # Worker与server交互的API，get/set event
    ├── helper
    ├── log             # 记录并push log到log server
    └── vcs             # SVC相关操作的接口（主要是clone、tag和commit相关）
        └── provider
```

> 目录结构不是很清晰，主要的source code跟其他的`docs`、`tests`、`scripts`、`vendor`等一起放在根目录下，比较零散，不便于识别和阅读。

## Server工作原理

### Server Startup

Server启动的入口是[cmd/server/server.go](https://github.com/caicloud/cyclone/blob/master/cmd/server/server.go)，启动时的主要工作却在[api/server/server.go](https://github.com/caicloud/cyclone/blob/master/api/server/server.go)中完成的。

[api/server/server.go](https://github.com/caicloud/cyclone/blob/master/api/server/server.go)
```golang
......
// PrepareRun prepare for apiserver running
func (s *APIServer) PrepareRun() (*PreparedAPIServer, error) {

	s.InitLog()
	cloud.Debug = s.Config.Debug
	logdog.Debugf("Debug mode: %t", s.Config.Debug)

	// init api doc
	if s.Config.ShowAPIDoc {
		// Open http://localhost:7099/apidocs and enter http://localhost:7099/apidocs.json in the api input field.
		config := swagger.Config{
			WebServices:    restful.RegisteredWebServices(), // you control what services are visible.
			WebServicesUrl: fmt.Sprintf(s.Config.CycloneAddrTemplate, s.Config.CyclonePort),
			ApiPath:        "/apidocs.json",

			// Optionally, specify where the UI is located.
			SwaggerPath:     "/apidocs/",
			SwaggerFilePath: "./node_modules/swagger-ui/dist",
		}
		swagger.InstallSwaggerService(config)
	}

	// init database
	err := s.InitStore()
	if err != nil {
		return nil, err
	}

	// init event manager
	err = s.initEventManager()
	if err != nil {
		return nil, err
	}

	// init rest api server
	rest.Initialize()

	return &PreparedAPIServer{s}, nil
}

......

// Run start a api server
func (s *PreparedAPIServer) Run(stopCh <-chan struct{}) error {

	// TODO: PostStartHooks

	// start log server
	go s.StartLogServer()

	// start server
	server := &http.Server{Addr: fmt.Sprintf(":%d", s.Config.CyclonePort), Handler: restful.DefaultContainer}
	logdog.Infof("cyclone server listening on %d", s.Config.CyclonePort)
	logdog.Fatal(server.ListenAndServe())
	// <-stopCh
	return nil
}
......
```

Server启动的主要工作：
* Init程序自身的log系统
* 集成Swagger UI和SPEC展示API
* Init MongoDB
* Init event manager，主要是ETCD
* Init REST API
* 启动log server用来向Kafka push/pull各阶段执行日志
* 启动service服务

### Log Server

Log server是整个server中比较重要的，也是相对比较复杂的组件。Log server是一个Web socket的server，根据socket中的data内容，进行相应的操作。首先是将`*websocket.Conn`封装成自定义的数据结构`WSSession`，然后开启一个for循环，重复地执行：
1. 更新Web socket的read deadline，在当前时间基础上加一定的时间`IdleCheckInterval`，默认是30s。
2. 从Web socket读取字节流，然后封装为`DataPacket`数据结构。
3. 然后通过`WSSession.OnReceive()`对读取的数据进行处理。如果处理成功，则更新Web socket的ActiveTime，继续读取后面字节流。
4. 如果数据读取出错，或者一定时间内没有读取到正确的数据，就关闭session。

[websocket/server.go](https://github.com/caicloud/cyclone/blob/master/websocket/server.go)
```golang
//StartServer start the websocket server
func StartServer() (err error) {
	scServerConfig := GetConfig()
	logdog.Infof("Start Websocket Server at Port:%d", scServerConfig.Port)

	hsServer := &http.Server{
		Addr:    fmt.Sprintf(":%d", scServerConfig.Port),
		Handler: websocket.Handler(webMessageHandle),
	}
	return hsServer.ListenAndServe()
}

//webMessageHandle handle the message receive from web client
func webMessageHandle(wsConn *websocket.Conn) {
	wssSession, err := CreateWSSession(wsConn)
	if err != nil {
		wsConn.Close()
		logdog.Error(err.Error())
		return
	}
	defer wssSession.OnClosed()

	nIdleCheckInterval := GetConfig().IdleCheckInterval
	for {
		//timeout
		err = wsConn.SetReadDeadline(time.Now().Add(time.Second * time.Duration(nIdleCheckInterval)))
		if err != nil {
			logdog.Error(err.Error())
			return
		}

		//receive msg
		var byrMesssage []byte
		err = websocket.Message.Receive(wsConn, &byrMesssage)
		if err == nil {
			if receiveData(wssSession, byrMesssage) {
				wssSession.UpdateActiveTime()
			}
			continue
		}

		//error and timeout handler
		e, ok := err.(net.Error)
		if !ok || !e.Timeout() {
			logdog.Error(err.Error())
			return
		} else {
			if wssSession.SessionTimeoverCheck() {
				return
			}
		}
	}
}
```

将Web socket字节流Unmarshal为一个map，然后根据其中的`action`来选择具体的执行逻辑。
支持3种类型的action：
* **watch_log**: Web client请求实时地watch log。会不断地从Kafka中pull日志，然后发送给web client，直到收到`stop`的请求。
* **heart_beat**: Client发送过来的心跳检测，不需额外处理。
* **worker_push_log**: 收到Worker push过来的log，直接push到Kafka。

> 所有log最终都是落到MongoDB中。引入Kafka的目的，仅仅是为了中间临时存一下log，主要满足2方面的需求：1. Web client watch log的时候，能够从Kafka实时地pull log；2. 所有阶段结束之后，一次性将log从Kafka中存储到MongoDB中。

[websocket/application.go](https://github.com/caicloud/cyclone/blob/master/websocket/application.go):
```golang
//AnalysisMessage analysis message receive from the web client.
func AnalysisMessage(dp *DataPacket) bool {
	sReceiveFrom := dp.GetReceiveFrom()
	jsonPacket := dp.GetData()

	defer unmarshalRecover(sReceiveFrom)
	var mapData map[string]interface{}

	if err := json.Unmarshal(jsonPacket, &mapData); err != nil {
		panic(err)
	}

	sAction := mapData["action"].(string)
	MsgHandler, bFoundAction := MsgHandlerMap[sAction]
	if bFoundAction {
		MsgHandler(sReceiveFrom, jsonPacket)
		return true
	}
	log.Infof("ws server recv unknow packet:%s", sAction)
	return false
}
......
// MsgHandler is the type for message handler.
type MsgHandler func(sReceiveFrom string, jsonPacket []byte)

//MsgHandlerMap is the map of web client message handler
var MsgHandlerMap = map[string]MsgHandler{
	"watch_log":       watchLogHandler,
	"heart_beat":      heartBeatHandler,
	"worker_push_log": workerPushLogHandler,
}

//watchLogHandler handle the watch log message
func watchLogHandler(sReceiveFrom string, jsonPacket []byte) {
	//Handle watch_log data
	log.Infof("receive watch_log packet")

	pWatchLog := &WatchLogPacket{}
	if err := json.Unmarshal(jsonPacket, &pWatchLog); err != nil {
		panic(err)
	}

	sTopic := CreateTopicName(pWatchLog.Api, pWatchLog.UserId,
		pWatchLog.ServiceId, pWatchLog.VersionId)
	wss := GetSessionList().GetSession(sReceiveFrom).(*WSSession)
	if "start" == pWatchLog.Operation {
		wss.SetTopicEnable(sTopic, true)
		go PushTopic(wss, pWatchLog)
	} else if "stop" == pWatchLog.Operation {
		wss.SetTopicEnable(sTopic, false)
	}

	byrResponse := PacketResponse(pWatchLog.Action, pWatchLog.Id,
		Error_Code_Successful)
	dpPacket := &DataPacket{
		byrFrame:  byrResponse,
		nFrameLen: len(byrResponse),
		sSendTo:   wss.sSessionID,
	}
	wss.Send(dpPacket)
}

//heartBeatHandler handle heart beat message
func heartBeatHandler(sReceiveFrom string, jsonPacket []byte) {
	//Handle heart_beat data
	log.Infof("receive heart_beat packet")
}

//pushLogHandler handle the watch log message
func workerPushLogHandler(sReceiveFrom string, jsonPacket []byte) {
	//Handle watch_log data

	workerPushLog := &WorkerPushLogPacket{}
	if err := json.Unmarshal(jsonPacket, &workerPushLog); err != nil {
		panic(err)
	}

	// log.Debugf("Worker log (%s): %s", workerPushLog.Topic, workerPushLog.Log)
	kafka.Produce(workerPushLog.Topic, []byte(workerPushLog.Log))
}
```

### Event Manager

API-server收到一些任务请求之后，首先写入ETCD中，然后由event manager watch ETCD中的`/events/unfinished/`，最后对这些event进行处理。引入ETCD的主要目的是防止任务太多或者任务执行时间过长，系统负载过高导致一些任务丢失的情况。其实可以简单地认为是一个任务队列，起到提高系统稳定性和可靠性的作用。

[event/manager.go](https://github.com/caicloud/cyclone/blob/master/event/manager.go)
```golang
// Init init event manager
// Step1: init event operation map
// Step2: new a etcd client
// Step3: load unfinished events from etcd
// Step4: create a unfinished events watcher
// Step5: new a remote api manager
func Init(wopts *cloud.WorkerOptions, cloudAutoDiscovery bool) {

	initCloudController(wopts, cloudAutoDiscovery)

	initOperationMap()

	etcdClient := etcd.GetClient()

	if !etcdClient.IsDirExist(EventsUnfinished) {
		err := etcdClient.CreateDir(EventsUnfinished)
		if err != nil {
			log.Errorf("init event manager create events dir err: %v", err)
			return
		}
	}

	GetList().loadListFromEtcd(etcdClient)
	initPendingQueue()

	go watchEtcd(etcdClient)
	go handlePendingEvents()

	remoteManager = remote.NewManager()
}
......
// watchEtcd watch unfinished events status change in etcd
func watchEtcd(etcdClient *etcd.Client) {
	watcherUnfinishedEvents, err := etcdClient.CreateWatcher(EventsUnfinished)
	if err != nil {
		log.Fatalf("watch unfinshed events err: %v", err)
	}

	for {
		change, err := watcherUnfinishedEvents.Next(context.Background())
		if err != nil {
			log.Fatalf("watch unfinshed events next err: %v", err)
		}

		switch change.Action {
		case etcd.WatchActionCreate:
			log.Infof("watch unfinshed events create: %s\n", change.Node)
			event, err := loadEventFromJSON(change.Node.Value)
			if err != nil {
				log.Errorf("analysis create event err: %v", err)
				continue
			}
			eventCreateHandler(&event)

		case etcd.WatchActionSet:
			log.Infof("watch unfinshed events set: %s\n", change.Node.Value)
			event, err := loadEventFromJSON(change.Node.Value)
			if err != nil {
				log.Errorf("analysis set event err: %v", err)
				continue
			}

			if change.PrevNode == nil {
				eventCreateHandler(&event)
			} else {
				preEvent, preErr := loadEventFromJSON(change.PrevNode.Value)
				if preErr != nil {
					log.Errorf("analysis set pre event err: %v", preErr)
					continue
				}
				eventChangeHandler(&event, &preEvent)
			}

		case etcd.WatchActionDelete:
			log.Infof("watch finshed events delete: %s\n", change.PrevNode)
			event, err := loadEventFromJSON(change.PrevNode.Value)
			if err != nil {
				log.Errorf("analysis delete event err: %v", err)
				continue
			}
			eventRemoveHandler(&event)

		default:
			log.Warnf("watch unknow etcd action(%s): %v", change.Action, change)
		}
	}
}

// eventCreateHandler handler when watched a event created
func eventCreateHandler(event *api.Event) {
	GetList().addUnfinshedEvent(event)
	pendingEvents.In(event)

	return
}

// eventChangeHandler handler when watched a event changed
func eventChangeHandler(event *api.Event, preEvent *api.Event) {
	GetList().addUnfinshedEvent(event)

	// event handle finished
	if !IsEventFinished(preEvent) && IsEventFinished(event) {
		postHookEvent(event)
		etcdClient := etcd.GetClient()
		err := etcdClient.Delete(EventsUnfinished + string(event.EventID))
		if err != nil {
			log.Errorf("delete finished event err: %v", err)
		}
	}
}

// eventChangeHandler handler when watched a event removed
func eventRemoveHandler(event *api.Event) {
	GetList().removeEvent(event.EventID)
}
......
// handlePendingEvents polls event queues and handle events one by one.
func handlePendingEvents() {
	for {
		if pendingEvents.IsEmpty() {
			time.Sleep(time.Second * 1)
			continue
		}

		event := *pendingEvents.GetFront()
		err := handleEvent(&event)
		if err != nil {
			if cloud.IsAllCloudsBusyErr(err) {
				log.Info("All system worker are busy, wait for 10 seconds")
				time.Sleep(time.Second * 10)
				continue
			}

			// remove the event from queue which had run
			pendingEvents.Out()

			event.Status = api.EventStatusFail
			event.ErrorMessage = err.Error()
			log.Error("handle event err", log.Fields{"error": err, "event": event})
			postHookEvent(&event)
			etcdClient := etcd.GetClient()
			err := etcdClient.Delete(EventsUnfinished + string(event.EventID))
			if err != nil {
				log.Errorf("delete finished event err: %v", err)
			}
			continue
		}

		// remove the event from queue which had run
		pendingEvents.Out()
		event.Status = api.EventStatusRunning
		ds := store.NewStore()
		defer ds.Close()
		if event.Operation == "create-version" {
			event.Version.Status = api.VersionRunning
			if err := ds.UpdateVersionDocument(event.Version.VersionID, event.Version); err != nil {
				log.Errorf("Unable to update version status post hook for %+v: %v", event.Version, err)
			}
		}
		SaveEventToEtcd(&event)
	}
}
```

Event manager会在server启动的时候进行init，init的主要操作是：
* initOperationMap()：为create-service和create-version这两种operation的event指定handler和postHook。
* initPendingQueue()：从ETCD中list所有event，并将pending状态的event加入pending queue中。
* watchEtcd(etcdClient)：单独起协程来watch ETCD中event的状态变化。
* handlePendingEvents()：单独起协程来处理pending queue中的任务。

watchEtcd(etcdClient)协程的主要工作是不断地watch ETCD中unfinished event的状态变化，包括create、update和delete，分别调用响应的handler进行处理：
* create：将该event添加到event list和pending event queue中。
* update：将该event添加到event list中，如果状态由unfinished变为finished，触发postHook，并从ETCD中删除该event。
* delete：将该event从event list中删除。

handlePendingEvents()协程的主要工作是不断地处理pending queue中的任务：
* 如果pending queue非空，获得第一个event进行处理
* 如果是cloud busy错误，就等待10s后继续处理该event
* 如果处理失败，就从pending queue和ETCD中删除该任务，同时更新event status，并执行postHook
* 如果处理成功，从pending queue中删除该任务，如果event是`create-version`操作，还会在MongoDB创建version记录
* 最后更新event的状态到ETCD中
