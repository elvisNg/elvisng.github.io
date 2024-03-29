---
layout: post
title: "Kubernetes-Introduce"
subtitle: 'kubernetes介绍'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - kubernetes
---



下面是简单介绍一下k8的各个组件，后面会详读源码展开介绍。

Kubernetes主要由以下几个核心组件组成：

- etcd保存了整个集群的状态；
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

![](https://raw.githubusercontent.com/elvisNg/elvisng.github.io/master/img/in-post/post-kubernetes-introduce/resource.png)





