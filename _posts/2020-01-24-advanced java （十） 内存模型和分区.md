---
layout:     post
title:      advanced java （十） 内存模型和分区
subtitle:   advanced java （十） 内存模型和分区
date:       2020-01-24
author:     ica10888
catalog: true
tags:
    - java
---

#  advanced java （十） 内存模型和分区

java 为了发挥缓存，多核的性能，提出了 java 内存模型（ Java memory model，JMM ）这一概念。每个线程都有自己的  **工作内存**  ，同时也存储所有变量的 **主内存** 。工作内存中存储的是主内存中存储的变量（ 这里指的是引用，也就是指向 jvm 堆中 ）的一份拷贝。

而在 工作内存和主内存的变量，通过一些原子操作来达到同步效果。

这里就涉及到访问这些变量的问题，需要满足 **原子性** 、**可见性** 和 **有序性** 。可以使用 `volatile` 修饰符来保证变量在各个线程中都是一致的，也就是保证读操作是正确，但是不保证写操作。 `volatile`  关键字同时也禁止了指令重排序优化，保证两个线程之间的先行发生 （happens-before ）原则。

JVM 规范中定义了五个分区：程序计数器，栈，本地栈，堆和方法区。在这里还加入了方法区中的运行时常量池和不属于 JVM 管理的直接内存。在启动java工程的时候，可以通过一些 java指令设置分区大小和比例。

### java 内存模型

java内存模型中包含工作内存和主内存。线程对于变量的操作都必须在工作内存中进行，而不会直接读取主内存的变量。同样不同线程的工作内存之间是相互独立的，不能直接读取另一个线程的工作内存，变量的传递都需要在主内存完成。

主内存和工作内存之间有8种原子操作

- **lock**：作用于主内存，将变量标识为一条线程独占的状态
- **unlock**：作用于主内存，解锁被锁定的变量，才能会被其他线程读取。
- **read**：作用于主内存，把一个变量的值从主内存传输到线程的工作线程中，以便随后的 load 使用
- **load**：作用于工作内存，把 read 操作从主内存中得到的变量值放入工作内存的变量副本中
- **use**：作用于工作内存的变量，在遇到取值字节码指令时，把工作内存中的变量值传递给执行引擎
- **assign**：作用于工作内存的变量，在遇到赋值字节码指令时，把从执行引擎得到的值赋给工作内存
- **store**：作用于工作内存的变量，把工作内存中的一个变量值传送到主内存中，以便随后的 write 操作使用
- **write**：作用于主内存的变量，把 store 操作从工作内存中得到的变量值放入主内存

其中 read 和 load 、store 和 write 、lock 和 unlock 操作是成对出现的。数据从工作内存同步回主内存时，需要先 assign 操作。同一时刻只允许一条线程对其进行 lock 操作。在进行 lock 操作时，将会清空工作内存中此变量的值，在执行引擎使用这个变量前需要重新执行 load 或 assign 操作初始化变量的值。在进行 unlock 操作之前，会先执行 执行 store 和 write 操作将变量同步到主内存。

### volatile 关键字

被  `volatile` 修饰的共享变量，具有以下两点特性

- 保证了不同线程对该变量操作的内存可见性
- 禁止指令重排序

也就是说，当一个线程改变了变量时，其他线程能够立即得知。保证了变量在各个线程之间是一致的。但是由于写操作不是原子性的，因此不是线程安全的。

java 内存模型的三个特征

- **原子性**：基本上可以认为 java 内存模型对变量的操作是原子的，通过一些原子操作指令来实现

- **可见性**：当一个线程修改了共享变量的值，其他线程能立刻得知这个修改，`volatile ` 对普通变量提供了可靠的可见性

- **有序性**： Java 在线程内是完全有序的，而在跨线程的观察中是无序的。 这是由于指令重排和工作内存决定的。而解决方案是 ，使用 `synchronized`  关键字或加锁

同时 `volatile ` 关键字禁止了指令重排，虽然无法保证写操作的原子性。相比于全局计计数器这种有写操作的变量，加锁是正确的选择。而如果是全局开关这样的变量，就应当选择 `volatile` 。

##### 重排序

指令重排序分为三种类型

- **编译器优化的重排序** ：编译器中，只要保证不改变单线程程序语义，就可以改变语句的执行顺序
- **指令级并行的重排序** ：计算机处理器中，采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，就可以改变机器指令的执行顺序
- **内存系统的重排序** ：由于处理器使用缓存和读写缓冲区，这使得加载和存储变量的操作可能是乱序执行

 **内存屏障**：一个实现是通过 lock 指令强制将前面操作完成的修改从 CPU 的 cache 中写入到内存。 

lock 指令 将本 CPU 的 Cache 写入内存，同时引起其他 CPU 的 Cache 中的相同位置**无效化**。 所有的CPU如果需要再次使用这个变量必须从 主内存 中重新获得。相当于在每次获取这个变量之前，都需要 read 操作和 load 操作。 

看得出来，`volatile` 修饰的变量，写操作基本一样，而读操作有一些额外的操作。不过 JVM 对锁有优化和消除。`synchronized` 不一定比 `volatile`  慢。

##### Happens-before 关系

 如果线程 A 与线程 B 满足 happens-before 关系，则线程 A 执行动作的结果对于线程 B 是可见的。也就是说，如果满足 happens-before 原则，那么就是线程安全的。

-  程序次序规则 ： 按照程序控制流顺序， 在一个线程内，前面的代码先行发生于后面的代码 
-  监视器锁规则 ：按照程序控制流顺序解锁，之后才能加锁
-   volatile 变量规则： 对于这个变量的写操作，之后对这个变量的读操作
-  传递性 ： A事件在B事件之后，B事件在C事件之后，那么一定A事件在C事件之后
-  start 规则 ：如果通过线程A启动线程B，那么线程A的启动先于线程B
-  join规则：  如果线程A执行线程B的join方法，那么线程B先于线程A完成
-  程序中断规则：  对线程 `interrupt()` 方法的调用先行发生于线程内部的代码检测到中断
-  对象终结规则 ： 一个对象的初始化完成先于其 `finalize()` 方法

如果满足上面的规则，那么就不需要加锁。 使用 `synchronized` 或 `volatile` ，分别对应了第二和第三条规则。 


##### 基于双检锁的单例模式

``` java
public class DoubleCheckSingleton {
    private static DoubleCheckSingleton instance;
    //私有的构造方法
    private DoubleCheckSingleton() {}
    public synchronized static DoubleCheckSingleton getErrorInstance(){
        if (instance==null){
            instance = new DoubleCheckSingleton();
        }
        return instance;
    }
}
```

由于新建实例 `instance = new DoubleCheckSingleton();`  这条语句不是原子性操作，因此如果单例模式需要在多线程环境下执行，则需要在新建实例的方法上加锁。但是这里效率高，因为每次访问这个方法都需要同步操作。

``` java
public class DoubleCheckSingleton {
    private volatile static DoubleCheckSingleton instance;
    //私有的构造方法
    private DoubleCheckSingleton() {}
    public static DoubleCheckSingleton getInstance(){
        if(instance == null){ //第一层检查
            synchronized (DoubleCheckSingleton.class){
                if(instance == null){ //第二层检查
                    instance = new DoubleCheckSingleton();
                }
            }
        }
        return instance;
    }
}
```

这里通过前一次检查，如果创建好了实例，就不需要经过同步代码了。需要注意的是，由于 java 内存模型，当一个线程对实例初始化时，其他线程是不可知的。因此需要 `volatile` 来修饰该变量。

### 内存分区

线程隔离的分别是程序计数器、虚拟机栈和本地方法栈，线程共享的是堆和方法区。

程序计数器（Program Counter） 计数器中存放的是 **虚拟机字节码的地址** ，在挂起的线程恢复时需要通过程序计数器恢复到运行时的地方（native 方法除外）。这也是jvm规范中唯一没有规定 OOM 的内存区域。

虚拟机栈 （VM Stack）保存的是 **局部变量表（ 八种基本类型的数据和对象的引用 ），操作数栈，动态链接，方法出口** 等信息 ，当使用栈深度大于所允许的深度，会抛出 ` StackOverflowError `  的异常。而动态扩展的虚拟机栈，如果无法申请到足够内存 ，会抛出 `OutOfMemoryError` 的异常。其每层栈的生命周期为方法从调用到返回，整个栈的生命周期等同线程的生命周期。

本地方法栈 （Native Method Stack）也就是  `native` 关键字修饰的方法。 HotSpot 中本地方法栈和虚拟机栈是在一起的。 

堆 （Heap） 大多数对象实例和数组都分配在堆上，如在栈上对象引用指向的对象。 在堆内部，可以根据 GC 算法分为 **新生代（划分为 Eden 空间，From Survivor 空间和 To Survivor 空间）和老生代** 。当然Eden空间也有可能划分出为个 Thread Local Allocation Buffer（TLAB ）。

方法区（Method Area） 保存的是已被 **加载的类信息、常量、静态变量、 JIT 编译后的代码** 等数据，也被称作非堆。在 Hotspot 早先的版本， 和字符串常量池（Interned Strings）一起被称为 **永久代** （ PermGen ）。 而后来 HotSpot 逐渐去掉了永久代。

##### HotSpot 去永久代

 在 java7 之后，移除了永久代中的常量池，同时将 符号引用移动到了  native heap ， 常量和静态变量移动到了  java heap  。在 java8 中，移除了永久区，使用本地内存来存储类元数据信息并称之为 **元空间 **（ metaspace ） 。  元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。 因此元空间的大小仅受本地内存限制。

这样的改变有以下优点

- 如果永久代发生了内存不足，则会触发 Full GC 以回收常量和加载的类，而使用 metaspace 可以提高 Full GC的性能。
- 不需要关注设置永久代空间的大小，metaspace  位于 Native Heap 中，几乎没有限制 。

以字符串常量池举例，来说明 java7 去除了常量池带来的影响

``` java
String s1 = new String("Misaka") + new String("Mikoto");
s1.intern(); 
String s2 = "MisakaMikoto";
System.out.println(s1 == s2);
```

在 java7 之前，会返回 `false`，在 java7 及其之后的版本，会返回 `true` 。 这是因为常量池已经被从永久代移到了一般的堆中，这意味着常量池不再存储一个对象。而是直接在常量池中存储一个引用，这个引用指向同一个对象。也就是说，常量池中存储的是一个指向 Java Heap 中某个对象的引用。 

##### 内存参数

在分配内存的时候，如果通过 CAS 操作来解决线程冲突的问题是可以的，当然也可以使用 TLAB 为每个线程分配一个，然后使用 TLAB 进行线程间的同步。通过 java 参数 `-XX:-UseTLAB` 来设置。 

如果有大量线程，而栈的深度不深（一般而言就是没有使用递归方法），可以将每个线程堆栈大小设置小一些，默认是 1M，可以设置成 `-Xss256k` 。

同样，可以通过 `-Xms<size>` 设置初始 Java 堆大小，`-Xmx<size>` 设置最大 Java 堆大小，需要注意的是需要保留内存一些内存给非堆空间。假如在 linux 里面如果还剩下 1g 内存，就不能将堆设置成 1g，不然 java 程序会内存溢出而退出。

元空间是没有限制的，可以通过 `-XX:MetaspaceSize=<size>` 设置元空间初始大小， `-XX:MaxMetaspaceSize=<size>` 设置元空间最大值。

可以通过 `-XX:SurvivorRatio=n` 设置 Eden 区与两个 Survivor 区的比值 ，默认是 8 : 1+1。

而在 docker 容器中，可以通过 `-XX:MaxRAMPercentage=75.0`  ，这是在 java8u191 加入的参数，针对的是能够感知到 `cgroup` 对内存的限制。相当于限制了堆内存的占比是 75% ，需要留一部分给非堆内存。同理还有参数 `InitialRAMPercentage` 和 `MinRAMPercentage` 。

要注意的是使用 Unsafe 方法或使用 NIO 的 `DirectByteBuffer`，有可能会导致直接内存的 OOM 。
