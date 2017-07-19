---
layout:     post
title:      "Kubernetes源码分析：Client-go"
subtitle:   "Kubernetes Source Code Reading: Client-go"
date:       2017-07-16
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Kubernetes
    - 源码分析
---

> The version of Kubernetes Client-go source code is **`release-4.0`**.

2015年8月使用Go语言搭建基于Kubernetes的PaaS平台的时候，还没有任何Kubernetes的Go client。
只能把Kubernetes的核心源码撸一遍，通过源码级别集成Kubernetes，直接调用Kubernetes的一些代码，来实现Kubernetes资源的增删改查，以及watch等操作。这种实现方式存在三个主要问题：
1. 必须对Kubernetes的核心源码非常熟悉；
2. 直接依赖Kubernetes源码，导致Kubernetes大版本升级的时候，可能需要更新依赖的Kubernetes源码；
3. 自己需要实现的代码量比较大。

最近基于Kubernetes官方提供的Go client [Client-go](https://github.com/kubernetes/client-go)又开发了一次PaaS平台，发现Client-go已经将Kubernetes的各种操作封装地很好，使用起来非常方便和清晰。因此，基本上一个星期的时间，就把Kubernetes相关的绝大部分功能都实现了。

Kubernetes官方从2016年8月份开始，将Kubernetes资源操作相关的核心源码抽取出来，独立出来一个项目[Client-go](https://github.com/kubernetes/client-go)，作为官方提供的Go client。Kubernetes的部分代码也是基于这个client实现的，所以对这个client的质量、性能等方面还是非常有信心的。目前，Kubernetes官方提供clients只有Go client和[Python Client](https://github.com/kubernetes-incubator/client-python)，而且他们已经还在[Kubernetes Clients](https://github.com/kubernetes-client)中开始尝试提供Java、Javascript等语言的clients，不过这些语言的clients还在不断完善中。像Java的Kubernetes client，官方的建议还是[fabric8io/kubernetes-client](https://github.com/fabric8io/kubernetes-client)。


## Kubernetes Clientset

众所周知，Kubernetes的API是非常庞大而且复杂的。庞大体现在操作的Resources种类非常多，例如：namespace、service、pod、node、deployment、daemonset、ingress、configmap等；复杂体现在API成熟的程度，例如：v1、v1beta1、v1alpha1等。为了方便对这些API进行有效管理，将它们根据功能进行分组：core组包括一些最核心的resource的操作，例如namespace、service、pod、node等；extensions组包括一些扩展的resource的操作，例如daemonset、deployment、replicaset、ingress等；rbac组包括一些权限控制相关的resource的操作，例如role、clusterrole等。

Kubernetes clientset就是这些组的client的集合，集合中的每个client只能操作相应的group中的resource，其他group中的resource是无法进行操作的。

[kubernetes/clientset.go](https://github.com/kubernetes/client-go/blob/release-4.0/kubernetes/clientset.go#L46-L122)
```go
type Interface interface {
	Discovery() discovery.DiscoveryInterface
	AdmissionregistrationV1alpha1() admissionregistrationv1alpha1.AdmissionregistrationV1alpha1Interface
	// Deprecated: please explicitly pick a version if possible.
	Admissionregistration() admissionregistrationv1alpha1.AdmissionregistrationV1alpha1Interface
	CoreV1() corev1.CoreV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Core() corev1.CoreV1Interface
	AppsV1beta1() appsv1beta1.AppsV1beta1Interface
	// Deprecated: please explicitly pick a version if possible.
	Apps() appsv1beta1.AppsV1beta1Interface
	AuthenticationV1() authenticationv1.AuthenticationV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Authentication() authenticationv1.AuthenticationV1Interface
	AuthenticationV1beta1() authenticationv1beta1.AuthenticationV1beta1Interface
	AuthorizationV1() authorizationv1.AuthorizationV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Authorization() authorizationv1.AuthorizationV1Interface
	AuthorizationV1beta1() authorizationv1beta1.AuthorizationV1beta1Interface
	AutoscalingV1() autoscalingv1.AutoscalingV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Autoscaling() autoscalingv1.AutoscalingV1Interface
	AutoscalingV2alpha1() autoscalingv2alpha1.AutoscalingV2alpha1Interface
	BatchV1() batchv1.BatchV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Batch() batchv1.BatchV1Interface
	BatchV2alpha1() batchv2alpha1.BatchV2alpha1Interface
	CertificatesV1beta1() certificatesv1beta1.CertificatesV1beta1Interface
	// Deprecated: please explicitly pick a version if possible.
	Certificates() certificatesv1beta1.CertificatesV1beta1Interface
	ExtensionsV1beta1() extensionsv1beta1.ExtensionsV1beta1Interface
	// Deprecated: please explicitly pick a version if possible.
	Extensions() extensionsv1beta1.ExtensionsV1beta1Interface
	NetworkingV1() networkingv1.NetworkingV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Networking() networkingv1.NetworkingV1Interface
	PolicyV1beta1() policyv1beta1.PolicyV1beta1Interface
	// Deprecated: please explicitly pick a version if possible.
	Policy() policyv1beta1.PolicyV1beta1Interface
	RbacV1beta1() rbacv1beta1.RbacV1beta1Interface
	// Deprecated: please explicitly pick a version if possible.
	Rbac() rbacv1beta1.RbacV1beta1Interface
	RbacV1alpha1() rbacv1alpha1.RbacV1alpha1Interface
	SettingsV1alpha1() settingsv1alpha1.SettingsV1alpha1Interface
	// Deprecated: please explicitly pick a version if possible.
	Settings() settingsv1alpha1.SettingsV1alpha1Interface
	StorageV1beta1() storagev1beta1.StorageV1beta1Interface
	StorageV1() storagev1.StorageV1Interface
	// Deprecated: please explicitly pick a version if possible.
	Storage() storagev1.StorageV1Interface
}

// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
	*discovery.DiscoveryClient
	*admissionregistrationv1alpha1.AdmissionregistrationV1alpha1Client
	*corev1.CoreV1Client
	*appsv1beta1.AppsV1beta1Client
	*authenticationv1.AuthenticationV1Client
	*authenticationv1beta1.AuthenticationV1beta1Client
	*authorizationv1.AuthorizationV1Client
	*authorizationv1beta1.AuthorizationV1beta1Client
	*autoscalingv1.AutoscalingV1Client
	*autoscalingv2alpha1.AutoscalingV2alpha1Client
	*batchv1.BatchV1Client
	*batchv2alpha1.BatchV2alpha1Client
	*certificatesv1beta1.CertificatesV1beta1Client
	*extensionsv1beta1.ExtensionsV1beta1Client
	*networkingv1.NetworkingV1Client
	*policyv1beta1.PolicyV1beta1Client
	*rbacv1beta1.RbacV1beta1Client
	*rbacv1alpha1.RbacV1alpha1Client
	*settingsv1alpha1.SettingsV1alpha1Client
	*storagev1beta1.StorageV1beta1Client
	*storagev1.StorageV1Client
}
```

整个Clientset的数据结构图如下：
![clientset](/img/in-post/k8s-clientgo/k8s-clientset-structure.png)

由Clientset获得操作deployment的client的流程图如下：
![clientset](/img/in-post/k8s-clientgo/k8s-clientset-deployment.png)

