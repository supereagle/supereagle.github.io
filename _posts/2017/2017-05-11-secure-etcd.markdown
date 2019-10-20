---
layout:     post
title:      "如何搭建安全的ETCD集群"
subtitle:   "How to Setup Secure ETCD Cluster"
date:       2017-05-11
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Security
    - ETCD
---

在容器云平台中，除了Docker以外，ETCD是另外一个必不可少的组件。许多知名开源项目都依赖ETCD作为其键值存储，例如[Kubernetes](https://github.com/kubernetes/kubernetes),[Swarm](https://github.com/docker/swarm)以及[Contiv](https://github.com/contiv/netplugin)等。
ETCD同时是一个非常关键性的组件，容器云平台中的其他组件无时无刻不在跟ETCD打交道。例如，Kubernetes将集群的所有配置，状态，资源等信息都存在ETCD集群中，甚至组件间不是通过API直接通信，而是通过Watch ETCD变化来通信，从而减少组件间的耦合。

ETCD集群除了要保证稳定性，同时还要保证其安全性。ETCD自身是支持安全模式的，下面介绍一下搭建安全的ETCD集群的经验。


## Generate Self-signed Certificates

### Install cfssl

[cfssl](https://github.com/cloudflare/cfssl)是ETCD官方推荐的CA生产工具。

```shell
$ curl -s -L -o /usr/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ curl -s -L -o /usr/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x /usr/bin/{cfssl,cfssljson}
```

### Generate Self-signed CA

```shell
$ cat ca-config.json 
{
  "signing": {
    "default": {
      "expiry": "100000h"
    },
    "profiles": {
      "server": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "100000h"
      },
      "client": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
$ cat ca-csr.json 
{
  "CN": "Etcd",
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "ETCD",
      "OU": "PaaS",
      "ST": "Shanghai"
    }
  ]
}
 
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

生成三个文件：
- ca-key.pem - private key for the CA
- ca.pem - certificate for the CA
- ca.csr - certificate signing request for the CA

### Generate Server Self-signed Key Pairs

```shell
$ cat server-csr.json
{
  "CN": "etcd-server",
  "hosts": [
    "localhost",
    "0.0.0.0",
    "127.0.0.1",
    "etcd1.master.com",
    "etcd2.master.com"
  ],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "O": "ETCD",
      "OU": "PaaS",
      "ST": "Shanghai"
    }
  ]
}
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server-csr.json | cfssljson -bare server
```

生成三个文件：server.pem, server-key.pem和server.csr。

> hosts需要包括允许访问ETCD Cluster的IP或者FQDN。

### Generate Client Self-signed Key Pairs

```shell
$ cat client-csr.json
{
  "CN": "etcd-client",
  "hosts": [
    ""
  ],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "ST": "Shanghai",
      "O": "ETCD",
      "OU": "PaaS"
    }
  ]
}
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json | cfssljson -bare client
```

生成三个文件：client.pem, client-key.pem和client.csr。

## Config ETCD Cluster

### Example

以在一台机器上搭建三个节点的集群为例，分别在1180，1280和1380分别起node1，node2和node3：

```shell
$ etcd --name=node3 --listen-client-urls=https://localhost:1379 --advertise-client-urls=https://localhost:1379 --listen-peer-urls=https://localhost:1380 --initial-advertise-peer-urls=https://localhost:1380 --initial-cluster=node1=https://localhost:1180,node2=https://localhost:1280,node3=https://localhost:1380 --initial-cluster-token=etcd-cluster-token --initial-cluster-state=new --cert-file=./server.pem --key-file=./server-key.pem --peer-cert-file=./server.pem --peer-key-file=./server-key.pem --trusted-ca-file=./ca.pem --peer-trusted-ca-file=./ca.pem --data-dir=./nodes/node3 --peer-client-cert-auth=true --client-cert-auth=true
```

额外需要的配置：--cert-file=./server.pem --key-file=./server-key.pem --peer-cert-file=./server.pem --peer-key-file=./server-key.pem --trusted-ca-file=./ca.pem --peer-trusted-ca-file=./ca.pem --peer-client-cert-auth=true --client-cert-auth=true

### Verification

```shell
$ etcdctl --cert-file=client.pem  --key-file=client-key.pem --ca-file=ca.pem --endpoints=https://0.0.0.0:1179,https://0.0.0.0:1279,https://0.0.0.0:1379 cluster-health
member 5a68dbeefb870ed1 is healthy: got healthy result from https://localhost:1179
member 772c76fe731a3914 is healthy: got healthy result from https://localhost:1379
member aa3bff8d4d84db66 is healthy: got healthy result from https://localhost:1279
cluster is healthy
$ etcdctl  --endpoints=https://0.0.0.0:1179,https://0.0.0.0:1279,https://0.0.0.0:1379 ls
Error:  x509: certificate signed by unknown authority
```

## Config Kubernetes for Secure ETCD Cluster

1. 复制CA文件到/etc/kubernetes/ssl
2. /etc/kubernetes/config添加配置：--etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/client.pem --etcd-keyfile=/etc/kubernetes/ssl/client-key.pem
	```
	KUBE_ETCD_SERVERS="--etcd-servers=https://0.0.0.0:1179,https://0.0.0.0:1279,https://0.0.0.0:1379 --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/client.pem --etcd-keyfile=/etc/kubernetes/ssl/client-key.pem"
	```

## Reference
- [ETCD Security Model](https://coreos.com/etcd/docs/latest/op-guide/security.html)
- [Generate self-signed certificates](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)
- [Setting up a secure etcd cluster behind a proxy](https://sdqali.in/blog/2016/11/11/setting-up-a-secure-etcd-cluster-behind-a-proxy/)