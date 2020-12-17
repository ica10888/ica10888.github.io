---
layout:     post
title:      advanced java （补三）NIO
subtitle:   advanced java （补三）NIO
date:       2020-12-16
author:     ica10888
catalog: true
tags:
    - java
---


# Java NIO

Java 在  jdk1.4 版本加入了  `java.nio` 包，提供了一系列改进的输入/输出处理的新特性，被统称为NIO( 即 New I/O ) 。

### 核心部分

NIO主要有三个核心部分组成：

- Buffer 缓冲区：一般来说，对于操作系统，如网卡的写操作过程如下：1. 将数据从网卡拷贝到系统空间；2.将数据从系统空间拷贝到用户空间（如果使用DirectByteBuffer，可以减少一次系统空间到用户空间的拷贝）。然后程序从缓冲区中读取数据。Buffer 的创建和销毁的成本很高，不宜维护，通常会用内存池来提高性能。
- Channel 管道：表示到实体如硬件设备、文件、网络套接字或可以执行一个或多个不同I/O操作的程序组件的开放的连接。从 Buffer 中读取数据或向 Buffer 中写入数据。
- Selector选择器：Selector 能够检测多个注册的  Channel 上是否有事件发生，如果有事件发生，便获取事件然后针对每个事件进行相应的响应处理。

### FileChannel （文件IO）

首先 磁盘IO 由于磁盘文件（也就是regular file）是没有就绪这过程的，它们随时是就绪的，模型上是这样的，因此不适用同步非阻塞（non-blocking io）。同步非阻塞适用于网络IO但是对硬盘IO不起作用，磁盘只能通过阻塞传输数据。

使用传统的 `java.io` 来读写文件数据，是面向 **数据流**  的输入输出，同时需要 OutputStream 和 InputStream 两种流 （但是实际上，旧的io包已经使用NIO重新实现过，同样具有效率）

``` java
String filePath = "/tmp/test.txt";
File file = new File(filePath);
//输出流写入数据
try {
    OutputStreamWriter writer = new OutputStreamWriter(new FileOutputStream(file));
    BufferedWriter br = new BufferedWriter(writer);
    //写入数据
    writer.write("aaaaaaa\n".toCharArray());
    writer.flush();

    writer.write("bbbbbbb\n".toCharArray());
    writer.flush();

    writer.write("ccccccc\n".toCharArray());
    writer.flush();
    writer.close();

} catch (IOException e) {
    e.printStackTrace();
}

//输入流读取数据
try {
    InputStreamReader reader = new InputStreamReader(new FileInputStream(file));
    BufferedReader br = new BufferedReader(reader);
    //读取数据
    char data[] = new char[8];
    while (br.read(data)  !=  -1){
        System.out.print(data);
    }
} catch (IOException e) {
    e.printStackTrace();
}

```

而如果使用 `java.nio`  ，是面向 **缓冲区** 的输入和输出，只需要一个缓冲就可以同时读写数据

``` java
//缓冲区，需要注意的是 HeapByteBuffer 是堆外内存。如果没有一个良好的设计，可能存在内存泄漏的问题
ByteBuffer bf = ByteBuffer.allocate(8);
try {
    // 通过缓冲写入数据
    FileChannel outFc  = new FileOutputStream("/tmp/test.txt",true).getChannel();
    bf.put("aaaaaaa\n".getBytes());
    bf.flip();
    outFc.write(bf);
    bf.clear();

    bf.put("bbbbbbb\n".getBytes());
    bf.flip();
    outFc.write(bf);
    bf.clear();

    bf.put("ccccccc\n".getBytes());
    bf.flip();
    outFc.write(bf);
    bf.clear();
    outFc.close();

    //通过缓冲读取数据
    FileChannel inFc = new FileInputStream("/tmp/test.txt").getChannel();
    while (inFc.read(bf)  !=  -1){
        System.out.print(new String(bf.array()));
        bf.clear();
    }
    outFc.close();

} catch (IOException e) {
    e.printStackTrace();
}
```

同时 nio 提供了`java.nio.flie` 类 ，来更方便实现文件路径的操作和对于文件的创建、移动、删除、获取信息等操作。

### ServerSocketChannel （网络IO，TCP）

下面是一个模拟HTTP请求，访问 `http://127.0.0.1:18081/`  , 返回 `Hello World` 的例子

``` java
// 创建网络服务端
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式 - 默认都是阻塞的
serverSocketChannel.socket().bind(new InetSocketAddress(18081)); // 绑定端口
System.out.println("启动成功");
while (true) {
    SocketChannel socketChannel = serverSocketChannel.accept(); // 获取新tcp连接通道
    // tcp请求 读取/响应
    if (socketChannel != null) {
        System.out.println("收到新连接 : " + socketChannel.getRemoteAddress());
        socketChannel.configureBlocking(false); // 默认是阻塞的,一定要设置为非阻塞
        try {
            ByteBuffer requestBuffer = ByteBuffer.allocateDirect(1024);
            while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
                // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                if (requestBuffer.position() > 0) {
                    break;
                }
            }
            if (requestBuffer.position() == 0) {
                continue;
            } // 如果没数据了, 则不继续后面的处理

            requestBuffer.flip();
            byte[] content = new byte[requestBuffer.limit()];
            requestBuffer.get(content);
            System.out.println(new String(content));
            System.out.println("收到数据,来自：" + socketChannel.getRemoteAddress());

            // 响应结果 200
            String response = "HTTP/1.1 200 OK\r\n" +
                "Content-Length: 11\r\n\r\n" +
                "Hello World";
            ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
            while (buffer.hasRemaining()) {
                socketChannel.write(buffer);// 非阻塞
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

##### 直接内存

之前提到过网络非阻塞io的 epoll 过程中可以通过用户空间和内核空间的地址映射到同一块物理内存上，减少内核态和用户态切换的开销，这也就是直接内存 `allocateDirect`的作用 。直接内存只适用于网络IO，文件IO如果使用，会抛出 `UnsupportedOperationException  `  的异常。需要注意的是，直接内存属于堆外内存，不受 JVM GC 管理，如果使用了不恰当的处理方法，可能会存在内存泄漏的问题。

### DataGramChannel（UDP）

DatagramChannel 是一个可以发送、接收UDP数据包的通道。通过  `send()` 方法从 `DatagramChannel` 发送数据 

``` java
ByteBuffer buf = ByteBuffer.allocateDirect(1024);
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(38081));
channel.receive(buf);

int byteSent = channel.send(buf, new InetSocketAddress("example.com", 80));
channel.connect(new InetSocketAddress(80));
```

### AsynchronousFileChannel （异步文件通道）



##### 通过Future读取数据

``` java
ByteBuffer buf = ByteBuffer.allocate(1024);
Path path = Paths.get("/tmp/test.txt");

AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path, StandardOpenOption.READ);
Future<Integer> operation = fileChannel.read(buf, 0);
while(!operation.isDone());
buf.flip();
byte[] data = new byte[buf.limit()];
buf.get(data);
System.out.println(new String(data));
buf.clear();
```

##### 通过Future写数据

``` java
ByteBuffer buf = ByteBuffer.allocate(1024);
Path path = Paths.get("/tmp/test.txt");

AsynchronousFileChannel fileChannel = AsynchronousFileChannel.open(path, StandardOpenOption.WRITE);
buf.put("test data\n".getBytes());
buf.flip();

Future<Integer> operation = fileChannel.write(buf, 0);
buf.clear();

while(!operation.isDone());
System.out.println("Write done");
```

同时也可以通过 `CompletionHandler` ，重写方法来读取和写数据



