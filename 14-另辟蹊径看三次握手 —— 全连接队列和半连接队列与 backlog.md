

关于三次握手，还有很多细节之前的文章没有详细介绍，这篇文章我们以 backlog 参数来深入研究一下建连的过程。通过阅读这篇文章，你会了解到下面这些知识：

*   backlog、半连接队列、全连接队列是什么
*   linux 内核是如何计算半连接队列、全连接队列的
*   为什么只修改系统的 somaxconn 和 tcp\_max\_syn\_backlog 对最终的队列大小不起作用
*   如何使用 systemtap 探针获取当前系统的半连接、全连接队列信息
*   iprouter 库中的 ss 工具的原理是什么
*   如何快速模拟半连接队列溢出，全连接队列溢出

注：本文中的代码和测试均在内核版本 3.10.0-514.16.1.el7.x86\_64 下进行。

半连接队列、全连接队列基本概念
---------------

为了理解 backlog，我们需要了解 listen 和 accept 函数背后的发生了什么。backlog 参数跟 listen 函数有关，listen 函数的定义如下：

    int listen(int sockfd, int backlog);


当服务端调用 listen 函数时，TCP 的状态被从 CLOSE 状态变为 LISTEN，于此同时内核创建了两个队列：

*   半连接队列（Incomplete connection queue），又称 SYN 队列
*   全连接队列（Completed connection queue），又称 Accept 队列

如下图所示。

![](https://user-gold-cdn.xitu.io/2019/6/28/16b9dae5efc47de8)

接下来开始详细介绍这两个队列相关的内容。

半连接队列（SYN Queue）
----------------

当客户端发起 SYN 到服务端，服务端收到以后会回 ACK 和自己的 SYN。这时服务端这边的 TCP 从 listen 状态变为 SYN\_RCVD (SYN Received)，此时会将这个连接信息放入「半连接队列」，半连接队列也被称为 SYN Queue，存储的是 "inbound SYN packets"。

![](https://user-gold-cdn.xitu.io/2019/2/23/16918ddaf0b49c7e)

服务端回复 SYN+ACK 包以后等待客户端回复 ACK，同时开启一个定时器，如果超时还未收到 ACK 会进行 SYN+ACK 的重传，重传的次数由 tcp\_synack\_retries 值确定。在 CentOS 上这个值等于 5。

一旦收到客户端的 ACK，服务端就开始**尝试**把它加入另外一个全连接队列（Accept Queue）。

### 半连接队列的大小的计算

这里使用 SystemTap 工具插入系统探针，在收到 SYN 包以后打印当前的 SYN 队列的大小和半连接队列的总大小。

TCP listen 状态的 socket 收到 SYN 包的处理流程如下

    tcp_v4_rcv
      ->tcp_v4_do_rcv
        -> tcp_v4_conn_request


这里注入 tcp\_v4\_conn\_request 方法，代码如下所示。

    probe kernel.function("tcp_v4_conn_request") {
        tcphdr = __get_skb_tcphdr($skb);
        dport = __tcp_skb_dport(tcphdr);
        if (dport == 9090)
        {
            printf("reach here\n");
            // 当前 syn 排队队列的大小
            syn_qlen = @cast($sk, "struct inet_connection_sock")->icsk_accept_queue->listen_opt->qlen;
            // syn 队列总长度 log 值
            max_syn_qlen_log = @cast($sk, "struct inet_connection_sock")->icsk_accept_queue->listen_opt->max_qlen_log;
            // syn 队列总长度，2^n
            max_syn_qlen = (1 << max_syn_qlen_log);
            printf("syn queue: syn_qlen=%d, max_syn_qlen_log=%d, max_syn_qlen=%d\n",
             syn_qlen, max_syn_qlen_log, max_syn_qlen);
            // max_acc_qlen = $sk->sk_max_ack_backlog;
            // printf("accept queue length limit: %d\n", max_acc_qlen)
            print_backtrace();
        }
    }


使用 stap 执行上面的脚本

    sudo stap -g syn_backlog.c


这样在收到 SYN 包以后可以打印当前syn 队列排队的连接个数和总大小了。

还是以之前的 echo 程序为例，listen 的 backlog 设置为 10，如下所示。

    int server_fd = //...
    
    listen(server_fd, 10 /*backlog*/)


启动 echo-server，监听 9090 端口。然后在另外一个机器上使用 nc 命令进行连接。

    nc 10.211.55.10 9090


此时在 stap 的输出中，已经可以看到当前的 可以看到syn 队列大小为 0，最大的队列长度是 `2^4=16`

![](https://user-gold-cdn.xitu.io/2020/2/24/17076a3dbfa629f4)

因此可以看到实际的 syn 并不是等于`net.ipv4.tcp_max_syn_backlog`的默认值为 128，而是将用户传入的 10 向上取了最接近的 2 的指数幂值 16。

接下来我们来看代码中是如何计算的，半连接队列的大小与三个值有关：

*   用户层 listen 传入的backlog
*   系统变量 `net.ipv4.tcp_max_syn_backlog`，默认值为 128
*   系统变量 `net.core.somaxconn`，默认值为 128

具体的计算见下面的源码，调用 listen 函数首先会进入如下的代码。

    SYSCALL_DEFINE2(listen, int, fd, int, backlog)
    {
        // sysctl_somaxconn 是系统变量 net.core.somaxconn 的值
    	int somaxconn = sysctl_somaxconn;
    	if ((unsigned int)backlog > somaxconn)
    		backlog = somaxconn;
    	sock->ops->listen(sock, backlog);
    }


通过 SYSCALL\_DEFINE2 代码可以得知，如果用户传入的 backlog 值大于系统变量 net.core.somaxconn 的值，用户设置的 backlog 不会生效，使用系统变量值，默认为 128。

接下来这个 backlog 值会被依次传递给 inet\_listen()->inet\_csk\_listen\_start()->reqsk\_queue\_alloc() 方法。在 reqsk\_queue\_alloc 方法中进行了最终的计算。精简后的代码如下。

    int reqsk_queue_alloc(struct request_sock_queue *queue,
    		      unsigned int nr_table_entries)
    {
        nr_table_entries = min_t(u32, nr_table_entries, sysctl_max_syn_backlog);
        nr_table_entries = max_t(u32, nr_table_entries, 8);
        nr_table_entries = roundup_pow_of_two(nr_table_entries + 1);
        	
        for (lopt->max_qlen_log = 3;
             (1 << lopt->max_qlen_log) < nr_table_entries;
             lopt->max_qlen_log++);
    }


代码中 nr\_table\_entries 为前面计算的 backlog 值，sysctl\_max\_syn\_backlog 为 net.ipv4.tcp\_max\_syn\_backlog 的值。 计算逻辑如下：

*   在 nr\_table\_entries 与 sysctl\_max\_syn\_backlog 两者中的较小值，赋值给 nr\_table\_entries
*   在 nr\_table\_entries 和 8 取较大值，赋值给 nr\_table\_entries
*   nr\_table\_entries + 1 向上取求最接近的最大 2 的指数次幂
*   通过 for 循环找不大于 nr\_table\_entries 最接近的 2 的对数值

下面来举几个实际的例子，以 listen(50) 为例，经过 SYSCALL\_DEFINE2 中计算 backlog 的值为 min(50, somaxconn)，等于 50，接下来进入 reqsk\_queue\_alloc 函数的计算。

    // min(50, 128) = 50
    nr_table_entries = min_t(u32, nr_table_entries, sysctl_max_syn_backlog);
    // max(50, 8) = 50
    nr_table_entries = max_t(u32, nr_table_entries, 8);
    // roundup_pow_of_two(51) = 64
    nr_table_entries = roundup_pow_of_two(nr_table_entries + 1);
      
    max_qlen_log 最小值为 2^3 = 8
    for (lopt->max_qlen_log = 3;
         (1 << lopt->max_qlen_log) < nr_table_entries;
         lopt->max_qlen_log++);
    经过 for 循环 max_qlen_log = 2^6 = 64


下面给了几个 somaxconn、max\_syn\_backlog、backlog 三者之间不同组合的最终半连接队列大小值。

| somaxconn | max\_syn\_backlog | listen backlog | 半连接队列大小 |
| --- | --- | --- | --- |
| 128 | 128 | 5 | 16 |
| 128 | 128 | 10 | 16 |
| 128 | 128 | 50 | 64 |
| 128 | 128 | 128 | 256 |
| 128 | 128 | 1000 | 256 |
| 128 | 128 | 5000 | 256 |
| 1024 | 128 | 128 | 256 |
| 1024 | 1024 | 128 | 256 |
| 4096 | 4096 | 128 | 256 |
| 4096 | 4096 | 4096 | 8192 |

可以看到:

*   在系统参数不修改的情形，盲目调大 listen 的 backlog 对最终半连接队列的大小不会有影响。
*   在 listen 的 backlog 不变的情况下，盲目调大 somaxconn 和 max\_syn\_backlog 对最终半连接队列的大小不会有影响

### 模拟半连接队列占满

以 somaxconn=128、tcp\_max\_syn\_backlog=128、listen backlog=50 为例，模拟的原理是在三次握手的第二步，客户端在收到服务端回复的 SYN+ACK 以后使用 iptables 丢弃这个包。这里实验的服务端是 10.211.55.10，客户端是 10.211.55.20，在客户端使用 iptables 增加一条规则，如下所示。

    sudo  iptables --append INPUT  --match tcp --protocol tcp --src 10.211.55.10 --sport 9090 --tcp-flags SYN SYN --jump DROP


这条规则的含义是丢弃来自 ip 为 10.211.55.10，源端口号为 9090 的 SYN 包，如下图所示。

![syn-queue-full](https://user-gold-cdn.xitu.io/2020/2/24/17076a3dc1909b4a)

接下来使用你喜欢的语言，开始发起连接就好了，这里选择了 go，代码如下：

    func main() {
    	for i := 0; i < 2000; i++ {
    		go connect()
    	}
    	time.Sleep(time.Minute * 10)
    }
    func connect() {
    	_, err := net.Dial("tcp4", "10.211.55.10:9090")
    	if err != nil {
    		fmt.Println(err)
    	}
    }


执行这个 go 程序，在服务端使用 netstat 查看当前 9090 端口的连接状态，如下所示。

    netstat -lnpa | grep :9090  | awk '{print $6}' | sort | uniq -c | sort -rn
         64 SYN_RECV
          1 LISTEN


可以观察到 SYN\_RECV 状态的连接个数的从 0 开始涨到 64，就不再上涨了，这里的 64 就是半连接队列的大小。

接下来我们来看全连接队列

全连接队列（Accept Queue）
-------------------

「全连接队列」包含了服务端所有完成了三次握手，但是还未被应用调用 accept 取走的连接队列。此时的 socket 处于 ESTABLISHED 状态。每次应用调用 accept() 函数会移除队列头的连接。如果队列为空，accept() 通常会阻塞。全连接队列也被称为 Accept 队列。

你可以把这个过程想象生产者、消费者模型。内核是一个负责三次握手的生产者，握手完的连接会放入一个队列。我们的应用程序是一个消费者，取走队列中的连接进行下一步的处理。这种生产者消费者的模式，在生产过快、消费过慢的情况下就会出现队列积压。

listen 函数的第二个参数 backlog 用来设置全连接队列大小，但不一定就会选用这一个 backlog 值，还受限于 somaxconn，等下会有更详细的内容说明全连接队列大小的计算规则。

`int listen(int sockfd, int backlog)`

如果全连接队列满，内核会舍弃掉 client 发过来的 ack（应用层会认为此时连接还未完全建立）

我们来模拟一下全连接队列满的情况。因为只有 accept 才会移除全连接的队列，所以如果我们只 listen，不调用 accept，那么很快全连接就可以被占满。

![](https://user-gold-cdn.xitu.io/2019/6/29/16ba09ba6e24b1c3)

为了贴近最底层的调用，这里用 c 语言来实现，新建一个 main.c 文件

    #include <stdio.h>
    #include <sys/socket.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <errno.h>
    #include <arpa/inet.h>
    
    int main() {
        struct sockaddr_in serv_addr;
        int listen_fd = 0;
        if ((listen_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
            exit(1);
        }
        bzero(&serv_addr, sizeof(serv_addr));
    
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        serv_addr.sin_port = htons(8080);
    
        if (bind(listen_fd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) == -1) {
            exit(1);
        }


​        
        if (listen(listen_fd, 50) == -1) {
            exit(1);
        }
        sleep(100000000);
        return 0;
    }


编译运行`gcc main.c; ./a.out`，使用前面的的 go 程序发起 connect，在服务端用 netstat 查看 tcp 连接状态

    netstat -lnpa | grep :9090  | awk '{print $6}' | sort | uniq -c | sort -rn
         51 ESTABLISHED
         31 SYN_RECV
          1 LISTEN


虽然并发发了很多请求，实际只有 51 个请求处于 ESTABLISHED 状态，还有大量请求处于 SYN\_RECV 状态。

另外注意到 backlog 等于 50，但是实际上处于 ESTABLISHED 状态的连接却有 51 个，后面会讲到。

客户端用 netstat 查看 tcp 有几百个连接，状态全是 ESTABLISHED，如下所示。

    Proto Recv-Q Send-Q Local Address           Foreign Address         State
    tcp        0      0 10.211.55.20:37732      10.211.55.10:9090       ESTABLISHED 23618/./connect
    tcp        0      0 10.211.55.20:37824      10.211.55.10:9090       ESTABLISHED 23618/./connect
    tcp        0      0 10.211.55.20:37740      10.211.55.10:9090       ESTABLISHED 23618/./connect
    ...


使用 systemstap 可以实时观察当前的全连接队列情况，探针代码如下所示。

    probe kernel.function("tcp_v4_conn_request") {
        tcphdr = __get_skb_tcphdr($skb);
        dport = __tcp_skb_dport(tcphdr);
        if (dport == 9090)
        {
            printf("reach here\n");
            // 当前 syn 排队队列的大小
            syn_qlen = @cast($sk, "struct inet_connection_sock")->icsk_accept_queue->listen_opt->qlen;
            // syn 队列总长度 log 值
            max_syn_qlen_log = @cast($sk, "struct inet_connection_sock")->icsk_accept_queue->listen_opt->max_qlen_log;
            // syn 队列总长度，2^n
            max_syn_qlen = (1 << max_syn_qlen_log);
            printf("syn queue: syn_qlen=%d, max_syn_qlen_log=%d, max_syn_qlen=%d\n",
             syn_qlen, max_syn_qlen_log, max_syn_qlen);
            ack_backlog = $sk->sk_ack_backlog;
            max_ack_backlog = $sk->sk_max_ack_backlog;
            printf("accept queue length, max: %d, current: %d\n", max_ack_backlog, ack_backlog)
        }
    }


使用 stap 执行这个探针，重新运行上面的测试，可以看到内核探针的输出结果。

    ...
    syn queue: syn_qlen=45, max_syn_qlen_log=6, max_syn_qlen=64
    accept queue length, max: 50, current: 14
    ...
    syn queue: syn_qlen=2, max_syn_qlen_log=6, max_syn_qlen=64
    accept queue length, max: 50, current: 51


这里也可以看出全连接队列的大小变化的情况，印证了我们前面的说法。

跟踪服务器端的一个包的结果如下：

![](https://user-gold-cdn.xitu.io/2020/2/24/17076a3dcaf0c3b8)

以下记客户端 10.211.55.20 为 A，服务端 10.211.55.10 为 B

*   1：客户端 A 发起 SYN 到服务端 B 的 9090 端口，开始三次握手的第一步
*   2：服务器 B 马上回复了 ACK + SYN，此时 服务器 B socket处于 SYN\_RCVD 状态
*   3：客户端 A 收到服务器 B 的 ACK + SYN，发送三次握手最后一步的 ACK 给服务器 B，自己此时处于 ESTABLISHED 状态，与此同时，由于服务器 B 的全连接队列满，它会丢掉这个 ACK，连接还未建立
*   4：服务端 B 因为认为没有收到 ACK，以为是自己在 2 中的 SYN + ACK 在传输过程中丢掉了，所以开始重传，期待客户端能重新回复 ACK。
*   5：客户端 A 收到 B 的 SYN + ACK 以后，确实马上回复了 ACK
*   6 ~ 13：但是这个 ACK 同样也会被服务器 B 丢弃，服务端 B 还是认为没有收到 ACK，继续重传重传的过程同样也是指数级退避的（1s、2s、4s、8s、16s），总共历时 31s 重传 5 次 `SYN + ACK` 以后，服务器 B 认为没有希望，一段时间后此条 tcp 连接就被系统回收了。

SYN+ACK重传的次数是由操作系统的一个文件决定的`/proc/sys/net/ipv4/tcp_synack_retries`，可以用 cat 查看这个文件

    cat /proc/sys/net/ipv4/tcp_synack_retries
    5


整个过程如下图所示：

![](https://user-gold-cdn.xitu.io/2019/2/23/16918ddaf0d4afa5)

### 全连接队列的大小

全连接队列的大小是 listen 传入的 backlog 和 somaxconn 中的较小值。

全连接队列大小判断是否满的函数是 /include/net/sock.h 中 的 sk\_acceptq\_is\_full 方法。

    static inline bool sk_acceptq_is_full(const struct sock *sk)
    {
    	return sk->sk_ack_backlog > sk->sk_max_ack_backlog;
    }


这里本身没有什么毛病，只是 sk\_ack\_backlog 是从 0 开始计算的，所以真正全连接队列大小是 backlog + 1。当你指定 backlog 值为 1 时，能容纳的连接个数会是 2。《Unix 网络编程卷一》87 页 4.5 节有详细的对比各个操作系统 backlog 与实际全连接队列最大数量之间的关系。

### ss 命令

ss 命令可以查看全连接队列的大小和当前等待 accept 的连接个数，执行 `ss -lnt` 即可，比如上面的 accept 队列满的例子中，执行 ss 命令的输出结果如下。

    ss -lnt | grep :9090
    State      Recv-Q Send-Q Local Address:Port               Peer Address:Port
    LISTEN     51     50           *:9090                     *:*


对于 LISTEN 状态的套接字，Recv-Q 表示 accept 队列排队的连接个数，Send-Q 表示全连接队列（也就是 accept 队列）的总大小。

我们来看看 ss 命令的底层实现。ss 命令的源码在 iproute2 项目里，它巧妙的利用了 netlink 与 TCP 协议栈中 tcp\_diag 模块通信获取 socket 的详细信息。tcp\_diag 是一个统计分析模块，可以获取内核中很多有用的信息，ss 输出中的 Recv-Q 和 Send-Q 就是从 tcp\_diag 模块中获取的，这两个值是等于 inet\_diag\_msg 结构体的 idiag\_rqueue 和 idiag\_wqueue。tcp\_diag 部分的源码如下所示。

    static void tcp_diag_get_info(struct sock *sk, struct inet_diag_msg *r,
    			      void *_info)
    {
    	struct tcp_info *info = _info;
    
    	if (inet_sk_state_load(sk) == TCP_LISTEN) {
    	   // 对应 Recv-Q
    		r->idiag_rqueue = READ_ONCE(sk->sk_ack_backlog); 
    		// 对应 Send-Q
    		r->idiag_wqueue = READ_ONCE(sk->sk_max_ack_backlog);	} else if (sk->sk_type == SOCK_STREAM) {
    		const struct tcp_sock *tp = tcp_sk(sk);
    		r->idiag_rqueue = max_t(int, READ_ONCE(tp->rcv_nxt) -
    					     READ_ONCE(tp->copied_seq), 0);
    		r->idiag_wqueue = READ_ONCE(tp->write_seq) - tp->snd_una;
    	}
    }


从上面的源码可以得知：

*   处于 LISTEN 状态的 socket，Recv-Q 对应 sk\_ack\_backlog，表示当前 socket 的完成三次握手等待用户进程 accept 的连接个数，Send-Q 对应 sk\_max\_ack\_backlog，表示当前 socket 全连接队列能最大容纳的连接数
*   对于非 LISTEN 状态的 socket，Recv-Q 表示 receive queue 的字节大小，Send-Q 表示 send queue 的字节大小

其它
--

### 多大的 backlog 是合适的

前面讲了这么多，应用程序设置多大的 backlog 是合理的呢？

答案是 It depends，根据不同过的业务场景，需要做对应的调整。

*   你如果的接口处理连接的速度要求非常高，或者在做压力测试，很有必要调高这个值
*   如果业务接口本身性能不好，accept 取走已建连的速度较慢，那么把 backlog 调的再大也没有用，只会增加连接失败的可能性

可以举个典型的 backlog 值供大家参考，Nginx 和 Redis 默认的 backlog 值等于 511，Linux 默认的 backlog 为 128，Java 默认的 backlog 等于 50

### tcp\_abort\_on\_overflow 参数

默认情况下，全连接队列满以后，服务端会忽略客户端的 ACK，随后会重传`SYN+ACK`，也可以修改这种行为，这个值由`/proc/sys/net/ipv4/tcp_abort_on_overflow`决定。

*   tcp\_abort\_on\_overflow 为 0 表示三次握手最后一步全连接队列满以后 server 会丢掉 client 发过来的 ACK，服务端随后会进行重传 SYN+ACK。
*   tcp\_abort\_on\_overflow 为 1 表示全连接队列满以后服务端直接发送 RST 给客户端。

但是回给客户端 RST 包会带来另外一个问题，客户端不知道服务端响应的 RST 包到底是因为「该端口没有进程监听」，还是「该端口有进程监听，只是它的队列满了」。

小结
--

这篇文章我们从 backlog 参数为入口来研究了半连接队列、全连接队列的关系。简单回顾一下。

*   半连接队列：服务端收到客户端的 SYN 包，回复 SYN+ACK 但是还没有收到客户端 ACK 情况下，会将连接信息放入半连接队列。半连接队列又被称为 SYN 队列。
*   全连接队列：服务端完成了三次握手，但是还未被 accept 取走的连接队列。全连接队列又被称为 Accept 队列。
*   半连接队列的大小与用户 listen 传入的 backlog、net.core.somaxconn、net.core.somaxconn 都有关系，准确的计算规则见上面的源码分析
*   全连接队列的大小是用户 listen 传入的 backlog 与 net.core.somaxconn 的较小值

上面所说的结论不应当都是对的，这也是我一直的观点：结论不重要，重要的是研究的过程。我更多的是想授之以渔，教会你一些工具和方法，如果你能举一反三的去研究一些问题，那便是极好的。

不要随意相信网上文章乱下的结论，包括我这篇。实验出真知，自己动手亲自验证一下。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788828729358)