# 三大组件
## 概览
|Channel|Buffer|Selector|
|---|---|-|
|FileChannel|ByteBuffer|-|
|DatagramChannel|IntBuffer|-|
|SocketChannel|ShortBuffer|-|
|ServerSocketChannel|LongBuffer|-|
|-|DoubleBuffer|-|
|-|FloatBuffer|-|

## 服务器设计
### 多线程版
 ![](assets/Pasted%20image%2020220927221208.png)
多线程版的缺点：

1. 线程多了，CPU调度起来就浪费资源

### 线程池版
![](assets/Pasted%20image%2020220927221306.png)
线程池版缺点：

1. 阻塞式IO，当某个线程对一个socket进行任何操作时，另外的socket只能干等

### Selector版
 ![](assets/Pasted%20image%2020220927221400.png)
Selector版的优点：
1. 非阻塞式IO，线程不会傻傻的等待某一个任务的完成。

## Buffer的使用
### 引入依赖
```xml
<dependencies>  
    <dependency>  
        <groupId>io.netty</groupId>  
        <artifactId>netty-all</artifactId>  
        <version>4.1.75.Final</version>  
    </dependency>  
    <dependency>  
        <groupId>org.projectlombok</groupId>  
        <artifactId>lombok</artifactId>  
        <version>1.18.24</version>  
    </dependency>  
    <dependency>  
        <groupId>com.google.code.gson</groupId>  
        <artifactId>gson</artifactId>  
        <version>2.8.9</version>  
    </dependency>  
    <dependency>  
        <groupId>com.google.guava</groupId>  
        <artifactId>guava</artifactId>  
        <version>19.0</version>  
    </dependency>  
    <dependency>  
        <groupId>ch.qos.logback</groupId>  
        <artifactId>logback-classic</artifactId>  
        <version>1.2.11</version>  
    </dependency>  
</dependencies>
```

### 使用案例
```java
@Slf4j
public class TestByteBuffer {  
    @Test  
    public void test() {  
        try {  
            FileChannel channel = new FileInputStream("data.txt").getChannel();  
            ByteBuffer buf = ByteBuffer.allocate(10);  
            int len = 0;  
            while ((len = channel.read(buf)) != -1) {  
                log.info("读到的字节数：{}",len);  
                while (buf.hasRemaining()) {  
                    // 切换成读模式  
                    buf.flip();  
                    byte b = buf.get();  
                    log.info("读到了:{}",(char) b);  
                }  
                // 切换成写模式  
                buf.clear();  
            }  
        } catch (FileNotFoundException e) {  
            throw new RuntimeException(e);  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```

**读写模式切换原理**
 ![](assets/Pasted%20image%2020220928095643.png)	

# 文件编程



# 网络编程



# NIO

## 多线程优化
### 多线程模型
 ![](assets/Pasted%20image%2020221002135136.png)

## Stream和Channel的联系
- stream不会自动缓冲数据，channel会利用系统提供的发送缓冲区、接收缓冲区（更加底层）
- stream仅支持阻塞API。
- 二者均为全双工通信

## IO模型
### 阻塞IO
 ![](assets/Pasted%20image%2020220930223125.png)	
### 非阻塞IO
 ![](assets/Pasted%20image%2020220930222712.png)
### 多路复用IO
 ![](assets/Pasted%20image%2020220930223305.png)

**同样都阻塞，那么多路复用IO和阻塞IO的区别是什么呢？**
看下面这两张图：

 ![](assets/Pasted%20image%2020220930225156.png)

 ![](assets/Pasted%20image%2020220930230949.png)

<p>他们的区别就是：阻塞IO模型，只能同时有一个操作(ACCPET、READ、WRITE)，其他操作要等待这个操作结束，并且阻塞IO模型需要等待数据和等待连接。IO多路复用模型，在进行一个操作的时候，其他操作也可以同时进行，并且IO多路复用模型只需要等待事件的触发，而数据已经提前准备好了，因此不用等待数据。</p> 



### 同步和异步

同步指的是只有一个线程在完成整个交互的过程。
异步指的是可以有一个线程去调用过程，另一个线程返回结果。

### 同步IO模型
上面提到的几种IO模型都是同步IO模型

### 异步IO模型
 ![](assets/Pasted%20image%2020220930234007.png)


## 零拷贝的进化史

 ![](assets/Pasted%20image%2020220930221630.png)


# Netty入门
## 几个概念的图解
channel:
 ![](assets/Pasted%20image%2020221001165633.png)

pipeline:
 ![](assets/Pasted%20image%2020221001165804.png)

eventloop:
 ![](assets/Pasted%20image%2020221001165855.png)

## channelFuture的连接问题
客户端的主要代码如下
```java
ChannelFuture channelFuture  = new Bootstrap()  
        .group(new NioEventLoopGroup())  
        .channel(NioSocketChannel.class)  
        .handler(new ChannelInitializer<NioSocketChannel>() {  
            @Override  
            protected void initChannel(NioSocketChannel sc) throws Exception {  
                sc.pipeline().addLast(new StringEncoder(Charsets.UTF_8));  
            }  
        })  
        .connect(new InetSocketAddress("localhost", 8080));
        // channelFuture.sync();
        Channel channel = channelFuture.channel();
```
上面代码中的connect()方法是一个异步非阻塞的方法，调用玩这个方法之后，不会马上进行连接，而是马上执行下面的`Channel channel = channelFuture.channel();`代码。如果先执行了``
`Channel channel = channelFuture.channel();`代码，会导致channel没有进行连接。因此，我们还要调用`channelFuture.sync();`这一句代码，来实现同步连接。

当然，还有另外一种方法，异步处理结果，在connect方法后添加如下代码：
```java
channelFuture.addListener(new ChannelFutureListener() {  
    @Override  
    public void operationComplete(ChannelFuture channelFuture) throws Exception {  
        channelFuture.channel();  
    }
});
```

## channelFuture处理关闭
下面这段代码主要想完成的功能是让用户不断输入信息，直到输入"q"才关闭连接，并且处理连接关闭后的操作。
```java
public static void main(String[] args) throws InterruptedException, IOException {  
    Channel channel = new Bootstrap()  
            .group(new NioEventLoopGroup())  
            .channel(NioSocketChannel.class)  
            .handler(new ChannelInitializer<NioSocketChannel>() {  
                @Override  
                protected void initChannel(NioSocketChannel sc) throws Exception {  
                    sc.pipeline().addLast(new StringEncoder(Charsets.UTF_8));  
                }  
            })  
            .connect(new InetSocketAddress("localhost", 8080))  
            .sync()  
            .channel();  
    new Thread(() -> {  
        Scanner sc = new Scanner(System.in);  
        while (true) {  
            String str = sc.next();  
            if (str == "q") {  
                channel.close();  
                break;
			}  
            channel.writeAndFlush(str);  
        }  
    }).start();  
	log.info("处理关闭后的操作。。。");
```
存在问题：
由于`close()`方法是一个异步非阻塞的方法，因此调用完close()方法之后，`log.info("处理关闭后的操作。。。");`可能在连接关闭前就执行了。

解决方法：
方式一
在`start()`方法后续添加如下代码：

```java
/// 关闭channel的方式1  
// 同步处理  
ChannelFuture closeFuture = channel.closeFuture();  
closeFuture.sync();  
log.info("处理关闭后的操作");
```

方式二
在`start()`方法后续添加如下代码：

```java
/// 关闭channel的方式2  
// 异步处理  
closeFuture.addListener(new ChannelFutureListener() {  
    @Override  
    public void operationComplete(ChannelFuture channelFuture) throws Exception {  
        log.info("处理关闭后的操作");  
    }  
});
```

虽然连接是关闭了（用户已经无法继续输入），但是客户端进程依然没有完全关闭。
原因是，NioEventGroup中的线程没有被关闭
修改代码如下：
```java
	// 1. 抽取变量出来
	NioEventLoopGroup group = new NioEventLoopGroup(); 
	// ...省略部分代码	
	closeFuture.addListener(new ChannelFutureListener() {  
	    @Override  
	    public void operationComplete(ChannelFuture channelFuture) throws Exception {  
	        log.info("处理关闭后的操作");  
			// 2. 调用shutdownGracefully()方法
			group.shutdownGracefully();
	    }  
	});
```

## 异步原理
场景一：
一个病人找一个医生完成看病的所有流程（如下图所示）
 ![](assets/Pasted%20image%2020221003112544.png)
特点：多线程。
优点：病人看病的流程少。
缺点：效率不算高，一段时间内，完成看病的人数比较少。

场景2：
将看病的流程细分出去，每一个医生完成单独的一项流程。
 ![](assets/Pasted%20image%2020221003113042.png)
特点：多线程
优点：效率较高，一段时间内，完成看病的人数比较多。
缺点：病人看病的流程多，流程切换比较频繁。

**Netty采用的异步和场景2类似。**

## Jdk的Future和Netty的Future
Jdk的Future案例代码：
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {  
    ExecutorService service = Executors.newFixedThreadPool(2);  
    Future<Integer> future = service.submit(() -> {  
        try {  
            Thread.sleep(1000);  
            return 50;  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
    });  
    System.out.println(future.get());  
}
```
Netty的Future案例代码：
```java
public static void main(String[] args) {  
    NioEventLoopGroup group = new NioEventLoopGroup();  
    EventLoop eventLoop = group.next();  
    Future<Integer> future = eventLoop.submit(() -> {  
        try {  
            Thread.sleep(1000);  
            return 80;  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
    });  
    future.addListener(new GenericFutureListener<Future<? super Integer>>() {  
        @Override  
        public void operationComplete(Future<? super Integer> future) throws Exception {  
            log.debug("接收结果：{}",future.getNow());  
        }  
    });  
}
```

## pipeline出站入站原理
 ![](assets/Pasted%20image%2020221003150929.png)


## ByteBuf
创建
```java
public static void main(String[] args) {  
    ByteBuf buf = ByteBufAllocator.DEFAULT.buffer();  
    System.out.println(buf);  
    StringBuilder sb = new StringBuilder();  
    for (int i = 0; i < 300; i++) {  
        sb.append("a");  
    }  
    buf.writeBytes(sb.toString().getBytes());  
    System.out.println(buf);  
}
```

### 直接内存vs堆内存
可以使用下面的代码来创建池化基于堆的 ByteBuf

```
ByteBuf buffer = ByteBufAllocator.DEFAULT.heapBuffer(10);
```

也可以使用下面的代码来创建池化基于直接内存的 ByteBuf

```
ByteBuf buffer = ByteBufAllocator.DEFAULT.directBuffer(10);
```

-   直接内存创建和销毁的代价昂贵，但读写性能高（少一次内存复制），适合配合池化功能一起用
-   直接内存对 GC 压力小，因为这部分内存不受 JVM 垃圾回收的管理，但也要注意及时主动释放

### 池化vs非池化
池化的最大意义在于可以重用ByteBuf，优点有

-   没有池化，则每次都得创建新的 ByteBuf 实例，这个操作对直接内存代价昂贵，就算是堆内存，也会增加 GC 压力
-   高并发时，池化功能更节约内存，减少内存溢出的可能

池化功能是否开启，可以通过下面的系统环境变量来设置

```
-Dio.netty.allocator.type={unpooled|pooled}
```

-   4.1 以后，非 Android 平台默认启用池化实现，Android 平台启用非池化实现
-   4.1 之前，池化功能还不成熟，默认是非池化实现

### 组成
 ![](assets/Pasted%20image%2020221003153213.png)

### 写入
方法列表，省略一些不重要的方法
|方法签名|含义|备注|
|--|--|------|
|writeBoolean(boolean value)|写入 boolean 值|用一字节 01|00 代表 true|false|
|writeInt(int value)|写入 int 值|Big Endian，即 0x250，写入后 00 00 02 50|
|writeBytes(ByteBuffer src)|写入 nio 的 ByteBuffer|| 
|int writeCharSequence(CharSequence sequence, Charset charset)|写入字符串||

先写入 4 个字节

```java
buffer.writeBytes(new byte[]{1, 2, 3, 4});
log(buffer);
```

结果是

```
read index:0 write index:4 capacity:10
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04                                     |....            |
+--------+-------------------------------------------------+----------------+
```

再写入一个 int 整数，也是 4 个字节

```java
buffer.writeInt(5);
log(buffer);
```

结果是

```
read index:0 write index:8 capacity:10
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 00 00 00 05                         |........        |
+--------+-------------------------------------------------+----------------+
```

还有一类方法是 set 开头的一系列方法，也可以写入数据，但不会改变写指针位置

### 扩容
再写入一个 int 整数时，容量不够了（初始容量指定了是 10），这时会引发扩容

```java
buffer.writeInt(6);
log(buffer);
```

扩容规则是

-   如何写入后数据大小未超过 512，则选择下一个 16 的整数倍，例如写入后大小为 12 ，则扩容后 capacity 是 16
-   如果写入后数据大小超过 512，则选择下一个 2^n，例如写入后大小为 513，则扩容后 capacity 是 2^10=1024（2^9=512 已经不够了）
-   扩容不能超过 max capacity 会报错

结果是

```
read index:0 write index:12 capacity:16
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 00 00 00 05 00 00 00 06             |............    |
+--------+-------------------------------------------------+----------------+
```

### 读取
例如读了 4 次，每次一个字节

```
System.out.println(buffer.readByte());
System.out.println(buffer.readByte());
System.out.println(buffer.readByte());
System.out.println(buffer.readByte());
log(buffer);
```

读过的内容，就属于废弃部分了，再读只能读那些尚未读取的部分

```
1
2
3
4
read index:4 write index:12 capacity:16
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 00 00 00 05 00 00 00 06                         |........        |
+--------+-------------------------------------------------+----------------+
```

如果需要重复读取 int 整数 5，怎么办？

可以在 read 前先做个标记 mark

```
buffer.markReaderIndex();
System.out.println(buffer.readInt());
log(buffer);
```

结果

```
5
read index:8 write index:12 capacity:16
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 00 00 00 06                                     |....            |
+--------+-------------------------------------------------+----------------+
```

这时要重复读取的话，重置到标记位置 reset

```
buffer.resetReaderIndex();
log(buffer);
```

这时

```
read index:4 write index:12 capacity:16
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 00 00 00 05 00 00 00 06                         |........        |
+--------+-------------------------------------------------+----------------+
```

还有种办法是采用 get 开头的一系列方法，这些方法不会改变 read index

### 内存释放
由于 Netty 中有堆外内存的 ByteBuf 实现，堆外内存最好是手动来释放，而不是等 GC 垃圾回收。

-   UnpooledHeapByteBuf 使用的是 JVM 内存，只需等 GC 回收内存即可
-   UnpooledDirectByteBuf 使用的就是直接内存了，需要特殊的方法来回收内存
-   PooledByteBuf 和它的子类使用了池化机制，需要更复杂的规则来回收内存

> 回收内存的源码实现，请关注下面方法的不同实现
> 
> `protected abstract void deallocate()`

Netty 这里采用了引用计数法来控制回收内存，每个 ByteBuf 都实现了 ReferenceCounted 接口

-   每个 ByteBuf 对象的初始计数为 1
-   调用 release 方法计数减 1，如果计数为 0，ByteBuf 内存被回收
-   调用 retain 方法计数加 1，表示调用者没用完之前，其它 handler 即使调用了 release 也不会造成回收
-   当计数为 0 时，底层内存会被回收，这时即使 ByteBuf 对象还在，其各个方法均无法正常使用

谁来负责 release 呢？

不是我们想象的（一般情况下）

```
ByteBuf buf = ...
try {
    ...
} finally {
    buf.release();
}
```

请思考，因为 pipeline 的存在，一般需要将 ByteBuf 传递给下一个 ChannelHandler，如果在 finally 中 release 了，就失去了传递性（当然，如果在这个 ChannelHandler 内这个 ByteBuf 已完成了它的使命，那么便无须再传递）
还有一个注意点：虽然说head和tail两个handler在最后会检查并释放存在的Bytebuf，但是吗，这个Bytebuf传到tail的时候，可能已经不是原来的那个ByteBuf了。那么就可能导致没有真正释放掉。

基本规则是，**谁是最后使用者，谁负责 release**

Tail释放未处理消息逻辑
```java
// io.netty.channel.DefaultChannelPipeline#onUnhandledInboundMessage(java.lang.Object)
protected void onUnhandledInboundMessage(Object msg) {
    try {
        logger.debug(
            "Discarded inbound message {} that reached at the tail of the pipeline. " +
            "Please check your pipeline configuration.", msg);
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

具体代码
```java
// io.netty.util.ReferenceCountUtil#release(java.lang.Object)
public static boolean release(Object msg) {
    if (msg instanceof ReferenceCounted) {
        return ((ReferenceCounted) msg).release();
    }
    return false;
}
```

### slice
【零拷贝】的体现之一，对原始 ByteBuf 进行切片成多个 ByteBuf，切片后的 ByteBuf 并没有发生内存复制，还是使用原始 ByteBuf 的内存，切片后的 ByteBuf 维护独立的 read，write 指针
 ![](assets/Pasted%20image%2020221003161801.png)

案例一：
```java
public static void main(String[] args) {  
    ByteBuf buf = ByteBufAllocator.DEFAULT.buffer(10);  
    buf.writeBytes(new byte[]{1,2,3,4,5,'a','b','c','d','e'});  
    log(buf);  
    ByteBuf buf1 = buf.slice(0, 5);  
    ByteBuf buf2 = buf.slice(5, 5);  
    buf1.setByte(0,6);  
    buf2.setByte(0,'f');  
    log(buf);  
    log(buf1);  
    log(buf2);  
}
```

结果：
```
read index:0 write index:10 capacity:10
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 05 61 62 63 64 65                   |.....abcde      |
+--------+-------------------------------------------------+----------------+
read index:0 write index:10 capacity:10
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 06 02 03 04 05 66 62 63 64 65                   |.....fbcde      |
+--------+-------------------------------------------------+----------------+
read index:0 write index:5 capacity:5
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 06 02 03 04 05                                  |.....           |
+--------+-------------------------------------------------+----------------+
read index:0 write index:5 capacity:5
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 66 62 63 64 65                                  |fbcde           |
+--------+-------------------------------------------------+----------------+

Process finished with exit code 0

```

案例二
原始ByteBuf为：
```
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 02 03 04                                        |...             |
+--------+-------------------------------------------------+----------------+
```
如果原始 ByteBuf 再次读操作（又读了一个字节）

```java
origin.readByte();
```

此时ByteBuf为：

```
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 03 04                                           |..              |
+--------+-------------------------------------------------+----------------+
```

因为这时的 slice 不受影响，因为它有独立的读写指针

**最后一点**
在实际编程的时候，要在原来的Bytebuf对象调用完slice()后（假设返回的对象名是buf1)，紧接着调用buf1的retain()方法，最终在代码的最后面调用buf1的release方法。

### duplicate

【零拷贝】的体现之一，就好比截取了原始 ByteBuf 所有内容，并且没有 max capacity 的限制，也是与原始 ByteBuf 使用同一块底层内存，只是读写指针是独立的

 ![](https://bright-boy.gitee.io/technical-notes/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/img/0012.png)

### copy
会将底层内存数据进行深拷贝，因此无论读写，都与原始 ByteBuf 无关


### CompositeByteBuf

【零拷贝】的体现之一，可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf，避免拷贝

有两个 ByteBuf 如下

```
ByteBuf buf1 = ByteBufAllocator.DEFAULT.buffer(5);
buf1.writeBytes(new byte[]{1, 2, 3, 4, 5});
ByteBuf buf2 = ByteBufAllocator.DEFAULT.buffer(5);
buf2.writeBytes(new byte[]{6, 7, 8, 9, 10});
System.out.println(ByteBufUtil.prettyHexDump(buf1));
System.out.println(ByteBufUtil.prettyHexDump(buf2));
```

输出

```
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 05                                  |.....           |
+--------+-------------------------------------------------+----------------+
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 06 07 08 09 0a                                  |.....           |
+--------+-------------------------------------------------+----------------+
```

现在需要一个新的 ByteBuf，内容来自于刚才的 buf1 和 buf2，如何实现？

方法1：

```
ByteBuf buf3 = ByteBufAllocator.DEFAULT
    .buffer(buf1.readableBytes()+buf2.readableBytes());
buf3.writeBytes(buf1);
buf3.writeBytes(buf2);
System.out.println(ByteBufUtil.prettyHexDump(buf3));
```

结果

```
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 05 06 07 08 09 0a                   |..........      |
+--------+-------------------------------------------------+----------------+
```

这种方法好不好？回答是不太好，因为进行了数据的内存复制操作

方法2：

```
CompositeByteBuf buf3 = ByteBufAllocator.DEFAULT.compositeBuffer();
// true 表示增加新的 ByteBuf 自动递增 write index, 否则 write index 会始终为 0
buf3.addComponents(true, buf1, buf2);
```

结果是一样的

```
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 05 06 07 08 09 0a                   |..........      |
+--------+-------------------------------------------------+----------------+
```

CompositeByteBuf 是一个组合的 ByteBuf，它内部维护了一个 Component 数组，每个 Component 管理一个 ByteBuf，记录了这个 ByteBuf 相对于整体偏移量等信息，代表着整体中某一段的数据。

-   优点，对外是一个虚拟视图，组合这些 ByteBuf 不会产生内存复制
-   缺点，复杂了很多，多次操作会带来性能的损耗

### Unpooled

Unpooled 是一个工具类，类如其名，提供了非池化的 ByteBuf 创建、组合、复制等操作

这里仅介绍其跟【零拷贝】相关的 wrappedBuffer 方法，可以用来包装 ByteBuf

```
ByteBuf buf1 = ByteBufAllocator.DEFAULT.buffer(5);
buf1.writeBytes(new byte[]{1, 2, 3, 4, 5});
ByteBuf buf2 = ByteBufAllocator.DEFAULT.buffer(5);
buf2.writeBytes(new byte[]{6, 7, 8, 9, 10});

// 当包装 ByteBuf 个数超过一个时, 底层使用了 CompositeByteBuf
ByteBuf buf3 = Unpooled.wrappedBuffer(buf1, buf2);
System.out.println(ByteBufUtil.prettyHexDump(buf3));
```

输出

```
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 05 06 07 08 09 0a                   |..........      |
+--------+-------------------------------------------------+----------------+
```

也可以用来包装普通字节数组，底层也不会有拷贝操作

```
ByteBuf buf4 = Unpooled.wrappedBuffer(new byte[]{1, 2, 3}, new byte[]{4, 5, 6});
System.out.println(buf4.getClass());
System.out.println(ByteBufUtil.prettyHexDump(buf4));
```

输出

```
class io.netty.buffer.CompositeByteBuf
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 01 02 03 04 05 06                               |......          |
+--------+-------------------------------------------------+----------------+
```


### 💡 ByteBuf 优势
-   池化 - 可以重用池中 ByteBuf 实例，更节约内存，减少内存溢出的可能
-   读写指针分离，不需要像 ByteBuffer 一样切换读写模式
-   可以自动扩容
-   支持链式调用，使用更流畅
-   很多地方体现零拷贝，例如 slice、duplicate、CompositeByteBuf
