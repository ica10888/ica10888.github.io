---
layout:     post
title:      advanced java （六） lamdba表达式
subtitle:   advanced java （六） lamdba表达式
date:       2020-01-02
author:     ica10888
catalog: true
tags:
    - java
---



# advanced java （六） lamdba表达式

lamdba表达式是java8带来的新特性，也是函数式思想的一种体现。

一方面来说，java8 将lamdba表达式 当作了一等公民（frist class），引入了这种代码风格后，大大简化了代码，提高了可读性。而另外一方面，提高了代码质量，如之前的 for循环，如果不经注意，容易写出更多时间复杂度和空间复杂度的实现，如果使用 lamdba ，如 stream API，可以强制地使用 `递归`和 `短路` 的思想，优化代码质量。

首先，会讲一讲 JDK里面的 Function 包，以及如何在 jvm里面实现的。使用匿名函数替代匿名内部类。以及lamdba表达式里面的变量会将外界变量隐式加上 `final`，一种`闭包`的体现。而不可变对象（immutable）可以带来一些优势。

然后是 `副作用`，即函数式思维的一种体现。而为什么必须要在lamdba表达式中捕获异常，甚至是不能有return语句。以及高阶函数的使用，以数学的方式表达如何使用默认方法 `andThen`和 `compose`。

接下来是 stream API 的使用。 （btw，其实我不喜欢`stream`这个命名，感觉像是IO处理相关的，或许可以像scala那样改成 `lazylist`） 如 构建`无限流` ，`mapreduce` ，即映射（map）和归约（reduce）思想。以及`call by name` 、`call by vaule` 和 `call by need`的区别。

最后进阶地讲一下什么是柯里化和反柯里化，以及箭头函数和右结合。



### 一等公民 lambda

常用 Function类 ，这里只列出引用类型，基本类型也有对应的类

| 类                | 表现形式                                                     |
| ----------------- | ------------------------------------------------------------ |
| Runnable          | 没有入参，也没有出参，即 `() -> {}`                          |
| Callable/Supplier | 没有入参，一个出参，即`() -> 1`                              |
| Consumer          | 一个入参，没有出参，即`(a) -> {}`                            |
| Function          | 一个入参，一个出参，即`(a) -> 1`                             |
| BiFunction        | 两个入参，一个出参，即`(a,b) -> 1`                           |
| Predicate         | 一个入参，返回值是一个 bool 类型，即`(a) -> true`，这里有点像英语里的be动词 |

以上类都被`@FunctionalInterface` 修饰，方法里面也许有默认方法和静态方法，但是普通方法 **有且仅有一个** ，而这个接口的抽象方法叫做`函数描述符`，类比就是英语里的谓语。

``` java
Supplier supplier = () -> 1;
System.out.println(supplier.get()); //1
```

### 匿名函数

使用 lambda表达式，可以替代以前的匿名内部类，简化代码

``` java
//java8之前的写法，使用匿名内部类
FutureTask task = new FutureTask<>(new Callable<String>() {
    @Override
    public String call() throws Exception {
        return "I am task";
    }
});
new Thread(task).start();

//使用匿名函数
FutureTask task = new FutureTask<>(() -> "I am task"); //这里没有对() -> "I am task" 命名
```

需要注意的是在匿名类 ，this代表类自身，而 lambda中代表包含类。

当然只能用在继承了函数式接口的匿名内部类里面， 可以看作针对仅仅涉及单一方法的语法糖。

在 jvm 里面，lambda直接翻译成了字节码，而其是使用字节码指令集 `invokedynamic指令` 来实现的，而非翻译回匿名内部类。当时也考虑过基于`MethodHandle`的`MethodHandleProxy`，但是因为不够理想而放弃了。

### immutable

在使用 lambda表达式 ，如果存在需要使用外部变量，这个外部变量会被隐式地被 `final` 修饰，即`闭包`，就是一个函数的实例，且它可以无限制地访问那个函数的非本地变量。

``` java
//这里num被final 修饰了，无法对其进行修改
Integer num = 114514;
Supplier supplier = () -> "stench numbers: " + num; 

//当然可以使用一个引用类型包装，虽然这个引用类型也会是不可变的，但是我们可以改变他成员变量的值
class Box { Integer num;}
Box o = new Box(); o.num = 114514;
Supplier supplier = () -> "stench numbers: " + o.num;
o.num = 1919810;
System.out.println(o.num); // 1919810

//或许使用一个线程安全的引用类型是更好的选择
AtomicInteger atom = new AtomicInteger(114514);
Supplier supplier = () -> "stench numbers: " + atom.get();
atom.set(1919810);
System.out.println(atom.get()); // 1919810

//当然，使用数组也没有问题的，这也是在java文档里面推荐的
Integer[] arr = new Integer[]{114514};
Supplier supplier = () -> "stench numbers: " + arr[0];
arr[0] = 1919810;
System.out.println(arr[0]); // 1919810
```

而不可变对象有什么好处呢，其实大多数函数式语言都使用不可变变量来避免副作用，如 scala 的 `val` 。但是这里，java是有其他的考虑，可以说是殊途同归。

我们来看看 java 字节码

``` java
Integer num = 114514;
Supplier supplier = () -> {
  Integer var0 = num;
  return "stench numbers: " + var0;
};
```

事实上，在这里，变量是局部变量的一个拷贝。这其实是java的一个trick，java不是第一次这样做了。

这是对于 java 内存模型的考虑，实例变量存在堆中，而局部变量是在栈上分配，将局部变量私有化，这样就不会将变量地址泄漏出去（逃逸分析），做到内存安全，这是 java 虚拟机栈所要求的。而达到这样的目的就需要全局变量不可变，让局部变量的值等于全局变量的值。

当然，顺带一说，immutable 可以给函数式编程带来几个好处

- 优化GC，一方面来说，80%的对象周期都极短，而不可变对象更好对其做分代GC，更重要的是，不可变对象没有环引用，引用计数很好做，而函数式语言的GC一般效率都很高(erlang 、haskell)
- 而且不可变对象加上无副作用的函数式编程，可以确认入参和出参的不可变对象的记录，思维模式变得简单了，而且便于写测试代码和debug。

- 无锁化，都是不可变对象，也就不存在共享变量的变化了，配合一些函数式的技巧，大大增强并发效率。

##### 副作用

副作用（side effect）简单来说就是对所处的环境有改变，这个环境可以是指等待输入数据，典型例子就是 readLine()，因为每次都是读取下一行，读入的文字都会不一样，就会造成不同的结果。

再比如说，命令行打印一行文字，事实上没有副作用的函数不允许改变外部成员变量，而打印文字也就是改变了控制台这个对象。(所以有这样一个比喻，要让打印控制台是纯的，那么就有两个世界，一个是控制台没有文字的世界，而另一个是控制台有文字的创造后的新世界)

甚至是像Random函数，众所周知，程序里面的随机数是伪随机的，而Random函数每次都会改变 seed ，而这个 seed 是一个外部变量，自然也是有副作用的。

没有改变外部环境，只有入参和出参，也被称作纯函数。如果一个方法既不修改它内嵌类的状态，也不修改其他对象的状态，使用return返回所有的计算结果，那么我们称其为纯粹的或无作用的。

这样的函数是具有`引用透明性` （referential transparency）的，如果一个函数只要传递同样的参数值，总是返回同样的结果。

打个比喻来说 `y = x + 1` 就是一个纯函数，每次输入相同的值后，输出结果是 **一定是相同** 的，而且没有对外界造成影响。 

在java里面，函数式是不会处理 return 和异常的，在lambda表达式里面，不允许返回方法。同样异常强制需要在lambda表达式里面捕获。

``` java
// 虽然 forEach 里面本质上使用的是 for，但是lambda表达式里面无法返回方法
public int getResult(){
  Arrays.asList(1,3,5)
    .forEach( result -> {
      if ( result == 3){
        return result ; //无法编译
      }
    });
  return 0;
}
//正确的写法
public int getResult(){
  List<Integer> list = Arrays.asList(1,3,5);
  for (Integer result : list) {
    if ( result == 3){
      return result ;
    }
  }
  return 0;
}

//而异常强制要在lambda表达式里面处理
Arrays.asList(1,3,5)
  .stream()
  .peek(r -> {
    try {
      System.out.println(r);
      Thread.sleep(500);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

  })
  .count();
```

##### 高阶函数

在接口`Function`，里面有两个默认方法  `andThen`和 `compose` 。当然，在其他函数接口里面也有，用于编写高阶函数 (high-order function)。

以数学公式举例

![](http://latex.codecogs.com/gif.latex?\\f(x)=x+5)

![](http://latex.codecogs.com/gif.latex?\\g(y)=y*2)

![](http://latex.codecogs.com/gif.latex?\\h(z)=z-7)


当我们需要将3个公式组合在一起的时候


![](http://latex.codecogs.com/gif.latex?\\h(g(f(x)))=(x+5)*2-7)

当然，这里看起来有一些反人类，我们修改成符合人们习惯的公式


![](http://latex.codecogs.com/gif.latex?\\f\circ%20g\circ%20h(x)=(x+5)*2-7)

前者便是 `compose` ，后者便是  `andThen` ，以下是代码实现

``` java
Function<Integer,Integer> f =  x -> x + 5 ;
Function<Integer,Integer> g =  y -> y * 2 ;
Function<Integer,Integer> h =  z -> z - 7 ;
System.out.println(" andThen function result: "  + f.andThen(g.andThen(h)).apply(20));
System.out.println(" compose function result: "  + h.compose(g.compose(f)).apply(20));

// andThen function result: 43
// compose function result: 43
```

需要注意的是，高阶函数是**可结合的** ，满足结合律。

显而易见

![](http://latex.codecogs.com/gif.latex?\\f\circ%20g\circ%20h(x)=(x+5)*2-7\\=(2x+10)-7\\=y*2-7\:(y=x+5))


``` java
Function<Integer,Integer> continues = null;
Function<Integer,Integer> results = null;

continues = g.andThen(h);
results = f.andThen(continues);
System.out.println(" ( f ◦ g ） ◦ h    function result: "  + results.apply(20));


continues = f.andThen(g);
results = continues.andThen(h);
System.out.println("  f ◦ （ g ◦ h ）  function result: "  + results.apply(20));
//( f ◦ g ） ◦ h   function result: 43
// f ◦ （ g ◦ h ）  function result: 43
```

高阶函数需要注意的就是类型推断，上个函数的出参和下个函数的接受入参类型是一样的。而多参情况，其实也是可以结合的，当然也可以使用元组来包装。

### Stream 

映射（map）方法

| 方法       | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| filter     | 筛选                                                         |
| map        | 转化一个对象成其他的对象                                     |
| distinct   | 去重                                                         |
| sorted     | 可以接受一个Comparator 接口，排序                            |
| peek       | 做一些处理，没有返回值，一般最后用count归约                  |
| limit/skip | 截断操作，limit获取其前N个元素，skip获取其后N个元素，limit常和无限流一起使用 |

归约（reduce）方法

| 方法                        | 作用                                                         |
| --------------------------- | ------------------------------------------------------------ |
| forEach                     | 做一些处理，没有返回值                                       |
| toArray                     | 返回数组                                                     |
| reduce                      | 规约操作，最后只返回一个值，其他的规约方法都可以用这个来实现，如果没有设置单位元，将返回一个Optional |
| collect                     | 即可以选择容器聚合（分别传入存放容器的构造，添加到容器的函数，聚合策略），也可以直接聚合到继承Collector的实体类里面，常和Collectors.toCollection(ArrayList::new)一起使用，这里可以传入任意实现 Collector 接口的类（必须是有构造方法的实体类，抽象类和接口无法new） |
| min/max                     | 返回最大值，最小值，返回的是Optional                         |
| count                       | 统计数量                                                     |
| anyMatch/allMatch/noneMatch | 判断匹配，最后返回的是一个boolean值                          |
| findFirst/findAny           | 寻找元素，但 findAny 在parallel操作里面可能不会返回第一个值，返回的是Optional |

##### 无限流

事实上在没有使用映射方法之前。无论有多少个映射方法，这个List都不会去求值的，这也是我们为什么能够使用Stream 去构建`无限流` 。以前我们使用的 `List`是 严格求值（strictness）的，即总是对它的参数求值，而 `Lazylist`是非严格求值（non-strictness）的，可以选择不对它一个或多个参数求值，即不使用规约方法，也就是说可以延迟执行的。

``` java
Stream<Integer> stream = Stream.iterate(0,t -> t+1); // 0，1，2，3 ...
//但是这里不会去赋值
```

##### 短路

我们常见的短路也就是 `&&`  和 `||` 操作符，举例来说，`( 1 == 2 ) && ( 2 == 2 )` 当我们执行上面的计算的时候，其实我们只计算了第一步，第一步返回 false 之后，我们就知道整个值都是 false 了，后面的计算就没有必要了，因此后面的表达式事实上是没有计算的，计算机也是这样处理的。 `( 1 == 1 ) || ( 3 == 2 )` , 或操作同理，只计算了第一个表达式，也就返回 true 了。

而映射操作 `filter` 起到了同样的效果，这样被筛选掉的值就不会参与后面的计算，减少了时间复杂度，而`递归` 让我们更好容易理解程序运行过程。

``` java
long time = System.currentTimeMillis();
Stream.iterate(0,t -> t+1)
    .limit(10)  // 0,1,2,3,4,5,6,7,8,9
    .filter(t -> t % 2 == 0)  //0,2,4,6,8 这里对于奇数就被短路了，不再参与后面的计算  
    .map(t -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return t + 100;}) // 100,102,104,106,108
    .forEach(t -> System.out.println(t));
System.out.println("spent time: " + (System.currentTimeMillis() - time));

//100
//102
//104
//106
//108
//spent time: 5058   这里只用了5秒
```

##### 并行流

如果声明使用`parallelStream()` 或者 `stream().parallel()` 即使用了并行流，并行流是多线程执行的，使用的`ForkJoinPool::commonPool` 线程池实现。使用中如果有共享变量，需要注意线程安全，或者使用`monoid` 技巧并发执行，也可以采用`CSP并发模型`来并发。

##### 扁平和流连接

flatmap 可以将几个小的list转换到一个大的list。

``` java
Arrays.asList(Arrays.asList(1,4,8),Arrays.asList(6,9,2))
    .stream()
    .flatMap(List::stream)
    .forEach(i -> System.out.print(i + " "));
//1 4 8 6 9 2 
```

使用 concat 将两个流连接到一起

``` java
Stream.concat(Arrays.asList(1,4,8).stream(), Arrays.asList(6,9,2).stream())
    .forEach(i -> System.out.print(i + " "));
//1 4 8 6 9 2 
```

##### 下游收集器

Collectors 可以对list规约，最后返回一个 map ，以下都是 `Collectors::toMap` 方法

使用 groupingBy 分组

``` java
Map<String, List<Integer>> map = Arrays.asList(1,1,3,3,3,5,8)
    .stream()
    .collect(Collectors.groupingBy(t-> "I am " + t));
System.out.println(map);
//{I am 3=[3, 3, 3], I am 1=[1, 1], I am 8=[8], I am 5=[5]}
```

使用 partitioningBy 分片

``` java
Map<Boolean, List<Integer>> map = Arrays.asList(1,1,3,3,3,5,8)
    .stream()
    .collect(Collectors.partitioningBy(t-> t == 1 ));
System.out.println(map);
//{false=[3, 3, 3, 5, 8], true=[1, 1]}
```

除此之外，还有`toSet` 、`toList`、`counting`  、`averaging`、 `mapping` 、`summarizing` 、`reducing` 等方法

##### call by need

| 调用          | 代码                                                         | 说明                             |
| ------------- | ------------------------------------------------------------ | -------------------------------- |
| call-by-value | `Integer i = 2 * 3;`                                         | 在调用函数时就预先计算了         |
| call-by-name  | `Supplier<Integer>  supplier = () -> 2 * 3;` <br>`supplier.get();` | 在调用函数时计算                 |
| call-by-need  | Stream API                                                   | 相比于call-by-name，只会求值一次 |

后面两者都可以被称作惰性求值 （Lazy Evaluation）。

call-by-value 即在 java 里面的 常见的赋值，如代码里，会直接在这一行求值，即 `i = 6`。

call-by-name 一般来说不会立即求值，而是到了调用的时候再进行求值，并且这个值会保存下来，在下次调用的时候就直接用这个值了。举个例子来说，tomcat 里面 ，当我们第一次访问页面的时候，会初始化 jsp ，然后返回结果，而之后再访问这个页面，就会直接返回这个结果。而如果这个页面一直没被访问过，那么就一直不会初始化。

java 里面实现起来会有点困难，而scala 里面提供了一个语法糖,使用 `i: => A`   来延迟调用，也被称作 `thunk`。 

不使用 thunk

``` scala
lazy val i  = { println("I am lazy"); 2 * 3} //这里使用 lazy 关键字，只有在调用的时候求值

println("begin")
def printValue (i: Int){ //这里就触发计算了 ，打印 I am lazy

    println("lazy function running")
    println(i) //只会打印计算后的结果
}
printValue(i)
println(i) //只会打印计算后的结果

//begin
//I am lazy
//lazy function running
//6
//6
```

使用 thunk

``` scala
lazy val i  = {println("I am thunk"); 2 * 3} //这里没有预先计算

println("begin")
def printValue (i: => Int){ //在传入过程中，并不会去触发计算，注意区别，这里是 ：=>

    println("thunk function running")
    println(i)   //开始计算，打印 I am thunk
}
printValue(i)
println(i)  //这里并不会打印 I am thunk ，这里只会打印6，即只会计算一次，这里是直接调用结果

//begin
//thunk function running
//I am thunk
//6
//6
```

Haskell 里面默认的赋值操作都是  call-by-name 的，区别于其他语言。

call-by-need 的意思是只能被求值一次，同样规约操作也只能执行一次，也就是Stream API 的体现。

``` java
Stream<Integer> stream = Stream.iterate(0,t -> t+1).limit(10);
stream.count(); //在这里是不会报错的
stream.count(); //抛出异常 java.lang.IllegalStateException: stream has already been operated upon or closed
```

这里相当于不允许 call-by-name 的函数再次执行计算过程。

### 进阶

##### 柯里化和反柯里化

柯里化就是把接受多个参数的函数变换成接受一个单一参数，而反柯里化就是把接受一个单一参数的函数变换成接受多个参数。

事实上，只要一个方法，能够返回 lambda 表达式，就可以实现 柯里化和反柯里化。但是 java写起来太啰嗦了，好比是在 java8 里面，还在用内部类写匿名函数，这里使用 scala 代码来解释。

还是以数学公式举例

![](http://latex.codecogs.com/gif.latex?\\f(x,y)=(x+5)*y)


这里是一个二元函数，我们希望把它转化成单参数的形式

既然，我们使用 `f` 能够表示是一个函数，那么 `fx` 也能表示一个函数，这里都是数学符号 （暴论）

![](http://latex.codecogs.com/gif.latex?\\fx(y)=(x+5)*y)

这里的 `fx` 其实是一个函数

![](http://latex.codecogs.com/gif.latex?\\fx=x+5)

这样就转换成了单一参数的形式，也就是柯里化，反过来就是反柯里化

scala代码实现

``` scala
//多个参数
def f (x:Int,y:Int): Int = { (x + 5) * y }
println(f(2,3))
//单一参数
def fx(x: Int) = (y: Int) => { (x + 5) * y }
println(fx(2)(3))
```

tabulate 和 fold 函数就是科里化的实践

``` scala
val result = List.tabulate(100)( x => x + 1)
.fold(0)((a,b) => a +b)
println(result)
//计算 1~100 的和 ，即 5050
```

柯里化的作用，只是为了 **简化lambda演算证明，把多元函数简化成一元函数，因为纯正的lambda演算是没有多元函数的** ，仅此而已。

##### 箭头函数的右结合

箭头函数是隐式右结合的

下面是柯里化和反柯里化的scala实现

``` scala
def curry[A,B,C](f:(A,B) => C): A=> ( B => C) = { //注意这里

    a:A =>  b:B => f(a,b)
}

def uncurry[A,B,C](f:A => B => C): (A,B) => C = {

    (a:A,b:B) => f(a)(b)
}

println(curry((x:Int,y:Int) => ( x + 5 ) * y).apply(2).apply(3))
println(uncurry((x:Int) => (y:Int) => ( x + 5 ) * y).apply(2,3))
```

需要注意的是 ，我们可以看到 `A => ( B => C)`  这样的 lambda 表达式， 这里 A,B,C 表示类型，假设都用 `Int` 代替： `Int => (Int => Int)`  ，这里表示入参是一个Int 类型参数，出参是一个 `Function[Int,Int]` 

而实际上，这里可以不用加括号，因为箭头函数是隐式右结合，可以写成 `Int => Int => Int`

而  `(Int => Int) => Int`  和前面两种是不等价的，表示入参是一个  `Function[Int,Int]`  类型参数，出参是一个Int 类型参数

柯里化和反柯里化的scheme实现

``` scheme
(define curry (lambda (f) (lambda (x) (lambda (y) (f x y)))))

(define uncurry (lambda (f) (lambda (x y) ( (f x) y))))

(display (((curry (lambda(x y) (* (+ x 5) y ))) 2 ) 3))
(display ((uncurry (lambda(x)( lambda(y) (* (+ x 5) y )))) 2 3 ))
```
