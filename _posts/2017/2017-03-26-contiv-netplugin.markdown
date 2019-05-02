---
layout:     post
title:      "Contiv源码分析：Netplugin"
subtitle:   "Contiv Source Code Reading: Netplugin"
date:       2017-03-26
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Contiv
    - Docker
    - 源码分析
---

> The version of Contiv source code is **`v1.0.0-alpha-01-28-2017.10-23-11.UTC`**.

## 程序启动

Netplugin首先读取并解析命令行参数，然后根据参数配置，New一个Agent，主要工作是有Agent完成。Agent进行一系列的初始化后，会不停地监听并处理网络状态存储（ETCD或者Consul）中的event。

[contiv/netplugin/netplugin/netd.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/netplugin/netd.go#L111)

```
func main() {
	// 读取并解析命令行参数
	...
	
	// initialize the config
	pluginConfig := plugin.Config{
		Drivers: plugin.Drivers{
			Network: "ovs",
			State:   stateStore,
		},
		Instance: core.InstanceInfo{
			HostLabel:  opts.hostLabel,
			CtrlIP:     opts.ctrlIP,
			VtepIP:     opts.vtepIP,
			UplinkIntf: opts.vlanIntf,
			DbURL:      opts.dbURL,
			PluginMode: opts.pluginMode,
		},
	}

	// Create a new agent
	ag := agent.NewAgent(&pluginConfig)

	// Process all current state
	ag.ProcessCurrentState()

	// post initialization processing
	ag.PostInit()

	// handle events
	if err := ag.HandleEvents(); err != nil {
		log.Infof("Netplugin exiting due to error: %v", err)
		os.Exit(1)
	}
}
```

命令行参数：
- **plugin-mode**：plugin mode，支持docker和kubernetes，默认是docker。其实还有mesos plugin，只不过这种plugin是默认开启的，不需要配置。
- **vtep-ip**：VTEP ip address，默认是本机IP。
- **ctrl-ip**：Local ip address to be used for control communication，默认是本机IP。
- **vlan-if**：VLAN接口。
- **cluster-store**：网络状态存储url，不支持多url的集群配置。目前支持ETCD和Consul两种，后面介绍中Netplugin的stateStore默认为ETCD。

## Netplugin功能组件

### Agent

NewAgent()主要根据Netplugin Config来进行Initialize各模块：
- **Service Registry**：创建bridge，能够AddService()和RemoveService()。
- **Cluster State**：根据`cluster-store`的url初始化KV store client。
- **Driver Plugins**：初始化StateDriver和NetworkDriver。
- **Orchestration Plugins**：根据`plugin-mode`参数，选择性初始化docker或者kubernetes plugin，mesos plugin是一定会初始化的。

[contiv/netplugin/netplugin/agent/agent.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/netplugin/agent/agent.go#L47)

```
// NewAgent creates a new netplugin agent
func NewAgent(pluginConfig *plugin.Config) *Agent {
	opts := pluginConfig.Instance
	netPlugin := &plugin.NetPlugin{}

	// Initialize service registry plugin
	svcPlugin, quitCh, err := svcplugin.NewSvcregPlugin(opts.DbURL, nil)
	if err != nil {
		log.Fatalf("Error initializing service registry plugin")
	}

	// init cluster state
	err = cluster.Init(opts.DbURL)
	if err != nil {
		log.Fatalf("Error initializing cluster. Err: %v", err)
	}

	// Init the driver plugins..
	err = netPlugin.Init(*pluginConfig)
	if err != nil {
		log.Fatalf("Failed to initialize the plugin. Error: %s", err)
	}

	// Initialize appropriate plugin
	switch opts.PluginMode {
	case "docker":
		dockplugin.InitDockPlugin(netPlugin, svcPlugin)

	case "kubernetes":
		k8splugin.InitCNIServer(netPlugin)

	case "test":
		// nothing to do. internal mode for testing
	default:
		log.Fatalf("Unknown plugin mode -- should be docker | kubernetes")
	}
	// init mesos plugin
	mesosplugin.InitPlugin(netPlugin)

	// create a new agent
	agent := &Agent{
		netPlugin:    netPlugin,
		pluginConfig: pluginConfig,
		svcPlugin:    svcPlugin,
		svcQuitCh:    quitCh,
	}

	return agent
}
```

ProcessCurrentState()会处理从ETCD中读取当前网络状态，并在节点上初始化当前的网络环境。
不然，一台临时新加入节点上的Pod，将无法正常与其他节点上的Pod进行正常通信。

PostInit()将该节点加入到Cluster Services中。Register三个services，netplugin支持两个RPC端口（9002和9003），以及netplugin.vtep的4789端口。
单独起一个Goroutine执行peerDiscoveryLoop()来进行Netmaster和Netplugin的服务发现。Watch ETCD中的两个Services：netmaster.rpc(master add/del event)，netplugin.vtep（node add/del event）。
最后启动service监听9090端口，主要是提供REST API来获得NetworkDriver的信息。

[contiv/netplugin/netplugin/agent/agent.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/netplugin/agent/agent.go#L179)

```
// PostInit post initialization
func (ag *Agent) PostInit() error {
	opts := ag.pluginConfig.Instance

	// Initialize clustering
	err := cluster.RunLoop(ag.netPlugin, opts.CtrlIP, opts.VtepIP, opts.HostLabel)
	if err != nil {
		log.Errorf("Error starting cluster run loop")
	}

	// start service REST requests
	ag.serveRequests()

	return nil
}
```

### StateDriver

Netplugin中最重要的两个driver，分别是StateDriver和NetworkDriver，都是通过Factory模式产生的，采用reflect机制，获得DriverType和ConfigType。

[contiv/netplugin/utils/driverfactory.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/utils/driverfactory.go#L11)

```
// implement utilities for instantiating the supported core.Driver
// (state, network and endpoint) instances

type driverConfigTypes struct {
	DriverType reflect.Type
	ConfigType reflect.Type
}

var networkDriverRegistry = map[string]driverConfigTypes{
	OvsNameStr: {
		DriverType: reflect.TypeOf(drivers.OvsDriver{}),
		ConfigType: reflect.TypeOf(drivers.OvsDriver{}),
	},
	// fakedriver is used for tests, so not exposing a public name for it.
	"fakedriver": {
		DriverType: reflect.TypeOf(drivers.FakeNetEpDriver{}),
		ConfigType: reflect.TypeOf(drivers.FakeNetEpDriverConfig{}),
	},
}

var stateDriverRegistry = map[string]driverConfigTypes{
	EtcdNameStr: {
		DriverType: reflect.TypeOf(state.EtcdStateDriver{}),
		ConfigType: reflect.TypeOf(state.EtcdStateDriverConfig{}),
	},
	ConsulNameStr: {
		DriverType: reflect.TypeOf(state.ConsulStateDriver{}),
		ConfigType: reflect.TypeOf(state.ConsulStateDriverConfig{}),
	},
	// fakestate-driver is used for tests, so not exposing a public name for it.
	"fakedriver": {
		DriverType: reflect.TypeOf(state.FakeStateDriver{}),
		ConfigType: reflect.TypeOf(state.FakeStateDriverConfig{}),
	},
}
```

StateDriver支持两种类型，分别是contiv/netplugin/state下面的EtcdStateDriver和ConsulStateDriver。它们都实现了[core.StateDriver](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/core/core.go#L108)接口，可以对stateStore中的状态进行Read、Write以及Watch操作。

Netplugin中的StateDriver是单例模式，全局唯一的gStateDriver。

[contiv/netplugin/utils/driverfactory.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/utils/driverfactory.go#L73)

```
// NewStateDriver instantiates a 'named' state-driver with specified configuration
func NewStateDriver(name string, instInfo *core.InstanceInfo) (core.StateDriver, error) {
	if name == "" || instInfo == nil {
		return nil, core.Errorf("invalid driver name or configuration passed.")
	}

	if gStateDriver != nil {
		return nil, core.Errorf("statedriver instance already exists.")
	}

	driver, err := initHelper(stateDriverRegistry, name)
	if err != nil {
		return nil, err
	}

	d := driver.(core.StateDriver)
	err = d.Init(instInfo)
	if err != nil {
		return nil, err
	}

	gStateDriver = d
	return d, nil
}

// GetStateDriver returns the singleton instance of the state-driver
func GetStateDriver() (core.StateDriver, error) {
	if gStateDriver == nil {
		return nil, core.Errorf("statedriver has not been not created.")
	}

	return gStateDriver, nil
}
```

### NetworkDriver

Netplugin中的NetworkDriver跟StateDriver不一样，不是单例模式。当ETCD中的网络FwdMode更新时，NetworkDriver可以调用Deinit()来cleanup整个网络，然后根据最新的network config重新New一个NetworkDriver。

[contiv/netplugin/utils/driverfactory.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/utils/driverfactory.go#L115)

```
// NewNetworkDriver instantiates a 'named' network-driver with specified configuration
func NewNetworkDriver(name string, instInfo *core.InstanceInfo) (core.NetworkDriver, error) {
	if name == "" || instInfo == nil {
		return nil, core.Errorf("invalid driver name or configuration passed.")
	}

	driver, err := initHelper(networkDriverRegistry, name)
	if err != nil {
		return nil, err
	}

	d := driver.(core.NetworkDriver)
	err = d.Init(instInfo)
	if err != nil {
		return nil, err
	}

	return d, nil
}
```

### Orchestration Plugin

Netplugin支持的Container Orchestration包括三种：Docker，Kubernetes和Mesos。它们的实现都在[contiv/netplugin/mgmtfn](https://github.com/contiv/netplugin/tree/v1.0.0-alpha-01-28-2017.10-23-11.UTC/mgmtfn)目录下。它们是在NewAgent()中，根据`plugin-mode`参数，选择性初始化docker或者kubernetes plugin，mesos plugin是一定会初始化的。下面主要介绍其中kubernetes plugin的实现。

NewAgent()中会调用k8splugin.InitCNIServer(netPlugin)来启动CNIServer，监听`/run/contiv/contiv-cni.sock`并相应`/ContivCNI.AddPod`和`/ContivCNI.DelPod`请求。

[contiv/netplugin/mgmtfn/k8splugin/cniserver.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/mgmtfn/k8splugin/cniserver.go#L198)

```
// InitCNIServer initializes the k8s cni server
func InitCNIServer(netplugin *plugin.NetPlugin) error {

	netPlugin = netplugin
	hostname, err := os.Hostname()
	if err != nil {
		log.Fatalf("Could not retrieve hostname: %v", err)
	}

	pluginHost = hostname

	// Set up the api client instance
	kubeAPIClient = setUpAPIClient()
	if kubeAPIClient == nil {
		log.Fatalf("Could not init kubernetes API client")
	}

	log.Debugf("Configuring router")

	router := mux.NewRouter()

	// register handlers for cni
	t := router.Headers("Content-Type", "application/json").Methods("POST").Subrouter()
	t.HandleFunc(cniapi.EPAddURL, makeHTTPHandler(addPod))
	t.HandleFunc(cniapi.EPDelURL, makeHTTPHandler(deletePod))
	t.HandleFunc("/ContivCNI.{*}", unknownAction)

	driverPath := cniapi.ContivCniSocket
	os.Remove(driverPath)
	os.MkdirAll(cniapi.PluginPath, 0700)

	go func() {
		l, err := net.ListenUnix("unix", &net.UnixAddr{Name: driverPath, Net: "unix"})
		if err != nil {
			panic(err)
		}

		log.Infof("k8s plugin listening on %s", driverPath)
		http.Serve(l, router)
		l.Close()
		log.Infof("k8s plugin closing %s", driverPath)
	}()

	//InitKubServiceWatch(netplugin)
	return nil
}
```


addPod()首先从req.Body中获得pod info，然后getEPSpec()根据pod labels（io.contiv.net-group，io.contiv.network，io.contiv.tenant）中获得网络信息。如果pod没有这些labels，就使用podInfo.setDefaults中的默认值。createEP()会分别调用netmaster（POST request）和netplugin（函数调用）创建endpoint。最后还需要进行一些列网络设置。

deletePod()跟addPod()操作类似，只不过获得网络信息之后，执行的是epCleanUp()，对Endpoint进行清理。

[contiv/netplugin/mgmtfn/k8splugin/driver.go](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/mgmtfn/k8splugin/driver.go#L376)

```
// addPod is the handler for pod additions
func addPod(r *http.Request) (interface{}, error) {

	resp := cniapi.RspAddPod{}

	logEvent("add pod")

	content, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Errorf("Failed to read request: %v", err)
		return resp, err
	}

	pInfo := cniapi.CNIPodAttr{}
	if err := json.Unmarshal(content, &pInfo); err != nil {
		return resp, err
	}

	// Get labels from the kube api server
	epReq, err := getEPSpec(&pInfo)
	if err != nil {
		log.Errorf("Error getting labels. Err: %v", err)
		setErrorResp(&resp, "Error getting labels", err)
		return resp, err
	}

	ep, err := createEP(epReq)
	if err != nil {
		log.Errorf("Error creating ep. Err: %v", err)
		setErrorResp(&resp, "Error creating EP", err)
		return resp, err
	}

	// convert netns to pid that netlink needs
	pid, err := nsToPID(pInfo.NwNameSpace)
	if err != nil {
		log.Errorf("Error moving to netns. Err: %v", err)
		setErrorResp(&resp, "Error moving to netns", err)
		return resp, err
	}

	// Set interface attributes for the new port
	err = setIfAttrs(pid, ep.PortName, ep.IPAddress, pInfo.IntfName)
	if err != nil {
		log.Errorf("Error setting interface attributes. Err: %v", err)
		setErrorResp(&resp, "Error setting interface attributes", err)
		return resp, err
	}

	// if Gateway is not specified on the nw, use the host gateway
	gwIntf := pInfo.IntfName
	gw := ep.Gateway
	if gw == "" {
		hostIf := netutils.GetHostIntfName(ep.PortName)
		hostIP, _ := netutils.HostIfToIP(hostIf)
		err = netPlugin.CreateHostAccPort(hostIf, ep.IPAddress, hostIP)
		if err != nil {
			log.Errorf("Error setting host access. Err: %v", err)
		} else {
			err = setIfAttrs(pid, hostIf, hostIP, "host1")
			if err != nil {
				log.Errorf("Move to pid %d failed", pid)
			} else {
				gw = hostGWIP
				gwIntf = "host1"
				// make sure service subnet points to eth0
				svcSubnet := contivK8Config.SvcSubnet
				addStaticRoute(pid, svcSubnet, pInfo.IntfName)
			}
		}

	}

	// Set default gateway
	err = setDefGw(pid, gw, gwIntf)
	if err != nil {
		log.Errorf("Error setting default gateway. Err: %v", err)
		setErrorResp(&resp, "Error setting default gateway", err)
		return resp, err
	}

	resp.Result = 0
	resp.IPAddress = ep.IPAddress
	resp.EndpointID = pInfo.InfraContainerID
	return resp, nil
}

// deletePod is the handler for pod deletes
func deletePod(r *http.Request) (interface{}, error) {

	resp := cniapi.RspAddPod{}

	logEvent("del pod")

	content, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Errorf("Failed to read request: %v", err)
		return resp, err
	}

	pInfo := cniapi.CNIPodAttr{}
	if err := json.Unmarshal(content, &pInfo); err != nil {
		return resp, err
	}

	// Get labels from the kube api server
	epReq, err := getEPSpec(&pInfo)
	if err != nil {
		log.Errorf("Error getting labels. Err: %v", err)
		setErrorResp(&resp, "Error getting labels", err)
		return resp, err
	}

	netPlugin.DeleteHostAccPort(epReq.EndpointID)
	err = epCleanUp(epReq)
	resp.Result = 0
	resp.EndpointID = pInfo.InfraContainerID
	return resp, err
}
```

[contiv/netplugin/mgmtfn/k8splugin/](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/mgmtfn/k8splugin/)目录下还有一些文件夹：
- **certs**：contivk8s需要的config，主要是与k8s建立通讯的配置文件contiv.json。
- **cniapi/api.go**：很多有用配置的default value。
- **contivk8s**：与netplugin进行通信的client。

### contivk8s

与netplugin进行通信的client。Kubelet创建pod的时候，会执行/opt/cni/bin/contivk8s。注意是执行，通过cmd的环境变量，传递`CNI_COMMAND`和`CNI_ARGS`参数。`CNI_COMMAND`是pod的操作：ADD/DEL；`CNI_ARGS`是pod information。
contivk8s通过unix sock(/run/contiv/contiv-cni.sock)，向localhost的netplugin发送REST请求。netplugin收到请求后，就执行相应的网络操作。

[contiv/netplugin/mgmtfn/k8splugin/contivk8s](https://github.com/contiv/netplugin/blob/v1.0.0-alpha-01-28-2017.10-23-11.UTC/mgmtfn/k8splugin/contivk8s/k8s_cni.go#L150)

```
func mainfunc() {
	pInfo := cniapi.CNIPodAttr{}
	cniCmd := os.Getenv("CNI_COMMAND")

	// Open a logfile
	f, err := os.OpenFile("/var/log/contivk8s.log", os.O_RDWR|os.O_CREATE|os.O_APPEND, 0666)
	if err != nil {
		logger.Fatalf("error opening file: %v", err)
	}
	defer f.Close()

	logger.SetOutput(f)
	log = getPrefixedLogger()

	log.Infof("==> Start New Log <==\n")
	log.Infof("command: %s, cni_args: %s", cniCmd, os.Getenv("CNI_ARGS"))

	// Collect information passed by CNI
	err = getPodInfo(&pInfo)
	if err != nil {
		log.Fatalf("Error parsing environment. Err: %v", err)
	}

	nc := clients.NewNWClient()
	if cniCmd == "ADD" {
		addPodToContiv(nc, &pInfo)
	} else if cniCmd == "DEL" {
		deletePodFromContiv(nc, &pInfo)
	}

}
```
