

这篇文章我们来介绍延迟确认。

首先必须明确两个观点：

*   不是每个数据包都对应一个 ACK 包，因为可以合并确认。
*   也不是接收端收到数据以后必须立刻马上回复确认包。

如果收到一个数据包以后暂时没有数据要分给对端，它可以等一段时间（Linux 上是 40ms）再确认。如果这段时间刚好有数据要传给对端，ACK 就可以随着数据一起发出去了。如果超过时间还没有数据要发送，也发送 ACK，以免对端以为丢包了。这种方式成为「延迟确认」。

这个原因跟 Nagle 算法其实一样，回复一个空的 ACK 太浪费了。

*   如果接收端这个时候恰好有数据要回复客户端，那么 ACK 搭上顺风车一块发送。
*   如果期间又有客户端的数据传过来，那可以把多次 ACK 合并成一个立刻发送出去
*   如果一段时间没有顺风车，那么没办法，不能让接收端等太久，一个空包也得发。

这种机制被称为延迟确认（delayed ack），思破哥的文章把延迟确认（delayed-ack）称为「**磨叽姐**」，挺形象的。TCP 要求 ACK 延迟的时延必须小于500ms，一般操作系统实现都不会超过200ms。

延迟确认在很多 linux 机器上是没有办法关闭的，

那么这里涉及的就是一个非常根本的问题：「收到数据包以后什么时候该回复 ACK」

什么时候需要回复 ACK
------------

[tcp\_input.c](https://elixir.bootlin.com/linux/v2.6.11/source/net/ipv4/tcp_input.c)

    static void __tcp_ack_snd_check(struct sock *sk, int ofo_possible)
    {
    	struct tcp_sock *tp = tcp_sk(sk);
    
    	    /* More than one full frame received... */
    	if (((tp->rcv_nxt - tp->rcv_wup) > tp->ack.rcv_mss
    	     /* ... and right edge of window advances far enough.
    	      * (tcp_recvmsg() will send ACK otherwise). Or...
    	      */
    	     && __tcp_select_window(sk) >= tp->rcv_wnd) ||
    	    /* We ACK each frame or... */
    	    tcp_in_quickack_mode(tp) ||
    	    /* We have out of order data. */
    	    (ofo_possible &&
    	     skb_peek(&tp->out_of_order_queue))) {
    		/* Then ack it now */
    		tcp_send_ack(sk);
    	} else {
    		/* Else, send delayed ack. */
    		tcp_send_delayed_ack(sk);
    	}
    }


可以看到需要立马回复 ACK 的场景有：

*   如果接收到了大于一个frame 的报文，且需要调整窗口大小
*   处于 quickack 模式（tcp\_in\_quickack\_mode）
*   收到乱序包（We have out of order data.）

其它情况一律使用延迟确认的方式

需要重点关注的是：tcp\_in\_quickack\_mode()

    /* Send ACKs quickly, if "quick" count is not exhausted
     * and the session is not interactive.
     */
    
    static __inline__ int tcp_in_quickack_mode(struct tcp_sock *tp)
    {
    	return (tp->ack.quick && !tp->ack.pingpong);
    }
    
    /* Delayed ACK control data */
    struct {
    	__u8	pending;	/* ACK is pending */
    	__u8	quick;		/* Scheduled number of quick acks	*/
    	__u8	pingpong;	/* The session is interactive		*/
    	__u8	blocked;	/* Delayed ACK was blocked by socket lock*/
    	__u32	ato;		/* Predicted tick of soft clock		*/
    	unsigned long timeout;	/* Currently scheduled timeout		*/
    	__u32	lrcvtime;	/* timestamp of last received data packet*/
    	__u16	last_seg_size;	/* Size of last incoming segment	*/
    	__u16	rcv_mss;	/* MSS used for delayed ACK decisions	*/ 
    } ack;


内核 tcp\_sock 结构体中有一个 ack 子结构体，内部有一个 quick 和 pingpong 两个字段，其中pingpong 就是判断交互连接的，只有处于非交互 TCP 连接才有可能即进入 quickack 模式。

什么是交互式和 pingpong 呢？

顾名思义，其实有来有回的双向数据传输就叫 pingpong，对于通信的某一端来说，`R-W-R-W-R-W...`（R 表示读，W 表示写）

延迟确认出现的最多的场景是 `W-W-R`（写写读），我们来分析一下这种场景。

延迟确认实际例子演示
----------

可以用一段 java 代码演示延迟确认。

服务端代码如下，当从服务端 readLine 有返回非空字符串（读到`\n 或 \r`）就把字符串原样返回给客户端

    public class DelayAckServer {
        private static final int PORT = 8888;
    
        public static void main(String[] args) throws IOException {
            ServerSocket serverSocket = new ServerSocket();
            serverSocket.bind(new InetSocketAddress(PORT));
            System.out.println("Server startup at " + PORT);
            while (true) {
                Socket socket = serverSocket.accept();
                InputStream inputStream = socket.getInputStream();
                OutputStream outputStream = socket.getOutputStream();
                int i = 1;
                while (true) {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
                    String line = reader.readLine();
                    if (line == null) break;
                    System.out.println((i++) + " : " + line);
                    outputStream.write((line + "\n").getBytes());
                }
            }
        }
    }


下面是客户端代码，客户端分两次调用 write 方法，模拟 http 请求的 header 和 body。第二次 write 包含了换行符（\\n)，然后测量 write、write、read 所花费的时间。

    public class DelayAckClient {
        public static void main(String[] args) throws IOException {
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("server_ip", 8888));
            InputStream inputStream = socket.getInputStream();
            OutputStream outputStream = socket.getOutputStream();
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
            String head = "hello, ";
            String body = "world\n";
    
            for (int i = 0; i < 10; i++) {
                long start = System.currentTimeMillis();
                outputStream.write(("#" + i + " " + head).getBytes()); 
                outputStream.write((body).getBytes()); 
                String line = reader.readLine(); 
                System.out.println("RTT: " + (System.currentTimeMillis() - start) + ": " + line);
            }
            inputStream.close();
            outputStream.close();
            socket.close();
        }
    }


运行结果如下

    javac DelayAckClient.java; java -cp . DelayAckClient
    RTT: 1: 
    RTT: 44: 
    RTT: 46: 
    RTT: 44: 
    RTT: 42: 
    RTT: 41: 
    RTT: 41: 
    RTT: 44: 
    RTT: 44: 
    RTT: 44: 


除了第一次，剩下的 RTT 全为 40 多毫秒。这刚好是 Linux 延迟确认定时器的时间 40ms 抓包结果如下:

![](https://user-gold-cdn.xitu.io/2019/4/9/16a006acf0a4e73f)

对包逐个分析一下 1 ~ 3：三次握手 4 ~ 9：第一次 for 循环的请求，也就是 W-W-R 的过程

*   4：客户端发送 "#0 hello, " 给服务端
*   5：因为服务端只收到了数据还没有回复过数据，tcp 判断不是 pingpong 的交互式数据，属于 quickack 模式，立刻回复 ACK
*   6：客户端发送 "world\\n" 给服务端
*   7：服务端因为还没有回复过数据，tcp 判断不是 pingpong 的交互式数据，服务端立刻回复 ACK
*   8：服务端读到换行符，readline 函数返回，会把读到的字符串原样写入到客户端。TCP 这个时候检测到是 pingpong 的交互式连接，进入延迟确认模式
*   9：客户端收到数据以后回复 ACK

10 ~ 14：第二次 for 循环

*   10：客户端发送 "#1 hello, " 给服务端。服务端收到数据包以后，因为处于 pingpong 模式，开启一个 40ms 的定时器，奢望在 40ms 内有数据回传
*   11：很不幸，服务端等了 40ms 定期器到期都没有数据回传，回复确认 ACK 同时取消 pingpong 状态
*   12：客户端发送 "world\\n" 给服务端
*   13：因为服务端不处于 pingpong 状态，所以收到数据立即回复 ACK
*   14：服务端读到换行符，readline 函数返回，会把读到的字符串原样写入到客户端。这个时候又检测到收发数据了，进入 pingpong 状态。

从第二次 for 开始，后面的数据包都一样了。 整个过程包交互图如下：

![](https://user-gold-cdn.xitu.io/2019/4/9/16a006ace9ddc4ef)

用 packetdrill 模拟延迟确认
--------------------

    --tolerance_usecs=100000
    0.000 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
    0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
    0.000 bind(3, ..., ...) = 0
    0.000 listen(3, 1) = 0
    
    0.000 < S 0:0(0) win 32792 <mss 1000, sackOK, nop, nop, nop, wscale 7>
    0.000 > S. 0:0(0) ack 1 <...>
    
    0.000 < . 1:1(0) ack 1 win 257
    
    0.000 accept(3, ..., ...) = 4
    
    + 0 setsockopt(4, SOL_TCP, TCP_NODELAY, [1], 4) = 0
    
    // 模拟往服务端写入 HTTP 头部: POST / HTTP/1.1
    +0 < P. 1:11(10) ack 1 win 257
    
    // 模拟往服务端写入 HTTP 请求 body: {"id": 1314}
    +0 < P. 11:26(15) ack 1 win 257
    
    // 往 fd 为4 的 模拟服务器返回 HTTP response {}
    + 0 write(4, ..., 100) = 100


​    
    // 第二次模拟往服务端写入 HTTP 头部: POST / HTTP/1.1
    +0 < P. 26:36(10) ack 101 win 257
    
    // 抓包看服务器返回
    
    +0 `sleep 1000000`


这个构造包的过程跟前面的思路是一模一样的，抓包同样复现了 40ms 延迟的现象。

![](https://user-gold-cdn.xitu.io/2019/4/9/16a006acecf83ba7)

可以设置关掉延迟确认吗？
------------

这个是我刚开始学习 TCP 的一个疑惑，既然是 TCP 的一个特性，那有没有一个开关可以开启或者关闭延迟确认呢？ 答案是否定的，大部分 Linux 实现上并没有开关可以关闭延迟确认。我曾经以为它是一个 sysctl 项，可是后来找了很久都没有找到，没有办法通过一个配置彻底关掉或者开启 Linux 的延迟确认。

当 Nagle 算法遇到延迟确认
----------------

Nagle 算法和延迟确认本身并没有什么问题，但一起使用就会出现很严重的性能问题了。Nagle 攒着包一次发一个，延迟确认收到包不马上回。

如果我们把上面的 Java 代码稍作调整，禁用 Nagle 算法可以试一下。

    Socket socket = new Socket();
    socket.setTcpNoDelay(true); // 禁用 Nagle 算法
    socket.connect(new InetSocketAddress("server ip", 8888));


运行 Client 端，可以看到 RTT 几乎为 0

    RTT: 1: 
    RTT: 0: 
    RTT: 1: 
    RTT: 1: 
    RTT: 0: 
    RTT: 1: 
    RTT: 1: 
    RTT: 0: 
    RTT: 1: 
    RTT: 0: 


抓包结果如下

![](https://user-gold-cdn.xitu.io/2019/4/9/16a006aceed2734f)

黑色背景部分的是客户端发送给服务端的请求包，可以看到在禁用 Nagle 的情况下，不用等一个包发完再发下一个，而是几乎同时把两次写请求发送出来了。服务端收到带换行符的包以后，立马可以返回结果，ACK 可以捎带过去，就不会出现延迟 40ms 的情况。

小结
--

这篇文章主要介绍了延迟确认出现的背景和原因，然后用一个实际的代码演示了延迟确认的具体的细节。到这里 Nagle 算法和延迟确认这两个主题就介绍完毕了。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788845506573)