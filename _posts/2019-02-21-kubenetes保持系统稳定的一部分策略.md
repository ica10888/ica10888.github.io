---
layout:     post
title:      kubenetes保持系统稳定的一部分策略
subtitle:   kubenetes保持系统稳定的一部分策略
date:       2019-01-03
author:     ica10888
catalog: true
tags:
    - kubernetes
    - linux
---

# kubenetes保持系统稳定的一部分策略

为了保持 kubenetes 集群的稳定性，分别从不同的层面加强系统的稳定性。

首先是在 linux 层面上管理，CGroup 是 Control Groups 的缩写，是 Linux 内核提供的一种可以限制、记录、隔离进程组 (process groups) 所使用的物力资源 (如 cpu memory i/o 等等) 的机制。 使用 CGroup 可以在 CPU 繁忙时分配给 kubelet 进程一部分CPU使用时间，预留内存和io资源等。 

然后是配置 kubelet 的启动参数，为 kube-reserved  和 system-reserved  预留计算资源。

接下来是制定 pod 驱逐策略，达到驱逐策略自动地驱逐 pod 地目的，让 pod 使用的 cpu 和内存资源不影响 kubelet、kube-proxy、kube-apiserver、kube-controller-manager、kube-scheduler 等关键进程 。

最后是尝试在 master 节点上加入 Taints PreferNoSchedule ，这个 Taints 的目的是尽量让 pod 不调度在 master 节点调度，但又不能使  pod  完全不能调度在 master 节点。

### CGroup 

systemctl 般有以下几个策略

``` shell
no（默认值）：退出后不会重启
on-success：只有正常退出时（退出状态码为0），才会重启
on-failure：非正常退出时（退出状态码非0），包括被信号终止和超时，才会重启
on-abnormal：只有被信号终止和超时，才会重启
on-abort：只有在收到没有捕捉到的信号终止时，才会重启
on-watchdog：超时退出，才会重启
always：不管是什么退出原因，总是重启
```

改变 systemctl 的重启策略 on-failure => always ，让 kubenetes 进程能够自动重启。

``` shell
sed -i 's/Restart=on-failure/Restart=always/g' /etc/systemd/system/kubelet.service
systemctl daemon-reload
service kubelet restart
sed -i 's/Restart=on-failure/Restart=always/g' /etc/systemd/system/kube-apiserver.service
systemctl daemon-reload
service kube-apiserver restart
sed -i 's/Restart=on-failure/Restart=always/g' /etc/systemd/system/kube-proxy.service
systemctl daemon-reload
service kube-proxy restart
sed -i 's/Restart=on-failure/Restart=always/g' /etc/systemd/system/kube-controller-manager.service
systemctl daemon-reload
service kube-controller-manager restart
sed -i 's/Restart=on-failure/Restart=always/g' /etc/systemd/system/kube-scheduler.service
systemctl daemon-reload
service kube-scheduler restart
```

这里分别设置了kubelet、kube-apiserver、kube-proxy、kube-controller-manager、kube-scheduler，有些 kubenetes  集群是将部分组件是在集群内部管理而不是 systemctl 管理的。

分配cpu时间资源给 kubenetes 进程。

``` shell 
sed -i '/^\[Service\]/a\CPUShares=500' /etc/systemd/system/kubelet.service
systemctl daemon-reload
service kubelet restart
cat /sys/fs/cgroup/cpu,cpuacct/system.slice/kubelet.service/cpu.share

sed -i '/^\[Service\]/a\CPUShares=300' /etc/systemd/system/kube-proxy.service
systemctl daemon-reload
service kube-proxy restart
cat /sys/fs/cgroup/cpu,cpuacct/system.slice/kube-proxy.service/cpu.shares
```

分别设置kubelet和kube-proxy进程的cpu.shares设置为500和300，并重启进程使配置生效。 

**cpu.share**

公式如下

```
per = cpu.share/sum(cpu.share)
```

##### 举个例子

一台机器上有三个进程，A的cpu.share为1000，而B、C的cpu.share都是500，当CPU饱和时，A可以使用50%CPU时间，而B、C两个进程则分别能使用25%的CPU时间。

**cpu.cfs_period_us、cpu.cfs_quota_us**

当需要对Cgroup进行硬性CPU限制时，可以使用这两个参数。它们会保证进程永远不会使用超过这个阈值的CPU时间。

cpu.cfs_period_us：cfs的含义前文解释了，period表示一段时间，而us则是时间单位µs。

cpu.cfs_quota_us：此参数可以设定在cpu.cfs_period_us时间内 cgroup 中所有任务可运行的时间总量，单位为微秒（µs）。

所以当cpu.cfs_period_us为1000000而cpu.cfs_quota_us设置为200000时，进程无论如何也不会使用超过200000/1000000=0.2个CPU。

### 为系统守护进程预留计算资源

包含有三部分，一个是系统保留值（System Reserved），一个是kubernetes 系统保留值（Kube Reserved），一个是驱逐阈值（Eviction Thresholds）。

- `system-reserved` 用于为诸如 `sshd`、`udev` 等系统守护进程争取资源预留。 
- `kube-reserved` 是为了给诸如 `kubelet`、`container runtime`、`node problem detector` 等 kubernetes 系统守护进程争取资源预留 。

- `Eviction Thresholds` 是为了在资源达到这个阈值之前提前触发驱逐 pod 的条件，memory.available默认的值是100Mi。

##### 配置 kubelet

``` shell
--kube-reserved=cpu=1,memory=1Gi,ephemeral-storage=1Gi
--system-reserved=cpu=1,memory=1Gi,ephemeral-storage=1Gi
--eviction-hard=memory.available<1Gi,nodefs.available<1Gi
--enforce-node-allocatable=pods
```

 移除信号（allocatable ） = 总共内存（NodeCapacity ） - system内存 -  kube内存 - 驱逐阈值

##### 举个例子

一台32Gi的机器

移除信号 = 32Gi -1Gi -1Gi -1Gi  = 29Gi

当实际内存达到29Gi时，开始执行驱逐策略，驱逐 pod。

而且没有通过 cgroup 限制 system 和 kube 的内存使用，这里 system 和 kube 都可以使用超过1Gi的内存。



需要注意的是，这里要十分小心地对`system-reserved`预留操作 ，因为它可能导致节点上的关键系统服务 CPU 资源短缺或因为内存不足（OOM）而被终止。

 因此在这里没有使用`--kube-reserved-cgroup=` ，不在cgroup层面对system 和  kube 添加限制。



### 驱逐策略

驱逐策略分为两种，`Soft Eviction Thresholds` 和`Hard Eviction Thresholds`

- 软驱逐：在达到移除信号后，将通过`pod.Spec.TerminationGracePeriodSeconds` 值 或者设置kubelet参数 `eviction-max-pod-grace-period `，在通过这二者中的最小值决定结束的时间优雅停止。如果都没有设定，将强制结束。

- 硬驱逐：在达到移除信号后，强制结束 pod。

  

驱逐 pod 的优先级

服务质量（QoS）等级 ：BestEffort > Burstable > Guaranteed

实际内存使用情况：使用更多的内存 > 使用更少的内存

##### QoS

BestEffort 就是没有配置 `spec.containers.resources` 的 pod。

Burstable  就是配置了 `spec.containers.resources` ，但是 limits > requests 的 pod。

Guaranteed 就是配置了 `spec.containers.resources` ，但是 limits 和 requests 相等的 pod。

简单来说，就是会先驱逐没有限制 CPU 和内存的BestEffort  pod。然后会驱逐有可能会超频使用 CPU 和内存的Burstable   pod 。最后限制多少cpu，就请求多少 CPU 的Guaranteed pod 是最不可能被优先驱逐。

驱逐策略提供了一个打分机制

| Quality of Service | oom_score_adj                                                |
| ------------------ | ------------------------------------------------------------ |
| Guaranteed       | -998                                                         |
| BestEffort       | 1000                                                         |
| Burstable        | min(max(2, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999) |

需要注意的是当 Qos 是 Guaranteed 类型时，CPU将采用 cpuset 而非 cpushare 的模式，意思是假设分配2c，程序将固定在宿主机的某2个核上，进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。

##### 举个例子

这里有3个pod ，由于内存不足，达成移除信号，开始驱逐。

pod1 是 Guaranteed 的，得分 -998

``` yaml
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 200Mi
```

pod2 是 Burstable 的，得分 996.95

machineMemoryCapacityBytes 表示linux 机器实际内存 32Gi

得分  = min(max(2, 1000 - (1000 * 100 * 1024  * 1024) / (32 * 1024  * 1024* 1024), 999)

memoryRequestBytes 越低，得分越高

``` yaml
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 100Mi
```

pod3 是 BestEffort 的，得分1000

``` yaml
    resources:
      # 这里什么也没有
```

分数越高，越容易被驱逐。

### 在 master 节点上加入Taint PreferNoSchedule 

需要尽量地不在 master 节点上部署 pod 以防止抢占 kube-proxy、kube-apiserver、kube-controller-manager、kube-scheduler 等的资源。

kubenetes 提供了 Taint 和 Toleration 来预防 一些 pod 是否能够部署在目标节点上。

有三种Taint

- `NoSchedule`：表示k8s将不会将Pod调度到具有该污点的Node上
- `PreferNoSchedule`：表示k8s将尽量避免将Pod调度到具有该污点的Node上
- `NoExecute`：表示k8s将不会将Pod调度到具有该污点的Node上，同时会将Node上已经存在的Pod驱逐出去

在 master 节点上加入Taint PreferNoSchedule

`kubectl taint nodes master_node_ip key1=value1:PreferNoSchedule`





### 参考

[kubernetes官网reserve-compute-resources](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)

[kubernetes官网out-of-resource](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)

[Kubernetes Pod调度进阶：Taints(污点)和Tolerations(容忍)](https://blog.frognew.com/2018/05/taint-and-toleration.html)
