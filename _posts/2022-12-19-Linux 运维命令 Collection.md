---
layout:     post
title:      Linux 运维命令 Collection
subtitle:   Linux 运维命令 Collection
date:       2022-12-19
author:     ica10888
catalog: true
tags:
    - linux
---


# Linux 运维命令 Collection

现代计算机基本上都是冯·洛伊曼架构，而对于基础的计算机资源，往往是木桶效应，也就是说，计算机程序的瓶颈一般是由于某一种或几种计算机资源不足而造成的性能问题，这个时候就需要通过一些 Linux命令去排除这些问题，通过增加资源或者替换服务器型号的方式，达到平衡服务器开支和程序稳定的目的。

一般而言，产生计算机瓶颈的资源一般是CPU，内存，磁盘和磁盘io，网络io。当然在一些特定程序下，GPU，文件操作符等，也会形成瓶颈。

- CPU:  
  **上下文切换**：先把前一个任务的 CPU 上下文（也就是 CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。而这些保存下来的上下文，会存储在系统内核中，并在任务重新调度执行时再次加载进来。这样就能保证任务原来的状态不受影响，让任务看起来还是连续运行。

  **特权等级**：指内核空间（Ring 0）具有最高权限，可以直接访问所有资源，包括硬件中断等。用户空间（Ring 3）只能访问受限资源。

  **孤儿进程与僵尸进程**：**僵尸进程**，这是多进程应用很容易碰到的问题。正常情况下，当一个进程创建了子进程后，它应该通过系统调用 wait() 或者 waitpid() 等待子进程结束，回收子进程的资源；而子进程在结束时，会向它的父进程发送 SIGCHLD 信号，所以，父进程还可以注册 SIGCHLD 信号的处理函数，异步回收资源。一旦父进程没有处理子进程的终止，还一直保持运行状态，那么子进程就会一直处于僵尸状态。大量的僵尸进程会用尽 PID 进程号，导致新进程不能创建，所以这种情况一定要避免。在父进程退出后，由 init 进程回收后也会消亡。**孤儿进程** todo

  **硬中断和软中断**：Linux 将中断处理过程分成了两个阶段，也就是上半部和下半部：上半部用来快速处理中断，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作。下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行。网卡接收到数据包后，会通过硬件中断的方式，通知内核有新的数据到了。对上半部来说，既然是快速处理，其实就是要把网卡的数据读到内存中，然后更新一下硬件寄存器的状态（表示数据已经读好了），最后再发送一个软中断信号，通知下半部做进一步的处理。上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行；而下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。实际上，上半部会打断 CPU 正在执行的任务，然后立即执行中断处理程序。而下半部以内核线程的方式执行，并且每个 CPU 都对应一个软中断内核线程，名字为 “ksoftirqd/CPU 编号”，比如说， 0 号 CPU 对应的软中断内核线程的名字就是 ksoftirqd/0。
  
- 内存

  **Buffer/Cache**：Buffers 是内核缓冲区用到的内存，对应的是 /proc/meminfo 中的 Buffers 值。Cache 是内核页缓存和 Slab 用到的内存，对应的是 /proc/meminfo 中的 Cached 与 SReclaimable 之和。Buffers 是对原始磁盘块的临时存储，也就是用来缓存磁盘的数据，通常不会特别大（20MB 左右）。这样，内核就可以把分散的写集中起来，统一优化磁盘的写入，比如可以把多次小的写合并成单次大的写等等。Cached 是从磁盘读取文件的页缓存，也就是用来缓存从文件读取的数据。这样，下次访问这些文件数据时，就可以直接从内存中快速获取，而不需要再次访问缓慢的磁盘。**Buffer 是对磁盘数据的缓存，而 Cache 是文件数据的缓存，它们既会用在读请求中，也会用在写请求中。**

  **直接内存**：

  **eBPF（extended Berkeley Packet Filters）机制**：

- 磁盘和磁盘io

  **虚拟文件系统 VFS（Virtual File System）**：Linux 内核在用户进程和文件系统的中间，引入了一个抽象层，这样，用户进程和内核中的其他子系统，只需要跟 VFS 提供的统一接口进行交互就可以了，而不需要再关心底层各种文件系统的实现细节。
  **零拷贝技术**：纵观 Linux 的零拷贝技术，相较于mmap、sendfile和 MSG_ZEROCOPY 等其他技术，splice 从使用成本、性能和适用范围等维度综合来看更适合在程序中作为一种通用的零拷贝方式。

- 网络io

  **TCP滑动窗口**：

这个时候，就需要我们使用一些Linux命令去排查问题

### 常用 Linux 运维命令

##### CPU

查询命令

| 命令                        | 使用方式                                                     | 常用参数                                                     |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| top 或者 uptime             | 一般查看平均负载，平均负载表示平均活跃进程数，和 cpu 个数相关 |                                                              |
| htop                        | 更直观的top！                                                |                                                              |
| atop                        | 更直观的top！                                                |                                                              |
| mpstat                      | 查看每个  cpu 使用情况                                       | -P ALL 5 监控所有CPU                                         |
| pidstat -u                  | 查看进程的 cpu 使用情况                                      | -u 5 1  间隔5秒后输出一组数据<br/>-p 指定线程号              |
| pidstat -w                  | 查看进程的上线文切换使用情况<br/> Cswch/s:每秒主动任务上下文切换数量  <br/>Nvcswch/s:每秒被动任务上下文切换数量 | -w 5 1  间隔5秒后输出一组数据<br/>-p 指定线程号              |
| vmstat                      | 查看进程的上线文切换使用情况<br/>si so :交换磁盘和内存的数据（kb/s）<br/>bi bo: 发送\接收 设备的块数<br/>in: 每秒的中断数，包括时钟中断<br/>cs:每秒的上下文切换数<br/>us sy:用户\系统进程使用的cpu时间（%）<br/>id :cpu空闲时间（%）<br/>wa:等待io所消耗的cpu时间（%） | 5  1 间隔5秒后输出1组数据                                    |
| pstree                      | 查看进程树                                                   | -p 查看进程号<br/>1005 查看1005 进程                         |
| perf                        | 记录性能事件，使用 Ctrl+ C 退出记录                          | record -F 99 -p 13204 -g -- sleep 30 -F 99表示每秒99次，-p 13204是进程号，即对哪个进程进行分析，-g表示记录调用  -a 采集所有cpus信息   <br/>report 查看报告<br/>top 实时观察<br/>stat 执行一个命令并收集其运行过程中的各个数据，提供一个程序运行情况的总体概览 |
| execsnoop                   | 监控短时进程（不断的崩溃、重启的进程）                       |                                                              |
| watch -d cat /proc/softirqs | 查看中断情况<br/>TIMER（定时中断）、NET_RX（网络接收）、SCHED（内核调度）、RCU（RCU 锁） |                                                              |
| sar -c                      | 查看CPU使用情况<br/>%user   用户空间的CPU使用 <br/>%nice   改变过优先级的进程的CPU使用率 <br/>%system   内核空间的CPU使用率 <br/>%iowait   CPU等待IO的百分比 <br/>%steal   虚拟机的虚拟机CPU使用的CPU <br/>%idle   空闲的CPU<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| sar -W                      | 查看系统swap分区统计情况<br/>pswpin/s    每秒从交换分区到系统的交换页面（swap page）数量 <br/>pswpott/s   每秒从系统交换到swap的交换页面（swap page）的数量<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| sar -q                      | 查看平均负载<br/>runq-sz    运行队列的长度（等待运行的进程数，每核的CP不能超过3个）<br/>plist-sz   进程列表中的进程和线程数的数量 <br/>ldavg-1\ldavg-5\ldavg-15  最后1分钟的CPU平均负载，即将多核CPU过去一分钟的负载相加再除以核心数得出的平均值，5分钟和15分钟以此类推<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/>-q 1 3 查看平均负载 |

模拟命令

| 命令     | 使用方式                                                     | 常用参数                                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| stress   | 模拟一个 cpu 使用 100% （usr）<br/>模拟 I/O 压力（iowait）<br/>模拟8 个进程争抢 cpu （wait） | -c 1 --timeout 600<br/>-i 1 --timeout 600<br/>-c 8 --timeout 600 |
| sysbench | 以10个线程运行5分钟的基准测试，模拟多线程切换                | --threads=10 --max-time=300 threads run                      |


``` bash
uptime
load average: 0.63, 0.83, 0.88 # 依次则是过去 1 分钟、5 分钟、15 分钟的平均负载

top - 23:15:11 up 8 days,  1:42,  8 users,  load average: 0.03, 0.04, 0.09
                  # 系统运行时间    #当前登录用户数  # 平均负载：
                                                # 如果平均值为 0.0，意味着系统处于空闲状态
                                                # 如果 1min 平均值高于 5min 或 15min 平均值，则负载正在增加
                                                # 如果 1min 平均值低于 5min 或 15min 平均值，则负载正在减少
                                                # 如果它们高于系统 CPU 的数量，那么系统很可能会遇到性能问题（视情况而定）
Tasks: 109 total,   1 running, 108 sleeping,   0 stopped,   0 zombie
       #进程总数      #正在运行   #睡眠             #停止        #僵尸
%Cpu(s):  0.3 us,  0.5 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
          #用户空间  #内核空间  #改变优先级 #空闲   #等待输入输出 #硬中断 #软中断  #偷取
KiB Mem :   951924 total,   118024 free,   223148 used,   610752 buff/cache
            #物理内存                           
KiB Swap:        0 total,        0 free,        0 used.   574748 avail Mem
            #交换区内存 
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1298 root      10 -10  136364  15104  11520 S   2.0  1.6 252:42.15 AliYunDun
#进程id  #用户 #优先级 #nice值                 #进程状态  
                    #负值表示高优先级          #R：进程在 CPU 的就绪队列中，正在运行或者正在等待运行
                    #正值表示低优先级          #D：进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断
                                            #Z：僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源
                                            #S：可中断状态睡眠，进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入R状态
                                            #I：Idle，空闲状态，对某些内核线程来说，它们有可能实际上并没有任何负载
                                            #进程的所有状态，以下不一定会见到
                                            #T：向一个进程发送SIGSTOP信号，它就会因响应这个信号变成暂停状态S；再向它发送 SIGCONT 信号，进程又会恢复运行
                                            #X：Dead的缩写，表示进程已经消亡
 1568 root      20   0  813000  12352   5384 S   0.3  1.3  13:08.60 aliyun-service
                        #VIRT=SWAP+RES 注意这里是SWAP
                        #VIRT：进程虚拟内存的大小，只要是进程申请过的内存，即便还没有真正分配物理内存，也会计算在内。
                        #RES：常驻内存的大小，也就是进程实际使用的物理内存大小，但不包括 Swap 和共享内存。
                        #SHR：共享内存的大小，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等。
    1 root      20   0  191008   3956   2620 S   0.0  0.4   0:05.41 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd
    4 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    5 root      20   0       0      0      0 S   0.0  0.0   0:03.35 kworker/u4:0
    6 root      20   0       0      0      0 S   0.0  0.0   0:01.98 ksoftirqd/0
    7 root      rt   0       0      0      0 S   0.0  0.0   0:00.01 migration/0



```



##### 内存

查询命令

| 命令       | 使用方式                                                     | 常用参数                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| free       | 查看内存                                                     |                                                              |
| pidstat -r | 查看进程的内存 使用情况 <br/> Minflt/s:任务每秒发生的次要错误，不需要从磁盘中加载页<br/> Majflt/s:任务每秒发生的主要错误，需要从磁盘中加载页 <br/>VSZ：虚拟地址大小，虚拟内存的使用KB <br/>RSS：常驻集合大小，非交换区五里内存使用KB | -r 5 1  间隔5秒后输出一组数据<br/>-p 指定线程号              |
| sar -r     | 查看内存使用情况 <br/>kbmemfree   空闲的物理内存大小 <br/>kbmemused   使用中的物理内存大小 %<br/>memused  物理内存使用率 <br/>kbbuffers  内核中作为缓冲区使用的物理内存大小，<br/>kbbuffers和kbcached:这两个值就是free命令中的buffer和cache. <br/>kbcached  缓存的文件大小 <br/>kbcommit   保证当前系统正常运行所需要的最小内存，即为了确保内存不溢出而需要的最少内存（物理内存+Swap分区） <br/>commit   这个值是kbcommit与内存总量（物理内存+swap分区）的一个百分比的值<br/> kbactive <br/>kbinact <br/>kbdirty<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| cachestat  | 查看缓存的命中率<br/>TOTAL ，表示总的 I/O 次数；<br/>MISSES ，表示缓存未命中的次数；<br/>HITS ，表示缓存命中的次数；<br/>DIRTIES， 表示新增到缓存中的脏页数；<br/>BUFFERS_MB 表示 Buffers 的大小，以 MB 为单位；<br/>CACHED_MB 表示 Cache 的大小，以 MB 为单位。 | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>  |
| cachetop   | 查看进程缓存的命中率<br/>MISSES ，表示缓存未命中的次数；<br/>HITS ，表示缓存命中的次数；<br/>DIRTIES， 表示新增到缓存中的脏页数；<br/>READ_HIT，读的缓存命中率。<br/>WRITE_HIT，写的缓存命中率。<br/> |                                                              |
| vmstat     | 查看内存使用情况<br/>swpd 使用的虚拟内存总量<br/>free   空闲的物理内存总量<br/>buff 用作缓冲区的内存总量 <br/>cache 用作高速缓存的内存总量<br/> | 5  1 间隔5秒后输出1组数据                                    |
| pmap       | 分析进程内存分布                                             | -d 1234 展示进程内存 - 设备信息<br/>-XX 1234 展示进程内存 - 所有信息 <br/> |
| pcstat     | 指定文件的缓存大小                                           |                                                              |
| memleak    | 内存泄漏定位                                                 |                                                              |
| valgrind   | 进程内存错误检查，内存初始化，泄漏，越界访问等各种内存错误   |                                                              |

模拟命令

| 命令                              | 使用方式           | 常用参数                                   |
| --------------------------------- | ------------------ | ------------------------------------------ |
| echo 3 > /proc/sys/vm/drop_caches | 清理缓存           |                                            |
| dd                                | 运行dd命令读取文件 | if=/dev/sda1 of=/dev/null bs=1M count=1024 |


``` bash
free -h
              total        used        free      shared  buff/cache   available
Mem:           929M        144M        139M        980K        645M        637M
Swap:            0B          0B          0B
```


##### 磁盘和磁盘io

查询命令

| 命令       | 使用方式                                                     | 常用参数                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| du     | 检查磁盘大小 | -sh ./* 查看当前目录各文件大小 |
| df     | 检查文件系统的磁盘空间占用情况 | -h 查询硬盘空间<br/>-ia 列出各文件系统的i节点使用情况<br/> -T 列出文件系统的类型<br/> |
| iostat     | 输出磁盘IO 和 CPU的统计信息<br/>device: 磁盘名称<br/>tps:每秒钟发送到的I/O请求数.<br/>Blk_read/s: 每秒读取的block数.<br/>Blk_wrtn/s: 每秒写入的block数.<br/>Blk_read: 读入的block总数.<br/>Blk_wrtn: 写入的block总数. | -d -x 5  1 表示显示所有磁盘I/O的指标<br/>-m 以 M 为单位显示<br/>-k 以 KB 为单位显示<br/> |
| pidstat -d | 查看进程的 io 使用情况<br/>kB_rd/s：每秒从磁盘读取的KB <br/>kB_wr/s：每秒写入磁盘KB <br/>kB_ccwr/s：任务取消的写入磁盘的KB。当任务截断脏的pagecache的时候会发生。 | 5 1  间隔5秒后输出一组数据<br/>-p 指定线程号                 |
| dstat | 更直观的iostat ！ |                                                              |
| sar -b     | 查看IO和传递速率<br/>tps  磁盘每秒钟的IO总数，等于iostat中的tps <br/>rtps  每秒钟从磁盘读取的IO总数 <br/>wtps  每秒钟从写入到磁盘的IO总数 <br/>bread/s  每秒钟从磁盘读取的块总数 <br/>bwrtn/s  每秒钟此写入到磁盘的块总数 <br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| sar -d     | 查看磁盘使用情况 <br/>tps  每秒I/O的传输总数  <br/>rd_sec/s  每秒读取的扇区的总数  <br/>wr_sec/s  每秒写入的扇区的总数  <br/>avgrq-sz  平均每次次磁盘I/O操作的数据大小（扇区）  <br/>avgqu-sz  磁盘请求队列的平均长度 await  从请求磁盘操作到系统完成处理，每次请求的平均消耗时间，包括请求队列等待时间，单位是毫秒（1秒等于1000毫秒），等于寻道时间+队列时间+服务时间  <br/>svctm  I/O的服务处理时间，即不包括请求队列中的时间  <br/>%util  I/O请求占用的CPU百分比，值越高，说明I/O越慢 <br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| iotop      | 类似 top 的工具，用来显示实时进程的磁盘活动。 |                                                              |
| filetop | 查看系统内核对文件读写情况（linux bbc） | -C 输出新内容时不清空屏幕 |
| opensnoop | 查看系统调用打开的所有文件（linux bbc） |                                                              |
| biotop              | top工具（linux bbc） |                                                              |
| blkstrace           | 块设备IO事件追踪                                             |                                                              |
| strace              | 进程IO事件追踪 | |
| slabtop | 目录项，索引文件，文件系统的缓存 | |
| cat /proc/slabinfo | slab是一个用于高效的内存分配对象的内存管理机制。与早些时候机制相比,它可以减少碎片造成的分配和回收。 | |
| cat /proc/meminfo | 内存信息 | |
| cat /proc/diskstats | 磁盘信息 | |
| cat /proc/${pid}/io | 进程io信息 | |
| tune2fs             | 显示和设置文件系统参数 | |
| hdparm              | 显示和设置磁盘参数 | |

模拟命令

| 命令 | 使用方式                            | 常用参数                                                     |
| ---- | ----------------------------------- | ------------------------------------------------------------ |
| fio  | 文件系统和磁盘 I/O 性能基准测试工具 | 随机读fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb<br/>随机写fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb<br/>顺序读fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb<br/>顺序写fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb |

``` bash
df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        455M     0  455M   0% /dev
tmpfs           465M     0  465M   0% /dev/shm
tmpfs           465M  596K  465M   1% /run
tmpfs           465M     0  465M   0% /sys/fs/cgroup
/dev/vda1        20G  2.6G   17G  14% /
tmpfs            93M     0   93M   0% /run/user/0
```

##### 网络io

查询命令

| 命令        | 使用方式                                                     | 常用参数                                                     |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ss          | 获取 socket 统计信息                                         | -anp 显示所有监听端口的进程<br/>-t 显示 TCP 协议的 sockets<br/>-u 显示 UDP 协议的 sockets<br/> |
| netstat     | 监控TCP/IP网络                                               | -atnlp 使用ip地址列出所有处理监听状态的TCP端口，且加上程序名<br/>-t 显示 TCP 协议端口<br/>-u 显示 UDP 协议端口<br/> |
| ifstat      | 统计网络接口活动状态                                         |                                                              |
| tcpdump     | 抓包工具                                                     | src 172.21.0.1 and tcp port 8080  -n -s 0 -A  打印来自 ip 172.21.0.1和端口 8080 的包并显示详细信息<br/>-nn -vv -X udp port 11000 udp抓包<br/>-i docker0 host 172.21.0.1 -nn 网卡和 dst/src 的数据<br/>-a 　　　将网络地址和广播地址转变成名字；<br/>-d 　　　将匹配信息包的代码以人们能够理解的汇编格式给出；<br/>-dd 　　　将匹配信息包的代码以c语言程序段的格式给出；<br/>-ddd 　　　将匹配信息包的代码以十进制的形式给出；<br/>-e 　　　在输出行打印出数据链路层的头部信息，包括源mac和目的mac，以及网络层的协议；<br/>-f 　　　将外部的Internet地址以数字的形式打印出来；<br/>-l 　　　使标准输出变为缓冲行形式；<br/>-n 　　　指定将每个监听到数据包中的域名转换成IP地址后显示，不把网络地址转换成名字；<br/>-nn：    指定将每个监听到的数据包中的域名转换成IP、端口从应用名称转换成端口号后显示<br/>-t 　　　在输出的每一行不打印时间戳；<br/>-v 　　　输出一个稍微详细的信息，例如在ip包中可以包括ttl和服务类型的信息；<br/>-vv 　　　输出详细的报文信息；<br/>-c 　　　在收到指定的包的数目后，tcpdump就会停止；<br/>-F 　　　从指定的文件中读取表达式,忽略其它的表达式；<br/>-i 　　　指定监听的网络接口；<br/>-p：    将网卡设置为非混杂模式，不能与host或broadcast一起使用<br/>-r 　　　从指定的文件中读取包(这些包一般通过-w选项产生)；<br/>-w 　　　直接将包写入文件中，并不分析和打印出来；<br/> -s          snaplen表示从一个包中截取的字节数。0表示包不截断，抓完整的数据包。默认的话 tcpdump 只显示部分数据包,默认68字节。<br/>-T 　　　将监听到的包直接解释为指定的类型的报文<br/>-X            告诉tcpdump命令，需要把协议头和包内容都原原本本的显示出来<br/> |
| sar -n DEV  | 查看系统的网络收发情况<br/>IFACE 表示网卡<br/>rxpck/s  每秒钟接受的数据包 <br/>txpck/s  每秒钟发送的数据库 <br/>rxKB/S  每秒钟接受的数据包大小，单位为KB <br/>txKB/S  每秒钟发送的数据包大小，单位为KB <br/>rxcmp/s  每秒钟接受的压缩数据包 <br/>txcmp/s  每秒钟发送的压缩包 <br/>rxmcst/s  每秒钟接收的多播数据包<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| sar -n EDVE | 网络设备通信失败信息<br/>rxerr/s  每秒钟接收到的损坏的数据包 <br/>txerr/s  每秒钟发送的数据包错误数 <br/>coll/s  当发送数据包时候，每秒钟发生的冲撞（collisions）数，这个是在半双工模式下才有 <br/>rxdrop/s  当由于缓冲区满的时候，网卡设备接收端每秒钟丢掉的网络包的数目 <br/>txdrop/s 当由于缓冲区满的时候，网络设备发送端每秒钟丢掉的网络包的数目 <br/>txcarr/s  当发送数据包的时候，每秒钟载波错误发生的次数 <br/>rxfram/s   在接收数据包的时候，每秒钟发生的帧对其错误的次数 <br/>rxfifo/s    在接收数据包的时候，每秒钟缓冲区溢出的错误发生的次数 <br/>txfifo/s    在发生数据包 的时候，每秒钟缓冲区溢出的错误发生的次数<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| sar -n SOCK | 统计socket连接信息<br/>totsck  当前被使用的socket总数 <br/>tcpsck  当前正在被使用的TCP的socket总数 <br/>udpsck   当前正在被使用的UDP的socket总数 <br/>rawsck  当前正在被使用于RAW的skcket总数 <br/>if-frag   当前的IP分片的数目 <br/>tcp-tw  TCP套接字中处于TIME-WAIT状态的连接数量 | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| sar -n TCP  | TCP连接的统计 <br/>active/s  新的主动连接<br/>passive/s  新的被动连接 <br/>iseg/s  接受的段 <br/>oseg/s  输出的段<br/> | 1 10 显示网络收发的报告，间隔1秒输出一组数据，刷新10次<br/>-o test 1 3  保存<br/>-f test    查看<br/> |
| ethtool     |                                                              |                                                              |
| mtr         |                                                              |                                                              |
| tcping      |                                                              |                                                              |
| nsenter     |                                                              | --target $PID --net -- lsof -i                               |


模拟命令

| 命令    | 使用方式                                       | 常用参数                              |
| ------- | ---------------------------------------------- | ------------------------------------- |
| ab      | 并发100个请求测试Nginx性能，总共测试1000个请求 | -c 100 -n 1000 https://www.baidu.com/ |
| python  | 端口8080测试                                   | -m SimpleHTTPServer 8080              |
| hping3  | 构建tcpip包工具                                |                                       |
| netperf | 时延网络测试工具                               |                                       |
| iperf3  | 重传包网络测试工具                             |                                       |


``` bash
netstat -atnlp
```

##### 其他-程序分析

查询命令

| 命令   | 使用方式      | 常用参数                                       |
| ------ | ------------- | ---------------------------------------------- |
| strace | 分析函数调用  | -p 进程号<br/>-f -p 9085 -T -tt -e fdatasync   |
| lsof   |               | -p 进程号<br/>-i:8080 查看8080端口的程序进程号 |
| dmesg  | 显示linux信息 |                                                |



​	

