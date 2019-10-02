---
layout:     post
title:      "容器化 MongoDB 集群"
subtitle:   "Containerized MongoDB Cluster"
date:       2018-05-26
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - DB
---


# MongoDB Cluster Architecture

MongoDB 有两种集群的架构，分别是 [replication](https://docs.mongodb.com/manual/replication/) 和 [sharding](https://docs.mongodb.com/manual/sharding/)。这两种架构各有侧重点，分别使用不同的应用场景：Replication 主要通过主从多副本，保证数据的可靠性；
Sharding 主要是通过数据的分片，保证数据的可用性和高并发。下面主要介绍如何容器化 replication 类型的 MongoDB 集群。

# MongoDB Cluster by Docker Compose

## Generate Key File

MongoDB Cluster 中的各 member 间需要通过 keyfile 进行内部的 authentication。产生 keyfile 的步骤如下：

```
# cd data
# openssl rand -base64 741 > mongodb-keyfile
```

## Setup Cluster

mongodb-cluster.yaml: 

```
version: '3.1'
services:
  mongo1:
    image: mongo:3.6.4
    volumes:
       - ./data/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
       - ./data/mongodb-keyfile:/data/config/mongodb-keyfile
    command:
    - mongod
    - "--replSet"
    - rs0
    - "--bind_ip"
    - 0.0.0.0
    - "--smallfiles"
    - "--noprealloc"
    - "--clusterAuthMode"
    - keyFile
    - "--keyFile"
    - "/data/config/mongodb-keyfile"
    expose:
       - "27017"
    ports:
       - "30000:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: Pwd123456
  mongo2:
    image: mongo:3.6.4
    volumes:
       - ./data/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
       - ./data/mongodb-keyfile:/data/config/mongodb-keyfile
    command:
    - mongod
    - "--replSet"
    - rs0
    - "--bind_ip"
    - 0.0.0.0
    - "--smallfiles"
    - "--noprealloc"
    - "--clusterAuthMode"
    - keyFile
    - "--keyFile"
    - "/data/config/mongodb-keyfile"
    expose:
       - "27017"
    ports:
       - "30001:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: Pwd123456
  mongo3:
    image: cmongo:3.6.4
    volumes:
       - ./data/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
       - ./data/mongodb-keyfile:/data/config/mongodb-keyfile
    command:
    - mongod
    - "--replSet"
    - rs0
    - "--bind_ip"
    - 0.0.0.0
    - "--smallfiles"
    - "--noprealloc"
    - "--clusterAuthMode"
    - keyFile
    - "--keyFile"
    - "/data/config/mongodb-keyfile"
    expose:
       - "27017"
    ports:
       - "30002:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: Pwd123456
```

说明：

* 使用的镜像版本是官方提供的 v3.6.4
* 分别将三个 MongoDB 的实例的 27017 端口，映射到主机的 30000 ~ 30002 端口
* 集群的 root 账户和密码分别是 admin 和 Pwd123456
* docker-entrypoint-initdb.d 目录中，可以放一些集群初始化的脚本。使用见 [Docker Hub MongoDB](https://hub.docker.com/_/mongo/)

Start MongoDB Cluster:

```
# docker-compose up -d
# docker ps
CONTAINER ID        IMAGE           COMMAND                  CREATED           STATUS              PORTS                      NAMES
0c5ffd781dfc        mongo:3.6.4   "docker-entrypoint.s…"   2 hours ago         Up About an hour    0.0.0.0:30000->27017/tcp   mongodbcluster_mongo1_1
f91aa2737b48        mongo:3.6.4   "docker-entrypoint.s…"   2 hours ago         Up About an hour    0.0.0.0:30001->27017/tcp   mongodbcluster_mongo2_1
78b454088500        mongo:3.6.4   "docker-entrypoint.s…"   2 hours ago         Up About an hour    0.0.0.0:30002->27017/tcp   mongodbcluster_mongo3_1
```

进入容器中，执行如下命令，组件集群：

```
# docker-compose exec mongo1 mongo
# rs.initiate()
# rs.add('mongo2:30001')
# rs.add('mongo3:30002')
# rs.status()
```

# MongoDB Cluster by Kubernetes

在 Kubernetes 中部署 MongoDB Cluster，主要是借助开源的工具 [mongo-k8s-sidecar](https://github.com/cvallance/mongo-k8s-sidecar)，后面简称为 Sidecar。
Sidecar 是通过 JavaScript 实现，主要是通过调用 Kubernetes API 实时 watch 集群中 MongoDB 实例状态，然后调用 MongoDB API 更新集群的 replica set config。
他提供了各种 StatefulSet、Emptydir 以及 Ceph RBD 多种部署的 example。

### Sidecar 原理

Sidecar 主要包含如下 4 个组件：

* Config：从 env 中读取 config
* K8s：从 k8s 中通过 label 选择所有 MongoDB pods
* Mongo：MongoDB client，提供 replica set 操作的 API
* Worker：根据 MongoDB pods 状态，来增删 replica set 中实例，从而维护 MongoDB Cluster 状态

worker.js 核心源码分析：

```
var workloop = function workloop() {
  if (!hostIp || !hostIpAndPort) {
    throw new Error('Must initialize with the host machine\'s addr');
  }

  //Do in series so if k8s.getMongoPods fails, it doesn't open a db connection
  async.series([
    k8s.getMongoPods,
    mongo.getDb
  ], function(err, results) {
    var db = null;
    if (Array.isArray(results) && results.length === 2) {
      db = results[1];
    }

    if (err) {
      return finish(err, db);
    }

    var pods = results[0];

    //Lets remove any pods that aren't running or haven't been assigned an IP address yet
    for (var i = pods.length - 1; i >= 0; i--) {
      var pod = pods[i];
      if (pod.status.phase !== 'Running' || !pod.status.podIP) {
        pods.splice(i, 1);
      }
    }

    if (!pods.length) {
      return finish('No pods are currently running, probably just give them some time.');
    }

    //Lets try and get the rs status for this mongo instance
    //If it works with no errors, they are in the rs
    //If we get a specific error, it means they aren't in the rs
    mongo.replSetGetStatus(db, function(err, status) {
      if (err) {
        if (err.code && err.code == 94) {
          notInReplicaSet(db, pods, function(err) {
            finish(err, db);
          });
        }
        else if (err.code && err.code == 93) {
          invalidReplicaSet(db, pods, status, function(err) {
            finish(err, db);
          });
        }
        else {
          finish(err, db);
        }
        return;
      }

      inReplicaSet(db, pods, status, function(err) {
        finish(err, db);
      });
    });
  });
};
```

主要 workloop 会定时地从 Kubernetes 集群中通过 label 来筛选 MongoDB 集群实例 pod。根据当前实例在集群的状态（notInReplicaSet，invalidReplicaSet 和 inReplicaSet），然后结合这些 pod 的状态，来更新 replica set config。

当前实例在集群的三种状态，分别采取的措施：
* notInReplicaSet：如果其他 pod 在 rs 中，不用操作，其他 pod 会将该 pod 加入到 rs 中；如果其他 pod 都不在 rs 中，触发 election。
* invalidReplicaSet：rs 失效，触发 election。如果没有赢得选举，就什么都不做；如果赢得选举，会重新 init replica set config。
* inReplicaSet：如果是 primary，则将其他 pod 添加到 rs；如果不是 primary 但还有其他 primary，则什么都不做；如果没有 primary，则触发 election。


## Setup Cluster

mongodb-statefulset.yaml

```
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    helm.sh/namespace: default
    helm.sh/path: mongo
    helm.sh/release: infra-mongo
  creationTimestamp: 2018-05-24T07:24:17Z
  generation: 51
  labels:
    controller.caicloud.io/chart: mongo
    controller.caicloud.io/release: infra-mongo
  name: infra-mongo-mongo-v1-0
  namespace: default
  ownerReferences:
  - apiVersion: release.caicloud.io/v1alpha1
    kind: Release
    name: infra-mongo
    uid: 10519cea-5cb1-11e8-8fec-5254000a3441
  resourceVersion: "1148816"
  selfLink: /apis/apps/v1/namespaces/default/statefulsets/infra-mongo-mongo-v1-0
  uid: 77ed3109-5f23-11e8-a120-525400d74dbf
spec:
  podManagementPolicy: OrderedReady
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      controller.caicloud.io/chart: mongo
      controller.caicloud.io/name: infra-mongo-mongo-v1-0
      controller.caicloud.io/release: infra-mongo
  serviceName: mgo-cluster
  template:
    metadata:
      annotations:
        helm.sh/namespace: default
        helm.sh/path: mongo
        helm.sh/release: infra-mongo
      creationTimestamp: null
      labels:
        controller.caicloud.io/chart: mongo
        controller.caicloud.io/name: infra-mongo-mongo-v1-0
        controller.caicloud.io/release: infra-mongo
    spec:
      containers:
      - args:
        - mongod
        - --replSet
        - rs0
        - --bind_ip
        - 0.0.0.0
        - --smallfiles
        - --noprealloc
        - --clusterAuthMode
        - keyFile
        - --keyFile
        - /data/config/mongodb-keyfile
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: MONGO_INITDB_ROOT_USERNAME
          value: admin
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: Pwd123456
        image: mongo:3.6.4
        imagePullPolicy: Always
        name: mongo
        ports:
        - containerPort: 27017
          name: tcp-27017
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /docker-entrypoint-initdb.d
          name: init-js
      - env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: MONGODB_USERNAME
          value: admin
        - name: MONGODB_PASSWORD
          value: Pwd123456
        - name: MONGODB_DATABASE
          value: admin
        - name: MONGO_SIDECAR_POD_LABELS
          value: controller.caicloud.io/release=infra-mongo
        - name: MONGO_PORT
          value: "27017"
        - name: KUBERNETES_MONGO_SERVICE_NAME
          value: mgo-cluster
        image: mongo-k8s-sidecar
        imagePullPolicy: Always
        name: mongo-sidecar
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /data/db
          type: ""
        name: mongo-storage
      - configMap:
          defaultMode: 420
          items:
          - key: init.js
            path: init.js
          name: init-js
        name: init-js
      - name: mongo-key
        secret:
          defaultMode: 384
          secretName: mongo-key
  updateStrategy:
    type: RollingUpdate
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    helm.sh/namespace: default
    helm.sh/path: mongo
    helm.sh/release: infra-mongo
  creationTimestamp: 2018-05-21T04:40:31Z
  labels:
    controller.caicloud.io/chart: mongo
    controller.caicloud.io/release: infra-mongo
    service.caicloud.io/export: "true"
  name: mgo-cluster
  namespace: default
  ownerReferences:
  - apiVersion: release.caicloud.io/v1alpha1
    kind: Release
    name: infra-mongo
    uid: 10519cea-5cb1-11e8-8fec-5254000a3441
  resourceVersion: "5456"
  selfLink: /api/v1/namespaces/default/services/mgo-cluster
  uid: 17ff6d28-5cb1-11e8-94c2-52540017abeb
spec:
  clusterIP: None
  ports:
  - name: tcp-27017
    port: 27017
    protocol: TCP
    targetPort: 27017
  selector:
    controller.caicloud.io/name: infra-mongo-mongo-v1-0
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
data:
  init.js: |
    var mgo = new Mongo('127.0.0.1:27017');
    var users = ["cyclone", "devops-admin", "cargo-admin"];

    for (i=0; i < users.length; i++) {
        user = users[i];
        db = mgo.getDB(user);
        u = db.getUser(user);
        print(pwd)
        if (u === null) {
            print("user is not found, add this user");
            db.createUser({'user': user, 'pwd': user, roles: [ { role: "dbOwner", db: user } ]});
        } else {
            print("user is found");
        }
    }
kind: ConfigMap
metadata:
  annotations:
    helm.sh/namespace: default
    helm.sh/path: mongo
    helm.sh/release: infra-mongo
  creationTimestamp: 2018-05-23T10:19:46Z
  labels:
    controller.caicloud.io/chart: mongo
    controller.caicloud.io/release: infra-mongo
  name: init-js
  namespace: default
  ownerReferences:
  - apiVersion: release.caicloud.io/v1alpha1
    kind: Release
    name: infra-mongo
    uid: 10519cea-5cb1-11e8-8fec-5254000a3441
  resourceVersion: "471813"
  selfLink: /api/v1/namespaces/default/configmaps/init-js
  uid: d13a6454-5e72-11e8-94c2-52540017abeb
```

备注：因为将 secret key mount 到 mongodb pod中，存在 permission 问题。 所以，自己构建 mongodb 的镜像，将 mongodb-keyfile 加入到镜像中。

# Reference

- [MongoDB Replica Set](https://docs.mongodb.com/manual/replication/)
- [Enforce Keyfile Access Control in a Replica Set](https://docs.mongodb.com/v3.2/tutorial/enforce-keyfile-access-control-in-existing-replica-set/)
- [Docker Hub MongoDB](https://hub.docker.com/_/mongo/)
- [MongoDB statefulset for kubernetes with authentication and replication](https://gist.github.com/thilinapiy/0c5abc2c0c28efe1bbe2165b0d8dc115)
- [Running MongoDB on Kubernetes with StatefulSets](https://kubernetes.io/blog/2017/01/running-mongodb-on-kubernetes-with-statefulsets/)
- [Deploying a MongoDB Replica Set as a GKE Kubernetes StatefulSet](http://pauldone.blogspot.com/2017/06/deploying-mongodb-on-kubernetes-gke25.html)