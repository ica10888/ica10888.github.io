---
layout:     post
title:      将 nexus 迁移到 kubernetes 集群
subtitle:   将 nexus 迁移到 kubernetes 集群
date:       2018-12-08
author:     ica10888
catalog: true
tags:
    - kubernetes
    - nexus
---

# 将 nexus 迁移到 kubernetes 集群

### helm 安装

使用 https://github.com/helm/charts 中的 stable/sonatype-nexus  安装到 kubernetes 集群中。

由于迁移之前的 nexus 版本是 3.9.0 ，需要修改 nexus 的镜像版本到 3.9.0 。

就3.9.0的以上版本来说， nexus 高版本（3.9.0 以上）对低版本（3.9.0）并不兼容。主要是使用的 elasticsearch 对低版本的文件结构无法识别。

然后暴露出服务就可以了。

### 复制文件

停止 nexus 服务。

使用`rsync -av nexus3/* 172.16.20.64:/data/nexus/`，将  sonatype-work/nexus3 以下几个文件和文件夹增量到 nexus  挂在的 kubernetes PersistentVolume下，最好先清空这几个文件和文件夹。

| 文件名            |
| ----------------- |
| backup            |
| blobs             |
| db                |
| elasticsearch     |
| generated-bundles |
| instances         |
| keystores         |



然后清空 cache 文件夹里面的内容。

### 启动服务

通过`kubectl scale -n nexus deployment --replicas=1 nexus-sonatype-nexus `，启动 nexus 就可以了。