# Netty

## 1 概述

Netty是一个基于异步、事件驱动的网络应用框架，快速开发高性能、高可靠性的网络IO程序

Netty主要针对在TCP协议下，面向Clients端的高并发应用，或Peer-to-Peer场景下的大量数据持续传输的应用

Netty本质是一个NIO框架，适用于服务器通讯相关的多种场景

### 1.1 Netty构成

![image-20201208101624768](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208101624768.png)

## 2 IO模型

### 2.1 IO模型概述

`IO模型`就是用什么样的通道进行数据的发送和接收，很大程度决定程序通信的性能

常用网络编程IO模式：BIO、AIO、NIO

- BIO(传统阻塞型)

  服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销

  ![image-20201208104249658](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208104249658.png)

- NIO(同步非阻塞)

  服务器实现模式为一个线程处理多个请求(连接)，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求就进行处理 

- AIO(NIO.2)(异步非阻塞)

  AIO 引入异步通道的概念，采用了 Proactor 模式；有效的请求才启动线程，特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用

### 2.2 BIO

#### 2.2.1 BIO简单流程

1. 服务器端启动一个ServerSocket
2. 客户端启动Socket对服务器进行通信，默认情况下服务器端需要对每个客户 建立一个线程与之通讯
3. 客户端发出请求后, 先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝
4. 如果有响应，客户端线程会等待请求结束后，再继续执行

#### 2.2.2 BIO应用实例

> 说明：使用`线程池`机制进行改善，实现多个客户端连接服务器

##### 2.2.2.1 创建服务器

```java
public class Server {
    private static Logger LOGGER = LoggerFactory.getLogger(Server.class);

    public static void main(String[] args) throws IOException {
        // 创建线程池
        ExecutorService pool = Executors.newCachedThreadPool();
        // 创建ServerSocket
        ServerSocket serverSocket = new ServerSocket(6666);

        LOGGER.info("服务器已启动");

        while (true) {
            final Socket socket = serverSocket.accept();
            LOGGER.info("连接到一个客户端");

            // 创建线程与之通信
            pool.execute(() -> handler(socket));
        }
    }

    // 用于和客户端通信
    public static void handler(Socket socket) {
        // 用于数据流的存储
        byte[] bytes = new byte[1024];
        // 通过Socket获取输入流
        try {
            InputStream is = socket.getInputStream();
            // 循环读取客户端发送的数据
            while (true) {
                int read = is.read(bytes);
                if (read != -1) {
                    // 输出客户端发送的数据
                    LOGGER.info(new String(bytes,0,read));
                } else {
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            LOGGER.info("关闭Socket连接");
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```reStructuredText
11:39:34.627 [main] INFO org.jiang.Server - 服务器已启动
```

从打印出的日志可以看出服务器已经在监听6666端口

##### 2.2.2.2 客户端连接服务器

> `telnet`指令就是查看某个端口是否可访问，Telnet协议是[TCP/IP协议](https://baike.baidu.com/item/TCP%2FIP协议)族中的一员，是Internet远程登录服务的标准协议和主要方式

通过终端连接服务器

```bash
telnet 127.0.0.1 6666
```

服务器打印日志

```reStructuredText
11:47:14.590 [main] INFO org.jiang.Server - 连接到一个客户端
```

##### 2.2.2.2 客户端发送数据

客户端已连接到服务器，终端通过快捷键`Ctrl+]`来发送数据

![image-20201208115255194](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208115255194.png)

服务端已经接受到对应的数据

```reStructuredText
11:49:29.447 [pool-1-thread-2] INFO org.jiang.Server - hello
```

##### 2.2.2.3 新的客户端连接并发送数据

客户端连接并发送

```reStructuredText
欢迎使用 Microsoft Telnet Client

Escape 字符为 'CTRL+]'


Microsoft Telnet> send ok
发送字符串 ok
Microsoft Telnet>
```

服务器接收到的数据

```reStructuredText
11:54:04.132 [main] INFO org.jiang.Server - 连接到一个客户端
11:54:15.669 [pool-1-thread-3] INFO org.jiang.Server - ok
```

##### 2.2.2.4 测试BIO模型阻塞

修改源代码进行测试

![image-20201208120551159](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208120551159.png)

![image-20201208120622300](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208120622300.png)

> 当服务器启动时，日志打印
>
> 12:03:37.134 [main] INFO org.jiang.Server - 等待连接...
>
> 当创建连接到服务器时，日志打印
>
> 12:46:39.822 [pool-1-thread-1] INFO org.jiang.Server - 等待读取数据...

#### 2.2.3 总结

从不同的客户端连接服务器时可以看出不同的连接对应的线程也不同，说明一个连接请求对应一个线程

### 2.3 NIO

#### 2.3.1 概述

核心三大组件：**Channel(**通道**)**，**Buffer(**缓冲区**)**, **Selector(****选择器**)** 

NIO面向**缓冲区，或者面向块**编程，数据读取到一个稍后处理的缓冲区，需要时可在缓冲区中前后移动，可以提供非阻塞式的高伸缩性网络

NIO使一个线程从某通道发送请求或者读取数据，但是仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而**不是保持线程阻塞**，所以直至数据可以读取之前，该线程可以继续进行其它操作

NIO可以做到用一个线程来处理多个操作，假设进入10000个请求，可以分配50或者100个线程来处理，而不像之前的阻塞IO那样必须分配10000个

HTTP2.0使用了多路复用的技术，做到同一个连接并发处理多个请求，而且并发请求的数量比HTTP1.1大了好几个数量级

##### 2.3.1.1 NIO模型

![image-20201208143155983](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208143155983.png)

> 所有的读写操作都是通过Buffer进行操作，Buffer存储部分数据，Channel读取Buffer中的数据

##### 2.3.1.2 NIO和BIO对比

- BIO 以流的方式处理数据,而 NIO 以块的方式处理数据,块 I/O 的效率比流 I/O 高出很多
- BIO 是阻塞的，NIO 则是非阻塞的
- BIO基于字节流和字符流进行操作，而 NIO 基于 Channel(通道)和 Buffer(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中；Selector(选择器)用于监听多个通道的事件(比如：连接请求，数据到达等)，因此使用**单个线程就可以监听多个客户端**通道

##### 2.3.1.3 NIO三大组件关系

- 每个Channel都会对应一个Buffer
- 每个Selector都会对应一个线程，一个线程对应多个Channel(连接)
- 多个Channel注册到Selector
- 程序切换到哪个Channel是由`事件`决定，Selector会根据不同的事件在各个通道切换
- Buffer本质就是一个内存块，底层就是一个数组，数据的读取或写入都是通过Buffer操作
- BIO中流要么是输入流，要么是输出流，不能进行双向操作；但是Buffer可以进行双向读写操作，使用filp方法切换
- Channel同样也是双向的，可以返回底层操作系统的情况

#### 2.3.2 Buffer使用案例

```java
public class BasicBuffer {
    private static Logger logger = LoggerFactory.getLogger(BasicBuffer.class);

    public static void main(String[] args) {
        // 创建一个Buffer，大小为5，存放int类型
        IntBuffer intBuffer = IntBuffer.allocate(5);

        // 向Buffer中存放数据
        intBuffer.put(new int[]{10,11,12,13,14});
        // 从Buffer中存储数据
        // Buffer读写切换
        intBuffer.flip();

        while (intBuffer.hasRemaining()) {
            logger.info(String.valueOf(intBuffer.get()));
        }
    }
}
```

输出结果

```reStructuredText
14:53:40.581 [main] INFO org.jiang.BasicBuffer - 10
14:53:40.583 [main] INFO org.jiang.BasicBuffer - 11
14:53:40.583 [main] INFO org.jiang.BasicBuffer - 12
14:53:40.583 [main] INFO org.jiang.BasicBuffer - 13
14:53:40.583 [main] INFO org.jiang.BasicBuffer - 14
```

#### 2.3.3 Buffer

##### 2.3.3.1 概述

缓冲区本质上是一个可以读写数据的内存块，可以理解成是一个**容器对象(含数组)**，该对象提供了**一组方法**可以更轻松地使用内存块，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况；Channel提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由Buffer

![image-20201208151126199](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208151126199.png)

##### 2.3.3.2 继承树

![image-20201208151543101](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208151543101.png)

##### 2.3.3.3 Buffer父类关键属性

```java
// Buffer父类具有的属性
// Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;
// 各个子类具有的属性(IntBuffer就是int数组)
final int[] hb;                  // Non-null only for heap buffers
```

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| capacity | 容量，即可以容纳的最大数据量；在缓冲区创建时被设定并且不能改变 |
| Limit    | 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作，且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变改值，为下次读写作准备 |
| Mark     | 标记                                                         |

> 对于`IntBuffer intBuffer = IntBuffer.allocate(5)`

hb也就是大小为5的数组，position为0，capacity为5，limit为5(不能对缓冲区超过极限位置limit进行读写操作)

> 对于`intBuffer.put(new int[]{10,11,12,13});`

![image-20201208153345475](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208153345475.png)

可以看出position为4，也就是存储了4个int数据，而capacity和limit依然为5

> 对于`intBuffer.flip()`

![image-20201208153501235](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208153501235.png)

可以看出当切换读写操作时，position会重新变为0，而limit就会改变为原来position的数据也就是4(本质上就是存储了多少数据)，而capacity不变

##### 2.3.3.4 常用方法

```java
public final int capacity( )//返回此缓冲区的容量
public final int position( )//返回此缓冲区的位置
public final Buffer position (int newPositio)//设置此缓冲区的位置
public final int limit( )//返回此缓冲区的限制
public final Buffer limit (int newLimit)//设置此缓冲区的限制
public final Buffer clear( )//清除此缓冲区, 即将各个标记恢复到初始状态，但是数据并没有真正擦除, 后面操作会覆盖
public final Buffer flip( )//反转此缓冲区
public final boolean hasRemaining( )//告知在当前位置和限制之间是否有元素
public abstract boolean isReadOnly( );//告知此缓冲区是否为只读缓冲区
public abstract boolean hasArray();//告知此缓冲区是否具有可访问的底层实现数组
public abstract Object array();//返回此缓冲区的底层实现数组
```

##### 2.3.3.5 ByteBuffer(最常用的Buffer)

```java
public abstract class ByteBuffer {
    //缓冲区创建相关api
    public static ByteBuffer allocateDirect(int capacity)//创建直接缓冲区
    public static ByteBuffer allocate(int capacity)//设置缓冲区的初始容量
    public abstract byte get( );//从当前位置position上get，get之后，position会自动+1
    public abstract byte get (int index);//从绝对位置get
    public abstract ByteBuffer put (byte b);//从当前位置上添加，put之后，position会自动+1
    public abstract ByteBuffer put (int index, byte b);//从绝对位置上put
 }
```

#### 2.3.3 Channel

##### 2.3.3.1 概述

通道可以同时进行读写，而流只能读或者只能写

通道可以实现异步读写数据

通道可以从缓冲读数据，也可以写数据到缓冲

![image-20201208155145069](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201208155145069.png)

常用的Channel子类：FileChannel、DatagramChannel、ServerSocketChannel 和 SocketChannel(**ServerSocketChanne 类似 ServerSocket , SocketChannel 类似 Socket**)

FileChannel 用于文件的数据读写，DatagramChannel 用于 UDP 的数据读写，ServerSocketChannel 和 SocketChannel 用于 TCP 的数据读写

##### 2.3.3.2 FileChannel

主要对本地文件进行IO操作

###### 常见方法

```java
public int read(ByteBuffer dst) ，从通道读取数据并放到缓冲区中
public int write(ByteBuffer src) ，把缓冲区的数据写到通道中
public long transferFrom(ReadableByteChannel src, long position, long count)，从目标通道中复制数据到当前通道
public long transferTo(long position, long count, WritableByteChannel target)，把数据从当前通道复制给目标通道
```

###### FileChannel写入数据到本地文件

```java
public class NioFileChannel {
    public static void main(String[] args) throws Exception {
        String str = "hello,test";
        // 创建输出流
        FileOutputStream fos = new FileOutputStream(".\\test.txt");
        // 通过输出流获取对应的FileChannel
        // FileChannel真实类型为FileChannelImpl
        FileChannel fileChannel = fos.getChannel();
        // 创建一个缓冲区ByteBuffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        // 将str放入到ByteBuffer中
        buffer.put(str.getBytes());
        // 反转ByteBuffer
        buffer.flip();
        // 将ByteBuffer中的数据写入到FileChannel中
        fileChannel.write(buffer);
        // 关闭流
        fileChannel.close();
    }
}
```

`buffer.put`将需要写入的内容放入缓冲区中，进而由Channel读取缓冲区中的内容，故缓冲区需要进行翻转，注意此时的`limit`会发生改变

![image-20201214154806763](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201214154806763.png)

> debug时可以看出当执行到`FileChannel fileChannel = fos.getChannel()`时输出流中存在一个channel也就是`fileChannel`，说明`fileChannel`是在`FileOutputStream`内部

![image-20201214155317003](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201214155317003.png)

###### FileChannel读取文件中的数据

```java
public class NioFileChannel02 {
    public static void main(String[] args) throws Exception{
        // 创建文件对象
        File file = new File(".\\test.txt");
        // 创建文件输入流
        FileInputStream fis = new FileInputStream(file);
        // 通过输入流获取对应的channel
        FileChannel fisChannel = fis.getChannel();
        // 创建缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(((int) file.length()));
        // 将通道中的数据读取到缓冲区
        fisChannel.read(buffer);
        // 将byteBuffer中的数据转换为字符串输出
        System.out.println(new String(buffer.array()));

        fisChannel.close();
    }
}
```

![image-20201214161205643](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201214161205643.png)

###### 使用一个Buffer完成文件的读写

```java
public class NioFileChannel03 {
    public static void main(String[] args) throws Exception{
        // 创建文件对象
        File file = new File(".\\test.txt");
        // 创建文件输入流
        FileInputStream fis = new FileInputStream(file);
        // 通过输入流获取对应的channel
        FileChannel fisChannel = fis.getChannel();
        // 创建文件输出流
        FileOutputStream fos = new FileOutputStream(".//test02.txt");
        // 获取输出流对应的channel
        FileChannel fosChannel = fos.getChannel();
        // 创建缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(5);
        // 循环读取数据
        while (true) {
            // 重要：清空buffer，重置limit和position
            buffer.clear();

            int temp = fisChannel.read(buffer);
            if (temp == -1) { // 读取完毕
                break;
            }
            // 将buffer中的数据写入到fosChannel中
            buffer.flip();
            fosChannel.write(buffer);
        }
        // 关闭channel
        fisChannel.close();
        fosChannel.close();
    }
}
```

> 重点是clear方法，清空buffer，对于clear方法

```java
public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
```

可以看出重置了position，并且重新设置limit为capacity

> 为什么必须调用clear方法清空？

如果不清空，一直执行到即将读取结束时，调用`fosChannel.write(buffer)`写入数据，此时limit和position相等，由于limit限制了读写的位置，因此返回的`int read`一定为0，并且一直循环为0(**因为在读取时position会不断变化，而即将读取完成时变化到的位置刚好等于limit**)

![image-20201214164118842](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201214164118842.png)

###### Channel拷贝文件

```java
public class NioFileChannel04 {
    public static void main(String[] args) throws Exception{
        // 创建文件对象
        File file = new File(".\\test.txt");
        // 创建文件输入流
        FileInputStream fis = new FileInputStream(file);
        // 通过输入流获取对应的channel
        FileChannel fisChannel = fis.getChannel();
        // 创建文件输出流
        FileOutputStream fos = new FileOutputStream(".//test03.txt");
        // 获取输出流对应的channel
        FileChannel fosChannel = fos.getChannel();
        // 使用transferFrom进行拷贝
        fosChannel.transferFrom(fisChannel,0,fisChannel.size());
    }
}
```

###### Buffer和Channel的注意事项

- ByteBuffer 支持类型化的put 和 get, put 放入的是什么数据类型，get就应该使用相应的数据类型来取出，否则可能有 BufferUnderflowException 异常

  ```java
  public class NioFileChannel05 {
      public static void main(String[] args) throws Exception{
          ByteBuffer buffer = ByteBuffer.allocate(5);
          buffer.putInt(100);
          buffer.getChar();
      }
  }
  ```

  ```reStructuredText
  Exception in thread "main" java.nio.BufferUnderflowException
  	at java.nio.Buffer.nextGetIndex(Buffer.java:509)
  	at java.nio.HeapByteBuffer.getChar(HeapByteBuffer.java:262)
  	at org.jiang.fileChannel.NioFileChannel05.main(NioFileChannel05.java:13)
  ```

  > ByteBuffer针对不同类型的数据有对应的put和get，必须匹配

- 可以将一个普通Buffer 转成只读Buffer

  ```java
  public class NioFileChannel05 {
      public static void main(String[] args) throws Exception{
          ByteBuffer buffer = ByteBuffer.allocate(5);
          buffer.putInt(100);
          buffer.flip();
          ByteBuffer byteBuffer = buffer.asReadOnlyBuffer();
      }
  }
  ```

  > 调用asReadOnlyBuffer方法转换为只读Buffer，该Buffer无法再进行put操作

- NIO 提供了 MappedByteBuffer 可以让文件直接在内存（堆外的内存）中进行修改， 而如何同步到文件由NIO 来完成

  ```java
  public class NioFileChannel06 {
      public static void main(String[] args) throws Exception{
          RandomAccessFile randomAccessFile = new RandomAccessFile(".\\test.txt", "rw");
          // 获取对应的通道
          FileChannel channel = randomAccessFile.getChannel();
          /*
            第一个参数代表使用的模式
            第二个参数表示可以修改的起始位置
            第三个参数表示映射到内存的字节大小
           */
          MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE, 0,5);
  
          mappedByteBuffer.put(3, ((byte) 'H'));
          mappedByteBuffer.put(1, ((byte) 9));
  
          randomAccessFile.close();
      }
  }
  ```

- Buffer的分散和聚合(Scattering和Gathering)

  NIO 支持通过多个Buffer (即 Buffer 数组) 完成读写操作
  
  > Buffer本质上就是字节数组，数组需要连续内存，在JVM的堆内存中开辟一块完整的空间供数组使用开销很大，因为内存一般都是**碎片化**的，故使用Buffer数组
  
  ```java
  public class ScatteringAndGathering {
      public static void main(String[] args) throws Exception{
          ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
          InetSocketAddress address = new InetSocketAddress(7000);
          // 绑定端口到socket并启动
          serverSocketChannel.socket().bind(address);
          // 创建Buffer数组
          ByteBuffer[] byteBuffers = new ByteBuffer[2];
          byteBuffers[0] = ByteBuffer.allocate(5);
          byteBuffers[1] = ByteBuffer.allocate(3);
          // 等待客户端连接
          SocketChannel socketChannel = serverSocketChannel.accept();
          // 假定从客户端接收8个字节
          int messageLength = 8;
          // 循环读取
          while (true) {
              int byteRead = 0;
  
              while (byteRead <= messageLength) {
                  long read = socketChannel.read(byteBuffers);
                  // 累计读取的字节数
                  byteRead += read;
                  System.out.println("byteRead = " + byteRead);
                  // 查看当前buffer的position和limit
                  Arrays.asList(byteBuffers).stream().map(buffer -> "position=" + buffer.position() + ",limit=" + buffer.limit()).forEach(System.out::println);
  
              }
              // 将所有的buffer进行反转
              Arrays.asList(byteBuffers).stream().forEach(Buffer::flip);
              // 将数据读出显示到服务端
              long byteWrite = 0;
              while (byteWrite <= messageLength) {
                  long write = socketChannel.write(byteBuffers);
                  byteWrite += write;
              }
              // 将所有的buffer进行clear
              Arrays.asList(byteBuffers).stream().forEach(Buffer::clear);
              System.out.println("byteWrite = " + byteWrite + ",messageLength=" + messageLength);
          }
      }
  }
  ```

#### 2.3.4 Selector

Selector可以实现使用一个线程处理多个的客户端连接

Selector能够检测多个注册的通道上是否有事件发生(注意：多个Channel以事件的方式可以注册到同一个Selector)

如果有事件发生，便获取事件然后针对每个事件进行相应的处理；这样就可以只用一个单线程去管理多个通道，也就是管理多个连接和请求

只有在连接/通道真正有读写事件发生时，才会进行读写，就大大地减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程

避免了多线程之间的上下文切换导致的开销

##### 2.3.4.1 特点

- Netty 的 IO 线程 NioEventLoop 聚合了 Selector(选择器，也叫多路复用器)，可以同时并发处理成百上千个客户端连接
- 当线程从某客户端 Socket 通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务
- 线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道
- 由于读写操作都是非阻塞的，这就可以充分提升 IO 线程的运行效率，避免由于频繁 I/O 阻塞导致的线程挂起
- 一个 I/O 线程可以并发处理 N 个客户端连接和读写操作，这从根本上解决了传统同步阻塞 I/O 一连接一线程模型

##### 2.3.4.2 Selector API

```java
public abstract class Selector implements Closeable { 
//得到一个选择器对象
public static Selector open();
//监控所有注册的通道，当其中有IO操作可以进行时，将对应的SelectionKey加入到内部集合中并返回，参数用来设置超时时间
public int select(long timeout);
//从内部集合中得到所有的 SelectionKey
public Set<SelectionKey> selectedKeys();	
}
```

相关方法说明

```java
selector.select()//阻塞
selector.select(1000);//阻塞1000毫秒，在1000毫秒后返回
selector.wakeup();//唤醒selector
selector.selectNow();//不阻塞，立马返还
```

##### 2.3.4.3 SelectorKey在NIO非阻塞网络编程中的作用

![NIO](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/NIO.png)

1. 当客户端连接时，会通过ServerSocketChannel 得到 SocketChannel

2. 将SocketChannel注册到Selector上，一个Selector上可以注册多个SocketChannel

   ```java
   public abstract class AbstractSelectableChannel extends SelectableChannel
   {
   	 public final SelectionKey register(Selector sel, int ops,
                                          Object att)
           throws ClosedChannelException
       {
           synchronized (regLock) {
               if (!isOpen())
                   throw new ClosedChannelException();
               if ((ops & ~validOps()) != 0)
                   throw new IllegalArgumentException();
               if (blocking)
                   throw new IllegalBlockingModeException();
               SelectionKey k = findKey(sel);
               if (k != null) {
                   k.interestOps(ops);
                   k.attach(att);
               }
               if (k == null) {
                   // New registration
                   synchronized (keyLock) {
                       if (!isOpen())
                           throw new ClosedChannelException();
                       k = ((AbstractSelector)sel).register(this, ops, att);
                       addKey(k);
                   }
               }
               return k;
           }
       }
   }
   ```

   > SocketChannel的父类AbstractSelectableChannel通过register方法将SocketChannel注册到Seletor上，`int opt`代表事件的类型，返回值为对应的`SelectionKey`对象

3. 注册后返回一个 SelectionKey, 会和该Selector 关联(集合)

4. Selector通过select方法进行监听，返回有事件发生的通道个数

5. 进一步可以通过Selector得到**有事件发生**的SelectionKey

   ```java
   public abstract class SelectorImpl extends AbstractSelector {
   	public Set<SelectionKey> selectedKeys() {
           if (!this.isOpen() && !Util.atBugLevel("1.4")) {
               throw new ClosedSelectorException();
           } else {
               return this.publicSelectedKeys;
           }
       }
   }
   ```

6. 再通过SelectionKey反向获取到对应的SocketChannel

   ```java
   public abstract class SelectionKey {
   	public abstract SelectableChannel channel();
   }
   ```

   > 看出可以通过`channel方法`获取到对应的SocketChannel

##### 2.3.4.4 NIO实现客户端和服务端通讯

- 服务端

  ```java
  public class NioServer {
      private static Logger logger = LoggerFactory.getLogger(NioServer.class);
  
      public static void main(String[] args) throws Exception{
          // 创建ServerSocketChannel
          ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
          // 创建Selector对象
          Selector selector = Selector.open();
          // 绑定端口并在服务端进行监听
          ServerSocket socket = serverSocketChannel.socket();
          socket.bind(new InetSocketAddress(6666));
          // 设置ServerSocketChannel为非阻塞模式
          serverSocketChannel.configureBlocking(false);
          // 将ServerSocketChannel注册到Selector，且关注事件为"注册监听"
          serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
          // 循环等待客户端连接
          while (true) {
              // 等待一秒，如果无事件发生就继续
              if (selector.select(1000) == 0) {
                  logger.info("服务器等待一秒，无连接");
                  continue;
              }
              // 如果有事件发生,获取到想过的SelectionKey集合
              Set<SelectionKey> selectionKeys = selector.selectedKeys();
              // 遍历集合获取通道
              Iterator<SelectionKey> iterator = selectionKeys.iterator();
              while (iterator.hasNext()) {
                  SelectionKey key = iterator.next();
                  // 根据key对应通道发生的事件做不同的处理SocketChannel
                  if (key.isAcceptable()) {
                      // 为该客户端生成一个SocketChannel
                      SocketChannel socketChannel = serverSocketChannel.accept();
                      logger.info("客户端连接成功");
                      // 将SocketChannel设置为非阻塞
                      socketChannel.configureBlocking(false);
                      // 将当前的SocketChannel注册到Selector，事件为OP_READ，并为该Channel绑定一个Buffer
                      socketChannel.register(selector,SelectionKey.OP_READ, ByteBuffer.allocate(1024));
                  }
                  if (key.isReadable()) {
                      // 通过SelectionKey获取对应的SocketChannel
                      SocketChannel channel = (SocketChannel) key.channel();
                      // 获取到Channel关联的Buffer
                      ByteBuffer buffer = (ByteBuffer) key.attachment();
                      // 将Channel中的数据读取到Buffer中
                      channel.read(buffer);
  
                      logger.info("从客户端读取到的数据为：" + new String(buffer.array()));
                  }
                  // 手动从集合中删除key，防止重复操作
                  iterator.remove();
              }
          }
      }
  }
  ```

- 客户端

  ```java
  public class NioClient {
      private static Logger logger = LoggerFactory.getLogger(NioServer.class);
  
      public static void main(String[] args) throws Exception{
          // 获取一个网络通道
          SocketChannel socketChannel = SocketChannel.open();
          // 设置通道非阻塞
          socketChannel.configureBlocking(false);
          // 提供服务器端的ip和端口
          InetSocketAddress address = new InetSocketAddress("127.0.0.1", 6666);
          // 连接服务器
          if (!socketChannel.connect(address)) {
              while (!socketChannel.finishConnect()) {
                  logger.info("连接需要时间，客户端不会阻塞，可以进行其他操作");
              }
          }
          // 如果连接成功，就发送数据
          String str = "hello";
          // wrap方法通过字节数组指定buffer大小
          ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
          // 将buffer数据写入Channel
          socketChannel.write(buffer);
  
          System.in.read();
      }
  }
  ```

##### 2.3.4.5 SelectionKey API

当有通道注册到Selector上是，实际上保存在Selector中的`keys`属性中，而`keys`就是一个HashSet

```java
public abstract class SelectorImpl extends AbstractSelector {
	protected Set<SelectionKey> selectedKeys = new HashSet();
    protected HashSet<SelectionKey> keys = new HashSet();
    private Set<SelectionKey> publicKeys;
    private Set<SelectionKey> publicSelectedKeys;
    
    public Set<SelectionKey> keys() {
        if (!this.isOpen() && !Util.atBugLevel("1.4")) {
            throw new ClosedSelectorException();
        } else {
            return this.publicKeys;
        }
    }
    
    public Set<SelectionKey> selectedKeys() {
        if (!this.isOpen() && !Util.atBugLevel("1.4")) {
            throw new ClosedSelectorException();
        } else {
            return this.publicSelectedKeys;
        }
    }   
}
```

通过调用`keys()`方法获取到的是publicKeys，而publicKeys就是注册到Selector上的所有key；而`selectedKeys()`方法获取到的是publicSelectedKeys，而publicSelectedKeys是注册到Selector上且有事件发生的所有key

对于SelectorKey的不同监听事件

```java
// 代表读操作，值为1 
public static final int OP_READ = 1 << 0; 
// 代表写操作，值为4
public static final int OP_WRITE = 1 << 2;
// 代表连接已经建立，值为8
public static final int OP_CONNECT = 1 << 3;
// 存在新的网络连接可以accept，值为16
public static final int OP_ACCEPT = 1 << 4;
```

SelectorKey相关方法

```java
public abstract Selector selector();//得到与之关联的Selector对象
public abstract SelectableChannel channel();//得到与之关联的通道
public final Object attachment();//得到与之关联的共享数据
public abstract SelectionKey interestOps(int ops);//设置或改变监听事件
public final boolean isAcceptable();//是否可以accept
public final boolean isReadable();//是否可以读
public final boolean isWritable();//是否可以写
```

##### 2.3.4.6 ServerSocketChannel和SocketChannel API

- ServerSocketChannel在服务器端监听新的客户端Socket连接

  ```java
  public static ServerSocketChannel open()//得到一个ServerSocketChannel通道
  public final ServerSocketChannel bind(SocketAddress local)//设置服务器端端口号
  public final SelectableChannel configureBlocking(boolean block)//设置阻塞或非阻塞模式，取值false表示采用非阻塞模式
  public SocketChannel accept()//接受一个连接，返回代表这个连接的通道对象
  public final SelectionKey register(Selector sel, int ops)//注册一个选择器并设置监听事件
  ```

- SocketChannel网络 IO 通道，具体负责进行读写操作NIO ;把缓冲区的数据写入通道，或者把通道里的数据读到缓冲区

  ```java
  public static SocketChannel open();//得到一个SocketChannel通道
  public final SelectableChannel configureBlocking(boolean block);//设置阻塞或非阻塞模式，取值false表示采用非阻塞模式
  public boolean connect(SocketAddress remote);//连接服务器
  public boolean finishConnect();//如果上面的方法连接失败，接下来就要通过该方法完成连接操作
  public int write(ByteBuffer src);//往通道里写数据
  public int read(ByteBuffer dst);//从通道里读数据
  public final SelectionKey register(Selector sel, int ops, Object att);//注册一个选择器并设置监听事件，最后一个参数可以设置共享数据
  public final void close();//关闭通道
  ```

##### 2.3.4.7 NIO群聊系统

- 服务端

  ```java
  public class Server {
      // 属性定义
      private Selector selector;
      private ServerSocketChannel listenChannel;
      private static final int PORT = 6667;
      private static final Logger LOGGER = LoggerFactory.getLogger(Server.class);
  
      public static void main(String[] args) throws Exception{
          // 启动客户端
          Server server = new Server();
          // 监听
          server.listen();
      }
  
      // 初始化
      public Server() {
          try {
              // 设置selector
              this.selector = Selector.open();
              // 设置channel
              listenChannel = ServerSocketChannel.open();
              // 绑定端口
              listenChannel.bind(new InetSocketAddress(PORT));
              // 设置非阻塞
              listenChannel.configureBlocking(false);
              // 将channel注册到selector
              listenChannel.register(selector, SelectionKey.OP_ACCEPT);
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  
      // 监听
      public void listen() {
          try {
              while (true) {
                  int count = selector.select();
                  if (count > 0) { // 有事件处理
                      // 遍历获取selectedKey
                      Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                      while (iterator.hasNext()) {
                          SelectionKey key = iterator.next();
                          // 对key的事件类型进行判断
                          if (key.isAcceptable()) {
                              SocketChannel socketChannel = listenChannel.accept();
                              // 设置channel非阻塞
                              socketChannel.configureBlocking(false);
                              // 将socketChannel注册到selector
                              socketChannel.register(selector,SelectionKey.OP_READ);
                              // 消息提示
                              LOGGER.info(socketChannel.getRemoteAddress().toString() + "上线");
                          }
                          if (key.isReadable()) {
                              // 读取客户端消息
                              readData(key);
                          }
                          // 从set中删除key
                          iterator.remove();
                      }
                  } else {
                      LOGGER.info("等待...");
                  }
              }
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  
      // 读取客户端消息
      private void readData(SelectionKey selectionKey) {
          // 定义socketChannel
          SocketChannel socketChannel = null;
          try {
              // 通过key获取channel
              socketChannel = (SocketChannel) selectionKey.channel();
              // 创建buffer
              ByteBuffer buffer = ByteBuffer.allocate(1024);
              // 将channel中的数据读取到buffer
              int count = socketChannel.read(buffer);
              // 判断是否读取到数据
              if (count > 0) {
                  String msg = new String(buffer.array());
                  LOGGER.info(msg);
                  // 向其他客户端转发消息
                  sendMsgToOther(msg,socketChannel);
              }
  
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  
      // 转发消息到其他客户端(需要排除自身的channel)
      private void sendMsgToOther(String msg,SocketChannel self) {
          LOGGER.info("服务器转发消息...");
          // 遍历所有注册到selector上的channel并排除自身
          selector.keys().stream().map(key -> ((SocketChannel) key.channel())).filter(channel -> channel == self).forEach(channel -> {
              try {
                  channel.write(ByteBuffer.wrap(msg.getBytes()));
              } catch (IOException e) {
                  try {
                      LOGGER.info(channel.getRemoteAddress().toString() + "已离线");
                      // 取消注册
                      channel.keyFor(selector).cancel();
                      // 关闭通道
                      channel.close();
                  } catch (IOException ioException) {
                      ioException.printStackTrace();
                  }
              }
          });
      }
  }
  ```

- 客户端

  ```java
  public class Client {
      // 定义属性
      private final static String HOST = "127.0.0.1";
      private final static int PORT = 6667;
      private Selector selector;
      private SocketChannel socketChannel;
      private String username;
  
      private static final Logger LOGGER = LoggerFactory.getLogger(Client.class);
  
      public static void main(String[] args) {
          // 启动客户端
          Client client = new Client();
          // 启动一个线程，每隔3秒从服务器读取数据
          new Thread(() -> {
              while (true) {
                  client.receiveInfo();
                  try {
                      Thread.sleep(3000);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
          }).start();
  
          // 客户端向服务端发送数据
          Scanner scanner = new Scanner(System.in);
  
          while (scanner.hasNextLine()) {
              String msg = scanner.nextLine();
              client.sendInfo(msg);
          }
      }
  
      // 初始化
      public Client() {
          try {
              selector = Selector.open();
              // 连接服务器
              socketChannel = SocketChannel.open(new InetSocketAddress(HOST, PORT));
              // 设置非阻塞
              socketChannel.configureBlocking(false);
              // 将channel注册到selector
              socketChannel.register(selector, SelectionKey.OP_READ);
              // 获取username
              username = socketChannel.getLocalAddress().toString();
  
              LOGGER.info("客户端" + username + "准备就绪");
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  
      // 向服务器发送消息
      public void sendInfo(String info) {
          try {
              int count = socketChannel.write(ByteBuffer.wrap(info.getBytes()));
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  
      // 读取从服务端回复的消息
      public void receiveInfo() {
          try {
              int readChannels = selector.select();
              if (readChannels > 0) { // 是否有可用通道
                  Set<SelectionKey> selectionKeys = selector.selectedKeys();
                  Iterator<SelectionKey> iterator = selectionKeys.iterator();
                  while (iterator.hasNext()) {
                      SelectionKey key = iterator.next();
                      if (key.isReadable()) {
                          SocketChannel channel = (SocketChannel) key.channel();
                          ByteBuffer buffer = ByteBuffer.allocate(1024);
                          // 读取缓冲区中的数据
                          channel.read(buffer);
                          // 输出buffer中的信息
                          LOGGER.info(new String(buffer.array()).trim());
                      } else {
                          LOGGER.info("没有可用通道");
                      }
                      iterator.remove();
                  }
              }
          } catch (IOException e) {
              e.printStackTrace();
          }
      }
  }
  ```

## 3 Netty

### 3.1 Netty概述

![图片1](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/图片1.png)

> 底层核心：零拷贝、核心库(API)、可扩展事件模型...
>
> 协议支持：HTTP&WS、gzip压缩/解压、大文件传输协议...
>
> 传输服务：Socket、HTTP隧道...

**Netty优势**

- 适用于各种传输类型的统一 API 阻塞和非阻塞 Socket
- 基于灵活且可扩展的事件模型，可以清晰地分离关注点
- 高度可定制的线程模型 - 单线程，一个或多个线程池
- 高性能、吞吐量更高：延迟更低；减少资源消耗；最小化不必要的内存复制
- 完整的 SSL/TLS 和 StartTLS 支持

### 3.2 Netty线程模型

> 当前存在的线程模型：

- 传统阻塞I/O模型
- Reactor模式
  - 单Reactor单线程
  - 单Reactor多线程
  - 主从Reactor多线程

Netty线程模型基于主从Reactor多线程模型进一步改进，其中主从Reactor多线程模型存在多个Reactor

> 传统阻塞I/O服务模型

![image-20201217112613250](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201217112613250.png)

- 图释

  黄色边框代表对象，蓝色边框代表线程，白色边框代表API

- 特点

  采用阻塞IO模式获取输入的数据

  每个连接都需要独立的线程完成数据的输入，业务处理,数据返回

- 存在问题

  当并发数很大，就会创建大量的线程，占用很大系统资源

  连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read 操作，造成线程资源浪费

> Reactor模式

- Reactor模式针对传统I/O问题的解决

  基于 I/O 复用模型：多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接；当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理

  基于线程池复用线程资源：不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，一个线程可以处理多个连接的业务

- 实现原理图

  ![image-20201217114254169](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201217114254169.png)

  Reactor 对应的叫法: 1. 反应器模式 2. 分发者模式(Dispatcher) 3. 通知者模式(notifier)

- Reactor架构图

  ![image-20201217114454755](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201217114454755.png)

  Reactor 模式通过一个或多个输入同时传递给服务处理器的模式(基于事件驱动)

  服务器端程序处理传入的多个请求,并将它们同步分派到相应的处理线程， 因此Reactor模式也叫 Dispatcher模式

  Reactor 模式使用IO复用监听事件, 收到事件后分发给某个线程(进程), 这就是网络服务器高并发处理关键

- Reactor核心组件

  Reactor(**也就是图中的ServiceHandler**)：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对 IO 事件做出反应

  Handlers：处理程序执行 I/O 事件要完成的实际事件，Reactor 通过调度适当的处理程序来响应 I/O 事件，处理程序执行非阻塞操作

#### 3.2.1 单Reactor模式

![image-20201217123621319](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201217123621319.png)

> 方案说明

- Select 是 I/O 复用模型标准网络编程 API，可以实现应用程序通过一个阻塞对象监听多路连接请求
- Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发
- 如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理
- 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应
- Handler 会完成 Read→业务处理→Send 的完整业务流程

> 优缺点

- 优点：模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成

- 缺点

  性能问题，只有一个线程，无法完全发挥多核 CPU 的性能；Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈

  可靠性问题，线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障

- 使用场景

  客户端的数量有限，业务处理非常快速，比如 Redis在业务处理的时间复杂度 O(1) 的情况

#### 3.2.2 单Reactor多线程

![image-20201217124722907](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201217124722907.png)

> 方案说明

- Reactor 对象通过select 监控客户端请求事件, 收到事件后，通过dispatch进行分发
- 如果建立连接请求, 则由Acceptor 通过accept 处理连接请求, 然后创建一个Handler对象处理完成连接后的各种事件
- 如果不是连接请求，则由Reactor分发调用连接对应的handler 来处理
- handler 只负责响应事件，不做具体的业务处理, 通过read 读取数据后，会分发给后面的worker线程池的某个线程处理业务
- worker 线程池会分配独立线程完成真正的业务，并将结果返回给handler
- handler收到响应后，通过send 将结果返回给client

> 优缺点

- 优点：可以充分的利用多核cpu 的处理能力
- 缺点：多线程数据共享和访问比较复杂， Reactor 处理所有的事件的监听和响应，在单线程运行， 在高并发场景容易出现性能瓶颈

#### 3.2.3 主从Reactor多线程

![image-20201217125222086](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201217125222086.png)

> 方案说明

- Reactor主线程 MainReactor 对象通过select 监听连接事件, 收到事件后，通过Acceptor 处理连接事件(<u>主线程只负责建立连接</u>)

- 当 Acceptor 处理连接事件后，MainReactor 将连接分配给SubReactor (<u>一个主线程下面并不是只有一个子线程，而是可能存在多个子线程</u>)
- Subreactor 将连接加入到连接队列进行监听,并创建handler进行各种事件处理
- 当有新事件发生时， Subreactor 就会调用对应的handler处理
- handler 通过read 读取数据，分发给后面的worker 线程处理
- worker 线程池分配独立的worker 线程进行业务处理，并返回结果
- handler 收到响应的结果后，再通过send 将结果返回给client

`特别说明：Reactor 主线程可以对应多个Reactor 子线程, 即MainRecator 可以关联多个SubReactor`

> 优缺点

- 优点

  父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理

  父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据

- 实例

  Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持

#### 3.2.4 Reactor模式总结

> 生活理解

- 单 Reactor 单线程，前台接待员和服务员是同一个人，全程为顾客服务
- 单 Reactor 多线程，1 个前台接待员，多个服务员，接待员只负责接待
- 主从 Reactor 多线程，多个前台接待员，多个服务生

> 优点

- 响应快，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的
- 可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销
- 扩展性好，可以方便的通过增加 Reactor 实例个数来充分利用 CPU 资源
- 复用性好，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性

### 3.3 Netty模型

![image-20201218142332734](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218142332734.png)

> 工作原理

- Netty抽象出两组线程池 BossGroup 专门负责接收客户端的连接, WorkerGroup 专门负责网络的读写
- BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup
- NioEventLoopGroup 相当于一个事件循环组, 这个组中含有多个事件循环 ，每一个事件循环是 NioEventLoop
- NioEventLoop 表示一个不断循环的执行处理任务的线程， 每个NioEventLoop 都有一个Selector , 用于监听绑定在其上的socket的网络通讯
- NioEventLoopGroup 可以有多个线程, 即可以含有多个NioEventLoop
- 每个Boss NioEventLoop 循环执行3个步骤
  - 轮询accept 事件
  - 处理accept 事件 , 与client建立连接 , 生成NioScocketChannel , 并将其注册到某个worker NIOEventLoop 上的 selector
  - 处理任务队列的任务 ， 即 runAllTasks
- 每个 Worker NIOEventLoop 循环执行3个步骤
  - 轮询read, write 事件
  - 处理I/O事件， 即read , write 事件，在对应NioScocketChannel 处理
  - 处理任务队列的任务 ， 即 runAllTasks
- 每个Worker NIOEventLoop  处理业务时，会使用pipeline(管道), pipeline 中包含了 channel , 即通过pipeline 可以获取到对应通道, 管道中维护了很多的 处理器

#### 3.3.1 Netty构建TCP服务

> 服务端

```java
public class Server {
    private static final Logger logger = LoggerFactory.getLogger(Server.class);

    public static void main(String[] args) {
        // 创建BossGroup和WorkGroup
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        try {
            // 创建服务端的启动对象，配置参数
            ServerBootstrap bootstrap = new ServerBootstrap();

            // 链式编程配置参数
            bootstrap.group(bossGroup,workGroup)  // 设置线程组
                .channel(NioServerSocketChannel.class) // 使用NioServerSocketChannel作为服务器通道的实现
                .option(ChannelOption.SO_BACKLOG,128) // 设置线程队列获得的连接个数
                .childOption(ChannelOption.SO_KEEPALIVE,true) // 设置保持活动连接状态
                .childHandler(new ChannelInitializer<SocketChannel>() { // 对workGroup的EventLoop对应的pipeline设置处理器
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().addLast(new ServerHandler());
                    }
                });

            logger.info("服务器已就绪...");

            // 绑定端口并生成同步对象
            try {
                ChannelFuture channelFuture = bootstrap.bind(6666).sync();
                // 对关闭通道进行监听
                ChannelFuture close = channelFuture.channel().closeFuture().sync();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
        }
    }
}
```

> 服务端处理器

```java
public class ServerHandler extends ChannelInboundHandlerAdapter {
    private static final Logger logger = LoggerFactory.getLogger(ServerHandler.class);
    /**
     * 当消息发送到pipeline时即调用handler用于读取消息
     * @param ctx ChannelHandlerContext上下文对象，其中包含管道、通道和handler
     * @param msg 消息
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        logger.info("server ctx=" + ctx);
        // 将msg转换为ByteBuffer，这里的Buffer属于Netty，而不是NIO中的Buffer
        ByteBuf byteBuf = (ByteBuf) msg;
        logger.info("服务端发送的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
    }

    // 数据读取完毕
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        // 将数据写入到缓冲区并刷新
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello client", CharsetUtil.UTF_8));
    }

    // 处理异常
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        // 关闭通道
        ctx.close();
    }
}
```

> 客户端

```java
public class Client {
    private static final Logger logger = LoggerFactory.getLogger(Client.class);

    public static void main(String[] args) {
        NioEventLoopGroup eventExecutors = new NioEventLoopGroup();

        try {
            // 创建客户端启动对象
            Bootstrap bootstrap = new Bootstrap();
            // 设置相关参数
            bootstrap.group(eventExecutors)  // 设置线程组
                    .channel(NioSocketChannel.class) // 设置客户端通道实现类
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new ClientHandler());  // 加入自定义处理器
                        }
                    });

            logger.info("客户端准备就绪...");

            // 启动客户端连接服务器
            try {
                ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 6666).sync();
                // 对关闭通道进行监听
                ChannelFuture closeFuture = channelFuture.channel().closeFuture().sync();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            eventExecutors.shutdownGracefully();
        }
    }
}
```

> 客户端处理器

```java
public class ClientHandler extends ChannelInboundHandlerAdapter {
    private static final Logger logger = LoggerFactory.getLogger(ClientHandler.class);

    // 当通道就绪就会触发该方法
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        logger.info("client ctx：" + ctx);
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello server", CharsetUtil.UTF_8));
    }

    // 读取服务端返回的信息
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 将msg转换为ByteBuffer，这里的Buffer属于Netty，而不是NIO中的Buffer
        ByteBuf byteBuf = (ByteBuf) msg;
        logger.info("客户端发送的消息：" + byteBuf.toString(CharsetUtil.UTF_8));
    }

    // 处理异常
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        // 关闭通道
        ctx.close();
    }
}
```

#### 3.3.2 Netty构建TCP服务源码分析

> 分析NioEventLoop

- 当执行到`EventLoopGroup bossGroup = new NioEventLoopGroup()`，bossGroup中包含的NioEventLoop为**机器核心数的2倍**

  **对应的源码**

  ```java
  private static final int DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2))
  ```

  如果NioEventLoopGroup的构造器参数为空时，源码追踪得到NioEventLoop的数量也就是`DEFAULT_EVENT_LOOP_THREADS`常量

  **结果展示**

  ![image-20201218164135149](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218164135149.png)

  ![image-20201218164359040](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218164359040.png)

  **指定NioEventLoopGroup线程个数**

  ```java
  // 即分配两个线程
  EventLoopGroup bossGroup = new NioEventLoopGroup(2)
  ```

  当NioEventLoopGroup分配线程对消息进行处理时，默认采用的轮询机制

> 分析NioEventLoop

![image-20201218165306335](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218165306335.png)

每一个NioEventLoop中都包含Selector、taskQueue以及executor，而Selector中包含SelectorKey

> ChannelHandlerContext、Channel和Pipeline之间的关系

- ChannelHandlerContext(上下文)

  ![image-20201218170617666](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218170617666.png)

  通过prev和next可以看出`ctx`是一个双向链表，可以获取到对应的channel和pipeline，且两者是对应关系，并且可以获得对应的handler

- Channel

  ![image-20201218170916673](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218170916673.png)

  可以看出channel可以获取到对应的selectionKey，pipeline以及对应的socket(IP和端口)，而且可以反向获取到对应的eventLoop

- Pipeline

  ![image-20201218171234301](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20201218171234301.png)

  通过head和tail可以看出pipeline是一个双向链表，可以获取到对应的channel，并且包含一个HashMap，其中保存的是对应处理器的信息