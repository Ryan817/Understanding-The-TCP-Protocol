

这篇文章我们来讲一个新的参数 SO\_LINGER，以一个小测验来开始今天的文章。 请看下面的代码：

    Socket socket = new Socket();
    InetSocketAddress serverSocketAddress = new InetSocketAddress("10.0.0.3", 8080);
    socket.connect(serverSocketAddress);
    
    byte[] msg = getMessageBytes(); 
    socket.getOutputStream().write(msg);
    
    socket.close();


会发现如下哪个选项的事情

1.  服务器收到 msg 所有内容
2.  服务器会收到 msg 部分内容
3.  服务器会抛出异常

简化为图如下：

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b90d9589384)

当我们调用 write 函数向内核写入一段数据时，内核会把这段时间放入一个缓冲区 buffer，如下图所示

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b90d978ff34)

关闭连接的两种方式
---------

前面有介绍过有两种方式可以关闭 TCP 连接

*   FIN：优雅关闭，发送 FIN 包表示自己这端所有的数据都已经发送出去了，后面不会再发送数据
*   RST：强制连接重置关闭，无法做出什么保证

当调用 socket.close() 的时候会发生什么呢？

正常情况下

*   操作系统等所有的数据发送完才会关闭连接
*   因为是主动关闭，所以连接将处于 TIME\_WAIT 两个 MSL

前面说了正常情况，那一定有不正常的情况下，如果我们不想等那么久才彻底关闭这个连接怎么办，这就是我们这篇文章介绍的主角 SO\_LINGER

SO\_LINGER
----------

Linux 的套接字选项SO\_LINGER 用来改变socket 执行 close() 函数时的默认行为。

linger 的英文释义有逗留、徘徊、继续存留、缓慢消失的意思。这个释义与这个参数真正的含义很接近。

SO\_LINGER 启用时，操作系统开启一个定时器，在定时器期间内发送数据，定时时间到直接 RST 连接。

SO\_LINGER 参数是一个 linger 结构体，代码如下

    struct linger {
        int l_onoff;    
        int l_linger;   
    };


第一个字段 l\_onoff 用来表示是否启用 linger 特性，非 0 为启用，0 为禁用 ，linux 内核默认为禁用。这种情况下 close 函数立即返回，操作系统负责把缓冲队列中的数据全部发送至对端

第二个参数 l\_linger 在 l\_onoff 为非 0 （即启用特性）时才会生效。

*   如果 l\_linger 的值为 0，那么调用 close，close 函数会立即返回，同时丢弃缓冲区内所有数据并立即发送 RST 包重置连接
*   如果 l\_linger 的值为非 0，那么此时 close 函数在阻塞直到 l\_linger 时间超时或者数据发送完毕，发送队列在超时时间段内继续尝试发送，如果发送完成则皆大欢喜，超时则直接丢弃缓冲区内容 并 RST 掉连接。

实验时间
----

我们用一个例子来说明上面的三种情况。

服务端代码如下，监听 9999 端口，收到客户端发过来的数据不做任何处理。

    import java.util.Date;
    public class Server {
    
        public static void main(String[] args) throws Exception {
            ServerSocket serverSocket = new ServerSocket();
            serverSocket.setReuseAddress(true);
            serverSocket.bind(new InetSocketAddress(9999));
    
            while (true) {
                Socket socket = serverSocket.accept();
                InputStream input = socket.getInputStream();
                ByteArrayOutputStream output = new ByteArrayOutputStream();
                byte[] buffer = new byte[1];
                int length;
                while ((length = input.read(buffer)) != -1) {
                    output.write(buffer, 0, length);
                }
                String req = new String(output.toByteArray(), "utf-8");
                System.out.println(req.length());
                socket.close();
            }
        }
    }


客户端代码如下，客户端往服务器发送 1000 个 "hel" 字符，代码最后输出了 close 函数调用的耗时

    import java.net.SocketAddress;
    
    public class Client {
        private static int PORT = 9999;
        private static String HOST = "c1";
    
        public static void main(String[] args) throws Exception {
            Socket socket = new Socket();
            
            socket.setSoLinger(false, 0);


​            
​            
​            
​    
            SocketAddress address = new InetSocketAddress(HOST, PORT);
            socket.connect(address);
    
            OutputStream output = socket.getOutputStream();
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < 10000; i++) {
                sb.append("hel");
            }
            byte[] request = sb.toString().getBytes("utf-8");
            output.write(request);
            long start = System.currentTimeMillis();
            socket.close();
            long end = System.currentTimeMillis();
            System.out.println("close time cost: " + (end - start));
        }
    }


> 情况#1 `socket.setSoLinger(false, 0)`

这个是默认的行为，close 函数立即返回，且服务器应该会收到所有的 30kB 的数据。运行代码同时 wireshark 抓包，客户端输出 close 的耗时为

    close time cost: 0


wireshark 抓包情况如下，可以看到完成正常四次挥手

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b90d97911b6)

整个发送的包大小为 30kB

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b90d96bca36)

> 情况#2 `socket.setSoLinger(true, 0)` 这种情况下，理论上 close 函数应该立刻返回，同时丢弃缓冲区的内容，可能服务端收到的数据只是部分的数据。

客户端终端的输出如下：

    close time cost: 0


服务端抛出了异常，输出如下：

    Exception in thread "main" java.net.SocketException: Connection reset
    	at java.net.SocketInputStream.read(SocketInputStream.java:210)
    	at java.net.SocketInputStream.read(SocketInputStream.java:141)
    	at java.net.SocketInputStream.read(SocketInputStream.java:127)
    	at Server.main(Server.java:21)


通过 wireshark 抓包如下：

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b90e76e83ec)

可以看到，没有执行正常的四次挥手，客户端直接发送 RST 包，重置了连接。

传输包的大小也没有30kB，只有14kB，说明丢弃了内核缓冲区的 16KB 的数据。

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b90ea987de8)

**情况#3 `socket.setSoLinger(true, 1);`**

这种情况下，close 函数不会立刻返回，如果在 1s 内数据传输结束，则皆大欢喜，如果在 1s 内数据没有传输完，就直接丢弃掉，同时 RST 连接

运行代码，客户端输出显示 close 函数耗时 17ms，不再是前面两个例子中的 0 ms 了。

    close time cost: 17


通过 wireshark 抓包可以看到完成了正常的四次挥手

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b91181c0190)

小结
--

这篇文章主要介绍了 SO\_LINGER 套接字选项对关闭套接字的影响。默认行为下是调用 close 立即返回，但是如果有数据残留在套接字发送缓冲区中，系统将试着把这些数据发送给对端，SO\_LINGER 可以改变这个默认设置，具体的规则见下面的思维导图。

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02b91ae14ef21)


[Source](https://juejin.im/book/6844733788681928712/section/6844733788832923656)