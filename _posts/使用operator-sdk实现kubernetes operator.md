---
layout:     post
title:      使用operator-sdk实现kubernetes operator 
subtitle:   使用operator-sdk实现kubernetes operator 
date:       2020-07-30
author:     ica10888
catalog: true
tags:
    - kubernetes
---


# 使用operator-sdk实现kubernetes operator 

尝试使用 operator-sdk 实现了一个简单的 kubernetes operator ：[multi-tenancy-operator](https://github.com/ica10888/multi-tenancy-operator)

简单控制和监控helm部署应用

### 为什么使用 kubernetes operator

在 kubernetes 当中，有一个十分重要的概念就是 CRD 模型，而 kubernetes operator 就是开发利用 CRD 模型实现自己需要的逻辑。事实上 kubernetes API 的很多思想也是基于使用 yaml 来管理环境状态， Kubernetes is the new database 。

而实时上就目前而言，很多数据库，中间件都在使用 operator 来实现管理集群状态。如 tidb-operator , mysql-operator ,  elasticsearch-operator (ECK)  , 这样使部署复杂集群（如主从，备份，集群多节点通讯等）和更改集群配置变得简单，这是一种趋势，值得kubernetes 团队去实现。

### 主要实现

事实上使用operator-sdk编写operator很简单，主要就使用了以下几个命令

``` shell
operator-sdk new <...>-operator # 在当前目录下生成一个 golang 基础项目
operator-sdk add ... # 生成一个类型的CRD对应的 go 文件，或者控制器
operator-sdk build <docker-registry>/<docker-name>:<tag> # 制作镜像
docker push <docker-registry>/<docker-name>:<tag> # 推送镜像
operator-sdk generated ... # 通过 code-generated 生成一些如 crd yaml 和 deepcopy go 文件
```

还有一些命令，可以去帮助里面查看

然后基本上就是主要实现一个 Reconcile 回调函数，一旦集群里面的  controller API 有更新，都会调用 这个函数，执行里面自己编写的逻辑，原理很简单。

然后就是 list & watch 机制。kubelet 的 podsync ， kube-scheduler 的 informer （不太准确，事实上 informer  也可以说是属于 client-go 的）都有使用这种机制。当然最简单的是使用 client-go 的 clientSet , 其实就是调用  kube-apiserver 的 URL 请求，如果去看对应就是一个 builder 设计模式拼接的 URL ，然后如 informer 的一个核心构造体 cache.ListWatch 也是这两个方法。

### 参考

[Operator-SDK简介- Operator](https://www.operator.org.cn/bian-xie-operator)

[Kubernetes is the new database](https://zhuanlan.zhihu.com/p/144273083)
