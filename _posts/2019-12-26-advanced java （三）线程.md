---
layout:     post
title:      advanced java （三）线程
subtitle:   advanced java （三）线程
date:       2019-12-26
author:     ica10888
catalog: true
tags:
    - java
---


# advanced java （三）线程

在java语言当中，有一个非常重要的概念便是线程。

线程是 java语言并发模型的基石。

多线程执行任务或许是不错的注意，如Tomcat中，默认保持200个线程处理连接，但是遇到IO密集任务的时候，线程或许不那么尽如人意，在200个连接都被占满后，将无法处理后来的连接。但是如果是CPU密集型任务，如复杂计算的时候，线程切换的开销就不那么明显，可以使用多线程并行执行计算任务。

一般来说，线程有五种状态。

- 初始化状态：新建一个线程对象

- 就绪状态：该状态的线程位于可运行线程池中，变得可运行，等待获取CPU的使用权

- 运行状态：开始运行

- 阻塞状态：线程因为某种原因放弃CPU使用权，暂时停止运行。

  - 等待： `wait()`,等待`notify()`、`notifyAll()`

  - 同步：进入锁池队列，等待获取到锁

  - 其他：`sleep()`、`join()`，或触发了I/O请求

    其中如果线程在 wait 等待状态或者 sleep 休眠状态被中断，（主动中断就是调用`interrupt()`）就会直接抛出 `InterruptedException` 异常。

- 死亡状态：结束生命周期

除初始化和死亡状态，其他状态都是可以相互转换的。

在 java 语言中，常使用线程的三种对象，Runnable(Thread) ，Callable ，Future（以及 CompletableFuture）。

而线程的创建和销毁会有很大的开销，为了解决这个问题，使用线程池对线程管理和监督，执行调度。分别会介绍以下几种线程池

-  `ThreadPoolExecutor`，传统线程池，具有多种实现，而子类`ScheduledThreadPoolExecutor`可以实现定时调度。
- `ForkJoinPool` java7加入的新线程池，工作窃取（work-stealing）算法的实现。同样 `Collection::parallelStream` 也是通过该种线程池实现的并行流。

ThreadLocal类可以创建线程局部变量的类，让每个线程只能访问自己的变量。以弱引用的形式保存线程局部变量。

### 线程对象

#### Runnable(Thread)

在 java8 之后，可以以lambda函数的形式创建线程，而替代了以前的内部类。

``` java
Thread thread =  new Thread(() -> System.out.println("I am thread"));
thread.start();
```

这是因为 `Runnable` 接口被`@FunctionalInterface` 修饰。

`thread.setDaemon(true)` 用来设置守护进程。

#### Callable

Callable 相比 Runnable 多了一个返回值，而 `Callable` 接口同样被`@FunctionalInterface` 修饰。

``` java
ExecutorService pool = Executors.newCachedThreadPool();
Future<String> future = pool.submit(() -> "I am Callable");
```

这里 Callable 和 Supplier 是同义的。

Callable 和 Runnable 的区别

- 一个是call()，一个是run()

- Callable的任务执行后可返回值，而Runnable的任务不返回值。

- call方法可以抛出异常，run方法不可以。
- 运行Callable任务可以拿到一个Future对象，Future 表示异步计算的结果。

#### Future

Future用来表示异步执行，并返回一个结果。`Future<V>` 是一个接口，一般使用实现类 `FutureTask<V>`

```java
FutureTask<String> task = new FutureTask<>(()->"I am FutureTask");
task.run();
```

`cancel(true)`用来取消任务，`isDone()` 和 `get()`获取返回值。

##### CompletableFuture

CompletableFuture 是Future的父类，相比Future通过阻塞或者轮询的方式获得结果，或许是更好的选择。

``` java
public T 	get(long timeout, TimeUnit unit) 
public T 	getNow(T valueIfAbsent) //回计算的结果或抛出异常，否则返回valueIfAbsent
public T 	join() //返回计算的结果或者抛出一个 CompletionException
```

初始化

``` java
CompletableFuture<String> task = CompletableFuture.supplyAsync(()->"I am FutureTask");
CompletableFuture<String> task2 = CompletableFuture.supplyAsync(()->"I am FutureTask",Executors.newSingleThreadExecutor()); //如果不指定 Executor ，将使用ForkJoinPool::commonPool
CompletableFuture<Void> task3 = CompletableFuture.runAsync(()->{}); //无入参和出参，即Runnable，返回 Void ，这是为了能够继续执行计算
```

回调执行

```java
task.thenApply(s -> s + " then Apply"); //I am FutureTask then Apply
task.thenApplyAsync(s -> s + " then Apply"); //以不同的线程执行，但是可能会选到同一个线程
task.thenApplyAsync(s -> s + " then Accept",Executors.newSingleThreadExecutor()); //指定 Executor
```

后两种方法 `Async`所有执行方法都有，后面不再叙述

``` java
task.thenRun(() -> {}); 
task.thenAccept(s -> {});
```

组合

``` java
task.thenCompose(s -> CompletableFuture.supplyAsync(()-> s + " compose me")); //I am FutureTask compose me
task.thenCombine(CompletableFuture.supplyAsync(()->"I am new FutureTask"),(a,b)-> a +" combine " + b); //I am FutureTask combine I am new FutureTask
```

``` java
CompletableFuture::thenAcceptBoth //前两个完成才会执行下一步
CompletableFuture::acceptEither //任意一个完成，就执行下一步
CompletableFuture::applyToEither //和acceptEither区别是有返回值
```

异常处理

``` java
CompletableFuture<String> exceptionTask = CompletableFuture.supplyAsync(()-> {throw new RuntimeException();});


exceptionTask.thenRun( ()-> {}); //抛出RuntimeException
exceptionTask.handle((s,t)-> s); //返回null，异常被catch了
exceptionTask.exceptionally(t -> "exceptionally"); //exceptionally
exceptionTask.whenComplete((s,t)-> {...}); //传递异常，没做处理,会抛出RuntimeException
```

（ CompletableFuture 是 functor，在自函子范畴，有半幺群，是monad）

CompletableFuture 可以帮助我们写出高效、异步、非阻塞的并行代码。

而 Promise 是更高级的封装。通过success方法可以成功去实现一个带有值的future。相反的，因为一个失败的Promise通过failure方法就会实现一个带有异常的future。可以去看看 Scala Promise 的实现。

### 线程池

#### ThreadPoolExecutor

实现一个ThreadPoolExecutor 需要的参数

| 参数                                | 作用                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| `int corePoolSize`                  | 核心线程数，真正可以同时运行的线程数量                       |
| `int maximumPoolSize`               | 最大线程数，当提交数大于 `corePoolSize + workQueue.size()`，会新建线程来执行。但是如果提交数大于`maximumPoolSize`,执行拒绝策略 |
| `long keepAliveTime`                | 工作线程空闲XX秒后被自动回收                                 |
| `BlockingQueue<Runnable> workQueue` | 分为有界队列和无界队列。`LinkedBlockingQueue` 在构造参数设置大小就有界，使用默认大小`Integer.MAX_VALUE`就无界了，或者使用`ArrayBlockingQueue`。`SynchronousQueue`内部容量是0，非公平模式。 |

使用不同的参数就可以初始化了，分别是

| new                     | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| newFixedThreadPool      | 指定大小的线程池                                             |
| newSingleThreadExecutor | 只有一个工作线程的线程池                                     |
| newCachedThreadPool     | 无容量线程池，核心线程数为0，工作线程空闲60秒后会被自动回收，启动线程数量无限制 |

拒绝策略

| 策略                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| AbortPolicy，默认策略 | 提交数超过 ` workQueue.size() + maximumPoolSize` ， 就会抛出异常 `RejectedExecutionException` 。 |
| CallerRunsPolicy      | 直接在 execute 方法的调用线程中运行被拒绝的任务              |
| DiscardPolicy         | 直接丢弃任务                                                 |
| DiscardOldestPolicy   | 丢弃队列中等待时间最长的任务，并执行当前提交的任务           |

如果线程池已经关闭，任务将被丢弃

##### 方法解析

```java
ThreadPoolExecutor::execute //提交一个任务到线程池
ThreadPoolExecutor::addWorker //(私有) 尝试添加一个新的工作线程,检查界限，然后对workercount做出相应调整
//Worker 是内部私有类
Worker::runWorker //循环从队列中获取任务并执行。在任务执行前后可调用beforeExecute和afterExecute处理执行前后的逻辑
Worker::tryTerminate //尝试终止线程池，如果线程池可被终止 SHUTDOWN -> TERMINATED（SHUTDOWN会运行队列任务，STOP会尝试中断）
```

##### ctl变量

使用低29位表示线程池中的线程数（workerCount），高3位表示线程池的运行状态(runState)

状态有RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED

Future转化成RunnableFuture，再执行

``` java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
  return new FutureTask<T>(runnable, value);
}
```

##### ScheduledThreadPoolExecutor 

重写了`execute(Runnable)`和`submit(Runnable)`方法来实现，可以周期执行任务

| new                              | 说明                     |
| -------------------------------- | ------------------------ |
| newScheduledThreadPool           | 可指定核心线程数的线程池 |
| newSingleThreadScheduledExecutor | 只有一个工作线程的线程池 |

#### ForkJoinPool

ForkJoinPool 并行的实现了分治算法（Divide-and-Conquer），而工作过程中使用了工作窃取（work-stealing）算法来让每个工作线程的都能工作。

- 分治算法：把任务递归的拆分为各个子任务，这样可以更好的利用系统资源，尽可能的使用所有可用的计算能力来提升应用性能。

- 工作窃取算法：线程池内的所有工作线程都尝试找到并执行已经提交的任务，或者是被其他活动任务创建的子任务

##### 伪共享状态

`@sun.misc.Contended`注解了线程池和工作队列，在CPU缓存（L1，L2，L3）以缓存行（cache line）为单位存储的，一般是64个字节大小，一般对象头是8字节变量，将将剩下的部分缓存行填充（Padding），这样多核频繁修改就不会影响性能，是优化多核执行的一种方式。

##### 构造函数

``` java
public ForkJoinPool(int parallelism, //并行度，默认为CPU数，最小为1
                    ForkJoinWorkerThreadFactory factory, //工作线程工厂
                    UncaughtExceptionHandler handler, //处理工作线程运行任务时的异常情况类，默认为null
                    boolean asyncMode) { //false:默认 true:子任务的执行遵循 FIFO 顺序并且任务不能被合并
    this(checkParallelism(parallelism),
         checkFactory(factory),
         handler,
         asyncMode ? FIFO_QUEUE : LIFO_QUEUE, //先进先出：后进先出
         "ForkJoinPool-" + nextPoolId() + "-worker-");
    checkPermission();
}
```

`ForkJoinPool::commonPool`  `parallelism`尝试获取cpu核心数 ，`asyncMode`使用`LIFO_QUEUE`，支持任务合并。

#####  workQueues

三种形式的出列操作：push、pop、poll(也叫steal)，push和pop只能被队列内部持有的线程调用，poll可被其他线程偷取任务时调用，使用CAS解决冲突问题。

直接提交的外部任务（submissions task），存放在 workQueues 的偶数槽位；通过内部 fork 分割的子任务（Worker task），存放在 workQueues 的奇数槽位。通过这种方式，简化GC。

`externalPush` 和`externalSubmit`提交外部任务，Submit会先初始化。

`ForkJoinPool.WorkQueue.push() `提交子任务。

##### 执行任务

`ForkJoinPool::scan` ：扫描并尝试偷取一个任务，使用`w.hint`进行随机索引（XorShifts），偷过来的Task 和队列里的都有可能被选中执行。

`WorkQueue::runTask()` ：标记`scanState`为正在执行状态，更新`currentSteal`为当前获取到的任务并执行它

`ForkJoinPool::deregisterWorker`：线程运行完毕之后终止线程或处理工作线程异常

##### Fork/Join

先Fork任务成子任务，后Join子任务

`ForkJoinPool::helpStealer`： 如果队列为空或任务执行失败，说明任务可能被偷，调用此方法来帮助偷取者执行任务。偷取者帮助我执行任务，我去帮助偷取者执行它的任务。

 `ForkJoinPool::tryCompensate`：补偿运行，如果没有足够的活动线程，tryCompensate()可能创建或重新激活一个备用的线程来为被阻塞的 joiner 补偿运行（currentJoin 记录引用）。

#### ThreadLocal

`ThreadLocal<T>`  是一个Map。线程将自身作为 key ，值作为 value 存入 `Thread` 对象中的 `ThreadLocalMap` 中。这样，就能够保证使用这个类包装的变量仅存在于当前线程。各个线程可以可以通过`set()`和`get()`来对操作各自的值。

`ThreadLocalMap`是在`ThreadLocal`中使用内部类，ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获取 value，value 放在了当前线程的一个ThreadLocalMap实例中，被线程持有。

##### 构造，避免hash碰撞

每个对象有一个 `final int threadLocalHashCode` 。这个变量的值由静态构造，来自一个 `AtomicInteger` 累加上一个魔法值。这个魔法值以尽量避免碰撞为依据。这个变量最终会被用在 `ThreadLocalMap` 的 hash 过程中。

##### Entry对ThreadLocal弱引用

假设一种情况，ThreadLocal存在内存泄漏。即在于如果使用线程池，线程不会被销毁，值被线程引用。

值被持有 -> ThreadLocalMap实例被持有 -> ThreadLocal实例被持有

那么原来的ThreadLocal将无法GC，内存泄漏。

实际上可以使用`弱引用` 解决这个问题。

``` java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

##### java的四种引用方式

| 引用方式                    | 方法                                                         | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 强引用(Strong Reference)    | `Object obj = new Object()`                                  | 如果没有移除 `obj`的引用，那么GC将无法回收                   |
| 软引用（Soft Reference）    | `SoftReference<Object> weakObj = new SoftReference<>(new Object());` | 软引用可以到达，那么这个对象会停留在内存更时间上长一些。当内存不足时的GC才会回收这些软引用可到达的对象,比弱引用好一些。 |
| 弱引用(Weak Reference)      | `WeakReference<Object> weakObj = new WeakReference<>(new Object())` | 弱引用不能阻挡垃圾回收器对其回收，使用get时有可能突然返回null |
| 虚引用（Phantom Reference） |                                                              | 虚引用指向的对象十分脆弱，我们不可以通过get方法来得到其指向的对象。它的唯一作用就是当其指向的对象被回收之后，自己被加入到引用队列，用作记录该引用指向的对象已被销毁。 |

##### 引用队列

软引用、弱引用和虚引用都可以传入一个`ReferenceQueue`对象，当该引用指向的对象被标记为垃圾的时候（这个时候已经不能获取到这个对象了），这个引用对象会自动地加入到引用队列里面。然后可以通过这个队列来处理其引用对象。










