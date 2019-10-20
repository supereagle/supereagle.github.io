---
layout:     post
title:      "结构化数据高效序列化方法：Google Protocol Buffers"
subtitle:   "An Efficient Serializing Method for Structured Data: Google Protocol Buffers"
date:       2017-04-19
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Protocol Buffers
    - Serialization
    - gRPC
---

Protocol Buffers是Google开源的一种针对结构化数据的跨平台、跨语言、可扩展的序列化方法。
目前，Protocol Buffers已经成为Google通用的数据模型，实际使用中有48K个不同message类型，并在ETCD，Docker和Kubernetes等热门开源项目中得到广泛使用。
Google开源的新RPC框架[gRPC](http://www.grpc.io/)，默认就是采用Protocol Buffers的数据传输协议，大大加快数据传输速度。


## Advantages
- **Flexible**：定义一次structured data，就可以在各种各样的数据流以及语言中，非常容易地read和write这些data
- **Extensible**：message中添加field，不会破坏backwards compatibility
- **Efficient**：相对于传统的XML存在诸多优势
  * are simpler
  * are 3 to 10 times smaller
  * are 20 to 100 times faster
  * are less ambiguous
  * generate data access classes that are easier to use programmatically

## Usage

### Protocol Compiler Installation
There are 2 installation ways：
- **C++ users**：[C++ Installation Instructions](https://github.com/google/protobuf/blob/master/src/README.md)
- **non-C++ users**：Install pre-built binary from [release page](https://github.com/google/protobuf/releases)

### Protobuf Runtime Installation
Protobuf Runtime（Protobuf Compiler Plugin）支持不同编程。
根据自己的编程语言，参考官方的[Protobuf Runtime Installation](https://github.com/google/protobuf#protobuf-runtime-installation)，选择相应的安装方法。

以Golang为例，其protobuf runtime安装非常简单：

```shell
$ go get -u github.com/golang/protobuf/protoc-gen-go
```

该命令执行两个操作：
- **Protobuf Runtime Library**：下载Golang最新的protobuf runtime library [golang/protobuf](https://github.com/golang/protobuf)到${GOPATH}/src目录下。可以通过调用该library的API对message进行write/read操作。
- **Protobuf Compiler Plugin**：安装Golang的protobuf compiler plugin `protoc-gen-go`到${GOPATH}/bin目录下。`protoc-gen-go`能够根据message类型的定义文件.proto，产生这些message在Golang语言中对应的message classes，用来访问和管理protocol buffer messages。

### Define Message Type
根据.proto文件的语法，在.proto文件中定义message类型。

**[Style Guide](https://developers.google.com/protocol-buffers/docs/style)**

遵循此Style规范，可以保证protocol buffer messages的定义，以及它们对应的classes的一致性，方便阅读。
- **Message**：Message name是驼峰格式；field name是下划线分隔的小写字母
- **Enums**：Enums name是驼峰格式；value name是下划线分隔的大写字母
- **RPC Services**：RPC services name和所有method，都是驼峰格式

**[Language Guide](https://developers.google.com/protocol-buffers/docs/proto)**

.proto文件的语法，通过protocol buffer language构造protocol buffer data。


### Generate Message Classes
根据.proto中定义message类型，不同语言的protobuf compiler plugin就可以产生这些message在该语言中对应的message classes。
如何理解产生的这些message classes，可以参考相应语言的[API Reference](https://developers.google.com/protocol-buffers/docs/reference/overview)。

### Write/Read Messages
根据支持Protocol Buffers的[Language List](https://github.com/google/protobuf/blob/master/docs/third_party.md)，选择合适的第三方API来使用这些message classes。

For example, Golang APIs for protocol buffers:
- **Writing a Message**：[proto.Marshal(pb Message)](https://github.com/golang/protobuf/blob/master/proto/encode.go)
- **Reading a Message**：[proto.Unmarshal(buf []byte, pb Message)](  https://github.com/golang/protobuf/blob/master/proto/decode.go)

## Proto3 vs Proto2

由于Protocol Buffers在开源前，已经在Google内部得到广泛使用，并且已经是Proto2版本，因此一开源就采用的是Proto2版本。
后来优化和改进的[Proto3](https://github.com/google/protobuf/releases/tag/v3.0.0)相对于Proto2，更加易用，并且支持更加广泛的语言。

选择Proto3的原则：
- 采用只有Proto3支持的language
- 使用新的gRPC
- 刚开始使用Protocol Buffers用户
- 因为API不兼容，不建议Proto2用户升级到Proto3，Proto2会继续长时间支持



## Reference
- [Protocol Buffers](https://developers.google.com/protocol-buffers/)
- [5 Reasons to Use Protocol Buffers Instead of JSON For Your Next Service](http://blog.codeclimate.com/blog/2014/06/05/choose-protocol-buffers/)
- [Serialization objects with protocol buffers in Golang](http://blog.ralch.com/tutorial/golang-proto-buffer/)
- [Go Demo for Protocol Buffers](https://github.com/supereagle/go-example/tree/master/protobuf)