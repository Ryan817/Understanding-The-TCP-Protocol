

最初这个问题是读者上一个小册中的一个留言提出的：「处于 ESTABLISHED 的连接，为什么还要响应 SYN 包？」，这篇文章就来聊聊这一部分的内容。

通过阅读这篇文章，你会了解到这些知识

*   ESTABLISHED 状态的连接收到乱序包会回复什么
*   Challenge ACK 的概念
*   ACK 报文限速是什么鬼
*   SystemTap 工具在 linux 内核追踪中的使用
*   包注入神器 scapy 的使用
*   RST 攻击的原理
*   killcx 等工具利用 RST 攻击的方式来杀掉连接的原理

接下来开始文章的内容。

scapy 实验复现现象
------------

实验步骤如下：

在机器 A(10.211.55.10) 使用 nc 启动一个服务程序，监听 9090 端口，如下所示。

    nc -4 -l 9090


机器 A 上同步使用 tcpdump 抓包，其中 -S 表示显示绝对序列号。

    sudo tcpdump -i any port 9090 -nn  -S


在机器 B 使用 nc 命令连接机器 A 的 nc 服务器，输入 "hello" 。

    nc 10.211.55.10 9090


使用 netstat 可以看到此次连接的信息。

    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 10.211.55.10:9090       10.211.55.20:50718      ESTABLISHED 9029/nc


在机器 B 上使用 scapy，模拟发送 SYN 包，scapy 脚本如下所示。

    send(IP(dst="10.211.55.10")/TCP(sport=50718, dport=9090, seq=10, flags='S'))


源端口号 sport 使用此次连接的临时端口号 50718，序列号随便写一个，这里 seq 为 10。

执行 scapy 执行上面的代码，tcpdump 中显示的包结果如下。

    // nc 终端中 hello 请求包
    18:41:51.956735 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [P.], seq 3219267420:3219267426, ack 2848436085, win 229, options [nop,nop,TS val 1094540820 ecr 12823113], length 6
    18:41:51.956787 IP 10.211.55.10.9090 > 10.211.55.20.50718: Flags [.], ack 3219267426, win 227, options [nop,nop,TS val 12827910 ecr 1094540820], length 0
    
    // scapy 的 SYN 包
    18:44:32.373331 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [S], seq 10, win 8192, length 0
    18:44:32.373366 IP 10.211.55.10.9090 > 10.211.55.20.50718: Flags [.], ack 3219267426, win 227, options [nop,nop,TS val 12988327 ecr 1094540820], length 0


可以看到，对于一个 SEQ 为随意的 SYN 包，TCP 回复了正确的 ACK 包，其确认号为 3219267426。

从 rfc793 文档中也可以看到：

> Linux 内核对于收到的乱序 SYN 报文，会回复一个携带了正确序列号和确认号的 ACK 报文。

这个 ACK 被称之为 Challenge ACK。

我们后面要介绍的杀掉连接工具 killcx 的原理，正是是基于这一点。

原因分析
----

为了方便说明，我们记发送 SYN 报文的一端为 A，处于 ESTABLISHED 状态接收 SYN 报文的一端为 B，B 对收到的 SYN 包回复 ACK 的原因是想让对端 A 确认之前的连接是否已经失效，以便做出一些处理。

对于 A 而已，如果之前的连接还在，对于收到的 ACK 包，正常处理即可，不再讨论。

如果 A 之前的此条连接已经不在了，此次 SYN 包是想发起新的连接，对于收到的 ACK 包，会立即回复一个 RST，且 RST 包的序列号就等于 ACK 包的序列号，B 收到这个合法的 RST 包以后，就会将连接释放。A 此时若想继续与 B 创建连接，则可以选择再次发送 SYN 包，重新建连，如下图所示。

![estab_syn](https://user-gold-cdn.xitu.io/2020/2/2/170047c916cc0d37)

接下来我们来看内核源码的处理，

内核源码分析
------

在这之前，我们需要先了解 SystemTap 工具的使用。SystemTap 是 Linux 中非常强大的调试探针工具，类似于 java 中的 javaagent instrument，可以获取一个内核函数运行时的入参变量、返回值、调用堆栈，甚至可以直接修改变量的值。这个工具详细的使用这里不展开，感兴趣的同学可以自行 Google。

接下来我们来使用 SystemTap 这个工具来给内核插入 probe 探针，以 3.10.0 内核为例，内核中回复的 ack 的函数在 net/ipv4/tcp\_output.c 的 tcp\_send\_ack 中实现。我们给这个函数插入调用探针，在端口号为 9090 时打印调用堆栈。新建一个 ack\_test.stp 文件，部分代码如下所示。

    %{


​    
​    
​    
    %}
    
    function tcp_src_port:long(sk:long)
    {
    	return __tcp_sock_sport(sk)
    }
    function tcp_dst_port:long(sk:long)
    {
    	return __tcp_sock_dport(sk)
    }
    function tcp_src_addr:long(sk:long)
    {
    	return ntohl(__ip_sock_saddr(sk))
    }
    function tcp_dst_addr:long(sk:long)
    {
    	return ntohl(__ip_sock_daddr(sk))
    }
    function str_addr:string(addr, port) {
            return sprintf("%d.%d.%d.%d:%d",
                           (addr & 0xff000000) >> 24,
                           (addr & 0x00ff0000) >> 16,
                           (addr & 0x0000ff00) >> 8,
                           (addr & 0x000000ff),
                           port
                    )
    }
    
    probe kernel.function("tcp_send_ack@net/ipv4/tcp_output.c")
    {
           src_addr = tcp_src_addr($sk);
           src_port = tcp_src_port($sk);
           dst_addr = tcp_dst_addr($sk);
           dst_port = tcp_dst_port($sk);
           if (dst_port == 9090 || src_port == 9090)
           {
                  printf("send ack : %s:->%s\n",
                         str_addr(src_addr, src_port),
                         str_addr(dst_addr, dst_port));
                  print_backtrace();
           }
    }


使用 stap 命令执行上面的脚本

    sudo stap -g ack_test.stp


​    

再次使用 scapy 发送一个 syn 包，内核同样会回复 ACK，此时 stap 输出结果如下。

    send ack : 10.211.55.10:9090:->10.211.55.20:50718
     0xffffffff815d0940 : tcp_send_ack+0x0/0x170 [kernel]
     0xffffffff815cb1d2 : tcp_validate_incoming+0x212/0x2d0 [kernel]
     0xffffffff815cb44d : tcp_rcv_established+0x1bd/0x760 [kernel]
     0xffffffff815d5f8a : tcp_v4_do_rcv+0x10a/0x340 [kernel]
     0xffffffff815d76d9 : tcp_v4_rcv+0x799/0x9a0 [kernel]
     0xffffffff815b1094 : ip_local_deliver_finish+0xb4/0x1f0 [kernel]
     0xffffffff815b1379 : ip_local_deliver+0x59/0xd0 [kernel]
     0xffffffff815b0d1a : ip_rcv_finish+0x8a/0x350 [kernel]
     0xffffffff815b16a6 : ip_rcv+0x2b6/0x410 [kernel]


可以看到这个 ACK 经过了下面这些函数调用。

    tcp_v4_rcv
      -> tcp_v4_do_rcv
        -> tcp_rcv_established
          -> tcp_validate_incoming
            -> tcp_send_ack


tcp\_validate\_incoming 函数精简后的部分代码如下所示。

    static bool tcp_validate_incoming(struct sock *sk, struct sk_buff *skb,
    				  const struct tcphdr *th)
    {	
    	// seq 不在窗口内
    	/* Step 1: check sequence number */
    	if (!tcp_sequence(tp, TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq)) {
    		// RST 标记没有设置
    		if (!th->rst) {
    			if (th->syn)
    				goto syn_challenge;
    		}
    		goto discard;
    	}
    	
    	/* step 4: Check for a SYN。 RFC 5961 4.2 : Send a challenge ack */
    	if (th->syn) {
    syn_challenge: // 处理 SYN Challenge 的情况
    		tcp_send_challenge_ack(sk, skb); // 
    		goto discard;
    	}


tcp\_send\_challenge\_ack 函数真正调用了 tcp\_send\_ack 函数。 这里的注释提到了 [RFC 5961 4.2](https://tools.ietf.org/html/rfc5961#section-4.2)，说的正是 Challenge ACK 相关的内容。

如果攻击者疯狂发送假的乱序包，接收端也跟着回复 Challenge ACK，会耗费大量的 CPU 和带宽资源。于是 RFC 5961 提出了 ACK Throttling 方案，限制了每秒钟发送 Challenge ACK 报文的数量，这个值由 net.ipv4.tcp\_challenge\_ack\_limit 系统变量决定，默认值是 1000，也就是 1s 内最多允许 1000 个 Challenge ACK 报文。

接下来使用 sysctl 将这个值改小为 1，如下所示。

    sudo sysctl -w net.ipv4.tcp_challenge_ack_limit="1"


这样理论上在一秒内多次发送一个 Challenge ACK 包，接下来使用 scapy 在短时间内发送 5 次 SYN 包，看看内核是否只会回复一个 ACK 包，scapy 的脚本如下所示。

    send(IP(dst="10.211.55.10")/TCP(sport=50718,dport=9090,seq=10,flags='S'), loop=0, count=5)


tcpdump 抓包结果如下。

    03:40:30.970682 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [S], seq 10, win 8192, length 0
    03:40:30.970771 IP 10.211.55.10.9090 > 10.211.55.20.50718: Flags [.], ack 3219267426, win 227, options [nop,nop,TS val 45146923 ecr 1094540820], length 0
    03:40:30.974889 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [S], seq 10, win 8192, length 0
    03:40:30.975004 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [S], seq 10, win 8192, length 0
    03:40:30.978643 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [S], seq 10, win 8192, length 0
    03:40:30.981987 IP 10.211.55.20.50718 > 10.211.55.10.9090: Flags [S], seq 10, win 8192, length 0


可以看到确实是只对第一个 SYN 包回复了一个 ACK 包，其它的四个 SYN 都没有回复 ACK。

小结
--

这篇文章介绍了为什么 ESTABLISHED 状态连接的需要对 SYN 包做出响应，Challenge ACK 是什么，使用 scapy 复现了现象，演示了 SystemTap 内核探针调试工具的使用，最后通过修改系统变量复现了 ACK 限速。

![](data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzMiIgaGVpZ2h0PSIzMiIgdmlld0JveD0iMCAwIDMyIDMyIj4KICAgIDxwYXRoIGZpbGw9Im5vbmUiIGZpbGwtcnVsZT0iZXZlbm9kZCIgc3Ryb2tlPSIjRkZGIiBzdHJva2Utd2lkdGg9IjIiIGQ9Ik0xOSAyM2wtNy02LjUgNy02LjUiLz4KPC9zdmc+Cg==)

![](data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzMiIgaGVpZ2h0PSIzMiIgdmlld0JveD0iMCAwIDMyIDMyIj4KICAgIDxwYXRoIGZpbGw9Im5vbmUiIGZpbGwtcnVsZT0iZXZlbm9kZCIgc3Ryb2tlPSIjRkZGIiBzdHJva2Utd2lkdGg9IjIiIGQ9Ik0xMyAyMmw3LTYuNUwxMyA5Ii8+Cjwvc3ZnPgo=)


[Source](https://juejin.im/book/6844733788681928712/section/6844733788845522957)