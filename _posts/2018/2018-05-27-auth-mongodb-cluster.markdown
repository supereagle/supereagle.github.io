---
layout:     post
title:      "容器化带权限控制的 MongoDB 集群"
subtitle:   "Containerized MongoDB Cluster with Authorization"
date:       2018-05-27
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - DB
---

搭建的 MongoDB cluster 的目的是为了保证高可靠和高可用，一个没有权限控制的 MongoDB cluster 是很难做到高可靠的。
任何人都可以在上面进行任何操作，且不说恶意攻击或损坏，就是人为的误操作，都有可能导致数据丢失的严重后果。
因此，在上一篇博客 [《容器化 MongoDB 集群》](/2018/05/26/mongodb-cluster/) 的基础上，本文继续深入探讨如何在容器化的 MongoDB cluster 上增加权限控制，使其真正做的高可靠。

## 权限控制目标

在容器化的 MongoDB cluster中，权限控制可以分为两个维度：一个是整个系统层面，即 root 账户；一个是数据库层面，即每个数据库都有各自单独的权限控制。
因此，要达到如下权限控制的目标：

* 集群有 root 账户权限控制
* 集群中预置一些 users，并分配合理权限

## MongoDB 集群在 k8s 上部署

### 准备 MongoDB 镜像

1. 生成 keyfile

```shell
$ openssl rand -base64 741 > mongodb-keyfile
```

2. 定制化 MongoDB 镜像

在官方的 MongoDB 镜像的基础上，增加 keyfile。Dockerfile 内容如下：

```dockerfile
FROM mongo:3.6.4

ADD mongodb-keyfile /data/config/mongodb-keyfile
RUN chown mongodb:mongodb /data/config/mongodb-keyfile
```

### K8s statefulset

mongo-statefulset.yaml：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: test
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo:3.6.4
          args:
            - mongod
            - "--replSet"
            - rs0
            - "--bind_ip"
            - 0.0.0.0
            - "--smallfiles"
            - "--noprealloc"
            - --clusterAuthMode
            - keyFile
            - --keyFile
            - /data/config/mongodb-keyfile
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: admin
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: 123456
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /mgo-data/db
            - mountPath: /docker-entrypoint-initdb.d
              name: mongo-init-js
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=test"
            - name: KUBERNETES_MONGO_SERVICE_NAME
              value: mongo
            - name: MONGODB_USERNAME
              value: admin
            - name: MONGODB_PASSWORD
              value: 123456
            - name: MONGODB_DATABASE
              value: admin
      volumes:
      - hostPath:
          path: /data/db
          type: ""
        name: mongo-persistent-storage
      - configMap:
          defaultMode: 420
          items:
          - key: mongo-init.js
            path: mongo-init.js
          name: mongo-init-js
        name: mongo-init-js
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-init-js
data:
  mongo-init.js: |
    var users = ["dev", "test", "product"];
    var pwd = "123456";
    var mgo = new Mongo('127.0.0.1:27017');
    var admindb = mgo.getDB("admin");
    admindb.auth("admin", pwd);

    for (i=0; i < users.length; i++) {
        user = users[i];
        db = admindb.getSiblingDB(user);
        u = db.getUser(user);
        if (u === null) {
            print("user is not found, add this user");
            db.createUser({'user': user, 'pwd': pwd, roles: [ { role: "dbAdmin", db: user } ]});
        } else {
            print("user is found, need no action");
        }
    }
```

基于官方 [statefulset](https://github.com/cvallance/mongo-k8s-sidecar/blob/master/example/StatefulSet/mongo-statefulset.yaml)，有如下优化：

* 使用上一步定制化的 MongoDB 镜像
* 将 storage 由 fast 类型的 storage class 换成了 hostPath
* 增加了 mongo-init-js configMap，来初始化 MongoDB 中用户
* Mongo 和 Sidecar 都增加 root 用户的 username 和 password
* Sidecar 中设置环境变量 `KUBERNETES_MONGO_SERVICE_NAME` 为 headless service name, 使 replica set 中的 node name 使用 statefulset 稳定的 network ID，而不是 pod IP

### 检查集群

查看集群资源：

```shell
$ kubectl get statefulset
NAME                     DESIRED   CURRENT   AGE
mongo                    3         3         58s
$ kubectl get svc
NAME                                                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mongo                                                          ClusterIP   None             <none>        27017/TCP        1m
$ kubectl get cm
NAME                 DATA      AGE
mongo-init-js        1         1m
$ kubectl get po
NAME                                                             READY     STATUS             RESTARTS   AGE
mongo-0                                                          2/2       Running            0          2m
mongo-1                                                          2/2       Running            0          2m
mongo-2                                                          2/2       Running            0          2m
```

进入集群某实例：

```shell
$ kubectl exec -it mongo-1 /bin/sh
$ mongo -u admin -p 123456 --authenticationDatabase admin
```

查看集群状态：

```shell
rs0:SECONDARY> rs.status()
{
	"set" : "rs0",
	"date" : ISODate("2018-05-27T15:18:00.619Z"),
	"myState" : 2,
	"term" : NumberLong(1),
	"syncingTo" : "mongo-2.mongo.default.svc.cluster.local:27017",
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1527434271, 1),
			"t" : NumberLong(1)
		},
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1527434271, 1),
			"t" : NumberLong(1)
		},
		"appliedOpTime" : {
			"ts" : Timestamp(1527434271, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1527434271, 1),
			"t" : NumberLong(1)
		}
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "mongo-2.mongo.default.svc.cluster.local:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 215,
			"optime" : {
				"ts" : Timestamp(1527434271, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1527434271, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-05-27T15:17:51Z"),
			"optimeDurableDate" : ISODate("2018-05-27T15:17:51Z"),
			"lastHeartbeat" : ISODate("2018-05-27T15:18:00.427Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-27T15:17:58.860Z"),
			"pingMs" : NumberLong(0),
			"electionTime" : Timestamp(1527434058, 2),
			"electionDate" : ISODate("2018-05-27T15:14:18Z"),
			"configVersion" : 4
		},
		{
			"_id" : 1,
			"name" : "mongo-0.mongo.default.svc.cluster.local:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 215,
			"optime" : {
				"ts" : Timestamp(1527434271, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1527434271, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-05-27T15:17:51Z"),
			"optimeDurableDate" : ISODate("2018-05-27T15:17:51Z"),
			"lastHeartbeat" : ISODate("2018-05-27T15:17:58.534Z"),
			"lastHeartbeatRecv" : ISODate("2018-05-27T15:17:58.982Z"),
			"pingMs" : NumberLong(1),
			"syncingTo" : "mongo-2.mongo.default.svc.cluster.local:27017",
			"configVersion" : 4
		},
		{
			"_id" : 2,
			"name" : "mongo-1.mongo.default.svc.cluster.local:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 229,
			"optime" : {
				"ts" : Timestamp(1527434271, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2018-05-27T15:17:51Z"),
			"syncingTo" : "mongo-2.mongo.default.svc.cluster.local:27017",
			"configVersion" : 4,
			"self" : true
		}
	],
	"ok" : 1,
	"operationTime" : Timestamp(1527434271, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1527434271, 1),
		"signature" : {
			"hash" : BinData(0,"O3HdMjCWSGU++pUngRJZR0KCIhI="),
			"keyId" : NumberLong("6560279334496501761")
		}
	}
}
```

查看集群用户信息：

```shell
rs0:PRIMARY> use admin
switched to db admin
rs0:PRIMARY> db.system.users.find().pretty()
{
	"_id" : "admin.admin",
	"user" : "admin",
	"db" : "admin",
	"credentials" : {
		"SCRAM-SHA-1" : {
			"iterationCount" : 10000,
			"salt" : "C0l9OxGYeet97PjV2elTJA==",
			"storedKey" : "e42xuW7vG71F5zG2Ad+GocaNMHc=",
			"serverKey" : "/CqJ6cL/+5+aHP4qMDQkomND8Mg="
		}
	},
	"roles" : [
		{
			"role" : "root",
			"db" : "admin"
		}
	]
}
{
	"_id" : "dev.dev",
	"user" : "dev",
	"db" : "dev",
	"credentials" : {
		"SCRAM-SHA-1" : {
			"iterationCount" : 10000,
			"salt" : "7O1IGsb4KmDwi2TzsBRSgQ==",
			"storedKey" : "5ILARc3E3hx75VU1D3i+XUrEMKU=",
			"serverKey" : "ZwBTqxF6vPYQD2pRGrxw+3+MovQ="
		}
	},
	"roles" : [
		{
			"role" : "dbAdmin",
			"db" : "dev"
		}
	]
}
{
	"_id" : "test.test",
	"user" : "test",
	"db" : "test",
	"credentials" : {
		"SCRAM-SHA-1" : {
			"iterationCount" : 10000,
			"salt" : "ZXT2TZfAJs0cRoRSo1re5w==",
			"storedKey" : "UHmlcglv8pP8uysX9cveWB84s7c=",
			"serverKey" : "3Z1WgpsFhuoB5YGurr7rANi036I="
		}
	},
	"roles" : [
		{
			"role" : "dbAdmin",
			"db" : "test"
		}
	]
}
{
	"_id" : "product.product",
	"user" : "product",
	"db" : "product",
	"credentials" : {
		"SCRAM-SHA-1" : {
			"iterationCount" : 10000,
			"salt" : "XaOpl7gstRFgFcyKX51NMQ==",
			"storedKey" : "zk5wrrO+v2nQ/CFrsdT+iL2YhI0=",
			"serverKey" : "6MHvAqVX8cCnvs2STSQql6XWR1M="
		}
	},
	"roles" : [
		{
			"role" : "dbAdmin",
			"db" : "product"
		}
	]
}
```

## Troubleshootings

1) MongoDB 集群配置 root 账户没有生效。

**原因：**
MongoDB root 账户是在 [entrypoint 启动脚本](https://github.com/docker-library/mongo/blob/master/3.6/docker-entrypoint.sh) 中添加的。K8s command 会覆盖 MongoDB image 中的 entrypoint。

**解决：**
需要将K8s command 改为 args。注意，只有同时设置 `MONGO_INITDB_ROOT_USERNAME` 和 `MONGO_INITDB_ROOT_USERNAME` 两个环境变量，root 账户才会添加成功。

2) MongoDB 加了 auth 之后，sidecar 没有权限访问 MongoDB。

**原因：**
Sidecar 需要配置 auth 才能访问 MongoDB。

**解决：**
Sidecar 加配置 auth 环境变量。

Error log from MongoDB:

```
2018-05-27T12:48:10.098+0000 I NETWORK  [listener] connection accepted from 127.0.0.1:55788 #17 (1 connection now open)
2018-05-27T12:48:10.098+0000 I NETWORK  [conn17] received client metadata from 127.0.0.1:55788 conn17: { driver: { name: "nodejs", version: "2.2.35" }, os: { type: "Linux", name: "linux", architecture: "x64", version: "3.10.0-693.el7.x86_64" }, platform: "Node.js v9.8.0, LE, mongodb-core: 2.1.19" }
2018-05-27T12:48:10.100+0000 I ACCESS   [conn17] Unauthorized: not authorized on admin to execute command { replSetGetStatus: {}, $db: "admin" }
2018-05-27T12:48:10.102+0000 I NETWORK  [conn17] end connection 127.0.0.1:55788 (0 connections now open)
```

Error log from sidecar:

```
Error in workloop { MongoError: not authorized on admin to execute command { replSetGetStatus: {}, $db: "admin" }
    at Function.MongoError.create (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/error.js:31:11)
    at /opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/pool.js:497:72
    at authenticateStragglers (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/pool.js:443:16)
    at Connection.messageHandler (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/pool.js:477:5)
    at Socket.<anonymous> (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/connection.js:333:22)
    at Socket.emit (events.js:180:13)
    at addChunk (_stream_readable.js:269:12)
    at readableAddChunk (_stream_readable.js:256:11)
    at Socket.Readable.push (_stream_readable.js:213:10)
    at TCP.onread (net.js:578:20)
  name: 'MongoError',
  message: 'not authorized on admin to execute command { replSetGetStatus: {}, $db: "admin" }',
  ok: 0,
  errmsg: 'not authorized on admin to execute command { replSetGetStatus: {}, $db: "admin" }',
  code: 13,
  codeName: 'Unauthorized',
  '$clusterTime':
   { clusterTime: Timestamp { _bsontype: 'Timestamp', low_: 0, high_: 0 },
     signature: { hash: [Binary], keyId: 0 } } }
```

3) MongoDB 加了 auth 之后，pod 无法加入到集群中。

**原因：**
MongoDB 加了 auth 之后，往集群中加节点，需要通过 keyFile 进行 internal authentication。

**解决：**
根据官方 MongoDB 官方文档，产生 keyFile。
同时添加 MongoDB 启动参数：

```
        - --clusterAuthMode
        - keyFile
        - --keyFile
        - /data/config/mongodb-keyfile
```

Error log from MongoDB:

```
2018-05-27T13:03:16.512+0000 I NETWORK  [listener] connection accepted from 192.168.68.95:39586 #149 (1 connection now open)
2018-05-27T13:03:16.539+0000 I NETWORK  [conn149] received client metadata from 192.168.68.95:39586 conn149: { driver: { name: "nodejs", version: "2.2.35" }, os: { type: "Linux", name: "linux", architecture: "x64", version: "3.10.0-693.el7.x86_64" }, platform: "Node.js v9.8.0, LE, mongodb-core: 2.1.19" }
2018-05-27T13:03:16.545+0000 I ACCESS   [conn149] Successfully authenticated as principal admin on admin
2018-05-27T13:03:16.547+0000 I NETWORK  [conn149] end connection 192.168.68.95:39586 (0 connections now open)
2018-05-27T13:03:16.966+0000 I NETWORK  [listener] connection accepted from 127.0.0.1:36716 #150 (1 connection now open)
2018-05-27T13:03:16.967+0000 I NETWORK  [conn150] received client metadata from 127.0.0.1:36716 conn150: { driver: { name: "nodejs", version: "2.2.35" }, os: { type: "Linux", name: "linux", architecture: "x64", version: "3.10.0-693.el7.x86_64" }, platform: "Node.js v9.8.0, LE, mongodb-core: 2.1.19" }
2018-05-27T13:03:16.972+0000 I ACCESS   [conn150] Successfully authenticated as principal admin on admin
2018-05-27T13:03:16.976+0000 I REPL     [conn150] replSetReconfig admin command received from client; new config: { _id: "rs0", version: 4, protocolVersion: 1, members: [ { _id: 0, host: "mongo-0.mongo.default.svc.cluster.local:27017", arbiterOnly: false, buildIndexes: true, hidden: false, priority: 1, tags: {}, slaveDelay: 0, votes: 1 }, { _id: 1, host: "mongo-1.mongo.default.svc.cluster.local:27017" }, { _id: 2, host: "mongo-2.mongo.default.svc.cluster.local:27017" } ], settings: { chainingAllowed: true, heartbeatIntervalMillis: 2000, heartbeatTimeoutSecs: 10, electionTimeoutMillis: 10000, catchUpTimeoutMillis: -1, catchUpTakeoverDelayMillis: 30000, getLastErrorModes: {}, getLastErrorDefaults: { w: 1, wtimeout: 0 }, replicaSetId: ObjectId('5b0aab9f4c07e29cb2c7459a') } }
2018-05-27T13:03:16.984+0000 I REPL     [conn150] replSetReconfig config object with 3 members parses ok
2018-05-27T13:03:16.986+0000 W REPL     [replexec-2] Got error (KeyNotFound: Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426193, 1) } with id: 6560244515196633089) response on heartbeat request to mongo-2.mongo.default.svc.cluster.local:27017; { ok: 1.0, hbmsg: "" }
2018-05-27T13:03:16.986+0000 W REPL     [replexec-2] Got error (KeyNotFound: Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426193, 1) } with id: 6560244515196633089) response on heartbeat request to mongo-1.mongo.default.svc.cluster.local:27017; { ok: 1.0, hbmsg: "" }
2018-05-27T13:03:16.986+0000 E REPL     [conn150] replSetReconfig failed; NodeNotFound: Quorum check failed because not enough voting nodes responded; required 2 but only the following 1 voting nodes responded: mongo-0.mongo.default.svc.cluster.local:27017; the following nodes did not respond affirmatively: mongo-2.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426193, 1) } with id: 6560244515196633089, mongo-1.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426193, 1) } with id: 6560244515196633089
2018-05-27T13:03:16.988+0000 I NETWORK  [conn150] end connection 127.0.0.1:36716 (0 connections now open)
2018-05-27T13:03:19.149+0000 I NETWORK  [listener] connection accepted from 192.168.66.91:34656 #151 (1 connection now open)
2018-05-27T13:03:19.150+0000 I NETWORK  [conn151] received client metadata from 192.168.66.91:34656 conn151: { driver: { name: "nodejs", version: "2.2.35" }, os: { type: "Linux", name: "linux", architecture: "x64", version: "3.10.0-693.el7.x86_64" }, platform: "Node.js v9.8.0, LE, mongodb-core: 2.1.19" }
2018-05-27T13:03:19.157+0000 I ACCESS   [conn151] Successfully authenticated as principal admin on admin
2018-05-27T13:03:19.161+0000 I NETWORK  [conn151] end connection 192.168.66.91:34656 (0 connections now open)
```

Error log from sidecar:

```
Db.prototype.authenticate method will no longer be available in the next major release 3.x as MongoDB 3.6 will only allow auth against users in the admin db and will no longer allow multiple credentials on a socket. Please authenticate using MongoClient.connect with auth credentials.
Addresses to add:     [ 'mongo-1.mongo.default.svc.cluster.local:27017',
  'mongo-2.mongo.default.svc.cluster.local:27017' ]
Addresses to remove:  []
replSetReconfig { _id: 'rs0',
  version: 3,
  protocolVersion: 1,
  members:
   [ { _id: 0,
       host: 'mongo-0.mongo.default.svc.cluster.local:27017',
       arbiterOnly: false,
       buildIndexes: true,
       hidden: false,
       priority: 1,
       tags: {},
       slaveDelay: 0,
       votes: 1 },
     { _id: 1,
       host: 'mongo-1.mongo.default.svc.cluster.local:27017' },
     { _id: 2,
       host: 'mongo-2.mongo.default.svc.cluster.local:27017' } ],
  settings:
   { chainingAllowed: true,
     heartbeatIntervalMillis: 2000,
     heartbeatTimeoutSecs: 10,
     electionTimeoutMillis: 10000,
     catchUpTimeoutMillis: -1,
     catchUpTakeoverDelayMillis: 30000,
     getLastErrorModes: {},
     getLastErrorDefaults: { w: 1, wtimeout: 0 },
     replicaSetId: 5b0aab9f4c07e29cb2c7459a } }
Error in workloop { MongoError: Quorum check failed because not enough voting nodes responded; required 2 but only the following 1 voting nodes responded: mongo-0.mongo.default.svc.cluster.local:27017; the following nodes did not respond affirmatively: mongo-1.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426143, 1) } with id: 6560244515196633089, mongo-2.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426143, 1) } with id: 6560244515196633089
    at Function.MongoError.create (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/error.js:31:11)
    at /opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/pool.js:497:72
    at authenticateStragglers (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/pool.js:443:16)
    at Connection.messageHandler (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/pool.js:477:5)
    at Socket.<anonymous> (/opt/cvallance/mongo-k8s-sidecar/node_modules/mongodb-core/lib/connection/connection.js:333:22)
    at Socket.emit (events.js:180:13)
    at addChunk (_stream_readable.js:269:12)
    at readableAddChunk (_stream_readable.js:256:11)
    at Socket.Readable.push (_stream_readable.js:213:10)
    at TCP.onread (net.js:578:20)
  name: 'MongoError',
  message: 'Quorum check failed because not enough voting nodes responded; required 2 but only the following 1 voting nodes responded: mongo-0.mongo.default.svc.cluster.local:27017; the following nodes did not respond affirmatively: mongo-1.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426143, 1) } with id: 6560244515196633089, mongo-2.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426143, 1) } with id: 6560244515196633089',
  ok: 0,
  errmsg: 'Quorum check failed because not enough voting nodes responded; required 2 but only the following 1 voting nodes responded: mongo-0.mongo.default.svc.cluster.local:27017; the following nodes did not respond affirmatively: mongo-1.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426143, 1) } with id: 6560244515196633089, mongo-2.mongo.default.svc.cluster.local:27017 failed with Cache Reader No keys found for HMAC that is valid for time: { ts: Timestamp(1527426143, 1) } with id: 6560244515196633089',
  code: 74,
  codeName: 'NodeNotFound',
  operationTime: Timestamp { _bsontype: 'Timestamp', low_: 1, high_: 1527426143 },
  '$clusterTime':
   { clusterTime: Timestamp { _bsontype: 'Timestamp', low_: 1, high_: 1527426143 },
     signature: { hash: [Binary], keyId: [Long] } } }
```

4) 通过 volume 的方式挂载 keyfile，MongoDB 因为打开 keyfile 存在 permission 问题，无法正常启动。

原本想参考 [thilinapiy‘s mongo-statefulset.yaml](https://gist.github.com/thilinapiy/0c5abc2c0c28efe1bbe2165b0d8dc115)，将 keyFile 存储为 secret，然后通过 volume 的方式挂载到 MongoDB中，不过这种方式存在 permission 的问题，导致 MongoDB 无法正常启动。下一个 troubleshooting 会详细分析。

MongoDB 的 entrypoint 通过 [gosu](https://github.com/tianon/gosu) 来使用 `mongodb` 用户启动 MongoDB。
Volume 挂载到 MongoDB 中的 keyfile 是属于 root 的 UID 和 GID，为 keyfile 配置 600 或 400 的权限，用户 `mongodb` 在启动 MongoDB 的时候没有权限访问开发 keyfile。想到的办法是改变挂载的 keyfile 的 UID 和 GID。
不过 [K8s Securitycontext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#podsecuritycontext-v1-core) 仅支持通过 fsGroup 参数设置 GID，不支持设置 volume 的UID。将 keyfile 的 GID 设置为 `mongodb`，同时配置660 或 440 来放开 group 权限，不过用户 `mongodb` 在启动 MongoDB 的时候又会提示权限 too open。所以，就进入了一个非常尴尬的境地，最终只好将 keyfile 直接打入镜像中。

这个问题的关键是 MongoDB 的 entrypoint 通过 `mongodb` 用户启动 MongoDB。问题1已经阐述了，是为了加 root 账户，所以才避免 MongoDB 的 entrypoint 被覆盖。
如果像 [thilinapiy‘s mongo-statefulset.yaml](https://gist.github.com/thilinapiy/0c5abc2c0c28efe1bbe2165b0d8dc115) 那样通过脚本的方式来加 root 账户，就不存在该问题了。

# Reference

- [Kubernetes sidecar for Mongo](https://github.com/cvallance/mongo-k8s-sidecar)
- [K8s: Define a Command and Arguments for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
- [MongoDB: Enforce Keyfile Access Control in a Replica Set](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/)
- [MongoDB statefulset for kubernetes with authentication and replication](https://gist.github.com/thilinapiy/0c5abc2c0c28efe1bbe2165b0d8dc115)
- [Simple Authentication always fails](https://github.com/cvallance/mongo-k8s-sidecar/issues/82)
