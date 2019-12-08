---
layout:     post
title:      Spring复用Bean类@Lookup注解
subtitle:   Spring复用Bean类@Lookup注解
date:       2018-12-29
author:     ica10888
catalog: true
tags:
    - java
    - Spring
---


# Spring复用Bean类@Lookup注解

在有的时候，实现了一个组件后，对于不同的 Task 使用 @Autowired 访问， 访问的都是同一个组件。这是因为 scope 是 singleton 的。而声明 scope 是prototype，并使用 @autowire 注入的话，还是不能实现我们的需求。Spring 提供了  @Lookup  来实现复用Bean类。

通过@Lookup可以实现对Bean类的复用。

### Bean 类

Counter .java

``` java
@Component
@Scope("prototype")
public class Counter {
    
    private String str;
    
    public Counter(String str) {
        this.str = str;
    }
    
    //一些实现方法
}
```

### Task

**注意这里要用 public 修饰符** ，不然 Spring 在初始阶段会报错

Task1

``` java
@Component
public class Task1 { 

    private Counter counter;
    
    @PostConstruct
    public void init(){
        counter = createCounter("Task1");
    }
    
    @Lookup
    public Counter createCounter(String str) {
        return null;
    }
}
```

Task2

``` java
@Component
public class Task2 { 

    private Counter counter;
     
    @PostConstruct
    public void init(){
        counter = createCounter("Task2");
    }
    
    @Lookup
    public Counter createCounter(String str) {
        return null;
    }
}
```

这里分别实现了2个 Counter ，并同时由 Spring 管理它们的生命周期。

### 什么是@Lookup

当我们调用一个被 `@Lookup`  注解的方法时，Spring 会返回一个值类型的实例。

实质上，Spring会 override  注解方法并使用我们方法的返回值和参数作为BeanFactory#getBean方法的入参。

`@Lookup`  用于

- 向一个单例bean注入一个原型bean(prototype-scoped bean)(类似于Provider)
- 程序化地注入依赖

### 使用@Lookup

当我们决定使用一个原型bean时，我们将要面对的是 单例 bean 要如何使用这些原型bean呢？
当然可以使用Provider，但是@Lookup在某些方面更合适一些。

以上面的例子为例

@Lookup 通过我们的 单例bean 获取 Counter 实例。

``` java
@Test
public void whenLookupMethodCalled_thenNewInstanceReturned() {
    // ... initialize context
    Task1 first = this.context.getBean(Task1.class);
    Task2 second = this.context.getBean(Task2.class);
        
    assertEquals(first, second); 
    assertNotEquals(first.getNotification(), second.getNotification()); 
}
```

为什么方法要返回 null呢。

这是因为，spring通过调用beanFactory.getBean(Task1.class)重写了这个方法，所以我们将它置空。

使用@Lookup注入的话，还可以通过构造器传参，其实就是spring调用beanFactory.getBean(class, name)实现的。



### 参考

[@Lookup Annotation in Spring](https://www.baeldung.com/spring-lookup)
