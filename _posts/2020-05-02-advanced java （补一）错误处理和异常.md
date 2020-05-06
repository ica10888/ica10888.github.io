---
layout:     post
title:      advanced java （补一）错误处理和异常
subtitle:   advanced java （补一）错误处理和异常
date:       2020-05-02
author:     ica10888
catalog: true
tags:
    - java
    - golang
    - scala
    - functional programming
---

# advanced java （补一）错误处理和异常

错误处理和异常机制是现代语言中常见的一种处理方式，用于处理已知或未知的错误。java 语言当中，除了不可恢复的  `Error` ，异常  `Exception` 分为检查型异常 （ Checked Exception ）和非检查型异常 ，非检查型异常也称作运行时异常 ( RuntimeException ) ，两种异常都继承 `Exception`。

同样,还有一种处理方式是返回一个非法的错误，如返回一个错误码。如 golang 语言里通过多返回值，可以认为 error 类型是一种错误码，也就是当判断 `if err != nil ` 的情况后，不再对返回的参数做处理。同样，golang 里面也有异常处理机制，也就是通过 panic 和 recover 来异常处理。

而在函数式语言当中，因为异常破坏了引用透明，需要统一错误处理逻辑。函数式语言提供一种 `Either` 的数据类型，它有两种类型： `Left` 和 `Right` ，正常情况下返回正常值，而当有异常发生的时候，返回值为 `None` 。 golang 语言中之前的 try 语法糖的 proposal 可以认为是基于这个思想，不过因为 golang 缺乏泛型，暂时还不好实现。

而对于异常的处理，有两种思想，分别是 Erlang 的 `任其崩溃` 和平时写业务逻辑代码时候的 `防御性编程思想` ，也是对错误处理和异常的两种态度。

### 检查型异常和运行时异常

Java 中继承 `Throwable` 类的有两个子类，分别是不可恢复的 `Error` 和异常 `Exception` 。异常的子类分为两种，分别是 `Checked Exception` 和  `RuntimeException` 。两者的区别在于一个需要在编译期做处理，不然无法通过编译，而另外一种可以不做处理，一般用于可以处理的不可预料错误。

以一个读取文件的异常处理为例，其中 `FileNotFoundException` 和 `IOException` 属于检查型异常，如果不做处理，将无法通过编译。

``` java
public static void readFile(String path){
  File file = new File(path);
  Long filelength = file.length();
  byte[] filecontent = new byte[filelength.intValue()];
  FileInputStream in = null;
  try {
    in = new FileInputStream(file);
    in.read(filecontent);
    in.close();
  } catch (FileNotFoundException e1) {
    //do something e1
  } catch (IOException e2) {
    //do something e2
  } catch (Exception e3){
    //do something e3
  } finally {
    //finally do something
  }
}
```

其中以 `try…catch…finally…` 的形式处理异常，无论是否捕获异常，都将处理 `finally` 里面的语句。而异常处理需要注意的是，父类必须在子类之后，不然无法捕获子类异常，无法编译。 `Exception` 必须在两个异常之后，可以用来捕获运行时异常。

当然在java7 加入了异常处理的新特性，一种 catch 语句的新语法

``` java
public static void readFile(String path){
  File file = new File(path);
  Long filelength = file.length();
  byte[] filecontent = new byte[filelength.intValue()];
  FileInputStream in = null;
  try {
    in = new FileInputStream(file);
    in.read(filecontent);
    in.close();
  } catch (IOException | RuntimeException e) {
    //do something e
  }
}
```

需要注意的是，由于  `FileNotFoundException` 是 `IOException` 子类，这种语法无法捕获这种继承关系。

或者可以不做处理，在上级方法处理异常

``` java
public static void main(String[] args) {
  try {
    readFile("test.txt");
  } catch (IOException | RuntimeException e) {
    //do something e
  }
}

public static void readFile(String path) throws FileNotFoundException,IOException,RuntimeException{
  File file = new File(path);
  Long filelength = file.length();
  FileInputStream in = null;
  byte[] filecontent = new byte[filelength.intValue()];
  in = new FileInputStream(file);
  in.read(filecontent);
  in.close();
}
```

如果不处理运行时异常，将会抛出堆栈信息，并在异常语句处停止。

事实上，检查型异常必须在编译期处理，在平时使用的过程中和语言最初的设计出现了一些偏差。检查型异常设计的时候是希望能够在编译期提前发现错误，但是可能会造成过度设计，意思是有一些情况处理可以使用 `if…else…` 来处理，异常本质上也是一种控制流语句，不过使用异常来处理会符合逻辑，更加直观一些。另外，检查型异常可以声明很多类型的异常，容易被滥用。在使用方法栈调用的时候，要么 `throws` 交给上一级方法处理，要么直接捕获处理，异常类型和处理过于繁杂。实际上，并不需要这种方式来处理异常，如 `SQLException` ，在异常的 `message` 里面表示异常发生的原因，而不需要声明多种检查型异常来强制处理。处理的时候只需要统一捕获检查型异常 `SQLException` 就可以了。

### golang 的错误处理和异常

##### 多返回值

golang语言的错误处理是和多返回值一起来使用的，也就是说返回值可以同时返回结果和错误。

``` go
func main() {
    _ ,err := division(2,0)
    if err != nil {
        fmt.Println("Err: ",err.Error())
    }
}


func division(a int ,b int) (result int,err error) {
    if(b == 0){
        err = errors.New("division by zero")
    } else {
        result = a / b
    }
    return
}

//Err:  division by zero
```

有的时候我们用一些特殊的错误码表示错误类型，如使用 A0001，09999 等表示不同的错误类型，这种被称作错误码。实际上，error 接口包含一个字符串参数，也可以说是一种错误码类型。

##### panic 和 recover

golang的异常处理是通过内置函数 `panic` 和 `recover` 来处理的，如果 panic 之后不处理异常，将会抛出堆栈信息，而 recover 一般在方法最后使用，一般包含在 defer 方法内。

``` go
func main() {
  division(2,0)
}

func division(a int ,b int) (result int) {
	defer func(){
		if err := recover(); err != nil {
      result = 0
			fmt.Println("recover Err: ",err)
		}
	}()
	if(b == 0){
		panic(errors.New("division by zero"))
	} else {
		result = a / b
	}
	return
}
//recover Err:  division by zero
```

但是这种处理并不是很好，因为结果 0 虽然表示了无法相除，但是可能会用于接下来的运算，接下来的运算会使用错误的结果继续执行，最后导致不可预料的错误。这里应当要么通过 panic 退出程序，要么停止后面的运算。

### 函数式异常处理

在函数式语言里面，异常带来了两个问题

- 异常破坏了引用透明并引入了上下文依赖，引用透明表示表达式可以被它引用的值替代。但是如果是一个有抛出异常的方法，在 try 语句块外面会抛出异常，但是如果替代了 try 语句块里面它引用的值后，就能够做异常处理，两种的返回结果就会不同。

- 异常不是类型安全的，比如我们一个函数的返回值是一个 Integer 类型，但是由于抛出了异常且不做处理，那么程序直接结束了，就没有返回值了。

##### Either类型

而 Either 作为一个重要的函数式语言的数据类型，可以用来处理异常。

先看一下 Either 的声明

``` scala
sealed abstract class Either[+A, +B] extends Product with Serializable {
  def left = Either.LeftProjection(this)
  def right = Either.RightProjection(this)

  def map[B1](f: B => B1): Either[A, B1] = this match {
    case Right(b) => Right(f(b))
    case _        => this.asInstanceOf[Either[A, B1]]
  }

  def toTry(implicit ev: A <:< Throwable): Try[B] = this match {
    case Right(b) => Success(b)
    case Left(a)  => Failure(a)
  }
}
```

Left 和 right 可以看做是 Either 的两个子类，通过模式匹配来判断结果。一般来说，使用 left 表示异常，right 表示返回的结果 (一语双关，right有正确的含义)。如果没有异常，那么将 right 通过函数处理后返回 right。如果有异常，将返回一个 left 的值，表示失败。如果是 left ，后面的函数将不作处理，依然返回这个 left。

Try 和 Either 类似，用于函数式语言的错误处理，实现组合这个抽象类的是 Future

``` scala
import scala.concurrent.ExecutionContext.Implicits.global
def div = (i:Int) => if (i % 2 != 0)  throw new ArithmeticException() else  i / 2
def part: PartialFunction[Try[Int], Unit] = {case f: Try[Int] => println(f)}
def fine: PartialFunction[Try[Int], Unit] = {case f: Try[Int] => println(f + "\nI am fine")}
def reco: PartialFunction[Throwable, Int] = {case f: Throwable => 0}

val producer = Future {12} andThen part map div andThen part map div andThen part map div andThen part map div andThen part recover reco andThen fine
Await.result(producer, Duration.Inf)
//Success(12)
//Success(6)
//Success(3)
//Failure(java.lang.ArithmeticException)	
//Failure(java.lang.ArithmeticException)
//Success(0)
//I am fine

val producer2 = Future {64} andThen part map div andThen part map div andThen part map div andThen part map div andThen part recover reco
Await.result(producer2, Duration.Inf)
//Success(64)
//Success(32)
//Success(16)
//Success(8)
//Success(4)
```

可以看见，返回 Success 和 Failure 两种类型，如果抛出异常，后面返回包装有异常的 Failure ，并 map 不再做处理，recover 函数相当于捕获异常并作处理，java 中  `CompletableFuture` 和这个是类似的。

##### golang try proposal

之前 golang 社区有一个讨论热闹的proposal，添加内建函数 `try`  和 `HandleErrorf` 方法，来解决代码繁琐的问题。

如果写过一段时间的golang代码，可以发现有相当一部分重复代码  `if err != nil `  来处理错误。

``` go
f, err := os.Open(filename)
if err != nil {
  return …, err  // zero values for other results, if any
}
```

通过 内建函数 `try` 语法来处理

``` go
f := try(os.Open(filename))
```

可以发现，这个方法和 Either 的思想非常相似。虽然 golang 是多返回值的，但是一旦有错误，便通过 if 来判断处理，不再执行后面的逻辑，虽然没有模式匹配和类型系统，不过也没有什么问题。这样做还有一个好处是，可以通过链式调用来处理异常，`res := try(try(try(foo()).bar()).baz())` ，emmm，可读性还行吧，比之前的  `if err != nil ` 好一点点。

当然，这个提议存在很多问题。一个是 golang 语言是多返回值的，两个返回值和函数和三个返回值的函数怎么链式调用，显然单返回值的语言就不存在这样的问题。其次，由于 golang 语言没有泛型，无法通过类型推断来判断本函数的返回类型是不是下一个函数的接受类型，虽然 `interface{}` 可以用来当做所有类型的子类型，但是这样无法在编译期发现类型错误，直到运行期才会发现错误，类型不安全。

总而言之，这个提议实现还很困难。

### 异常处理思想

##### Let it crash

函数式语言 erlang 有一个很出名的异常处理思想，一般称作任其崩溃 (Let it crash)。这种思想认为如果程序发生了错误，抛出了异常，应当不去处理。而是暴露出问题，停止处理，因为可能由于处理方法不正确，一般是 `catch` 里面没有正确而完整地处理方式，而后面会将错就错，可能会造成令人迷惑的结果。如果崩溃掉，那么及时修复这个 bug 就可以了。不过你的上级多半会很不高兴，做好挨骂的准备吧 :-) 。另外，还存在一个问题，大多数语言对于结束和启动一个处理线程往往开销很大，大并发的程序遇到问题就会很快崩溃掉。erlang 能够使用的主要原因一个是基于 actor 模型，创建和销毁开销不大，另外是基于函数式语言的思想，希望没有副作用。所以很多情况下这种思想虽然很理想，但并不是很好。

##### 防御性编程

防御性编程指的是通过很多的异常捕捉和异常处理，提高代码的可读性，并预防不可能会发生的错误，减少 bug 和问题。也就是说通过添加异常处理，捕获大多数运行时异常，这些异常往往不经常发生，但是可以减少灾难性的影响，但是这样也会存在问题，因为是自己都不知道可能会发生什么错误，通过一个子类捕获所有可能的异常，并没有区分并分别处理，也没有正确的处理逻辑，可能导致结果不正确或者被隐藏。
