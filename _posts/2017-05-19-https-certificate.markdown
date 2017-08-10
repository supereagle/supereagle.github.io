---
layout:     post
title:      "HTTPS原理解析及自建证书生产"
subtitle:   "Security communication between microservices"
date:       2017-05-19
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Security
---

## HTTPS原理

HTTPS是基于HTTP协议的更加安全的协议。HTTPS在HTTP协议与TCP协议间加一层SSL协议，负责对这两层间通信的数据进行加密（HTTP内容加密后传到TCP）和解密（TCP内容加密后传到HTTP）。

HTTPS虽然通过SSL对通信数据进行加密，但是并不能保证足够安全。因为无法确认服务器的安全性，很有可能访问的就是钓鱼网站。因此，HTTPS必须结合SSL和CA证书，来保证传输和认证的安全。

## 概念

- **SSL（Secure Socket Layer，安全套接字层）**：是Netscape公司设计的主要用以保障在Internet上数据传输之安全，利用数据加密(Encryption)技术，可确保数据在网络上之传输过程中不会被截取。它已被广泛地用于Web浏览器与服务器之间的身份认证和加密数据传输。
- **TLS（Transport Layer Security，传输层安全协议）**：用于两个应用程序之间提供保密性和数据完整性。它建立在SSL 3.0协议规范之上，是SSL 3.0的后续升级版本，可以理解为SSL 3.1。
- **HTTPS（Hypertext Transfer Protocol Secure，超文本传输安全协议）**：在HTTP（超文本传输协议）基础上，添加基于SSL/TLS协议的安全层。HTTP协议直接放置在TCP协议之上，而HTTPS提出在HTTP和TCP中间加上一层加密层。从发送端看，这一层负责把HTTP的内容加密后送到下层的TCP，从接收方看，这一层负责将TCP送来的数据解密还原成HTTP的内容。
- **对称加密（Symmetric Cryptography）**：加密和解密时使用相同的密钥，或是使用两个可以简单地相互推算的密钥。主要用于维持专属的通讯联系，要求通讯双方拥有相同的密钥。常见的对称加密算法有DES、3DES、Blowfish、IDEA、RC4、RC5、RC6和AES。
- **非对称加密（Asymmetric Cryptography）**：需要一对密钥：公钥和私钥。用其中一个密钥加密之后，必须用其对应的另外一个密钥才能解密。常见的非对称加密算法有：RSA、ECC（移动设备用）、Diffie-Hellman、El Gamal、DSA（数字签名用）。

> 对称加密与非对称加密比较：
> * 由于非对称加密虽然更加安全，但是速度远远慢于对称加密，所以它一般用于密钥交换。双方通过公钥算法协商出一份密钥，然后通过对称加密来通信。
> * 非对称加密可以实现数字签名，但是对称加密却无法实现。

## HTTPS具体实现

### TLS握手详解

![drawing](/img/in-post/https-certificate/tls-handshake-of-https.png)

1. ClientHello
客户端主要向服务器提供以下信息:
* 支持的安全协议版本
* 一个客户端生成的随机数
* 支持的加密方法
* 支持的压缩方法

2. SeverHello
服务器的回应包含以下内容：
* 确认使用的加密通信协议版本
* 一个服务器生成的随机数
* 确认使用的加密方法
* 服务器证书

3. ClentDone
向服务器发送以下信息：
* 随机数Premaster Secret
* 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
* 客户端握手结束通知

4. ServerDone
向客户端最后发送以下信息：
* 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
* 服务器握手结束通知，表示服务器的握手阶段已经结束

> 通过Wireshark抓包获得详细内容，可以参考[《SSL/TLS 握手过程详解》](http://www.jianshu.com/p/7158568e4867)。

Client和Server经过TLS握手最终都会产生一个相同的Session Key，对session中的通讯内容进行加密和解密。Session Key是通过上面TLS握手过程中的三个随机数产生的：Client Random，Server Random以及Premaster Secret。

### CA数字证书

TLS握手中的非对称加密部分，需要用到CA数字证书，来保证TLS握手过程不会被恶意地窃听、篡改和冒充。CA数字证书是KPI（Public Key Infrastructure，公开密钥基础设施）的一个重要组件。KPI是一套安全标准，利用公开密钥技术，解决网络安全的一种基础设施。

KPI主要组件：
- **CA（Certificate Authority，证书授权中心）**：负责管理数字证书
- **RA（Registration Authority，注册管理中心）**：将CA证书中的用户身份与公开密钥关联起来
- **数字证书（Digital Certificates ）**：可公开，分发过程中不可被篡改，整个KPI体系基于数字证书来保证安全

CA证书

功能：认证服务器的安全性。身份认证和密钥分发。负责管理数字证书：颁发，更新，查询，作废和归档。

来源：
- **自签**：自己颁发，免费，手动导入证书
- **国际公认证书机构申请**：申请，收费，证书自动加到浏览器中。

分类（难道依次增加）：
- DV（Domain Validation）
- OV（Organization Validation）
- EV（Extended Validation）

数字证书

经CA签名的包含公开密钥拥有者信息以及公开密钥的文件，作用是证明证书中列出的用户合法拥有证书中列出的公开密钥。

自签CA证书

国际公认证书机构申请的证书，需要花费时间和金钱的成本，甚至有可能还申请不下来。而且国际公认证书机构只能签署域名证书，不支持签署IP证书。因此，对于部分应用场景，例如小型网站或博客、企业内部网络访问，可以考虑通过自签CA证书来构建安全网络。

## Reference
- [SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
- [OpenSSL 与 SSL 数字证书概念贴](https://segmentfault.com/a/1190000002568019)
- [SSL/TLS 握手过程详解](http://www.jianshu.com/p/7158568e4867)
- [TLS完全指南](https://github.com/k8sp/tls)
