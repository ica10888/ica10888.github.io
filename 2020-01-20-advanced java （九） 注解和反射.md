---
layout:     post
title:      advanced java （九） 注解和反射
subtitle:   advanced java （九） 注解和反射
date:       2020-01-20
author:     ica10888
catalog: true
tags:
    - java
---


# advanced java （九） 注解和反射

注解是一种 元数据（matadata），元数据为描述数据的数据，主要是描述数据属性的信息。

一般来说注解提供了四个注解对自定义注解进行配置，分别是

- `@Documented` 是否添加到 Javadoc 中。
- `@Target` 注解可以添加的范围（类名，方法名等）
- `@Retention` 注解的生命周期
- `@Inherited` 用于配置继承后的子类是否也享有该注解。

在 java当中一般会使用反射来对注解过的对象做处理，比如说 Spring 框架里面的 `@Bean` 

同样 java 的反射机制很强大，不仅可以获取到类，类加载器，方法名，私有变量等等。同样还可以通过 `invoke` 来调用方法，甚至是一些 `private` 修饰的方法。

java 中使用的是 双亲委派模型 (parents delegate model)，实现对 class 的加载。

最后会说一下 Spring 框架中的 `BeanFactory` 和 `FactoryBean` 是通过反射来注册 `Bean` 的，以及 Spring 框架中 aop 和 ioc 思想。

### 元注解

当声明使用  `@interface` ，就可以自定义注解，注解本身没有意义，一般和反射搭配使用，常见的就是 Spring  框架。

@Documented 

一般用于文档，会被 javadoc 之类的工具处理。

@Target

| ElementType               | 范围                           |
| ------------------------- | ------------------------------ |
| TYPE                      | 用于描述接口、类、枚举、注解   |
| FIELD                     | 用于描述字段、枚举的常量       |
| METHOD                    | 用于描述方法                   |
| PARAMETER                 | 用于描述方法参数               |
| CONSTRUCTOR               | 用于描述构造器                 |
| LOCAL_VARIABLE            | 用于描述局部变量               |
| ANNOTATION_TYPE           | 用于描述注解                   |
| PACKAGE                   | 用于描述包                     |
| TYPE_PARAMETER  ( java8 ) | 用于描述泛型参数               |
| TYPE_USE ( java8 )        | 只要是形态名称，都可以进行标注 |

说明了注解所修饰的对象范围

@Retention

| RetentionPolicy | 范围                                                         |
| --------------- | ------------------------------------------------------------ |
| SOURCE          | 注解仅存在于源码中，在class字节码文件中不包含                |
| CLASS （默认）  | 注解会在 class 字节码文件中存在，但 Class Loader 不会加载，运行时无法获得 |
| RUNTIME         | 注解会在 class 字节码文件中存在，在运行时可以通过反射获取    |

注解有三种生命周期，比如说 `@Override`  一般用于检查重载，所以它的生命周期是 `SOURCE` 。而`CLASS` 一般会和注解处理器(Annotation Processor) 一起使用，它是 javac 的一个工具，它用来在编译时扫描和处理注解。而大多数需要使用反射接口的注解，如 Spring 框架里面的大量注解，会使用`RUNTIME` 。

@Inherited

Inherited作用是，使用此注解声明出来的自定义注解，在使用此自定义注解时，如果注解在类上面时，子类会自动继承此注解，否则的话，子类不会继承此注解。只对类有效。

### 使用注解

注解不支持继承，在注解中可以声明各种类型的变量。可以在注解中声明枚举类型，可以使用泛型，也可以对注解嵌套。可以声明类型或者类型的数组。

``` java
@Target({ElementType.LOCAL_VARIABLE,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface AwesomeAnnotation{
    enum Status {FIXED,NORMAL};
    String[] value();
    String desc() default "";
    Class<? extends Annotation> annotation() default  Annotation.class;
    Status status() default  Status.NORMAL;
    Retention retention() default @Retention(RetentionPolicy.RUNTIME);
}
```

如果没有设定默认值，那么使用注解的时候必须要提供元素的值。 `value` 可以省略 key=value 的语法，这被称作快捷方式。

``` java
@AwesomeAnnotation({"awesome","good"})
String s1;
@AwesomeAnnotation(value = "awesome",desc = "s2",annotation = AwesomeAnnotation.class)
String s2;
@AwesomeAnnotation(value = "awesome",status = AwesomeAnnotation.Status.FIXED)
String s3;
@AwesomeAnnotation(value = "awesome",retention = @Retention(RetentionPolicy.CLASS))
String s4;
```

在 java8 中，新增了对类型注解和重复注解的支持。

类型注解也就是  `TYPE_PARAMETER`  和  `TYPE_USE`

重复注解可以在同一方法、属性、类等类型中多次使用同一个注解。这样会比之前的写法更简单方便，可读性更强。

之前的写法

``` java
public @interface SubAnnotation {
    String value();
}
```

数组保存注解

``` java
public @interface RepeatableAnnotations {
    SubAnnotation[] value();
}
```

使用多个注解

``` java
@RepeatableAnnotations({@SubAnnotation("sub1"),@SubAnnotation("sub2")})
String s1;
```

而 java8 就简化了这种写法

使用 `@Repeatable`

``` java
@Repeatable(RepeatableAnnotations.class)
public @interface SubAnnotation {
    String value();
}
```

使用多个注解

``` java
@SubAnnotation("sub1")
@SubAnnotation("sub2")
String s1;
```

不过 `@Repeatable` 不能和 `@Retention(RetentionPolicy.RUNTIME)` 一起使用， `@Repeatable` 限定了只能在 `ClASS` 及其级别以下使用。


### 反射

反射是 java 中一个十分强大的特性，运行时可以检测和修改它本身状态或行为，也可以获取元数据信息。可以用来简化代码，或者一些强大的功能，使代码更加灵活。不过反射的运行效率并不高。

反射其实是语义同像（Semantic Homoiconicity）的一种体现。反射将对程序以透明的机制，暴露给了开发者，让我们有能力去操纵它，能够更灵活地完成工作。虽然一定程度上破坏了封装，但是任何事物都有两面性。封装建立抽象屏障，降低程序的复杂度。事实上通过反射，将很多难以预测的问题留到运行时，动态地去解决。也就是说，给我们关闭了正面的大门，却留了一条后路。

获取 Class 类有三种方法

``` java
Class aClass = String.class;
String str = new String("Hello");
Class getClass = str.getClass();
Class forNameClass = Class.forName("java.lang.String");
```

可以通过 Class 获取构造方法对象 ，并新建对象

可以获取相应 Class 的局部变量

可以获取对象的方法，并调用对象上相应方法

可以获取运行时的注解

``` java
class Apple implements Serializable{
    public Apple(int wight, String color) {
        this.wight = wight;
        this.color = color;
    }
    protected static String type = "Fruit";
    private int wight = 100;
    @NotNull // RetentionPolicy.CLASS
    @Resource(name = "color") // RetentionPolicy.RUNTIME
    public String color = "red";
    private void setWight(int wight) {
        this.wight = wight;
    }

    @Override
    public String toString() {
        return "Apple{" +
                "wight=" + wight +
                ", color='" + color + '\'' +
                '}';
    }
}
```

调用反射接口创建对象

``` java
Class aClass = Apple.class;

Apple apple = (Apple) aClass.getConstructor(int.class,String.class).newInstance(50 ,"green");
System.out.println(apple);
//Apple{wight=50, color='green'}
```

获取局部变量，需要注意的是 `getFields()` 只会获取 public 修饰的字段

``` java
Field[] fields = aClass.getDeclaredFields();
Arrays.asList(fields).forEach(f -> {
    System.out.println(f.getType() +  ": [" + f.getName() + "]  modifier:" + f.getModifiers() );
});

//int: [wight]  modifier:2
//class java.lang.String: [color]  modifier:1
//class java.lang.String: [type]  modifier:12

System.out.println(aClass.getFields().length);
// 1
```
modifier 表示该字段的修饰符，规则如下

PUBLIC: 1
PRIVATE: 2
PROTECTED: 4
STATIC: 8
FINAL: 16
SYNCHRONIZED: 32
VOLATILE: 64
TRANSIENT: 128
NATIVE: 256
INTERFACE: 512
ABSTRACT: 1024
STRICT: 2048

当然 强大的 java 反射机制甚至可以调用方法，在某些场景下，可以绕开 java 的限制，调用私有方法

其中 `getMethods()` 是获取本类和父类，父接口的所有 `public` 修饰的方法，而 `getDeclaredMethods()` 获取本类中 `private` 、 `protected` 、默认以及 `public` 修饰的方法。

需要注意的是，如果要调用私有方法，需要修改 showPrivate 权限，不然会抛出 ` IllegalAccessException: Class hello can not access a member of class Apple with modifiers "private" ` 的异常

``` java
Class aClass = Apple.class;
Apple apple = new Apple(70,"yellow");
Method method = aClass.getDeclaredMethod("setWight",int.class);
method.setAccessible(true);
method.invoke(apple,120);
System.out.println(apple.toString());
// Apple{wight=120, color='yellow'}
```

同时反射可以获取运行时的注解，需要注意的是，只有运行时的注解（ `RetentionPolicy.RUNTIME` ）才会存在 runtime 阶段，而注解里面的局部变量可以继续通过反射获得。

`getDeclaredAnnotations()` 和 `getAnnotations()` 的区别在于 `getDeclaredAnnotations()` 会忽略继承的注解，也就是 父类存在的 `@Inherited` 修饰的注解。


``` java
Class aClass = Apple.class;
Field color = aClass.getDeclaredField("color");
System.out.println(color.getAnnotations().length); //1  因为只有 @Resource 是运行时注解， 而 @NotNull 不是
Resource resource =  color.getAnnotation(Resource.class);
System.out.println(resource.name()); //color
```

同样可以获取实现了哪些接口

``` java
Class aClass = Apple.class;
Arrays.asList(aClass.getInterfaces()).forEach(i -> {
    System.out.println(i.getName());
});
// java.io.Serializable
```

被 `Enclosing` 修饰的反射方法的使用对象是内部类或匿名内部类，内部类或匿名内部类可以通过 `getEnclosingClass()` 获取外部类， 通过 `getEnclosingConstructor()` 获取被使用的构造方法， `getEnclosingMethod()` 获取被哪个方法定义。

当然，反射还有处理数组和枚举类的API。

还可以通过 `getClassLoader()` 获取类加载器。

通过反射，就可以通过输入一些字段，达到构建一个对象的目的。这也就是说，通过一个 xml 配置文件，就能够生成一个 Bean 实例，这也就是 Spring 框架的一部分原理。

### 双亲委派模型

##### 类加载过程

在编译成字节码后，虚拟机将 CLASS 文件加载到内存，然后验证、准备，解析（这三个阶段统称为连接）和初始化，最后形成能被虚拟机使用的 java 类型，这就是虚拟机的类加载机制。

![](http://www.debugger.wiki/sourceimg/200106/00e120d92efd57d001d4226e89cf9c55.jpg)

类加载的时机

- 遇到 new、getstatic、putstatic、invokestatic 这四条字节码指令的时候
- 在反射调用的时候
- 初始化的类的父类如果没有初始化，会先初始化父类
- 虚拟机启动时，会初始化主类 （ `mian()` ） 
- 动态语言支持

类加载的生命周期

- 加载：通过类的全限定名获取此类的二进制字节流，然后将其静态储存结构转化为运行时的数据结构（到方法区），生成Class对象，提供访问入口
- 验证：验证和加载是交叉进行的。因为 Class 文件并不完全是 源码编译而来，所以需要验证来确保虚拟机的安全。通过基本格式的验证（0xCAFEBASE 、版本号等），元数据验证 （语义分析，如是否继承了不允许继承的父类等），字节码验证 （能否正常工作），符号引用验证 （能否找到引用的类，能否正常访问应用的对象）
- 准备：正式分配内存并初始化值
- 解析：将常量池内的符号引用替换为直接引用的过程。也就是将字面量转换为句柄。对累或接口，字段，类方法，接口方法的解析
- 初始化：类加载的最后一步，真正开始执行字节码。初始化的过程是执行类构造器 `<clinit>()` 的过程。对默认值，static变量，成员变量初始化。
- 使用：虚拟机使用类
- 卸载：结束生命周期



##### 类加载器

在 java 中 ，有三个Class Loader类型

- 启动类加载器： 加载的是 JVM 自身需要的类，这个类加载使用 C++ 语言实现的，是虚拟机自身的一部分。
- 扩展类加载器： 加载 "标准库扩展" 部分，由Java语言实现的，在以前的版本中位于 jre/lib/ext 。
- 应用程序加载器： 加载系统类 `classpath` 路径下的类，一般情况下该类加载是程序中默认的类加载器。

![](https://img-blog.csdn.net/20180630110324712?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01pbnQ2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


双亲委派模式 （parents delegation model）就是说在类加载请求的时候，总是由父类加载器去完成加载任务，只有父类加载器无法完成时，子类加载器才会尝试自己去加载。

采用双亲委派模式的是好处是 Java 类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载。





### aop和ioc

##### aop和ioc思想

依赖注入（Dependency Injection） 和 控制反转（Inversion of Control）是 Spring 框架中很重要的两个知识点。依赖注入是种实现控制反转用于解决依赖性的设计模式。目的是解耦。

依赖注入是一种设计模式，目的是解除程序组件之间的一些直接耦合。也就是依赖注入到 IOC 容器，把有依赖关系的类放入容器中。

在 Spring 框架当中，具体的做法是使用 `Bean` 、 `FactoryBean` 和 `BeanFactory`。`Bean` 就是由 Spring 容器初始化、装配及管理的对象。将原本的 `Bean` 类依赖 `Bean` 的具体实现 ，改成 `Bean` 类依赖一个接口和一个 IOC 容器。IOC 容器也就是 `BeanFactory` ，用于实例化、定位、配置应用程序中的对象及建立这些对象间的依赖，同时 `Bean` 的具体实现内容 ，也就是 `FactoryBean` ， 实现了 `Bean` 类的接口。

然后将 `Bean` 的具体实现的创建工厂 ( `FactoryBean` ) 注册到 IOC 容器 ( `BeanFactory` )，最后 `Bean` 只需要给IOC 容器 ( `BeanFactory` ) 一个标识索取 `Bean` ，IOC 容器将回调  `Bean` 的具体实现的创建工厂 ( `FactoryBean` ) ，生产出 `BeanFactory` 的实例之后交给开发者。

控制反转是一种设计代码的思路。

举一个例子来说，就是框架（Framework）和库（Library）的区别。在我们使用库的时候，会去主动调用库里面的方法，来达到我们的目的。而 Spring 框架做到了控制反转。也就是说，库是一种 "正向" 控制，而框架是反过来调用开发者的代码。


##### BeanFactory和FactoryBean的区别

`BeanFactory` 是接口 , 一般使用的高级容器 `ApplicationContext` 接口继承了这个接口（ 其实是继承了多个接口，其中 `ListableBeanFactory`,  `HierarchicalBeanFactory` 都继承了 `BeanFactory` ），所有Bean 由它来管理。

`FactoryBean` 在IOC容器的基础上给 `Bean` 的实现加上了一个简单工厂模式和装饰模式，让我们可以获取 `Bean` ，是通过反射机制实现的，很多 `Bean` 类都通过 `FactoryBean` 生产出实例。

`ApplicationContext` 继承的接口，如 `EnvironmentCapable` 、 `MessageSource` 、 `ApplicationEventPublisher` 、 `ResourcePatternResolver` 等描述了它的意义。 同样在 Spring 基础框架里面，有很多不同类型的 `FactoryBean` 实现不同的功能，通过注入 `Bean` 的形式。同时 IOC 容器 还会管理 `Bean` 类之间的依赖关系。



