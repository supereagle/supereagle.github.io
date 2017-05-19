---
layout:     post
title:      "HTTPS原理解析及自建证书生产"
subtitle:   "Security the communication between microservices"
date:       2017-05-19
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Security
---

# Category

- HTTPS原理
- 概念
- Reference

## HTTPS原理

HTTPS是基于HTTP协议的更加安全的协议。HTTPS在HTTP协议与TCP协议间加一层SSL协议，负责对这两层间通信的数据进行加密（HTTP内容加密后传到TCP）和解密（TCP内容加密后传到HTTP）。

HTTPS虽然通过SSL对通信数据进行加密，但是并不能保证足够安全。因为无法确认服务器的安全性，很有可能访问的就是钓鱼网站。因此，HTTPS必须结合SSL和CA证书，来保证传输和认证的安全。

## 概念

- **SSL（Secure Socket Layer，安全套接字层）**：是Netscape公司设计的主要用以保障在Internet上数据传输之安全，利用数据加密(Encryption)技术，可确保数据在网络上之传输过程中不会被截取。它已被广泛地用于Web浏览器与服务器之间的身份认证和加密数据传输。
- **TLS（Transport Layer Security，传输层安全协议）**：用于两个应用程序之间提供保密性和数据完整性。它建立在SSL 3.0协议规范之上，是SSL 3.0的后续升级版本，可以理解为SSL 3.1。
- **HTTPS（Hypertext Transfer Protocol Secure，超文本传输安全协议）**：在HTTP（超文本传输协议）基础上，添加基于SSL/TLS协议的安全层。HTTP协议直接放置在TCP协议之上，而HTTPS提出在HTTP和TCP中间加上一层加密层。从发送端看，这一层负责把HTTP的内容加密后送到下层的TCP，从接收方看，这一层负责将TCP送来的数据解密还原成HTTP的内容。

对称加密

非对称加密

## Reference

