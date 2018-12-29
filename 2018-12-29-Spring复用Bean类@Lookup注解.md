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

