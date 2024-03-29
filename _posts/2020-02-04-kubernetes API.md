---
layout:     post
title:      kubernetes API
subtitle:   kubernetes API
date:       2020-02-04
cover:      'https://raw.githubusercontent.com/ica10888/banner/master/8b8affb4ba9715b94fed691d2a722f1417ef6a16.jpg@1320w_720h.webp'
author:     ica10888
catalog: true
tags:
    - kubernetes
---


# kubernetes API

kubernetes API 是 kubernetes 集群的系统描述性配置的基础。存储在 etcd 中，也是操作kubernetes  集群的基本单位。

在这里主要描述几个主要的资源

pod 是启动容器运行时的基本单位， **控制器API** 包括 

- `ReplicaSets`，是 ReplicationController 的增强版，一个基础的控制器
- `Deployment` ，拥有更多的功能，包括声明式的更新以及许多其他有用功能
- `StatefulSet` ，维护了一个固定的 ID，可以顺序执行pod，一般用于有顺序要求的集群组件，如 es 集群
- `DaemonSet`，在每一个 node 节点上启动一个 pod 的副本，一般用于运行集群存储和日志收集
- `Job` 和 `CronJob` ，定时启动一个 pod ，完成一些任务

**存储API** 可以将数据持久化，其中 `Volume` 和 `Persistent Volumes` 是唯一相互绑定的关系，`StorageClass` 限制了在绑定过程中策略，在前面两者声明了相同的 `StorageClass` 时，才能相互绑定。

存储类型是多样化的，有基于主机磁盘或内存的 emptyDir ，同理包括 `ConfigMap` 和 `Secret` 。也可以挂载在宿主机上的 HostPath  ，也有云平台支持的网络挂载卷或者如 Ceph 等支持。

**网络API** 包括基本单位 `Service` ，可以负载均衡到对应的多个相同的 pod 上，有 4 种类型

- ClusterIP，基础类型，通过 `Service`  IP 加对应的端口访问
- NodePort，**每个 node 节点** 的端口都可以访问这个服务
- LoadBalancer
- ExternalName

无论是集群内部还是外部 都可以通过对应规则的 DNS 服务器寻找 IP 地址

`Ingress` 本质上是一个 lua 写的 nginx ，也就是 openresty 项目。lua是一个动态语言，动态配置可以实现外界访问内部集群，同时具有 nginx 的性能。

**基于角色权限控制API** 可以让我们有限制地访问 kubernetes API 资源。而 **自定义资源API** 可以自定义我们需要的kubernetes API ，并生成对应的 rest 接口。两者结合，配合上一个不断轮询访问获取 kubernetes API 变化的服务，能够实现一些功能，比如探查到变化，可以创建或改变其他的 kubernetes API 。

### 控制器API

Pod 是 kubernetes 集群最小编排单位。

Pod 在被创建出来的时候，会先创建一个中间容器 Infra ，这个容器占用极少的资源，使用的镜像是 `k8s.gcr.io/pause` ，而其他用户容器加入到 Infra  的网络空间，它们之间可以使用 localhost，使用同一个 IP ，并且 Pod 的生命周期只和 Infra 容器一致 。

其中 Init Containers 的生命周期，先于所有的 Containers。

在使用控制器的时候，可以设置滚动更新的规则

``` yaml
    strategy:
      rollingUpdate:
        maxSurge: 15%
        maxUnavailable: 15%
```

每次滚动更新 15 % 总数的 Pod

可以设置 `HorizontalPodAutoscaler` ，实现弹性伸缩

设置亲和性，让 Pod 部署到具有亲和性标签的 node 节点上，并让相同 Pod 不会出现在同一 node 节点上

``` yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: affinity.namespace
                operator: In
                values:
                - {{ .Values.affinity.namespace }}
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - {{ template "name" . }}
            topologyKey: kubernetes.io/hostname
```

NodeSelector 已经逐步废弃了，不应使用。

Deployment 部署的都是相同实例，无法解决部署具有主从关系、主备关系的服务。这个时候就需要使用 StatefulSet 。

StatefulSet 部署出来的 Pod，以编号结尾，从0 开始叠加。前一个 Pod 成功运行，才会启动下一个 Pod 。

### 存储API

Volume 和 Persistent Volumes 是唯一相互绑定的关系，如果需要指定绑定关系，那么可以添加一个 `Persistent Volume Claim` 的声明，并在对应的 Volumes 声明这个 `StorageClass`

```` yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
````

其中至少需要 1G 以上 Volume 才能绑定 ，当然这里的大小只是声明，并没有实际限制

而这里的 accessModes 的 ReadWriteOnce 表示只能有一个 Pod 使用这个 PV，是可读写的。ReadOnlyMany 表示能够被多个  Pod 使用，但是是只读。ReadWriteMany 表示能够被多个 Pod 读写。

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-disks
mountOptions:
- vers=4.0
- noresvport
parameters:
  archiveOnDelete: "true"
provisioner: cluster.local/nfs-provisioner-nfs-client-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

这是一个 nfs 的 StorageClass ，能够使用  `storageClassName: fast-disks` 声明的 Volume 

这里比较特殊，pv 是由 nfs-provisioner 提供的。PV 有多种方式，还有 local volumes 、glusterFS 、ceph 等等，有些可以直接声明 PV
 
一般而言是通过声明相同字段的 storageClassName 来区分绑定 PVC 和 PV
 
 ``` yaml
 kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi 
  storageClassName: local-storage
  
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 15Gi
  local:
    fsType: ext4
    path: /data/some-path
  storageClassName: local-storage

```

### 网络API

##### Service

实际上 Service 是由 kube-proxy 组件，加上 iptables 来共同实现负载均衡

``` shell
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

假如说有 3 个 Pod ，那么会有 3 条 iptables 记录。可以看到，第一条选中的概率是 1/3，然后未选中的，经过第二条的概率，有 1/2 可能性被选中。最后一条兜底一定会选中。

这样每个 Pod 选中的概率都是 1/3。

而设置了 NodePort 可以通过 宿主机 + 端口 来访问，NodePort 同样是一条 iptables  规则

具体实现是执行一次源地址转换（Source Network Address Translation，SNAT），将这个 IP 包的源地址替换成了 Pod 宿主机的 IP 地址。

而对返回的包同样要做 SNAT，防止客户端可能报错。

也可以将 `spec.externalTrafficPolicy` 字段设置为 local ，这样除了这个宿主机，访问其他宿主机会被 DROP 掉。

LoadBalancer 可以起到同样的效果，会启动一个网络负载平衡器，但是需要支持的 cloud provider 才行。

ExternalName 是在 coredns 里面增加一个 CNAME 记录

通过 coredns  访问 `<svc-name>.<namespace>.svc.cluster.local`

返回 `externalName` 配置的域名

##### DNS

在  kubernetes 集群内部，可以通过域名解析到 service 的 cluster-ip。

一般规则是，在同一命名空间下，可以通过

`<svc-name>`  解析到对应 IP

如果需要访问其他命名空间的 Service

那么通过  `<svc-name>.<namespace>` 来解析

对于集群外部，可以配置 kubernetes 的 DNS 服务器

``` shell
[root@node1 ~]# kubectl get pod -n kube-system | grep dns
coredns-8484b67f96-ncxzh                        1/1     Running     0          56d
coredns-8484b67f96-xj298                        1/1     Running     3          189d
```

解析规则是

`<svc-name>.<namespace>.svc.cluster.local`

还有一种特殊的 Service ，没有 cluster-ip ，称作 Headless Service ，一般对于固定的 Pod 名称，如 StatefulSet 使用

可以通过  `<pod-name>.<svc-name>.<namespace>.svc.cluster.local` 来访问

##### Ingress

Ingress 相当于可以通过域名来访问内部服务，可以通过设置 `metadata.annotations`  不同的 `kubernetes.io/ingress.class` 来启动多个 Ingress 集群

这里写一下特殊规则的常用配置

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
    kubernetes.io/ingress.class: nginx-public
    nginx.ingress.kubernetes.io/auth-snippet: |
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    nginx.ingress.kubernetes.io/proxy-body-size: 20m
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: backend-frontend-service-ingress
  namespace: prod
spec:
  rules:
  - host: ica10888.com
    http:
      paths:
      - backend:
          serviceName: backend-frontend-service
          servicePort: 8080
        path: /api(/|$)(.*)
```

这里通过注解，就像 nginx 一样能够配置 server 相关配置，也能够配置限制最大文件大小 20m

通过重新路径可以访问后端服务

也就是说访问 `ica10888.com/api/info` 相当于访问 `localhost:8080/info`

更多的配置规则查阅 Ingress 文档


###  基于角色权限控制API

 基于角色的访问控制 ( Role-based access control，RBAC ) 是一种以角色为基础的访问控制模型 。简单来说，就是将对象分为三种类型，分别是角色，被作用者和绑定关系。

- 角色（Role）: 对应的 kubernetes API 是 `Role`  和 `ClusterRole` ，将定义对象的操作权限 （增删查改等），以及作用的 API 对象和版本。

- 绑定关系（  RoleBinding ）：对应的 kubernetes API 是  `RoleBinding`  和 `ClusterRoleBinding` ，也就是角色和被作用者之间的关系。
- 被作用者（ Subject ）：对应的 kubernetes API 是  `ServiceAccount`  ，保存的是一个密钥，密钥的权限也就是绑定的角色定义的权限。

Role

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
```

`Role`  定义了作用的 API 对象和版本，以及允许的操作，但是限定了只能在自己的命名空间下起作用。而 `ClusterRole` 可以对不限定命名空间的对象使用。

RoleBinding

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: leader-locking-nfs-client-provisioner
subjects:
- kind: ServiceAccount
  name: nfs-client-provisioner
  namespace: default
```

`RoleBinding` 中一个角色绑定多个被作用者。其中也限定了只能在自己的命名空间下起作用。而 `ClusterRoleBinding` 可以对不限定命名空间的对象使用。

如果 subjects 的 `kind` 是 `Group`  ，通过 `system:serviceaccounts:<Namespace名字>` 可以实现 **用户组** 的绑定。

ServiceAccount

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: default
secrets:
- name: nfs-client-provisioner-token-4kkk5
```

其中在同一命名空间下有一个  `nfs-client-provisioner-token-4kkk5` 的 `secret` 。保存 namespace 、ca.crt 和  token。

pod 如果使用 ServiceAccount ,会声明如下信息

``` yaml
    serviceAccount: nfs-client-provisioner
    serviceAccountName: nfs-client-provisioner
```

这样 pod 将会挂载密钥到  /var/run/secrets/kubernetes.io/serviceaccount 路径下，使用该密钥就可以对集群有限制地访问。

### 自定义资源API

 自定义资源（CustomResourceDefinition，CRD）允许对Kubernetes API 的扩展 ，也就是说，可以使用 kubectl 像源生 API 一样 操作这些新定义的 API。

``` yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  labels:
    app: istio-pilot
  name: gateways.networking.istio.io
spec:
  conversion:
    strategy: None
  group: networking.istio.io
  names:
    categories:
    - istio-io
    - networking-istio-io
    kind: Gateway
    listKind: GatewayList
    plural: gateways
    shortNames:
    - gw
    singular: gateway
  scope: Namespaced
  version: v1alpha3
  versions:
  - name: v1alpha3
    served: true
    storage: true
```

也就是说，定义了一个版本号为 `v1alpha3` 的 API `gateways.networking.istio.io` 。可以像使用  kubectl 去操作这个 API。

当然本身 新的 API 是没有意义的，一般而言会通过 RBAC 一起使用。通过部署一些 `pod` ，轮询去访问 kubernetes 集群里面 新定义的 CRD，通过字段来执行不同的操作。这个 `pod` 一般被称作  Operator  。

举个例子来说，我们经常使用的 `kubectl top `  指令也就是使用的  Operator  来执行的（虽然这里有一点小小的区别，`v1beta1.metrics.k8s.io`  使用的不是 CRD 而是 k8s 本身的资源对象，可以使用 `kubectl get apiservice` 来查看 ）。

``` shell
[root@node1 ~]# kubectl get pod -n kube-system  | grep metrics-server
metrics-server-6f868c8b55-65pqp             1/1     Running            1          93d
```

也就是说，我们查询cpu和内存的过程 都是通过这个 Operator  进行操作的。这个 Operator  先定义了对 `pod` 的访问权限，和 metrics 相关资源的权限，然后通过  `serviceAccount`  就能访问和操作这些资源 。

