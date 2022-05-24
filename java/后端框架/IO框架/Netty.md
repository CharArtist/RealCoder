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

## Netty

