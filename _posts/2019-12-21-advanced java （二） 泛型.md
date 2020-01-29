---
layout:     post
title:      advanced java （二） 泛型
subtitle:   advanced java （二） 泛型
date:       2019-12-21
author:     ica10888
catalog: true
tags:
    - java
---

# advanced java  泛型

泛型，即“参数化类型”。泛型一方面来说，可以提高安全性，可以在编译期提前发现错误，另一方面，定义类型后，可以提前确认类型，避免强转问题。

一个强大的类型系统，可以给程序带来鲁棒性和可维护性。

一般来说，泛型有四种使用方法，即除了`类型限定`，还有`类型变异`（分为向上和向下）和`通配符`。因为java里面是伪泛型的实现，泛型只存在于编译期，在运行时，需要做一些处理。在 jvm 中，存在 `类型擦除` 的问题，在类型擦除的时候，通过 `桥方法` ，达到一些 work around 。泛型在与java其他特性在一起时候，会出现一些问题，需要有处理方式。

### 泛型

| 类型                  | 代码            | 表现形式 |
| --------------------- | --------------- | -------- |
| 不变（invariant）     | `<T>`           | 类型限定 |
| 协变（covariant）     | `<T extends U >` | 类型变异，T是U的子类 |
| 逆变（contravariant） | `<T super U >`  | 类型变异，T是U的父类 |
| 量化  （quantification) | `<?>`           | 通配符   |

前三种都是变性 （Variance），表达了类层次结构和多态类型之间的关系。

##### 不变

``` java
interface Delicious{}
interface GoodLooking{}
class Fruit {}
class Apple extends Fruit implements Delicious,GoodLooking {}
class Durian extends Fruit implements Delicious {}
```
大多数情况下，子类型可以隐性的转换为父类型

``` java
ArrayList<Fruit> Fruits = new ArrayList<>();
Fruits.add(new Fruit());
Fruits.add(new Apple());
```
##### 协变和逆变

协变可以限定并传入父类。而逆变的时候，需要做一些处理，即需要`PECS法则`（Producer extends Comsumer super）。

``` java
public static <T extends Fruit> void doSomethingA(Optional<T> some){
    some.get();
}

public static <OT extends Optional<? super Apple>> void doSomethingB(OT some){
    some.get();
}
```

``` java
doSomethingA(Optional.of(new Apple()));
doSomethingB(Optional.of(new Fruit()));
```

协变和逆变可以是自己，即`<T extends Fruit>` ，T可以是`Fruit`

##### 量化

即可以传入所有类型。

``` java
public static void doSomethingC(Optional<?> some){
    some.get();
}
```

``` java
doSomethingC(Optional.of(new Apple()));
doSomethingC(Optional.of(1.0f));
```

`<?>`和`<Object>` 的区别在于其便利性，`List<?>`不能添加任何元素。

``` java
ArrayList<Object> listA = new ArrayList<>();
listA.add(new Apple());
ArrayList<?> listB = new ArrayList<>();
listB.add(new Apple()); //非法，编译错误
```

另外，使用一个通配符来捕获类型或许是更好的选择。

``` java
public static <T> void doSomethingC(Optional<T> some){
    some.get();
}
```

##### & 符号

``` java
public static <T extends Fruit & Delicious & GoodLooking> void doSomethingD(Optional<T> some){
    some.get();
}
```

``` java
doSomethingD(Optional.of(new Apple()));
doSomethingD(Optional.of(new Durian())); //非法，编译错误
```

`extends` 可以通过 & 符号表示多继承关系。但是继承类必须放在第一个位置。其中`Durian`没有 `GoodLooking`接口，会无法通过编译。泛型`extends` 可以只继承接口，如`<T extends Delicious>`也是可行的。

### 类型擦除

在java5 加入泛型之后，为了向前兼容性，采取了类型擦除。java的泛型只在源码中存在，编译后的字节码会替换成原生类型（Raw Type）。

``` java
ArrayList<Apple> apples = new ArrayList<>();
ArrayList<Fruit> fruits = new ArrayList<>();
System.out.println(apples.getClass()==fruits.getClass()); //true
apples.add(new Apple());
System.out.println(apples.get(0));
```
类型擦除后

``` java
ArrayList apples = new ArrayList<>();
ArrayList fruits = new ArrayList<>();
System.out.println(apples.getClass()==fruits.getClass());
apples.add(new Apple());
System.out.println((Apple) apples.get(0));
```

`ArrayList<Apple> `和 `ArrayList<Fruit>`  都变成了原生类型 `ArrayList<E>`

##### 桥方法

桥方法是编译器在泛型类型和协变返回类型中做的一个 trick ，用来消除类型擦除带来的影响。

``` java
class StringList<String> extends ArrayList<String> {
    public boolean add(String e) { return false;}
}

ArrayList<String> strings = new StringList();
```

这里就会存在泛型方法和多态冲突的问题了，由于父类 `ArrayList<T>`，类型擦除后，会生成 `boolean add(Object e)` ，而子类是 `boolean add(String e)` ,无法继承。

解决方法是，java字节码会偷偷加上一个方法

``` java
public boolean add(Object e) {
    add((String)e);
    return false;
}
```

但是还会有一个问题

``` java
class StringList<String> extends ArrayList<String> {
    public String get(int index) { return (String) "";}
}
```

其父类的实现是`E get(int index)`,类型擦除后，就是 `Object get(int index)` ，这样返回类型父类是`Object`,子类是`String` ，就又出现冲突了。

那在字节码里面这样实现就可以了。以下两个方法的类型签名是相同的，都是`get(int)`，在代码里面是无法实现的，但是在jvm字节码里可以存在。 

``` java
public String get(int index) {...}
public Object get(int index) {return get(index);}
```

##### 实例化泛型变量

泛型无法实例化对象

``` java
public static <T> void create(T some){
    T o = new T(); //非法，编译错误
    T[] arr = new T[10]; //非法，编译错误
}
```

或许可以使用反射机制来实现，或者像`ArrayList`的内部实现，使用`Object[]`。

``` jav
T o = (T) new Object();
Object[] arr = new Object[10];
T frist = (T) arr[0];
```

##### 数组

在有泛型时，初始化数组需要一些技巧

``` java
@SuppressWarnings("unchecked")
Optional<T>[] optionals = (Optional<T>[]) new Optional<?>[100];
```

##### 异常

`Throwable` 的子类不能是泛型的。

``` java
class SomeException extends RuntimeException {}

public static <T extends RuntimeException> void throwSomeException(T ex) throws T {
    T o = (T) new RuntimeException();
    throw o;
}

throwSomeException(new SomeException()); //这里会抛出RuntimeException而非SomeException
```

 `catch(T ex)`是非法的。

``` java
public static <T extends Exception> void throwSomeException(T ex) {
    T o = (T) new RuntimeException();
    try {
        throw o;
    } catch (T ex) { //非法，编译错误
        ex.printStackTrace();
    }
}
```

