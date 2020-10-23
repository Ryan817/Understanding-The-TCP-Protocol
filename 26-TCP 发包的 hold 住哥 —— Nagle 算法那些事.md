

从这篇文章开始，我们来讲大名鼎鼎的 Nagle 算法。同样以一个小测验来开始。

关于下面这段代码

    Socket socket = new Socket();
    socket.connect(new InetSocketAddress("localhost", 9999));
    OutputStream output = socket.getOutputStream();
    byte[] request = new byte[10];
    for (int i = 0; i < 5; i++) {
        output.write(request);
    }


说法正确的是：

*   A. TCP 把 5 个包合并，一次发送 50 个字节
*   B. TCP 分 5 次发送，一次发送 10 个字节
*   C. 以上都不对

来做一下实验，客户端代码如下

    public class NagleClient {
        public static void main(String[] args) throws Exception {
            Socket socket = new Socket();
            SocketAddress address = new InetSocketAddress("c1", 9999);
            socket.connect(address);
            OutputStream output = socket.getOutputStream();
            byte[] request = new byte[10];
            
            for (int i = 0; i < 5; i++) {
                output.write(request);
            }
            TimeUnit.SECONDS.sleep(1);
            socket.close();
        }
    }


服务端代码比较简单，可以直接用 `nc -l 9999` 启动一个 tcp 服务器 运行上面的 NagleClient，抓包如下

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eab60168bb9)

可以看到除了第一个包是单独发送，后面的四个包合并到了一起，所以文章开头的答案是 C

那为什么是这样的呢？这就是我们今天要讲的重点 Nagle 算法。

nagle 算法
--------

简单来讲 nagle 算法讲的是减少发送端频繁的发送小包给对方。

Nagle 算法要求，当一个 TCP 连接中有在传数据（已经发出但还未确认的数据）时，小于 MSS 的报文段就不能被发送，直到所有的在传数据都收到了 ACK。同时收到 ACK 后，TCP 还不会马上就发送数据，会收集小包合并一起发送。网上有人想象的把 Nagle 算法说成是「hold 住哥」，我觉得特别形象。

算法思路如下：

    if there is new data to send
      if the window size >= MSS and available data is >= MSS
        send complete MSS segment now
      else
        if there is unconfirmed data still in the pipe
          enqueue data in the buffer until an acknowledge is received
        else
          send data immediately
        end if
      end if
    end if


默认情况下 Nagle 算法都是启用的，Java 可以通过 `setTcpNoDelay(true);`来禁用 Nagle 算法。

还是上面的代码，修改代码开启 TCP\_NODELAY 禁用 Nagle 算法

    省略...
    Socket socket = new Socket();
    socket.setTcpNoDelay(true);
    省略...


再次抓包

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eab61eae538)

可以看到几乎同一瞬间分 5 次把数据发送了出去，不管之前发出去的包有没有收到 ACK。 Nagle 算法开启前后对比如下图所示

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eab67e29995)

用 packetdrill 来演示 Nagle 算法
--------------------------

如果不想写那么长的 Java 代码，可以用 packetdrill 代码来演示。同样的做法是发送端短时间内发送 5 个小包。先来看 Nagle 算法开启的情况

      1  --tolerance_usecs=100000
      2 0.000 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
      3 // 0.010 setsockopt(3, SOL_TCP, TCP_NODELAY, [1], 4) = 0
      4
      5 0.100...0.200 connect(3, ..., ...) = 0
      6
      7 // Establish a connection.
      8 0.100 > S 0:0(0) <mss 1460,sackOK,TS val 100 ecr 0,nop,wscale 7>
      9 0.200 < S. 0:0(0) ack 1 win 32792 <mss 1100,nop,wscale 7>
     10 0.200 > . 1:1(0) ack 1
     11
     12 +0 write(3, ..., 10) = 10
     13 +0 write(3, ..., 10) = 10
     14 +0 write(3, ..., 10) = 10
     15 +0 write(3, ..., 10) = 10
     16 +0 write(3, ..., 10) = 10
     17
     18  +0.030 < . 1:1(0) ack 11 win 257
     19  +0.030 < . 1:1(0) ack 21 win 257
     20  +0.030 < . 1:1(0) ack 31 win 257
     21  +0.030 < . 1:1(0) ack 41 win 257
     22  +0.030 < . 1:1(0) ack 51 win 257
     23
     24 +0 `sleep 1000000`


先注释掉第三行，关闭 TCP\_NODELAY，用 packetdrill 执行脚本`sudo packetdrill nagle.pkt`抓包结果如下

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eab6fb73b7b)

结果如我们预期，第一个包正常发送，等第 1 次包收到 ACK 回复以后，后面的 4 次包合并在一起发送出去。

现在去掉第三行的注释，禁用 Nagle 算法，重新运行抓包

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eab64a55a0c)

可以看到这次发送端没有等对端回复 ACK，就把所有的小包一个个发出去了。

一个典型的小包场景：SSH
-------------

一个典型的大量小包传输的场景是用 ssh 登录另外一台服务器，每输入一个字符，服务端也随即进行回应，客户端收到了以后才会把输入的字符和响应的内容显示在自己这边。比如登录服务器后输入`ls`然后换行，中间包交互的过程如下图

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eab763863bd)

1.  客户端输入`l`，字符 `l` 被加密后传输给服务器
2.  服务器收到`l`包，回复被加密的 `l` 及 ACK
3.  客户端输入`s`，字符 `s` 被加密后传输给服务器
4.  服务器收到`s`包，回复被加密的 `s` 及 ACK
5.  客户端输入 enter 换行符，换行符被加密后传输给服务器
6.  服务器收到换行符，回复被加密的换行符及 ACK
7.  服务端返回执行 ls 的结果
8.  客户端回复 ACK

Nagle 算法的意义在哪里
--------------

Nagle 算法的作用是减少小包在客户端和服务端直接传输，一个包的 TCP 头和 IP 头加起来至少都有 40 个字节，如果携带的数据比较小的话，那就非常浪费了。就好比开着一辆大货车运一箱苹果一样。

![](https://user-gold-cdn.xitu.io/2019/4/23/16a49eac0e76757b)

Nagle 算法在通信时延较低的场景下意义不大。在 Nagle 算法中 ACK 返回越快，下次数据传输就越早。

假设 RTT 为 10ms 且没有延迟确认（这个后面会讲到），那么你敲击键盘的间隔大于 10ms 的话就不会触发 Nagle 的条件：只有接收到所有的在传数据的 ACK 后才能继续发数据，也即如果所有的发出去的包 ACK 都收到了，就不用等了。如果你想触发 Nagle 的停等（stop-wait）机制，1s 内要输入超过 100 个字符。因此如果在局域网内，Nagle 算法基本上没有什么效果。

如果客户端到服务器的 RTT 较大，比如多达 200ms，这个时候你只要1s 内输入超过 5 个字符，就有可能触发 Nagle 算法了。

**Nagle 算法是时代的产物**：Nagle 算法出现的时候网络带宽都很小，当有大量小包传输时，很容易将带宽占满，出现丢包重传等现象。因此对 ssh 这种交互式的应用场景，选择开启 Nagle 算法可以使得不再那么频繁的发送小包，而是合并到一起，代价是稍微有一些延迟。现在的 ssh 客户端已经默认关闭了 Nagle 算法。

小结
--

这篇文章主要介绍了非常经典的 Nagle 算法，这个算法可以有效的减少网络上小包的数量。Nagle 算法是应用在发送端的，简而言之就是，对发送端而言：

*   当第一次发送数据时不用等待，就算是 1byte 的小包也立即发送
*   后面发送数据时需要累积数据包直到满足下面的条件之一才会继续发送数据：
    *   数据包达到最大段大小MSS
    *   接收端收到之前数据包的确认 ACK

不过 Nagle 算法是时代的产物，可能会导致较多的性能问题，尤其是与我们下一篇文章要介绍的延迟确认一起使用的时候。很多组件为了高性能都默认禁用掉了这个特性。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788841328648)