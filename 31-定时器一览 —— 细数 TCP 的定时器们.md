

TCP 为每条连接建立了 7 个定时器：

*   连接建立定时器
*   重传定时器
*   延迟 ACK 定时器
*   PERSIST 定时器
*   KEEPALIVE 定时器
*   FIN\_WAIT\_2 定时器
*   TIME\_WAIT 定时器

大部分定时器在前面的文章已经介绍过了，这篇文章来总结一下。

0x01 连接建立定时器（connection establishment）
--------------------------------------

当发送端发送 SYN 报文想建立一条新连接时，会开启连接建立定时器，如果没有收到对端的 ACK 包将进行重传。

可以用一个最简单的 packetdrill 脚本来模拟这个场景


    +0   socket(..., SOCK_STREAM, IPPROTO_TCP) = 3


​    
    +0 connect(3, ..., ...) = -1


抓包结果如下

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db345b63f36)

在我的电脑上，将重传 6 次（间隔 1s、2s、4s、8s、16s、32s），6 次重试以后放弃重试，connect 调用返回 -1，调用超时，

这个值是由/proc/sys/net/ipv4/tcp\_syn\_retries决定的， 在我的 Centos 机器上，这个值等于 6

整个过程如下：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db33ad849aa)

如果是用 Java 语言就会返回`java.net.ConnectException: Connection timed out`异常

0x02 重传定时器（retransmission）
--------------------------

第一个定时器讲的是连接建立没有收到 ACK 的情况，如果在发送数据包的时候没有收到 ACK 呢？这就是这里要讲的第二个定时器重传定时器。重传定时器在之前的文章中有专门一篇文章介绍，重传定时器的时间是动态计算的，取决于 RTT 和重传的次数。

还是用 packetdrill 脚本的方式来模拟

    0   socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
    +0  setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
    +0  bind(3, ..., ...) = 0
    +0  listen(3, 1) = 0


​    
    +0  < S 0:0(0) win 4000 <mss 1000>
    +0  > S. 0:0(0) ack 1 <...>
    +.1 < . 1:1(0) ack 1 win 4000
    +0  accept(3, ..., ...) = 4


​    
    +0  write(4, ..., 1000) = 1000


​    
​    
​    
    +0 `sleep 1000000`


抓包结果如下

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db34688732e)

重传时间间隔是指数级退避，直到达到 120s 为止，重传次数是15次（这个值由操作系统的 `/proc/sys/net/ipv4/tcp_retries2` 决定)，总时间将近 15 分钟。

整个过程如下图

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db344f0a4c2)

0x03 延迟 ACK 定时器
---------------

在 TCP 收到数据包以后在没有数据包要回复时，不马上回复 ACK。这时开启一个定时器，等待一段时间看是否有数据需要回复。如果期间有数据要回复，则在回复的数据中捎带 ACK，如果时间到了也没有数据要发送，则也发送 ACK。在 Centos7 上这个值为 40ms。这里在延迟确认章节有详细的介绍，不再展开。

![](https://user-gold-cdn.xitu.io/2019/5/29/16b040aecfbd6973)

0x04 坚持计时器（persist timer）
-------------------------

坚持计时器这个翻译真是很奇葩，下面我用 Persist 定时器来讲述。

Persist 定时器是专门为零窗口探测而准备的。我们都知道 TCP 利用滑动窗口来实现流量控制，当接收端 B 接收窗口为 0 时，发送端 A 此时不能再发送数据，发送端此时开启 Persist 定时器，超时后发送一个特殊的报文给接收端看对方窗口是否已经恢复，这个特殊的报文只有一个字节。

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db33d817bb4)

0x05 保活定时器（keepalive timer）
---------------------------

如果通信以后一段时间有再也没有传输过数据，怎么知道对方是不是已经挂掉或者重启了呢？于是 TCP 提出了一个做法就是在连接的空闲时间超过 2 小时，会发送一个探测报文，如果对方有回复则表示连接还活着，对方还在，如果经过几次探测对方都没有回复则表示连接已失效，客户端会丢弃这个连接。

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db3485c3a88)

0x06 FIN\_WAIT\_2 定时器
---------------------

四次挥手过程中，主动关闭的一方收到 ACK 以后从 FIN\_WAIT\_1 进入 FIN\_WAIT\_2 状态等待对端的 FIN 包的到来，FIN\_WAIT\_2 定时器的作用是防止对方一直不发送 FIN 包，防止自己一直傻等。这个值由`/proc/sys/net/ipv4/tcp_fin_timeout` 决定，在我的 Centos7 机器上，这个值为 60s

![](https://user-gold-cdn.xitu.io/2019/5/29/16b040aecfc3c926)

0x07 TIME\_WAIT 定时器
-------------------

TIME\_WAIT 定时器也称为 2MSL 定时器，可能是这七个里面名气最大的，主动关闭连接的一方在 TIME\_WAIT 持续 2 个 MSL 的时间，超时后端口号可被安全的重用。

![](https://user-gold-cdn.xitu.io/2019/5/29/16b03db46b133519)

TIME\_WAIT存在的意义有两个：

*   可靠的实现 TCP 全双工的连接终止（处理最后 ACK 丢失的情况）
*   避免当前关闭连接与后续连接混淆（让旧连接的包在网络中消逝）

小结
--

以上就是 TCP 的 7 个定时器的全部内容，每一个的细节都在之前的文章中有详细的介绍，如果有不太明白的地方可以翻阅


[Source](https://juejin.im/book/6844733788681928712/section/6844733788849700877)