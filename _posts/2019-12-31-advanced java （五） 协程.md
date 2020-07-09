---
layout:     post
title:      advanced java （五） 协程
subtitle:   advanced java （五） 协程
date:       2019-12-31
author:     ica10888
cover:      'https://raw.githubusercontent.com/ica10888/banner/master/558b21ab2e4dba013a23d0e692c8650d805a839e.jpg@1320w_938h.webp'
catalog: true
tags:
    - java
    - functional programming
    - golang
---

#  advanced java （五） 协程

什么是协程？

一方面来说，协程有很多种有很多种说法，有的叫 `纤程` 的（微软提出的，主要是windows平台），又有的说协程是 **CPS**（Continuation-passing style）变换的实现。对于前者，无论是协程还是纤程，实质上是同一种概念，一般称作Fiber。后者是受`lambda 演算` 影响而提出的概念，lisp系语言的实现，一般称作**Coroutine** 。Coroutine 为Fiber 的实现提供了理论支持。

两者的区别在于Coroutine 是没有 **调度器**（scheduler），而在Fiber中调度器是一个重要的组件。

协程还有一种说法，协程是 **callback 的语法糖** ，以同步程序的形式写异步代码。在没有调度器的情况下，确实，从理论上来说，CPS 变换的目的 ，就是将     callback 消除。

协程相比于线程，区别在于 **协程的主动退让机制和线程的抢占式形式** 。当有大量线程抢占的时候，由于大量活跃线程的抢占，内核的上下文切换往往会造成cpu资源紧张，消耗资源。而协程是被动调度的，当执行完后，会退让出去，会按照队列顺序执行接下来其他协程的 **continuation** （后面会提到什么是continuation）。

java有一个很出名的协程库 **quasar**  ，提供高并发平台轻量级的用户态线程（ high-performance lightweight threads） 。要注意的是，此线程非彼线程。 **用户态线程** 只是在逻辑上顺序执行代码块，并非是独立的 **内核态线程** 。

本文首先会从理论上来说明协程的机制，然后再说明在现代语言库的实现方式。

### Continuation

 一般来讲，Continuation（延续） 表示的是 **剩余的计算** 的概念，换句话说就是 **接下来要执行的代码** 。

举个例子来说

``` scheme
(display
    ((lambda (obj)
        (display (string-append "吃饭 ， 使用 " obj)) (newline)
        "吃饱了")
            ((lambda(obj)
                (display (string-append "做菜 ， 使用 " obj)) (newline)
                "食物") 
                    ((lambda(obj)
                        (display (string-append "买菜 ， 使用 " obj)) (newline)
                        "食材")
"菜篮子"))))
;;; 买菜 ， 使用 菜篮子
;;; 做菜 ， 使用 食材
;;; 吃饭 ， 使用 食物
;;; 吃饱了
```
理解一下

step1:

``` scheme
((lambda(obj)
    (display (string-append "买菜 ， 使用 " obj))(newline)
    "食材")
"菜篮子")
```

买菜后，的延续就是

``` scheme
(lambda(x) (display
    ((lambda (obj)
        (display (string-append "吃饭 ， 使用 " obj)) (newline)
        "吃饱了")
            ((lambda(obj)
                (display (string-append "做菜 ， 使用 " obj)) (newline)
               "食物") x )))) ;这里是一个callback，入参是"食材"，是上次计算完成的结果
```

接下来的计算就是 做菜，吃饭

step2：

``` scheme
((lambda(obj)
    (display (string-append "做菜 ， 使用 " obj)) (newline)
    "食物")
"食材")
```

做菜后，的延续就是

``` scheme
(lambda(x) (display
    ((lambda (obj)
      (display (string-append "吃饭 ， 使用 " obj)) (newline)
       "吃饱了") x ))) ;这里是一个callback，入参是"食物"，是上次计算完成的结果
```

接下来的计算就是 吃饭的延续

step3：

``` scheme
(lambda (x)(display "吃饱了"))
```

同理，使用java代码来理解一下

``` java
Function<String,String> buy = o -> {
  System.out.println("买菜 ， 使用 "+ o);
  o = "食材";
  return  o;
};
Function<String,String> cook = o -> {
  System.out.println("做菜 ， 使用 "+ o);
  o = "食物";
  return  o;
};
Function<String,String> eat = o -> {
  System.out.println("吃饭 ， 使用 "+ o);
  o = "吃饱了";
  return  o;
};

String s  =  eat.apply(cook.apply(buy.apply("菜篮子")));
System.out.println(s);

//买菜 ， 使用 菜篮子
//做菜 ， 使用 食材
//吃饭 ， 使用 食物
//吃饱了
```

### cps变换

 CPS是将控制流显式表示为continuation的一种编程风格， 简单来理解就是显式使用函数表示函数返回的后续操作。

在这里，我们将 continuation 作为参数传递进来，即 `f` ，可以理解成 continuation 是一个函数，调用后会执行接下来的运算。方法中调用自己的 continuation 来完成计算。

``` java
String s  = doEat(doCook(doBuy("菜篮子",buy),cook),eat);
System.out.println(s);
//买菜 ， 使用 菜篮子
//做菜 ， 使用 食材
//吃饭 ， 使用 食物
//吃饱了
```

``` java
public static String doBuy (String o ,Function<String,String> f){
    return  f.apply(o);
}
public static String doCook (String o ,Function<String,String> f){
    return  f.apply(o);
}
public static String doEat (String o ,Function<String,String> f){
    return  f.apply(o);
}
```

从上面的代码可以看出，模拟一个人买菜，做菜，吃饭顺序执行的情景 。

##### 回调地狱

这种代码在流程多了之后，就很有可能形成  **回调地狱** 或者 **回调金字塔** ，我们需要的是符合人类直觉的代码，这种风格的代码需要封装。 

实际上这种情况是不可避免的，因为上一次计算返回的结果要作为下次计算的参数，参数需要写在方法后面，这样就造成了一种结果，写在后面的代码会先执行。我们理解的顺序是  `doBuy -> doCook -> doEat ` ，事实上的代码是 `doEat(doCook(doBuy("菜篮子",buy),cook),eat)`

##### 保存 continuation

可以在执行完上一步后，将此步的 continuation 保存下来，假设我们有能力在此步中断，然后在上次中断处重新开始执行 ，就可以去做其他事情。

如执行完 `doBuy("菜篮子",buy)` 之后，可以将状态保存下来，也就是 `食材`  和它的延续（保存上下文），下次执行 `doCook("食材",cook)`就可以了。

比如买好菜后，可以先看一会电视，将衣服放入洗衣机等，然后再去做菜。这样就异步地执行了多个任务。

**continuation 的代码是不是和callback很像呢？**

因为 continuation 的特性很重要，有的语言（scheme/racket）将其作为一等公民（ first-class ）。

### call/cc

使用call/cc，让我们的假设成为现实

 `call/cc` 的全名是  `call-with-current-continuation` ，它可以捕捉当前环境下的 current continuation 并利用它做各种各样的事情，如改变控制流等。

假设这里有一个计算过程  

``` scheme
(* (+ 4 5) 7)
```

``` scheme
(define (f current-continuation)
  (current-continuation (+ 4 5)))

(display (f (lambda (x) (* x 7)))) ; displays 63
;;;  (lambda (x) (* x 7)) 相当于传入一个 t -> t * 7 匿名函数 ,这个匿名函数相当于 continuation，也相当于 callback
;;; 最后一步的返回值是 4 + 5 的计算结果 9 ，和接下来的计算 * 7  ，于是就直接打印 63


(display (call-with-current-continuation f)) ; displays 9
;;; callcc 接受的是一个入参是continuation，或者说 callback ，一个出参的函数
;;; callcc 打印的是 (return (+ 4 5)) 的返回值，即执行回调，或者说continuation，之前的地方，返回的结果也就是 9 
;;; 回想下计算过程，先计算 4 + 5 ，然后是 t ->  t * 7 (continuation) ，也就是说 callcc 将正常的顺序调换过来了，先得到了 9 
;;; callcc 是解释器里面实现 
;;; current-continuation 在scheme里不是关键字或者方法，这里只是传入参数的命名
```

callcc 本质是改变控制流，除了可以实现协程( coroutine ) ，还可以实现非本地退出( non-local exit )、多任务( multi-tasking )等。

我们用之前的例子来说明

``` scheme
(let ([context "undefined"]) ;保存上下文

(set! context "菜篮子")

(set! context
       (call-with-current-continuation (lambda (
                                                ;传入 continuation ，x -> "做菜,食材" -> "吃饭,食物" -> "吃饱了" 的一个callback，入参是 "菜篮子" 
                                                ;事实上并没有执行，这也是为什么说 callcc 本质是改变控制流，在 continuation 之前返回了
                                                (lambda(x) (display
                                                    ((lambda (obj)
                                                        (display (string-append "吃饭 ， 使用 " obj)) (newline)
                                                        "吃饱了")
                                                            ((lambda(obj)
                                                                (display (string-append "做菜 ， 使用 " obj)) (newline)
                                                               "食物") x ))))
                                                 ) 
        ((lambda (obj)
            (display (string-append "买菜 ， 使用 " obj)) (newline)
             "食材")  context ))))
         
(display "看一会电视，将衣服放入洗衣机") (newline)    
         
(set! context
       (call-with-current-continuation (lambda (
                                                ;传入 continuation ，x -> "吃饭,食物" -> "吃饱了" 的一个callback，入参是 "食材"
                                                ;也就是将上一次的 continuation 保存状态，然后接下来执行，直到遇到下一个 continuation
                                                (lambda(x) (display
                                                    ((lambda (obj)
                                                      (display (string-append "吃饭 ， 使用 " obj)) (newline)
                                                       "吃饱了") x )))   
                                                )
        ((lambda (obj)
            (display (string-append "做菜 ， 使用 " obj)) (newline)
             "食物")  context ))))
(set! context
                                                ;这里是 (lambda (x)(display "吃饱了")) ，也就是最后一个 continuation
                                                ;事实上，这里只需要是个回调函数就可以了，因为callcc 在接下来的延续（current-continuation）之前返回
       (call-with-current-continuation (lambda (whatever-current-continuation) 
        ((lambda (obj)
            (display (string-append "吃饭 ， 使用 " obj)) (newline)
             "吃饱了")  context ))))



(display context)
)

;;; 买菜 ， 使用 菜篮子
;;; 看一会电视，将衣服放入洗衣机
;;; 做菜 ， 使用 食材
;;; 吃饭 ， 使用 食物
;;; 吃饱了

```

这时可以发现callcc 具有两个能力

- 一个是消除了回调地狱，以符合人类习惯的顺序执行。
- 一个是具有暂停连续回调函数的能力，这样我们就可以先保存上下文，然后在中间插入其他执行，如看电视，将衣服放入洗衣机。

##### 阴阳谜题

``` scheme
(let* ((yin
         ((lambda (cc) (display #\@) cc) (call-with-current-continuation (lambda (c) c))))
       (yang
         ((lambda (cc) (display #\*) cc) (call-with-current-continuation (lambda (c) c)))))
    (yin yang))
```
执行结果是

``` scheme
@*@**@***@****@*****@******@*******@********  ;;; infinite
```

这里 yin 和 yang 交替进行，而 yang 由于 `上下文环境`不同，yang 每次会较上一次多一层嵌套，打印 `*` 依次增加

call/cc 具有改变控制流的能力，即可以跳到任意执行回调的地方 （就像是 goto ，或者指令集 jmp ）。这样，多个回调就可以拆开，我们将回调作为中断处，有了从中断处重新开始执行的能力。

**是不是有一种，两个线程交替执行的感觉呢？**

### 循环结构和尾递归优化

在现代高级语言中，常常会使用一种数据结构叫循环语句，这便是java中的关键字 for while 和 do。

如果将一段代码做cps变换，就是可以将一个任务分割成多个任务顺序执行。当遇到循环语句的时候，同样可以做到。

``` java
for (int i = 1;i <= 1000000;i++){
    System.out.println("第" + i + "次执行");
}
```

没错，使用**递归**就可以了。

``` java
public static void main(String[] args) {

    Function<Integer,Integer> func = (Integer num) -> {
        System.out.println("第" + num + "次执行");
        num ++;
        return num;
    };
    say(1,func);

}

public static Integer say(Integer num, Function<Integer,Integer> func) {
    if (num <= 1000000){
        num = func.apply(num);
        say(num,func);
    }
    return num;
}
```

上面这段代码会抛出 StackOverflowError 错误。

这是因为在 jvm 运行时的数据区域中有一个 java 虚拟机栈，当执行java方法时会进行压栈弹栈的操作。jvm 规定了栈的最大深度，当执行时栈的深度大于了规定的深度，就会抛出上面的错误。

如果使用了具有 **尾递归优化** 的语言，就不会存在上面的问题了。

但是，这里需要做一个小小的变换。尾递归，其实就是将递归方法中的需要的  **所有状态**  通过方法参数传入下一次调用中。

``` java
public static Integer say(Integer num, Function<Integer,Integer> func) {
    if (num <= 1000000){
        num = func.apply(num);
        say(num,func);
    }else {
      return num;
    }
}
```

换成 scheme，一个具有尾递归的语言

``` scheme
(define (say num func)
    (if (<= num 1000000)
        (say (func num) func)
         num ; 这里隐式存在 else
))

(say 1 (lambda(x) 
   (display (string-append "第" (number->string x) "次执行"))(newline)
   (+ x 1)
))
```

### 从原理到实现

为什么在协程之前会讲解Continuation 、CPS变换、递归等 lambda 演算相关的知识呢？

**协程框架是通过CPS概念实现的。从原理上，cps变换允许我们将每个协程都分割成多个顺序任务，每个协程的子任务交替执行，就达到了异步的效果。**

（ lambda 演算是等价于图灵机的，这种计算机程序和数学证明之间的紧密联系 ，被称作Curry-Howard同构。准确地说应该是 lambda-mu 演算 ，因为 callcc 是 Peirce’s law ，这又是另外个故事了... ）

### quasar 协程库

在java语言中有一种特殊的控制流语句，即异常处理

``` java
public  static void main(String[] args) {
    try {
        throwException();
    } catch (Exception e) {
        e.printStackTrace();
    }finally {
        System.out.println("finally");
    }

}
public  static void throwException () throws Exception{
    throw new Exception("some Exception");
}
```

以及标号(label)

``` java
OUT:
for(  ;  ;  ){
    for(  ;  ;  ){
        break OUT;
    }
}
```

两者其本质上也是一种控制流语句，异常和标号控制流的处理是 java 协程库实现必须要注意的问题。

##### goto关键字

在java中 goto 是保留关键字， 但在语言中并没有用到它 。字节码也没有具体实现。

这是因为 goto 会破坏工程的约束和实践，危害结构化设计。

quasar 是使用 java agent，以字节码织入的形式将 goto 加入到字节码中。

goto模拟的是控制流，将代码块分割成多段的形式，实现CPS变换。

以以下代码为例

``` java
static void doAll1() throws SuspendExecution, InterruptedException{
    for(int i=0;i<2;i++){
        doAll2();
        System.out.println("add");
    }
}

static void doAll2() throws SuspendExecution, InterruptedException{

    for(int i=0;i<2;i++){
        System.out.println("dss");
    }
}


static public void main(String[] args) throws SuspendExecution,ExecutionException, InterruptedException {
    new Fiber<Void>(new SuspendableRunnable() {
        @Override
        public void run() throws SuspendExecution, InterruptedException {
            for(int i=0;i<2;i++){
                doAll1();
                System.out.println("add");
            }
        }
    }).start();
}
```

goto 织入到字节码

``` java
static void doAll1()
        throws SuspendExecution, InterruptedException
    {
        Object obj = null;
        Stack stack;
        if((stack = Stack.getStack()) == null) goto _L2; else goto _L1
_L1:
        boolean flag = true;
        stack.nextMethodEntry();
        JVM INSTR tableswitch 1 1: default 36
    //                   1 72;
           goto _L3 _L4
_L3:
        if(!stack.isFirstInStackOrPushed())
            stack = null;
_L2:
        int i;
        boolean flag1 = false;
        i = 0;
_L9:
        if(i >= 2) goto _L6; else goto _L5
_L5:
        if(stack == null) goto _L8; else goto _L7
_L7:
        stack.pushMethod(1, 1);
        Stack.push(i, stack, 0);
        boolean flag2 = false;
_L4:
        i = stack.getInt(0);
_L8:
        doAll2();
        System.out.println("add");
        i++;
          goto _L9
_L6:
        if(stack != null)
            stack.popMethod();
        return;
        if(stack != null)
            stack.popMethod();
        throw ;
    }
 
    static void doAll2()
        throws SuspendExecution, InterruptedException
    {
        for(int i = 0; i < 2; i++)
            System.out.println("dss");
 
    }
 
    public static void main(String args[])
        throws SuspendExecution, ExecutionException, InterruptedException
    {
        (new Fiber(new SuspendableRunnable() {
 
            public void run()
                throws SuspendExecution, InterruptedException
            {
                Object obj = null;
                Stack stack;
                if((stack = Stack.getStack()) == null) goto _L2; else goto _L1
_L1:
                boolean flag = true;
                stack.nextMethodEntry();
                JVM INSTR tableswitch 1 1: default 36
            //                           1 72;
                   goto _L3 _L4
_L3:
                if(!stack.isFirstInStackOrPushed())
                    stack = null;
_L2:
                int i;
                boolean flag1 = false;
                i = 0;
_L9:
                if(i >= 2) goto _L6; else goto _L5
_L5:
                if(stack == null) goto _L8; else goto _L7
_L7:
                stack.pushMethod(1, 1);
                Stack.push(i, stack, 0);
                boolean flag2 = false;
_L4:
                i = stack.getInt(0);
_L8:
                QuasarIncreasingEchoApp.doAll1();
                System.out.println("add");
                i++;
                  goto _L9
_L6:
                if(stack != null)
                    stack.popMethod();
                return;
                if(stack != null)
                    stack.popMethod();
                throw ;
            }
 
        }
)).start();
```

##### 主动退让

在执行完单个协程的一段代码之后，需要主动退让出来，让后面队列的其他协程的子任务执行，quasar 采取的方式是抛出异常。

抛出 `SuspendExecution` 异常。

同时 Fiber 在父类方法上 定义了 `@Suspendable` 注解， 在`suspendable方法链内`Fiber的父类会调用`Fiber.park`  ，抛出`SuspendExecution`异常，从而来停止`主线程`的运行，好让Quasar的调度器执行调度。  Fiber自己捕获  `SuspendExecution`异常后，会执行调度器接下来的任务。

##### forkjoinpool

之前我们提到 Fiber 和 Continuation 的区别在于有没有调度器，调度器的作用是在多核环境中可以并发执行，充分利用cpu资源。 就绪队列实现协程的上下文环境，即保存接下来要执行的代码。

Quasar的调度器是用的 线程池 forkjoinpool。这是在java 1.7中加入的新线程池。Quasar 有自己  forkjoinpool的实现。forkjoinpool线程数的默认值是当前机器的核心数。每个线程占用一个cpu核心，并有自己的任务队列。 当一个线程处理完了队列中的任务之后，它会试图窃取其他线程队列的任务（work stealing）。如果有大量的协程执行，实际上就是将协程自己的子任务加入到核心个数的主线程各自的任务队列，由其执行。

##### blocking io 

在使用协程的时候可能遇到线程阻塞的代码。

``` java
while (true){
    byte data[] = new byte[1024];
    if (inputStream.read(data) != -1) {
        System.out.println(data);
    }
}
```

如果有多个如上代码的协程执行，并同时阻塞（inputStream 一直没有数据输入），那么确实会阻塞掉主线程的执行。虽然有工作窃取算法，但当核心个数的主线程都被阻塞的时候，可以想象，是forkjoinpool 所有的线程都被阻塞了，就无法执行新的协程了。

当有线程被阻塞的时候，Quasar会发出警告。

`warning hogging the cpu or blocking a thread.`

Quasar无法处理这种情况，使用Quasar库的时候要避免这种情况发生。如果存在这种情况，或许Quasar不是正确的选择。

### goroutine

 Golang 语言最大的特色就是从语言层面使用协程支持并发（准确地说是并行执行） 。其中 goroutine 的实现是使用的PMG模型。大多数的协程都是相似的。

![PMG模型]( https://pic3.zhimg.com/80/67f09d490f69eec14c1824d939938e14_hd.jpg )

- M：M是对内核级线程的封装，数量即 runtime.GOMAXPROCS， 数量对应真实的CPU数 。
- G:  代表一个goroutine ,具有自己的栈。
- P: 代表调度的上下文，可以把它看做一个局部的调度器。

 M:N  表示多个goroutine在多个内核线程上跑， 而P维护着一个队列，执行调度。

##### 阻塞处理

当然 goroutine 做了更多的事情， 当一个 OS 线程 M0 陷入阻塞时 ，会去运行队列中其他的 `G` 。也就是说，当 goroutine 遇到 io 阻塞或者调用 `runtime.Gosched()` 方法的时候，会主动从队列中退让出来。

比如 epoll 中，对于没有 ready 的非阻塞 fd 执行网络操作时, linux 内核不阻塞线程, 会直接返回 `EAGAIN`。这个时候将协程状态设置为 wait ，放入 `M` 的 `local runqueue` 的待运行中，运行其他可运行的 `G` 。待运行的 `G` 直到当有数据可读的时候唤醒他，继续进行调度 。

其中 epoll 事件管理线程即 golang 虚拟机的 `sysmon` 线程，也就是说，可以以同步的形式写异步代码，在使用协程做网络开发的时候，开发者即使写的是同步阻塞代码，也在不知不觉中使用了 epoll 模型。

而文件 io 阻塞的时候，由于 non-blocking 适用于网络 io 但是对硬盘 io 不起作用，需要做一些特殊处理来避免阻塞队列。

##### 内存优化

使用span机制减少内存碎片。

 Go 对于 GC 后回收的内存页, 并不是马上归还给操作系统, 而是会延迟归还, 用于满足未来的内存需求。

#####  work stealing

同样，goroutine 里面也使用了 work stealing 算法。首先会从`global runqueue`中获取`G`，然后也会去窃取其他`P` 的`local runqueue`，系统也会定期检查 `global runqueue` ，防止饿死。

##### 有栈协程

初始化的时候需要分配好栈，同时可以复用。

相比于使用对象指针的调度方式，栈寄存器的调用栈要浅一些。

相比于java线程 `1M`的内存空间，为了支持成千上万的 goroutine 同时运行，会分配一段`8K`字节的内存用于栈供goroutine运行使用。而不够用时，使用连续栈（continuous stacks）扩展栈空间。（相比于之前的方案，分段栈(segmented stacks)，创建一个两倍于原stack大小的新stack，并将旧栈拷贝到其中）,同时能栈缩小（小于 1/4 触发）。

相比于 Quasar 无栈协程 （stackless）的实现方式，有栈协程更复杂也更有效率。由于程序计数器，上下文切换周期更少，方法栈也可以存在协程的栈中。

go 在之前的版本不支持抢占式调度，在密集CPU时，可能会调度延迟。在 go 1.14 版本稳定了异步抢占式的调度模型。（erlang...）

### JDK 其他实现
阿里云（Alibaba JDK）和华为（Huawei JDK ）的在jvm上都有协程的实现。而 Project Loom 是由前 quasar  项目的主要设计开发人员 Ron Pressler 主导的，已经加入Oracle，可以期待官方的实现。
