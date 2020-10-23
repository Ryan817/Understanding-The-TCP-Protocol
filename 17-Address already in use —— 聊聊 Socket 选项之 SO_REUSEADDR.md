

前面介绍到四次挥手的时候有讲到，**主动断开**连接的那一端需要等待 2 个 MSL 才能最终释放这个连接。一般而言，主动断开连接的都是客户端，如果是服务端程序重启或者出现 bug 崩溃，这时服务端会主动断开连接，如下图所示

![](https://user-gold-cdn.xitu.io/2019/2/25/169230452f26de90)

因为要等待 2 个 MSL 才能最终释放连接，所以如果这个时候程序马上启动，就会出现`Address already in use`错误。要过 1 分钟以后才可以启动成功。如果你写了一个 web 服务器，崩溃以后被脚本自动拉起失败，需要等一分钟才正常，运维可能要骂娘了。

下面来写一段简单的代码演示这个场景是如何产生的。

    public class ReuseAddress {
        public static void main(String[] args) throws IOException {
            ServerSocket serverSocket = new ServerSocket();
            
            serverSocket.setReuseAddress(false);
            serverSocket.bind(new InetSocketAddress(8080));
            System.out.println("reuse address: " + serverSocket.getReuseAddress());
            while (true) {
                Socket socket = serverSocket.accept();
                System.out.println("incoming socket..");
                OutputStream out = socket.getOutputStream();
                out.write("Hello\n".getBytes());
                out.close();
            }
        }
    }


这段代码的功能是启动一个 TCP 服务器，客户端连上来就返回了一个 "Hello\\n" 回去。

使用 javac 编译 class 文件`javac ReuseAddress.java;`，然后用 java 命令运行`java -cp . ReuseAddress`。使用 nc 命令连接 8080 端口`nc localhost 8080`，应该会马上收到服务端返回的"Hello\\n"字符串。现在 kill 这个进程，马上重启这个程序就可以看到程序启动失败，报 socket bind 失败，堆栈如下：

    Exception in thread "main" java.net.BindException: 地址已在使用 (Bind failed)
    	at java.net.PlainSocketImpl.socketBind(Native Method)
    	at java.net.AbstractPlainSocketImpl.bind(AbstractPlainSocketImpl.java:387)
    	at java.net.ServerSocket.bind(ServerSocket.java:375)
    	at java.net.ServerSocket.bind(ServerSocket.java:329)
    	at ReuseAddress.main(ReuseAddress.java:18)


将代码修改为`serverSocket.setReuseAddress(true);`，再次重复上面的测试过程，再也不会出现上述异常了。

0x02 为什么需要 SO\_REUSEADDR 参数
---------------------------

服务端主动断开连接以后，需要等 2 个 MSL 以后才最终释放这个连接，重启以后要绑定同一个端口，默认情况下，操作系统的实现都会阻止新的监听套接字绑定到这个端口上。

我们都知道 TCP 连接由四元组唯一确定。形式如下

{local-ip-address:local-port , foreign-ip-address:foreign-port}

一个典型的例子如下图

![](https://user-gold-cdn.xitu.io/2019/2/25/169230452a3ad54a)

TCP 要求这样的四元组必须是唯一的，但大多数操作系统的实现要求更加严格，只要还有连接在使用这个本地端口，则本地端口不能被重用（bind 调用失败）

启用 SO\_REUSEADDR 套接字选项可以解除这个限制，默认情况下这个值都为 0，表示关闭。在 Java 中，reuseAddress 不同的 JVM 有不同的实现，在我本机上，这个值默认为 1 允许端口重用。但是为了保险起见，写 TCP、HTTP 服务一定要主动设置这个参数为 1。

0x03 是不是只有处于 TIME\_WAIT 才允许端口复用？
--------------------------------

查看 Java 中 ServerSocket.setReuseAddress 的文档，有如下的说明

    /**
     * Enable/disable the {@link SocketOptions
     * socket option.
     * <p>
     * When a TCP connection is closed the connection may remain
     * in a timeout state for a period of time after the connection
     * is closed (typically known as the {@code TIME_WAIT} state
     * or {@code 2MSL} wait state).
     * For applications using a well known socket address or port
     * it may not be possible to bind a socket to the required
     * {@code SocketAddress} if there is a connection in the
     * timeout state involving the socket address or port.
    * /


假设因为网络的原因，客户端没有回发 FIN 包，导致服务器端处于 FIN\_WAIT2 状态，而非 TIME\_WAIT 状态，那设置 SO\_REUSEADDR 还会生效吗？

![](https://user-gold-cdn.xitu.io/2019/2/25/169252c596fbaac0)

来做一个实验，现在有两台机器c1（充当客户端），c2（充当服务器）。在客户端 c1 利用防火墙拦截掉所有发出的 FIN 包：`sudo iptables --append OUTPUT --match tcp --protocol tcp --dport 8080 --tcp-flags FIN FIN --jump DROP`。 在c1 上使用`nc c2 8080`发起 tcp 连接，随后杀掉 c2 的进程， 因为服务端收不到客户端发过来的 FIN 包，也即四次挥手中的第 3 步没能成功，服务端此时将处于 FIN\_WAIT2 状态。

    ya@c2 ~$ sudo netstat -lnpa  | grep 8080
    tcp6       0      0 10.211.55.10:8080       10.211.55.5:39664       FIN_WAIT2   -


将 SO\_REUSEADDR 设置为 1，重复上面的测试过程，将发现不会出现异常。将 SO\_REUSEADDR 设置为 0，则会出现 Address already in use 异常。

因此，不一定是要处于 TIME\_WAIT 才允许端口复用的，只是大都是情况下，主动关闭连接的服务端都会处于 TIME\_WAIT。如果不把 SO\_REUSEADDR 设置为 1，服务器将等待 2 个 MSL 才可以重新绑定原端口

0x04 为什么通常不会在客户端上出现
-------------------

通常情况下都是客户端主动关闭连接，那客户端那边为什么不会有问题呢？

因为客户端都是用的临时端口，这些临时端口与处于 TIME\_WAIT 状态的端口恰好相同的可能性不大，就算相同换一个新的临时端口就好了。

小结
--

这篇文章主要讲了 SO\_REUSEADDR 套接字属性出现的背景和分析，随后讲解了为什么需要 SO\_REUSEADDR 参数，以及为什么客户端不需要关心这个参数。

如果你看这篇文章有什么疑问，欢迎你在留言区留言。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788832940039)