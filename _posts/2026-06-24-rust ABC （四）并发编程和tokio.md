---
layout:     post
title:      rust ABC （四）并发编程和tokio
subtitle:   rust ABC （四）并发编程和tokio
date:       2026-06-24
author:     ica10888
catalog: true
tags:
    - rust
---

# rust ABC （四）并发编程和tokio

rust作为一个功能强大的语言，支持多种编程范式，同时支持抢夺式进程和协程，同时对于共享变量提供了线程锁，引用计数，管道的同步和通讯机制，具备强大的功能。同时可以使用`tokio`，一个三方库，更强大的并发运行时，替换 `std` 库中的并发。由于基本上 `tokio` 成了rust程序的默认并发运行时，这里主要讲 `tokio` ，不过这里的 `tokio`  和  `std`  的函数签名基本是相同的，比如 `spawn` 和 `task` 等，因此无须有多余的担心。最后会讲 async 和 await 关键字，用于实现异步任务

### tokio 的 runtime 和 task

runtime 也就是 抢夺式进程 ，task 也就是协程。

其中 runtime 创建方式如下

``` rust
use tokio;

fn main() {
  // 创建带有线程池的runtime
  let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(8)  // 8个工作线程
    .enable_io()        // 可在runtime中使用异步IO
    .enable_time()      // 可在runtime中使用异步计时器(timer)
    .build()            // 创建runtime
    .unwrap();

    rt.block_on(async {
        println!("before sleep: {}", Local::now().format("%F %T.%3f"));
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;
        println!("after sleep: {}", Local::now().format("%F %T.%3f"));
    });

}

```

也可以直接使用 runtime

``` rust
fn main() {
    tokio::spawn(async {
        println!("before sleep: {}", Local::now().format("%F %T.%3f"));
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;
        println!("after sleep: {}", Local::now().format("%F %T.%3f"));
    });

}	
```

task 又被称作Asynchronous green-threads(异步的绿色线程)，创建方式如下

``` rust
use chrono::Local;
use std::thread;
use tokio::{self, task, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();
    let _guard = rt.enter();
    task::spawn(async {
        time::sleep(time::Duration::from_secs(3)).await;
        println!("task over: {}", now());
    });

    thread::sleep(time::Duration::from_secs(4));
}
```

或者直接使用 task

``` rust
fn main() {
    tokio::task::spawn(async {
        println!("before sleep: {}", Local::now().format("%F %T.%3f"));
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;
        println!("after sleep: {}", Local::now().format("%F %T.%3f"));
    });

}	
```

### 共享变量

并发程序中如何处理并发问题，一般来说有两种思路。一种是java中同步机制，使用锁来控制并发。另一种是go的协程机制，通过通讯来共享内存，而不是共享内存来通讯。

##### 同步

`tokio::sync`模块提供了几种状态同步的机制：

- Mutex: 互斥锁
- RwLock: 读写锁
- Notify: 通知唤醒机制
- Barrier: 屏障
- Semaphore: 信号量

然后就是使用互斥锁，一般签名如下

``` rust
let mutex = Arc::new(Mutex::new());
let rwlock = Arc::new(RwLock::new());
```

这里就不说明使用方式了，一般有编程基础的都会使用，这里只需要注意两点

- 如果使用的`tokio` 运行时，需要使用`tokio::sync`模块中的锁。官方文档中建议，如非必须，应使用标准库的Mutex或性能更高的`parking_lot`提供的互斥锁，而不是 tokio 的Mutex。
- 需要使用线程安全的引用计数 `Arc` ，是为了处理Rust语言设计中内存模型问题

然后关于栅栏类，提供了以下方式

- Notify提供了一种简单的通知唤醒功能，它类似于只有一个信号灯的信号量。
- Barrier是一种让多个并发任务在某种程度上保持进度同步的手段。
- Semaphore信号量可以保证在某一时刻最多运行指定数量的并发任务。

举例如下

``` rust
    let barrier = Arc::new(Barrier::new(3));


    for t in 0..3 {
        let c = barrier.clone();
        tokio::task::spawn(async move {
            for i in 1..100 {
                tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
                println!("开始执行第{}个", i);
            }
            c.wait().await;
            println!("after wait");
        });
    }

    println!("主线程完成");
```

##### 通讯

也就是go中的管道（channel），go中的管道有无界管道和有界管道两种类型，同样Rust也是支持无界管道和有界管道。在此基础之上，go中常用的就是多对一，一种类型，rust中有一对一、一对多、多对一、多对多四种类型，但是使用最多的是多对一这一类型。

- oneshot：单Sender、单Receiver以及单消息，简单来说就是一次性的通道。
- mpsc：有多个发送者发送多个消息，且只有一个接收者（使用最多的类型）。 其中`mpsc::channel()` 创建有界通道，`mpsc::unbounded_channel()` 创建无界通道。
- watch：只能有单个Sender，可以有多个Receiver，且通道永远只保存一个数据。Sender每次向通道中发送数据时，都会修改通道中的那个数据。
- broadcast：一种广播通道，可以有多个Sender端以及多个Receiver端，可以发送多个数据，且任何一个Sender发送的每一个数据都能被所有的Receiver端看到。

一个异步的例子

``` rust
use tokio::{ self, runtime::Runtime, sync };

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        let (tx, mut rx) = sync::mpsc::channel::<i32>(10);

        for i in 1..=10 {
            let tx = tx.clone();
            tokio::spawn(async move {
                if tx.send(i).await.is_err() {
                    println!("receiver closed");
                }
            });
        }
        drop(tx);

        while let Some(i) = rx.recv().await {
            println!("received: {}", i);
        }
    });
}
```

### async 和 await 关键字

简单来说如果写过一些前端，那么就能够理解这两个关键字，这两个关键字就是协程的一种实现方式，将主任务拆分成多个子任务，并且通过CPS 的变换，消除了callback。

简单来说Rust就是以下几个特性

- 在语句中使用了await的函数，需要使用async修饰函数。
- 具有传染性，调用async修饰的函数，也需要async修饰。
- await是关键字，使用是在语句的末尾，这种使用方式在编程语言中不是很常见，是无奈中最好的一种选择
- 用于异步io场景，在进行io的时候，能够切换到其他协程，让io不阻塞

一个文件io的场景如下

``` rust
use tokio::{self, fs::File, io::{self, AsyncReadExt}, runtime};

fn main() {
    let rt = runtime::Runtime::new().unwrap();
    rt.block_on(async {
        let f1 = File::open("a.log").await.unwrap();
        let f2 = File::open("b.log").await.unwrap();
        let mut f = f1.chain(f2);

        let mut data = String::new();
        let n = f.read_to_string(&mut data).await.unwrap();
        println!("data {} bytes: {:?}", n, data);
    });
}
```



