---
layout:     post
title:      advanced java （一） 继承和接口
subtitle:   advanced java （一） 继承和接口
date:       2019-12-18
author:     ica10888
catalog: true
tags:
    - java
---


#  advanced java （一） 继承和接口

java8之前，java为了避免多继承产生的问题，采用的方式是使用一个类只能继承一个父类，而可以实现多个接口。父类可以是抽象类，使用 ` abstract `关键字修饰，抽象类不能实例化对象。父类也可以继承普通类，类里的方法有实现，而接口里面的方法没有实现。

java8 之后，接口里面增加了静态方法和默认方法，需要用 `static` 和 `default` 关键字修饰，这样就可以在接口里面写方法的实现。但是这样不可避免地引入了菱形继承的问题。

另外还有一种`鸭子类型 `（Duck typing）的形式，一种宽泛的类型设计。而 `tarit `也是一种不错的设计， 可以避免抽象类的单一继承的限制 。

### 面向对象思想

 在面向对象编程中，有一些设计原则，能让我们能够高效地复用代码和设计架构。

以面向对象的思想思考继承的设计，其中需要符合的一个最重要的原则便是open-close原则，即对扩展开放，对修改封闭 （open For Extension,closed For Modification）。 具体实现以通过继承方式来重用，而接口勿需实现。对于已存在的实现修改是封闭的，但是新的实现不必实现原有的接口，即 `@overwrite`。  

 软件设计中，除开闭原则外，还有几个重要的设计思想，统称面向对象的七大基本原则

- 开闭原则：对扩展开放，对修改封闭
- 里式替换原则：程序中的所有用基类的地方，都可以用子类代替
- 依赖倒转原则：依赖于抽象而不依赖于具体
- 接口隔离原则：将大接口分散成小接口
- 单一指责原则：一个类的功能尽量单一
- 最小知识原则：一个对象应该尽量少的了解其他对象

里式替换提到了子类的可替代性，即子类可以用于父类，比如 `Car car = new BMW()` 。依赖倒转指的是需要的是抽象，比如需要一个List的时候，只需要的是List的增加，插入，删除节点等抽象，而不是链表或者跳表的具体实现。接口隔离即将接口拆开，一个类可以实现多个接口，具有多个性质。 单一指责和最小知识很容易理解，这里不再叙述。

### 静态方法和默认方法
在 java8 之前，由于在接口上无法实现方法，采用的是伴随类的折中方式。

如Collections 伴随类 之于Collection接口，  Executors 伴随类 之于Executor 接口。

伴随类不实现对应的接口，这样就降低了耦合度，同样也不能复用接口。继承抽象类，抽象类再实现接口或许是不错的主意，但是这样就丧失了接口的意义，因为一个类是无法继承多个抽象类的。

而使用伴随类的时候，如sort，如下使用

``` java
List<Integer> list =  Arrays.asList(4,10,2,5);
Collections.sort(list);
System.out.println(list); //[2, 4, 5, 10]
```

或许在接口里面实现是更好的选择

``` java
List<Integer> list =  Arrays.asList(4,10,2,5);
list.sort(Integer::compare);
System.out.println(list); //[2, 4, 5, 10]
```

java8在List 接口中，加入了默认方法 `sort`。

``` java
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```

而在 java8 以后Collection接口加入的方法，如 `stream` 和 `parallelStream` ,采用的都是静态方法，不再加入到伴随类中。

同样的，静态方法也可以加入到接口当中， 只是我们不能在实现类中重写它们，这样可以避免一些问题。

以Function 接口为例子

``` java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
    default <V> Function<V, R> compose(...) {...}
    default <V> Function<T, V> andThen(...) {...}
    static <T> Function<T, T> identity() {return t -> t;}
```

`@FunctionalInterface` 只允许存在一个非静态方法或默认方法的方法，即 `apply`。

`compose` 和 `andThen `是默认方法，如果有类继承 Function 或接口实现 Function，这两个方法是可以被重写的。(这两个方法可以用在高阶函数上)

`identity` 不能被重写，如果父类有此方法，会报错。（ identity 证明范畴存在单位元 ）

##### 菱形继承
由于默认方法可以被重写，这样菱形继承的问题就被带入进来了。

解决菱形继承问题的三条规则 

- 类中的方法优先级最高 

- 如果无法依据第一条进行判断，那么子接口的优先级更高 

- 最后，如果还是无法判断，继承了多个接口的类必须通过显式覆盖和调用期望的方法，显式地选择使用哪一个默认方法的实现。 


以下为例

``` 
 动物
 / \
人  鸭
 \ /
 鸭人
```

第一条规则

``` java
interface Animal {
    default void speak(){}
}

interface Duck extends Animal{
    default void speak(){ System.out.println("quaaaaaack");}
}

class Person implements Animal{
    public void speak(){ System.out.println("hello");}
}

class UnknownCreature extends Person implements Duck{}
// hello
```

第二条规则

``` java
interface Animal {
    default void speak(){}
}

interface Duck extends Animal{
    default void speak(){ System.out.println("quaaaaaack");}
}

interface Person extends Duck{ //???
    default void speak(){ System.out.println("hello");}
}

class UnknownCreature implements Person,Duck{}
// hello
```

第三条规则

``` java
interface Animal {
    default void speak(){}
}

interface Duck extends Animal{
    default void speak(){ System.out.println("quaaaaaack");}
}

interface Person extends Animal{
    default void speak(){ System.out.println("hello");}
}

class UnknownCreature implements Person,Duck{
    public void speak(){ Person.super.speak();}
}
// hello
```

如果三条都不符合，如最后没有覆盖 speak 方法，将无法编译。

### golang

##### 鸭子类型

 只要它会像鸭子一样叫，跑起来像鸭子，我们就把它看作鸭子 。

在 java8 之前的单继承多实现机制，由于接口无法实现方法，无法将一个方法实现传递至所有的父类，是属于扩展不友好，可以认为是封闭的设计。而鸭子类型一方面来说让我们方便扩展，但是由于没有对修改封闭，可能会造成一些困扰。

以一下代码举例

``` go
type flower interface {
	plant()  //种植
}
type flag interface {
	plant()  //安置
}

type Rose struct {
	Name string
}

func (a *Rose) plant() { //既是种植又是安置
	println("find a(n) " +  a.Name + " , then put it on soil")
}
```

其中 Rose 同时实现了植物的种植和旗帜的安置，如果遇到相当多的接口的时候，虽然可以通过现代IDE去查询实现了哪个接口，但是会造成语义的混乱。

鸭子类型的好处是容易修改接口，比如我们在flower 接口里面加入新的方法 `water()`，上级实现加入新方法的实现就可以了。

### trait
scala 的 trait  以混入（mix in）实现，而不是实现接口。

即 如果有一个类，我们可以声明他的很多特性，当我们需要使用其中一两个特性的时候，将这一两个特性混入进来就可以了。

当然，需要先使用`extends`声明特性是这个类的，如果未声明，就是最大的子类 `AnyRef`。

``` scala
class Animal {
  def head(): Unit = ???
  def body(): Unit = ???
  def legs(s: String): Unit = println(s)

}
trait TwoLegs extends Animal { override def legs(s: String) = super.legs(s + " with two legs") }
trait FourLegs extends Animal { override def legs(s: String) = super.legs(s + " with four legs") }

class Person extends Animal with TwoLegs
class Cat extends Animal with FourLegs

object  Main {
  def main(args: Array[String]): Unit = {
  
    val person  = new Person ; person.legs("person") // person with two legs
    
    val cat  = new Cat ; cat.legs("cat") //cat with four legs
  }
}
```

##### 瘦接口 VS 胖接口

Java8之前没有默认方法，都是瘦接口。而加入默认方法后，为了与旧库的兼容，也不能加入太多的默认方法。用抽象类又会失去接口的意义。

##### 堆叠 trait

需要注意混入的顺序是会有影响的，`super` 的顺序是由多重继承线性化的顺序决定的。

``` scala

class InsectA extends Animal with FourLegs with TwoLegs
class InsectB extends Animal with TwoLegs with FourLegs

object  Main {
  def main(args: Array[String]): Unit = {
  
    val insectA  = new InsectA ; insectA.legs("insectA") //insectA with two legs with four legs
    
    val insectB  = new InsectB ; insectB.legs("insectB") //insectB with four legs with two legs
    
  }
}
```

