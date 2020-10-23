

前面的文章中介绍过一个应用层的数据包会经过传输层、网络层的层层包装，交给网络接口层传输。假设上层的应用调用 write 等函数往 socket 写入了 10KB 的数据，TCP 会如何处理呢？是直接加上 TCP 头直接交给网络层吗？这篇文章我们来讲讲这相关的知识

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e12cc8358)

最大传输单元（Maximum Transmission Unit, MTU）
--------------------------------------

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e208c4f86)

数据链路层传输的帧大小是有限制的，不能把一个太大的包直接塞给链路层，这个限制被称为「最大传输单元（Maximum Transmission Unit, MTU）」

下图是以太网的帧格式，以太网的帧最小的帧是 64 字节，除去 14 字节头部和 4 字节 CRC 字段，有效荷载最小为 46 字节。最大的帧是 1518 字节，除去 14 字节头部和 4 字节 CRC，有效荷载最大为 1500，这个值就是以太网的 MTU。因此如果传输 100KB 的数据，至少需要 （100 \* 1024 / 1500) = 69 个以太网帧。

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e260cd0cd)

不同的数据链路层的 MTU 是不同的。通过`netstat -i` 可以查看网卡的 mtu，比如在 我的 centos 机器上可以看到

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e147f6672)

IP 分段
-----

IPv4 数据报的最大大小为 65535 字节，这已经远远超过了以太网的 MTU，而且有些网络还会开启巨帧（Jumbo Frame）能达到 9000 字节。 当一个 IP 数据包大于 MTU 时，IP 会把数据报文进行切割为多个小的片段(小于 MTU），使得这些小的报文可以通过链路层进行传输

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e1987f568)

IP 头部中有一个表示分片偏移量的字段，用来表示该分段在原始数据报文中的位置，如下图所示

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e185162dc)

下面我们 wireshark 来演示 IP 分段，wireshark 开启抓包，在命令行中执行

    ping -s 3000 www.baidu.com
    
    输出：
    PING www.a.shifen.com (14.215.177.39): 3000 data bytes
    Request timeout for icmp_seq 0
    Request timeout for icmp_seq 1
    Request timeout for icmp_seq 2


在 wireshark 的显示过滤器中输入`ip.addr==14.215.177.39`

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e4beb82ae)

通过`man ping`命令可以看到`ping -s`命令会增加 8byte 的 ICMP 头，所以`ping -s 3000` IP 层实际会发送 3008 字节。

> \-s packetsize Specify the number of data bytes to be sent. The default is 56, which translates into 64 ICMP data bytes when combined with the 8 bytes of ICMP header data. This option cannot be used with ping sweeps.

先看第一个包

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e4ee94a57)

这个包是 IP 分段包的第一个分片，`More fragments: Set`表示这个包是 IP 分段包的一部分，还有其它的分片包，`Fragment offset: 0`表示分片偏移量为 0，IP 包的 payload 的大小为 1480，加上 20 字节的头部正好是 1500

第二个包的详情截图如下

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e4dcc0174)

同样`More fragments`处于 set 状态，表示后面还有其它分片，`Fragment offset: 185`这里并不是表示分片偏移量为 185，wireshark 这里显示的时候除以了 8，真实的分片偏移量为 185 \* 8 = 1480

第三个包的详情截图如下

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e51db4162)

可以看到`More fragments`处于 Not set 状态，表示这是最后一个分片了。`Fragment offset: 370`表示偏移量为 370 \* 8 = 2960，包的大小为 68 - 20（IP 头部大小） = 48

三个分片如下图所示

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e54362613)

前面我们提到 IP 协议不会对丢包进行重传，那么 IP 分段中有分片丢失、损坏的话，会发生什么呢？ 这种情况下，目标主机将没有办法将分段的数据包重组为一个完整的数据包，依赖于传输层是否进行重传。

利用 IP 包分片的策略，有一种对应的网络攻击方式`IP fragment attack`，就是一直传`More fragments = 1`的包，导致接收方一直缓存分片，从而可能导致接收方内存耗尽。

网络中的木桶效应：路径 MTU
---------------

一个包从发送端传输到接收端，中间要跨越很多个网络，每条链路的 MTU 都可能不一样，这个通信过程中最小的 MTU 称为「路径 MTU（Path MTU）」。就好比开车有时候开的是双向 4 车道，有时候可能是乡间小路一样。

比如下图中，第一段链路 MTU 大小为 1500 字节，第二段链路 MTU 为 800 字节，第三段链路 MTU 为 1200 字节，则路径 MTU 为三段 MTU 的最小值 800。

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e55a91949)

路径 MTU 就跟木桶效应是一个道理，木桶的盛水量由最短的那条短板决定，路径 MTU 也是由通信链条中最小的 MTU 决定。

实际模拟路径 MTU 发现
-------------

用下面的代码可以用来测试路径 MTU 发现，为了方便，每行前面加了行号

    0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
    0.000 bind(3, ..., ...) = 0
    0.000 listen(3, 1) = 0
    
    0.100 < S 0:0(0) win 32792 <mss 1460,nop,wscale 7>
    0.100 > S. 0:0(0) ack 1 <mss 1460,nop,wscale 7>
    0.200 < . 1:1(0) ack 1 win 257
    0.200 accept(3, ..., ...) = 4


​    
​    
    +0.2 write(4, ..., 1460) = 1460
    
    +0.0 > P. 1:1461(1460) ack 1


​    
    +0.01 < icmp unreachable frag_needed mtu 1200 [1:1461(1460)]


​    
    +.0 > . 1:1161(1160) ack 1
    +0.0> P. 1161:1461(300) ack 1


​    
    +0.1 < . 1:1(0) ack 1461 win 257
    
    +0 `sleep 1000000`


其中在发送了 1460 大小的数据以后，这第一个数据包在 IP 层设置了不分段，之后收到一个 ICMP 告知的报文过大错误

运行抓包如下图

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e84446756)

*   1 ~ 3：三次握手
*   4：发送长度为 1460 的数据，这个数据包设置了不允许分片`Don't fragment: Set`
*   5：发送端收到 ICMP 包，告知包太大需要分片，下一个分片的大小按照 MTU=1200 来计算
    
    ![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e81b1e6df)
    
*   6：TCP 为了避免底层分片立刻拆包重发数据包，这次包大小为 1200 - 40 = 1160
*   7：发送端发送剩下的 300 字节（1460 - 1160）
*   8：确认所有的数据

整个过程如下图所示

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e887eea11)

因为有 MTU 的存在，TCP 每次发包的大小也限制了，这就是下面要介绍的 MSS。

TCP 最大段大小（Max Segment Size，MSS）
-------------------------------

TCP 为了避免被发送方分片，会主动把数据分割成小段再交给网络层，最大的分段大小称之为 MSS（Max Segment Size）。

    MSS = MTU - IP header头大小 - TCP 头大小


这样一个 MSS 的数据恰好能装进一个 MTU 而不用分片。

在以太网中 TCP 的 MSS = 1500（MTU） - 20（IP 头大小） - 20（TCP 头大小）= 1460

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e8c79596f)

我们来抓一个包来实际看一下，下面是下载一个 png 图片的 http 请求包 当三次握手建立一个 TCP 连接时，通信的双方会在 SYN 报文里说明自己允许的最大段大小。

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e8a06ec69)

可以看到 TCP 的包体数据大小为 1448，因为TCP 头部里包含了 12 字节的选项（Options）字段，头部大小从之前的 20 字节变为了 32 字节，所以 TCP 包体大小变为了：1500（以太网 MTU） - 20（IP 固定表头大小） - 20（TCP 固定表头大小） - 12（TCP 表头选项） = 1448

为什么有时候抓包看到的单个数据包大于 MTU
----------------------

写一个简单的代码来测试一下。

在服务端（10.211.55.10）使用`nc -l 9999` 启动一个 tcp 服务器

    nc -l 9999


在一台机器（10.211.55.5）记为 c1，使用 tcpdump 抓包开启抓包

    sudo tcpdump -i any port 9999 -nn


执行下面的 java 代码，往服务端 c2 写 100KB 的数据

    Socket socket = new Socket();
    socket.connect(new InetSocketAddress("c2", 9999));
    OutputStream out = socket.getOutputStream();
    byte[] bytes= new byte[100 * 1024];
    out.write(bytes);
    System.in.read();


抓包文件显示如下

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73e93840f8d)

可以看到包的长度达到了 14k，远超 MTU 的大小，为什么可以这样呢？

这就要说到 TSO（TCP Segment Offload）特性了，TSO 特性是指由网卡代替 CPU 实现 packet 的分段和合并，节省系统资源，因此 TCP 可以抓到超过 MTU 的包，但是不是真正传输的单个包会超过链路的 MTU。

使用`ethtool -k`可以查看这个特性是否打开，比如`ethtool -k eth0`输出如下

![](https://user-gold-cdn.xitu.io/2020/2/3/1700a73ebd4feda9)

TCP 套接字选项 TCP\_MAXSEG
---------------------

TCP 有一个 socket 选项 TCP\_MAXSEG，可以用来设置此次连接的 MSS，如果设置了这个选项，则 MSS 不能超过这个值。我们来看看实际的代码，还是以 echo server 为例，在 bind 之前调用 setsockopt 设置 socket 选项。完整的代码见：[github.com/arthur-zhan…](https://github.com/arthur-zhang/tcp-book-code-examples/blob/master/tcp-option-maxseg/test.c)

    int main(int argc, char *argv[]) {
      int port = atoi(argv[1]);
      int mss = atoi(argv[2]);
    
      // ...
      int tcp_maxseg = mss;
      socklen_t tcp_maxseg_len = sizeof(tcp_maxseg);
    
      // 设置 TCP_MAXSEG 选项
      if ((err = setsockopt(server_fd, IPPROTO_TCP, TCP_MAXSEG, &tcp_maxseg, tcp_maxseg_len)) < 0) {
        error_quit("set TCP_MAXSEG failed, code: %d\n", err);
      }
    
      if (bind(server_fd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        error_quit("could not bind socket");
      }
    
      if (listen(server_fd, 128) < 0) {
        error_quit("Could not listen on socket\n");
      }
    
      printf("server start, listening on %d\n", port);
    
      while (1) {
        socklen_t client_len = sizeof(cli_addr);
    
        if ((client_fd = accept(server_fd, (struct sockaddr *)&cli_addr, &client_len)) < 0) {
          error_quit("could not establish new connection\n");
        }
    
        while (1) {
          int read = recv(client_fd, buf, BUFFER_SIZE, 0);
          if (!read) break;
          if (read < 0) error_quit("read failed\n");
          if (send(client_fd, buf, read, 0) < 0) error_quit("write failed\n");
        }
      }
    }


编译运行上面的代码。

    gcc test.c -o echo-server
    ./echo-server 9999 100


在使用 nc 或者 telnet 连接这个 9999 端口服务，使用 tcpdump 查看抓包结果如下。

![](https://user-gold-cdn.xitu.io/2020/2/4/17010aaef8da1721)

可以看到经过代码的设置，三次握手中的 MSS 已经从 1460 变为了 100。那 MSS 允许的范围是多少呢？如果设置一个很小的 MSS，比如 50，会出现 setsockopt 失败的情况，如下所示。

    ./echo-server 9999 50
    set TCP_MAXSEG failed, code: -1


经过快速的二分法，很快就可以定位出来 setsockopt 合法的范围 88~32767，接下来我们来看看内核对这一部分是如何处理的。内核处理 setsockopt 的函数在 `do_tcp_setsockopt@net/ipv4/tcp.c`，

    static int do_tcp_setsockopt(struct sock *sk, int level,
    		int optname, char __user *optval, unsigned int optlen)
    {
        
        switch (optname) {
        case TCP_MAXSEG:
    		/* Values greater than interface MTU won't take effect. However
    		 * at the point when this call is done we typically don't yet
    		 * know which interface is going to be used */
    		if (val < TCP_MIN_MSS || val > MAX_TCP_WINDOW) {
    			err = -EINVAL; // -22
    			break;
    		}
    		tp->rx_opt.user_mss = val;
    		break;
        }
    }


常量 TCP\_MIN\_MSS 的值为 88，常量 MAX\_TCP\_WINDOW 的值为 32768，因此不在 88~32767 直接的 MSS 值会设置失败。

> 为什么 TCP\_MAXSEG 的下界是 88？

这是因为 TCP 头包含了 20 字节的固定长度和 40 字节的可选参数，所以 TCP 头的最大长度是 60，IP 头最大长度也是 60。

为了保证在 TCP 头占满 60 字节、IP 头占满 60 字节的情况下，至少还能发 8 字节的数据，MSS 至少要等于 (MAX\_IP\_HDR + MAX\_TCP\_HDR + MIN\_IP\_FRAG) - (MIN\_IP\_HDR + MIN\_TCP\_HDR) = (60+60+8) - (20+20) = 88 字节。

那 MSS 设置一个比较大的值，比如 30000，实际 MSS 是 30000 吗？

执行前面的程序，使用 setsockopt 将 MSS 设置为 30000，如下所示。

    ./echo-server 9999 30000


再次在使用 nc 或者 telnet 连接这个 9999 端口服务，使用 tcpdump 查看抓包结果如下。

![](https://user-gold-cdn.xitu.io/2020/2/4/17010aaf0222521f)

可以看到这时 MSS 没有变为 30000，依旧是 1460。这是因为调用 setsockopt 时并不知道后面会使用哪个网卡。后面真正发送 SYN 时，会根据设备的 MTU 重新计算最终的 MSS。

小结
--

这篇文章主要介绍了几个比较基础的概念，IP 数据包长度在超过链路的 MTU 时在发送之前需要分片，而 TCP 层为了 IP 层不用分片主动将包切割成 MSS 大小。

作业题
---

1、TCP/IP 协议中，MSS 和 MTU 分别工作在哪一层？

2、在 MTU=1500 字节的以太网中，TCP 报文的最大载荷为多少字节？


[Source](https://juejin.im/book/6844733788681928712/section/6844733788816179207)