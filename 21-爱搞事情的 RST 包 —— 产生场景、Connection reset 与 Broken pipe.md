

这篇文章我们来讲解 RST，RST 是 TCP 字发生错误时发送的一种分节，下面我们来介绍 RST 包出现常见的几种情况，方便你以后遇到 RST 包以后有一些思路。

在 TCP 协议中 RST 表示复位，用来**异常的**关闭连接，发送 RST 关闭连接时，不必等缓冲区的数据都发送出去，直接丢弃缓冲区中的数据，连接释放进入`CLOSED`状态。而接收端收到 RST 段后，也不需要发送 ACK 确认。

RST 常见的几种情况
-----------

我列举了常见的几种会出现 RST 的情况

#### 端口未监听

这种情况很常见，比如 web 服务进程挂掉或者未启动，客户端使用 connect 建连，都会出现 "Connection Reset" 或者"Connection refused" 错误。

![](https://user-gold-cdn.xitu.io/2019/6/19/16b6dd21761b9ffd)

这样机制可以用来检测对端端口是否打开，发送 SYN 包对指定端口，看会不会回复 SYN+ACK 包。如果回复了 SYN+ACK，说明监听端口存在，如果返回 RST，说明端口未对外监听，如下图所示

![](https://user-gold-cdn.xitu.io/2019/6/19/16b6dd217748a3d1)

#### 一方突然断电重启，之前建立的连接信息丢失，另一方并不知道

这个场景在前面 keepalive 那里介绍过。客户端和服务器一开始三次握手建立连接，中间没有数据传输进入空闲状态。这时候服务器突然断电重启，之前主机上所有的 TCP 连接都丢失了，但是客户端完全不知晓这个情况。等客户端有数据有数据要发送给服务端时，服务端这边并没有这条连接的信息，发送 RST 给客户端，告知客户端自己无法处理，你趁早死了这条心吧。

整个过程如下图所示：

![](https://user-gold-cdn.xitu.io/2019/6/19/16b6dd2177aff16c)

#### 调用 close 函数，设置了 SO\_LINGER 为 true

如果设置 SO\_LINGER 为 true，linger 设置为 0，当调用 socket.close() 时， close 函数会立即返回，同时丢弃缓冲区内所有数据并立即发送 RST 包重置连接。在 SO\_LINGER 那一节有详细介绍这个参数的含义。

RST 包如果丢失了怎么办？
--------------

这是一个比较有意思的问题，首先需要明确 **RST 是不需要确认的**。 下面假定是服务端发出 RST。

在 RST 没有丢失的情况下，发出 RST 以后服务端马上释放连接，进入 CLOSED 状态，客户端收到 RST 以后，也立刻释放连接，进入 CLOSED 状态。

如下图所示

![](https://user-gold-cdn.xitu.io/2019/6/19/16b6dd2176699765)

如果 RST 丢失呢？

服务端依然是在发送 RST 以后马上进入`CLOSED`状态，因为 RST 丢失，客户端压根搞不清楚状况，不会有任何动作。等到有数据需要发送时，一厢情愿的发送数据包给服务端。因为这个时候服务端并没有这条连接的信息，会直接回复 RST。

如果客户端收到了这个 RST，就会自然进入`CLOSED`状态释放连接。如果 RST 依然丢失，客户端只是会单纯的数据丢包了，进入数据重传阶段。如果还一直收不到 RST，会在一定次数以后放弃。

如下图所示

![](https://user-gold-cdn.xitu.io/2019/6/19/16b6dd22e77ed16b)

Broken pipe 与 Connection reset by peer
--------------------------------------

Broken pipe 与 Connection reset by peer 错误在网络编程中非常常见，出现的前提都是连接已关闭。

Connection reset by peer 这个错误很好理解，前面介绍了很多 RST 出现的场景。

`Broken pipe`出现的时机是：在一个 RST 的套接字继续写数据，就会出现`Broken pipe`。

下面来模拟 Broken pipe 的情况，服务端代码非常简单，几乎什么都没做，完整的代码见：[Server.java](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_rst/Server.java)

    public class Server {
        public static void main(String[] args) throws Exception {
            ServerSocket serverSocket = new ServerSocket(9999);
            Socket socket = serverSocket.accept();
            OutputStream out = socket.getOutputStream();
            while (true) {
                BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                String line = reader.readLine();
                System.out.println(">>>> process " + line);
                out.write("hello, this is server".getBytes());
            }
        }


使用`javac Server.java; javac -cp . Server`编译并运行服务端代码。

客户端代码如下，完整的代码见：[Client.java](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_rst/Client.java)

    public class Client {
        public static void main(String[] args) throws Exception {
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("c2", 9999));
    
            OutputStream out = socket.getOutputStream();
    
            System.out.println("start sleep. kill server process now!");


​            
            TimeUnit.SECONDS.sleep(5);
    
            System.out.println("start first write");
            
            out.write("hello".getBytes());
    
            TimeUnit.SECONDS.sleep(2);
            System.out.println("start second write");
            
            out.write("world".getBytes());
    
            System.in.read();
        }
    }


思路是先三次握手建连，然后马上 kill 掉服务端进程。客户端随后进行了两次 write，第一次 write 会触发服务端发送 RST 包，第二次 write 会抛出`Broken pipe`异常

    start sleep. kill server process now!
    start first write
    start second write
    Exception in thread "main" java.net.SocketException: Broken pipe
    	at java.net.SocketOutputStream.socketWrite0(Native Method)
    	at java.net.SocketOutputStream.socketWrite(SocketOutputStream.java:109)
    	at java.net.SocketOutputStream.write(SocketOutputStream.java:141)
    	at Client.main(Client.java:25)


抓包见下图，完整的 pcap 文件见：[broken\_pipe.pcap](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_rst/broken_pipe.pcap)

![](https://user-gold-cdn.xitu.io/2019/6/19/16b7073dc10282fd)

那 Broken pipe 到底是什么呢？这就要从 SIGPIPE 信号说起。

当一个进程向某个已收到 RST 的套接字执行写操作时，内核向该进程发送一个 SIGPIPE 信号。该信号的默认行为是终止进程，因此进程一般会捕获这个信号进行处理。不论该进程是捕获了该信号并从其信号处理函数返回，还是简单地忽略该信号，写操作都将返回 EPIPE 错误（也就Broken pipe 错误）,这也是 Broken pipe 只在写操作中出现的原因。

![](https://user-gold-cdn.xitu.io/2019/6/19/16b7073dca9493c8)

相比于 Broken pipe，Connection reset by peer 这个错误就更加容易出现一些了。一个最简单的方式是把上面代码中的第二次 write 改为 read，就会出现 `Connection reset`，完整的代码见：[Client2.java](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_rst/Client2.java)

运行日志如下：

    start sleep. kill server process now!
    start first write
    start second write
    Exception in thread "main" java.net.SocketException: Connection reset
    	at java.net.SocketInputStream.read(SocketInputStream.java:209)
    	at java.net.SocketInputStream.read(SocketInputStream.java:141)
    	at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
    	at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
    	at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
    	at java.io.InputStreamReader.read(InputStreamReader.java:184)
    	at java.io.BufferedReader.fill(BufferedReader.java:161)
    	at java.io.BufferedReader.readLine(BufferedReader.java:324)
    	at java.io.BufferedReader.readLine(BufferedReader.java:389)
    	at Client.main(Client.java:28)


小结
--

这篇文章主要介绍了 RST 包相关的内容，我们来回顾一下。首先介绍了 RST 出现常见的几种情况

*   端口未监听
*   连接信息丢失，另一方并不知道继续发送数据
*   SO\_LINGER 设置丢弃缓冲区数据，立刻 RST

然后介绍了两个场景的错误 Connection reset 和 Broken pipe 以及背后的原因，RST 包的案例后面还有一篇文章会介绍。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788837134344)