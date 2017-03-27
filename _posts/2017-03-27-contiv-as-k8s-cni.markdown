---
layout:     post
title:      "Kubernetes集群采用Contiv网络"
subtitle:   "Use Contiv as CNI Network in Kubernetes Cluster"
date:       2017-03-27
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Kubernetes
    - CNI
    - Contiv
---

# Category

- [Contiv Configuration](#contiv-configuration)
	- [Contiv Master](#contiv-master)
	- [Contiv Netplugin](#contiv-netplugin)
- [Kubernetes Configuration](#kubernetes-configuration)
- [Reference](#reference)

## Contiv Configuration

### Contiv Master

1. 启动netmaster

	```
	netmaster -debug -cluster-mode kubernetes -cluster-store etcd://${ETCD_HOST}:${ETCD_PORT} > /var/log/netmaster.log 2>&1 &
	```

2. 配置netmaster主机名

	netctl默认通过http://netmaster:9999与netmaster通信，不配置netmaster主机名的话，需要通过参数--netmaster指定

	```
	# cat /etc/hosts
	127.0.0.1   localhost
	::1         localhost
	...
	${NETMASTER_HOST_IP} netmaster
	```

3. 创建network

	```
	netctl network create -e vlan -p 1014 -s 10.10.14.0/22 -g 10.10.14.1 default-net
	```

	**`备注`**：Contiv通过pod label来选择网络。如果pod label没有设置网络，就采用默认的default-net。

### Contiv Netplugin

	```
	netplugin -vlan-if bond0 -plugin-mode kubernetes -fwd-mode bridge -store-url=http://${ETCD_HOST}:${ETCD_PORT} -local-ip=${HOST_IP} -debug > /var/log/netplugin.log 2>&1 &
	```

	**`备注`**：`-vlan-if`表示的是VLAN的上行端口，注意Contiv默认与其连接的bridge是contivVlanBridge。


## Kubernetes Configuration

每个Kubernetes nodes上都要运行Contiv Netplugin。

Kubelet配置：
- **添加config**：--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin
- **下载loopback binary**：从[CNI Release](https://github.com/containernetworking/cni/releases)下载loopback binary到/opt/cni/bin中，否则netplugin无法启动
- **添加CNI配置**：
	```
	# cat /etc/cni/net.d/contiv.conf
	{
	  "cniVersion": "0.1.0",
	  "name": "contiv-net",
	  "type": "contivk8s.bin"
	}
	```

Netplugin配置：（用于构建K8s client）
```

# cat /opt/contiv/config/contiv.json 
{
  "K8S_API_SERVER": "http://k8s.master.com:8080",
  "K8S_CA": "/etc/kubernetes/certs/ca.crt",
  "K8S_KEY": "/etc/kubernetes/certs/kubecfg.key",
  "K8S_CERT": "/etc/kubernetes/certs/kubecfg.crt"
}
```

## Reference
- [Kubernetes Network Plugins](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/)
- [Container Networking Interface Proposal](https://github.com/containernetworking/cni/blob/master/SPEC.md)