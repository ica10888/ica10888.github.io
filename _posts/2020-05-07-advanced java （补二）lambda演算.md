---
layout:     post
title:      advanced java （补二）lambda演算
subtitle:   advanced java （补二）lambda演算
date:       2020-05-07
author:     ica10888
catalog: true
tags:
    - scala
    - functional programming
---

# advanced java （补二）lambda演算

谈到 lambda 演算一般来说就是比较偏学术界的东西了，一般来说，和编程语言理论 ( programming language theory，PLT) 息息相关，这一门学科用于编程语言的设计和实现。但是对于工程来讲还是有一些距离，不过适当了解一下历史，还是挺有意思的。

计算机语言最开始源于数学，数学家阿隆佐·邱奇在20世纪30年代提出了λ演算，然后将这个理论发扬光大的是 lisp 系的语言，其中如对函数的抽象，应用，递归等对语言有很大的的影响。通过函数可以定义语言的非形式化描述，可以用 λ 表达式表示现代语言，如 邱奇 `α-等价` , `β-归约` , `η-变换`  。

同样 `邱奇数` 把数据和运算符嵌入到lambda演算内，这个可以了解一下。

而 `Y-组合子` 允许匿名函数足够达成递归的作用，证明了 lambda 演算和计算机程序是等价的。

当然，这些知识可能对工程上作用不大，这些深入后是数学家们的范畴。不过对于有好奇心的人，还是有吸引力的。

### 非形式化描述

在λ演算中，每个表达式（λx.x）都代表一个函数，这个函数有一个参数(x)，并且会返回一个值。因为可以科里化，因此都可以单个参数的形式，同时是左结合的。

##### α-等价

α-等价表达的是，被绑定变量的名称是不重要的。

也就是说 λx.x和 λy.y 是同一个函数，当然更复杂的情况 λx.(λx.x) x 和 λy.(λx.x) y 也是一样的。

``` scala
def func1 = (x:Int) => 5 + x
def func2 = (y:Int) => 5 + y
```

也就是说 func1 和 func2 是同样的函数，虽然参数名不一样，但是是等价。(这里准确地说是 x => x，或者表达式应该是 λx.x 5，不过也没有影响)

参数名不重要，也就可以写成这样。

``` scala
_ => 5 + _
```

##### β-归约

β-归约表示，在应用函数时，可以直接对结果中相应的变量进行替换。

定义是 (λx.E)M -> E[x:=M]

举个例子来说  (λx.λa.xa)(λz.zy)  可以规约成  (λa.λz.zya) ， 其中 M 是 (λz.y)，但是需要注意规约前后的作用域要相同

``` scala
val y = 100
def lx_la_xa = (x:Int => Int) => (a:Int) =>  x andThen ( (t:Int) => a * t )
def lz_zy = (z:Int) => z + y
def func1 = lx_la_xa apply lz_zy
def func2 = (a:Int) => (z:Int) => a * ( z + y )
```

其中 func2 就是 func1 规约后的结果，将 lx_la_xa 中的变量 x 替换成了 lz_zy


##### η-变换

η-变换的外延性指的是对于任一给定的参数，当且仅当两个函数得到的结果都一致，则它们将被视同为一个函数。

对所有的 a 有 (λx .f x) a == f a，可得λx .f x == f ，即 η-归约

``` scala
def f:Function1[Int,Int] = (t:Int) => 54 + t
def func1 = (x:Int) => {(f andThen ( t => t )).apply(x) }
println(f.apply(24) == func1.apply(24))
```

对于 apply 任意值都打印 true ，func1 和 f 是等价的

### 丘奇逻辑

丘奇编码把数据编码到lambda演算，可以用来表示自然数和布尔逻辑。

##### 丘奇数

| 自然数 | lambda表达式      |
| ------ | ----------------- |
| 0      | λf.λx.x           |
| 1      | λf.λx.f x         |
| 2      | λf.λx.f (f x)     |
| 3      | λf.λx.f (f (f x)) |

也就是说自然数表示高阶函数的阶级，加法也就是两个高阶函数的结合，自然也就是阶级相加，乘法也就是m加上n次，以及前缀（ pred ），后继（ succ ），指数和减法

##### 丘奇布尔值

| 布尔逻辑 | lambda表达式      |
| ------ | ----------------- |
| true      |  λa.λb.a          |
| false     | λa.λb.b        |
| and      | λp.λq.p q p    |
| or      | λp.λq.p p q |
| if      | λp.λa.λb.p a b |

布尔值可以参与与或运算。同理 not 和 xor 也可以实现，以及谓词是 ISZERO

### Y-组合子

Y-组合子的定义如下：

Y = λf.(λx.(f(x x))(λx.(f(x x))))

也就是说匿名函数也是可递归的，证明了 Curry–Howard 同构。

用斐波那契数列来演示

``` scala
def fib(n:Int): Long = {
  @annotation.tailrec
  def recursion(m:Int,frist:Long,last:Long): (Long,Long) = {
    if(m==1)
    (frist,last)
    else
    recursion(m-1,last,frist+last)
  }
  recursion(n,0,1)._2
}
```

将 recursion 改写成 arrow function 的形式

``` scala
class Point (val m:Int,val frist:Int,val last:Int) {}
val fib20 = {
  def recursion:(Point => Point) = point => {
    if(point.m == 1)
      point
    else
      recursion.apply(new Point(point.m - 1 , point.last , point.frist + point.last))
  }
  recursion.apply(new Point(20,0,1))
}
println(fib20.last)
```

然后将函数作为参数传入

``` scala
class PointWithFunc (val m:Int,val frist:Int,val last:Int,val f:PointWithFunc => PointWithFunc) {}
val fib20 = {
  def recursion:(PointWithFunc => PointWithFunc) = point => {
    if (point.m == 1)
      point
    else
      point.f .apply(new PointWithFunc(point.m - 1, point.last, point.frist + point.last, point.f))
  }

  recursion.apply(new PointWithFunc(20, 0, 1, ???))
}
```

显而易见，这里最后需要传入 f ,同时主参数无法传入 f，需要写一个新的函数

``` scala
class PointWithFunc(val m: Int, val frist: Int, val last: Int, val f: PointWithFunc => PointWithFunc) {}
class Point(val m: Int, val frist: Int, val last: Int) {}
val fib20 = {
  def recursion:(PointWithFunc => PointWithFunc) = point => {
    if (point.m == 1)
      point
    else
      point.f .apply(new PointWithFunc(point.m - 1, point.last, point.frist + point.last, point.f))
  }

  def result:(Point => PointWithFunc) = point => {
    recursion.apply(new PointWithFunc(point.m, point.frist, point.last, recursion))
  }
  
  result.apply(new Point(20, 0, 1))
}
println(fib20.last)
```

通过引用透明法则，对函数进行替换并匿名

``` scala
class PointWithFunc(val m: Int, val frist: Int, val last: Int, val f: PointWithFunc => PointWithFunc) {}
class Point(val m: Int, val frist: Int, val last: Int) {}
val fib20 = {
    {point:Point => {
    {point:PointWithFunc => {
      if (point.m == 1)
        point
      else
        point.f .apply(new PointWithFunc(point.m - 1, point.last, point.frist + point.last, point.f))
      }}.apply(new PointWithFunc(point.m, point.frist, point.last,
      {point:PointWithFunc => {
        if (point.m == 1)
          point
        else
          point.f .apply(new PointWithFunc(point.m - 1, point.last, point.frist + point.last, point.f))
      }}
    ))
  }}.apply(new Point(20, 0, 1))
}
println(fib20.last)
```

可以发现，函数都被匿名了，并且是递归的。


