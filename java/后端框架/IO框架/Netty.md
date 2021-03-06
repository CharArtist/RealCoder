# Netty

Netty是一款基于NIO（Nonblocking I/O，非阻塞IO）开发的网络通信框架，相对于BIO（Blocking I/O，阻塞IO），并发性能得到了很大提高。

## Java NIO

Java NIO 主要有三大核心部分：Channel(通道)，Buffer(缓冲区), Selector(选择器)。传统 IO 基于字节流和字符流进行操作，而 NIO 基于 Channel和 Buffer (缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector(选择器) 用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。

一个简单的网络 IO 服务器的案例如下：

**服务端**

```java
public class NIOServer {

    private static final int BUF_SIZE = 1024;
    private static final int PORT = 8080;
    private static final int TIMEOUT = 3000;

    public static void main(String[] args) {
      
        Selector selector = null; // selector 选择器
        ServerSocketChannel ssc = null; // 服务端用来监听的 socket channel 

        try {
            selector = Selector.open(); // 创建 selector
            ssc = ServerSocketChannel.open(); // 创建用于监听的 server socket channel
            ssc.socket().bind(new InetSocketAddress(PORT)); // 监听的 server socket channel 绑定本机端口
            ssc.configureBlocking(false); // 监听的 server socket channel 设置为非阻塞，select() 方法就不会阻塞
            // 将监听的 server socket channel 注册到 selector，并且注册感兴趣的事件 OP_ACCEPT
          	// 一共有 4 种事件：
            // 1. OP_ACCEPT：ServerSocketChannel 关心的事件，表示接受客户端连接
            // 2. OP_CONNECT：SocketChannel 关心的事件，表示连接就绪
            // 3. OP_WRITE：SocketChannel 关心的事件，表示 channel 可写
            // 4. OP_READ：SocketChannel 关心的事件，表示 channel 可读
          	ssc.register(selector, SelectionKey.OP_ACCEPT); 
            while(true) { // 不停机服务
              	// 非阻塞式 select
                if(selector.select(TIMEOUT) == 0){ // 一直没有连接就 continue
                    System.out.println("==");
                    continue;
                }
              	// 有事件来了就处理，key 就是事件详细信息
                // 第一次 selector 中只有 ServerSocketChannel 的 OP_ACCEPT 事件，即第一次走到这里说明是有客户端连接了
                Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
                while(iter.hasNext()) {
                    SelectionKey key = iter.next(); 
                    if(key.isAcceptable()) handleAccept(key); // ServerSocketChannle 处理连接
                    if(key.isReadable()) handleRead(key); // SocketChannel 处理读
                    if(key.isWritable() && key.isValid()) handleWrite(key); // SocketChannel 处理写
                    if(key.isConnectable()) 
                      	System.out.println("isConnectable = true"); // SocketChannel 处理可连接
                    iter.remove(); // 处理完了就移除该事件
                }
            }
        } catch(IOException e) {
            e.printStackTrace();
        } finally {
            try{
                if (selector!=null) selector.close(); // 关闭 selector 选择器
                if (ssc!=null) ssc.close(); // 关闭 server socket channel
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

  	// ServerSocketChannel 处理客户端连接
    private static void handleAccept(SelectionKey key) throws IOException {
        // 拿到 ServerSocketChannel
        ServerSocketChannel ssChannel = (ServerSocketChannel)key.channel();
      	// 调用 ServerSocketChannel 的 accept 方法，并新建一个 SocketChannel 负责该连接的 IO
        SocketChannel sc = ssChannel.accept();
        sc.configureBlocking(false);
      	// SocketChannel 注册到 selector，并注册其关心的可读事件
      	// 可以将一个对象附加到 SelectionKey 上，这样就能方便的识别某个给定的通道。例如可以附加与通道一起使用的 buffer
        sc.register(key.selector(), SelectionKey.OP_READ, ByteBuffer.allocateDirect(BUF_SIZE));
    }

  	// SocketChannel 处理可读
    private static void handleRead(SelectionKey key) throws IOException {
      	// 拿到 SocketChannel
        SocketChannel sc = (SocketChannel)key.channel();
        ByteBuffer buf = (ByteBuffer)key.attachment();
        long bytesRead = sc.read(buf); // 读数据到 buffer
        while(bytesRead > 0){
            buf.flip(); // buffer 反转为读模式
            while(buf.hasRemaining()) {
                System.out.print((char)buf.get()); // 读 buffer 中的数据
            }
            System.out.println();
            // 清空 buffer 以供下一次读，这里其实不是真的清空，而是对 buffer 中的 position、capacity 进行调整
          	// 这里对 buffer 的操作其实是有讲究的，包括前面的 flip 和这里的 clear
          	// clear 完了 buffer 就又是写模式了
          	buf.clear(); 
            bytesRead = sc.read(buf); // 循环从 SocketChannel 里读数据写到 buffer 里
        }
        if(bytesRead == -1) sc.close();
    }
		
  	// SocketChannel 处理可写
    private static void handleWrite(SelectionKey key) throws IOException {
        ByteBuffer buf = (ByteBuffer)key.attachment();
        buf.flip(); // buffer 反转为写模式
      	// 拿到 SocketChannel
        SocketChannel sc = (SocketChannel) key.channel();
        while(buf.hasRemaining()) {
            sc.write(buf); // 从 buffer 往 SocketChannel 写
        }
        buf.compact();
    }
}
```

**客户端**

```java
// 只需要用一个简单的 BIO 客户端连接就可以了，不需要 NIO 的客户端
public class Client extends Thread {

    // 定义一个Socket对象
    Socket socket = null;

    public Client(String host, int port) {
        try {
            // 根据服务器的IP地址和端口号创建Socket对象
            socket = new Socket(host, port);
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        // 客户端一连接就可以写数据给服务器了
        new sendMessThread().start();
        super.run();
        try {
            // 读 Socket 里面的数据
            InputStream s = socket.getInputStream();
            byte[] buf = new byte[1024];
            int len = 0;
            while ((len = s.read(buf)) != -1) {
                System.out.println(new String(buf, 0, len));
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //往Socket里面写数据，需要新开一个线程
    class sendMessThread extends Thread{
        @Override
        public void run() {
            super.run();
            // 写操作，从命令行读
            Scanner scanner=null;
            OutputStream os= null;
            try {
                scanner=new Scanner(System.in);
                os= socket.getOutputStream();
                String in = "";
                do {
                    in=scanner.next();
                    os.write((""+in).getBytes());
                    os.flush();
                } while (!in.equals("bye"));
            } catch (IOException e) {
                e.printStackTrace();
            }
            scanner.close();
            try {
                os.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    public static void main(String[] args) {
        // 创建客户端并启动
        Client clientTest = new Client("127.0.0.1", 8080);
        clientTest.start();
    }
}
```

### NIO 封装的内存映射 mmap ：MappedByteBuffer

Java NlO 中 的Channel (通道) 就相当于操作系统中的内核缓冲区，有可能是读缓冲区，也有可能是网络缓冲区，而一般的 Buffer 就相当于操作系统中的用户缓冲区。Java NIO 还提供给了 MappedByteBuffer，其底层就是 Linux 的 mmap 系统调用。

MapppedByteBuffer 是 FileChannel 专有的，只能通过调用 FileChannel 的map 方法获取，是一种用于直接将内核态缓冲直接映射到用户态的方式，之前的网络 IO 基础中介绍过，可以减少一次内核态到用户态拷贝，在传输大文件时性能要好很多。

```java
RandomAccessFile aFile = new RandomAccessFile("/xxx/yyy/zzz","rw");;
FileChannel fc = aFile.getChannel();
MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_ONLY, 0, aFile.length());
```

使用 MappedByteBuffer类要注意的是：mmap 的文件映射，在 full gc 时才会进行释放。当 close 时，需要手动清除内存映射文件，可以反射调用 sun.misc.Cleaner 方法。

但是如果其后要通过 SocketChannel 发送，还是需要 CPU 进行数据的拷贝（要从内核缓冲区拷贝到 Sokect 缓冲区）。小文件使用 MappedByteBuffer 效率不高，在并发不高的情况下效率也不高。

### NIO 封装的 sedndfile ：transferTo

FileChannel 的 transferTo 方法直接将当前通道内容传输到另一个通道，没有涉及到任何 Buffer 操作，NIO 中 的 Buffer 是 JVM 堆或者堆外内存，但不论如何他们都是操作系统内核空间的内存。 transferTo 方法的实现方式就是通过系统调用 sendfile ，sendfile 减少了两次用户态和内核态的切换，且高版本的 Linux 还可以通过网卡的 SG-DMA 控制器直接将文件从内核缓冲区拷贝到网卡，无需拷贝到 Socket 缓冲区，此过程无需 CPU 参与。

```java
// 这里其实就是零拷贝的一种实现，但是底层是不是零拷贝还需要看操作系统
// 操作系统提供 sendfile 零拷贝系统调用，则 transferTo 会通过 sendfile 系统调用实现
FileChannel sourceChannel = new RandomAccessFile("/xxx/yyy/zzz", "rw").getChannel();
SocketChannel socketChannel = SocketChannel.open(socketAddress);
sourceChannel.transferTo(0, sourceChannel.size(), socketChannel);
```

## Reactor 模型

I/O 多路复用 + 线程池就是 Reactor 模式的基本设计思想。线程池+BIO的方式存在的问题是连接数较大时，线程切换带来的开销会导致处理 IO 的效率变得很慢， I/O 多路复用让单线程就可以监听处理所有的 fd ，但是对于多核 CPU 来说，这不利于充分利用 CPU 资源，因此可以开辟专用的线程池用于 I/O 处理，一个线程负责多个 fd 的读写事件。这就是工程的艺术，amazing。

### 单 Reactor 单线程

<img src="/Users/xuping/Library/Application Support/typora-user-images/image-20220530220500107.png" alt="image-20220530220500107" style="zoom:50%;margin-left:0" />

### 单 Reactor 多线程

<img src="/Users/xuping/Library/Application Support/typora-user-images/image-20220530220550110.png" alt="image-20220530220550110" style="zoom:50%;margin-left:0" />

### 多 Reactor 多线程

<img src="/Users/xuping/Library/Application Support/typora-user-images/image-20220530220628284.png" alt="image-20220530220628284" style="zoom:50%;margin-left:0" />

## Proactor 模型

<img src="/Users/xuping/Library/Application Support/typora-user-images/image-20220530220804443.png" alt="image-20220530220804443" style="zoom:50%;margin-left:0" />

* 可惜的是，在 Linux 下的异步 I/O 是不完善的， `aio` 系列函数是由 POSIX 定义的异步操作接口，不是真正的操作系统级别支持的，而是在用户空间模拟出来的异步，并且仅仅支持基于本地文件的 aio 异步操作，网络编程中的 socket 是不支持的，这也使得基于 Linux 的高性能网络程序都是使用 Reactor 方案。

* 而 Windows 里实现了一套完整的支持 socket 的异步编程接口，这套接口就是 `IOCP`，是由操作系统级别实现的异步 I/O，真正意义上异步 I/O，因此在 Windows 里实现高性能网络程序可以使用效率更高的 Proactor 方案。

## Netty

Netty 是一个客户端、服务端 NIO 框架，提供异步的、事件驱动的网络 IO 工具，允许快速简单的开发网络应用程序，简化了网络编程模型。

简单的 Server 端 demo（从官网摘抄过来的，英文直接翻译为中文，英文的逻辑表达比很多中文博客都要简洁清晰）：

```java
public class NettyServer {

    public static void main(String[] args) throws Exception {
        int port = 8080;
        new Server(port).run();
    }

    static class Server {
        private int port;
        public Server(int port) {
            this.port = port;
    }

      public void run() throws Exception {
        // NioEventLoopGroup 是一个处理 I/O 操作的多线程事件循环，Netty 为不同类型的传输提供了各种 EventLoopGroup 实现
        // 在这个例子中，我们正在实现一个服务器端应用程序，因此将使用两个 NioEventLoopGroup
        // 第一个，通常称为“boss”，接受传入连接；第二个，通常称为“worker”，作为工作人员
        // 一旦老板接受连接并将接受的连接注册到工作人员，工作人员就会处理接受的连接的流量
        // 使用多少线程以及它们如何映射到创建的通道取决于 EventLoopGroup 实现，可以通过构造函数进行配置
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
          // ServerBootstrap 是一个设置服务器的辅助类
          // 您可以直接使用 Channel 设置服务器。但是，请注意，这是一个乏味的过程，在大多数情况下您不需要这样做
          ServerBootstrap b = new ServerBootstrap();
          b.group(bossGroup, workerGroup)
            // 在这里，我们指定使用 NioServerSocketChannel 类来实例化一个新的 Channel 来接受传入的连接。
            .channel(NioServerSocketChannel.class) 
            // 此处指定的处理程序将始终由新接受的 Channel 评估
            // ChannelInitializer 是一个特殊的处理程序，旨在帮助用户配置新的 Channel
            // 您很可能希望通过添加一些处理程序（例如 DiscardServerHandler）来配置新 Channel 的 ChannelPipeline 来实现您的网络应用程序。随着应用程序变得复杂，您可能会向管道添加更多处理程序，并最终将这个匿名类提取到顶级类中
            .childHandler(new ChannelInitializer<SocketChannel>() {
              @Override
              protected void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline().addLast(new DiscardServerHandler());
              }
            })
            // 您还可以设置特定于 Channel 实现的参数。我们正在编写一个 TCP/IP 服务器，因此我们可以设置套接字选项，例如 tcpNoDelay 和 keepAlive。请参阅 ChannelOption 的 apidocs 和具体的 ChannelConfig 实现，以了解有关支持的 ChannelOptions 的概述。
            .option(ChannelOption.SO_BACKLOG, 128)
            // 你注意到 option() 和 childOption() 了吗？ option() 用于接受传入连接的 NioServerSocketChannel。 childOption() 用于父 ServerChannel 接受的 Channels，在本例中为 NioSocketChannel。
            .childOption(ChannelOption.SO_KEEPALIVE, true);
          // 我们现在准备出发。剩下的就是绑定到端口并启动服务器。这里，我们绑定到本机所有网卡（网卡）的8080端口。您现在可以根据需要多次调用 bind() 方法（使用不同的绑定地址。）
          ChannelFuture f = b.bind(port).sync();
          // 等待服务器套接字关闭。在此示例中，这不会发生，但您可以这样做以正常关闭服务器。
          f.channel().closeFuture().sync();
        } finally {
          workerGroup.shutdownGracefully();
          bossGroup.shutdownGracefully();
        }
      }
    }

  	// DiscardServerHandler 扩展了 ChannelInboundHandlerAdapter，它是 ChannelInboundHandler 的一个实现。 ChannelInboundHandler 提供了可以覆盖的各种事件处理程序方法。目前，扩展 ChannelInboundHandlerAdapter 就足够了，而不是自己实现处理程序接口。
    static class DiscardServerHandler extends ChannelInboundHandlerAdapter{
      	// 我们在这里覆盖了 channelRead() 事件处理方法。每当从客户端接收到新数据时，都会使用接收到的消息调用此方法。在本例中，接收到的消息类型为 ByteBuf。
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ByteBuf in = (ByteBuf) msg;
            try {
                while (in.isReadable()) {
                    System.out.print((char) in.readByte()); // 打印一下接受到的消息，别的什么都不做
                    System.out.flush();
                }
            } finally {
              	// ByteBuf 是一个引用计数对象，必须通过 release() 方法显式释放。请记住，释放任何传递给处理程序的引用计数对象是处理程序的责任。通常，channelRead() 中会这样做
                ReferenceCountUtil.release(msg);
            }
        }
				// 当 Netty 由于 I/O 错误或由于处理事件时抛出的异常时，使用 Throwable 调用 exceptionCaught() 事件处理程序方法。在大多数情况下，应记录捕获的异常并在此处关闭其关联的 Channel，尽管此方法的实现可能会有所不同，具体取决于您要如何处理异常情况。例如，您可能希望在关闭连接之前发送带有错误代码的响应消息。
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            cause.printStackTrace();
            ctx.close();
        }
    }
}
```

