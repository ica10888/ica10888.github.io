---
layout:     post
title:      Tekton 问题排查和 github 社区
subtitle:   Tekton 问题排查和 github 社区
date:       2020-03-22
author:     ica10888
catalog: true
tags:
    - CI/CD
    - tekton
    - kubernetes
---

# Tekton 问题排查和 github 社区

Tekton 是一个 kubernetes 源生的 CICD 工具，本质上是 kubernetes Operator  的一种实现方式。使用了基于角色的访问控制 ( Role-based access control，RBAC ) 的一种以角色为基础的访问控制模型 ，和自定义资源（CustomResourceDefinition，CRD）允许对Kubernetes API 的扩展。

### 基本原理

``` shell
[root@node1 ~]# kubectl get pod -n tekton-pipelines
NAME                                          READY   STATUS    RESTARTS   AGE
tekton-dashboard-68d675b654-gj66x             1/1     Running   1          17d
tekton-pipelines-controller-d69766ddd-q4zw9   1/1     Running   1          19d
tekton-pipelines-webhook-64b4998468-wrwq9     1/1     Running   0          4d1h
```

在这里 的 `pod` 也就是之前说的 Operator  的概念。dashboard 应该不会用上，应该直接操作 k8s API 就可以了。controller  用来监听上述CRD的事件，执行Tekton的各种CI/CD逻辑，一个webhook用于触发后创建CRD资源。 

Tekton 中自定义了几种 CRD ，分别可以对应 jenkins 里面的一些对象

| Tekton           | jenkins                           | 描述                                                         |
| ---------------- | --------------------------------- | ------------------------------------------------------------ |
| PipelineResource | Build with Parameters             | 也就是需要传入的参数                                         |
| Pipeline         | Pipeline                          | 也就是一个流水线，流水线中定义的变量就是PipelineResource 传入的参数，通过改变变量来运行不同的 Pipeline，流水线中可以定义多个不同的阶段 |
| PipelineRun      | 执行一次Pipeline                  | 相当于执行一次流水线                                         |
| Task             | Pipeline 的 Stage                 | 也就是流水线的一个阶段的定义，重新在一个新的 pod 里执行      |
| TaskRun          | 执行一次Pipeline 的其中一个 Stage | 一般而言会执行一次流水线，也可以单独执行流水线的某个阶段     |

### 排查问题

本周，在使用 Tekton 的时候，遇到了一个严重的问题。也就是在使用 Tekton 的 pipelineRun 的时候，会突然创建出一堆无用的 PVC 。而这些 PVC 自动地绑定了 PV 资源，造成了一些影响。

经过 google 和 github 的搜索，我发现了有人在一个 `issues` 中提到了类似的问题，和我遇到的问题大致相似。

https://github.com/tektoncd/pipeline/issues/937

太长不看系列，基本来说就是在第一次关闭这个 `issues` 之后，发现问题并没有解决，又重新打开了这个  `issues`  。然后在一个新的 `issues` 里进行了讨论

https://github.com/tektoncd/pipeline/issues/1673

之后在这个 PR 提交并修复了这个问题

https://github.com/tektoncd/pipeline/pull/1545

如果看相关提交，可以看见是加入了一个判断参数，修复了这个问题

而这次 PR 合并的日期是 2019年 11 月 13 日，然后我发现我使用的 tekton 版本在这个时间点之前，然后就升级了一下 tekton 版本，解决了这个问题。

### github 贡献

顺带一提，如何对 github 上的开源项目贡献

一般来说，不同的社区的贡献规则是大同小异的，具体的需要看各社区的文档

以 kubernetes 社区为例

https://kubernetes.io/zh/docs/contribute/start/

太长不看系列

- 一定要遵守 [CNCF 社区规范](https://github.com/cncf/foundation/blob/master/code-of-conduct.md)

- 不仅仅是 PR ，修复文档，提出新功能的 `issues` ，帮助别人解决问题也是一种贡献
- 提出 `issues` 或者通过 `pull request` 提交代码

事实上，一般来说，如果发现了大型项目的代码存在 bug 或问题，应该先 fork 一份到自己仓库，然后 commit 提交代码，给原项目提出 PR

需要注意的是 **礼貌沟通和认真阐述问题是一切贡献的前提，特别是这种国际化的项目，一定要用英语交流**  （即使是国人的项目，之前我提 `issues` ，也是可以看出对方是中国人，但是也用英语交流）

这个时候，应当提出一个 `issues` , 在填写的时候按照原本的格式，一般是几个问题，如实填写就行了。然后社区人员会反馈，一般来说就是有没有人遇到过相同的问题，或者同意 PR

这个时候就可以提交 PR 了

提交之后，一般会再讨论一下，然后就是 review 过程了

如果没有人注意到 PR , 像 kubernetes 这种使用了 bot 的，可以用 `/cc` 来召唤

这里写一下常用的交流命令

- WIP   Work in progress, do not merge yet. // 开发中
- LGTM Looks good to me. // Riview 完别人的 PR ，没有问题
- PTAL Please take a look. // 帮我看下，一般都是请别人 review 自己的 PR
- CC Carbon copy // 一般代表抄送别人的意思
- RFC  —  request for comments. // 我觉得这个想法很好, 我们来一起讨论下
- IIRC  —  if I recall correctly. // 如果我没记错
- ACK  —  acknowledgement. // 我确认了或者我接受了,我承认了
- NACK/NAK — negative acknowledgement. // 我不同意

这个时候就会有人来 review 代码

这里可以看到相关的人

https://github.com/kubernetes/kubernetes/blob/master/.github/OWNERS

然后就是投票了，不同的社区设置不一样，有的需要一个人同意，有的需要多人同意（之前我就有一次特别尴尬的 PR ，我看到项目的创始人同意了我的 PR，然后我就把这个 PR 关了，事实上代码并没有合并进去，那个项目需要多人同意才行，我就重新打开了这个 PR ，然后等另外个人同意了，代码才合并了进去）

review 正式通过了，代码就合并进去了，一般来说 reviewer 会打出 `/lgtm`

这个时候就是一名项目的 committer 了，进一步可以成为 contributor / collaborator

顺带说一下 PR 讨论的时候不通过的问题，一般来说是补充测试代码

这个时候需要保持仍然是一个 commit 提交，但是补充代码需要再做一次 commit 提交，需要将这两次 commit 合并，需要使用 git 命令

``` shell
git rebase -i HEAD~2
```

将两次 commit 合并，这里会打开一个 vi 界面

``` shell
git push -f
```

然后在原有的 PR 上提过去就行了，没必要开一个新 PR
