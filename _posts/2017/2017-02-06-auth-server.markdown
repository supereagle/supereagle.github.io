---
layout:     post
title:      "微服务架构中Auth Server设计"
subtitle:   "基于OAuth 2.0的Pluggable认证授权系统的设计"
date:       2017-02-06
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - OAuth
    - Microservices
---

## 微服务架构中的Auth Server
微服务架构的系统中，每个服务只负责单一的一小块业务。如果该业务中的资源涉及到权限控制，就需要通过统一的Auth Server进行权限管理。多个服务中的业务逻辑不一样，涉及需要保护的资源也不一样。如果Auth Server中使用统一的一套权限管理规则，就会存在复杂、冗余和不易扩展的问题。通过Pluggable的Auth Server设计，使每个业务的认证授权是可插拔的。因此，每个微服务中只需要关心和实现自己业务相关的权限管理，同时Auth Server中各业务的权限管理相互独立、互不干扰。

## Auth Server系统设计

#### OAuth流程

![drawing](/img/in-post/auth-server/auth-sequence-diagram.png)

**OAuth组件**

- *Resource Owner* : 资源访问权限的赋予者
- *Client* : 资源请求者
- *Resource Server* : 资源持有者，能够根据token接受并响应资源请求
- *Auth Server* : 资源访问权限的认证、授权者
	* AuthN : 用户认证，判断用户身份是否合法
	* AuthZ : 用户授权，判断用户是否有资源访问的权限
- *Auth Plugins* :
	* Ticket Gen : 在Resource Server端，如果请求的token不存在或者无效，根据请求生成ticket
	* Ticket Parser : 在Auth Server端，解析ticket
	* Token Gen : 在Auth Server端，在权限认证结束后，根据认证结果生成对应的token

**OAuth流程**
1. Resource Owner对资源进行赋权
2. Client请求Resource Server上的资源：
请求被Resource Server接受后，先判断token是否存在以及是否有效。如果不存在或者无效，就由Ticket Gen根据请求信息生产ticket，并将该ticket附带在向Client返回的401响应中。

3. Client请求Auth Server请求token：
Client接收到401响应后，从响应中拿到ticket，然后根据ticket中信息向Auth Server请求token。

4. Auth Server通过认证、授权后生成token：
Auth Server接收到token请求后，通过Ticket Parser解析和校验ticket，然后根据ticket中的资源访问信息和用户信息依次进行认证和授权。认证判断用户身份是否合法，授权则判断用户是否有资源访问的权限。如果认证或授权失败，则返回401响应。否则，根据认证授权结果生成token并返回给Client。

#### Plugable Auth Server系统设计

![drawing](/img/in-post/auth-server/auth-sever-architecture.png)

上图示例中，被保护的资源为Products和Orders，分别有Products Server和Orders Server两个微服务对其提供服务。它们使用统一的Auth Server进行权限认证和授权。Auth Server中Ticket Parser和Authn/Authz都是Pluggable的，使用统一Token Gen生产token。其实Token Gen也可以是Pluggable，可以根据Service选择不同的token产生策略。

**Ticket结构**

示例：realm="http://oath-server.taobao.com/api/token",service="products",scope="products:foods:view"

- Realm : Auth server url
- Service : 指定使用的auth service，即auth server中的authn/authz plugin
- Scope : 主要是resource type，name以及actions

**Token结构**

Token由经过编码的3部分组成，之间通过"."分割。

- Header : 
- Claims : 一组resource及其对应的actions
- Signature : 

## Reference

- [User-Managed Access (UMA) Profile of OAuth 2.0](https://docs.kantarainitiative.org/uma/draft-uma-core-v1_0_1.html)
- [The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
- [Authorization In Docker Registry](https://github.com/docker/libtrust)

