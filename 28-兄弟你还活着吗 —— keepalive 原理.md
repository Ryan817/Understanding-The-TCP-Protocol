

一个 TCP 连接上，如果通信双方都不向对方发送数据，那么 TCP 连接就不会有任何数据交换。这就是我们今天要讲的 TCP keepalive 机制的由来。

永远记住 TCP 不是轮询的协议
----------------

网络故障或者系统宕机都将使得对端无法得知这个消息。如果应用程序不发送数据，可能永远无法得知该连接已经失效。假设应用程序是一个 web 服务器，客户端发出三次握手以后故障宕机或被踢掉网线，对于 web 服务器而已，下一个数据包将永远无法到来，但是它一无所知。TCP 不会采用类似于轮询的方式来询问：小老弟你有什么东西要发给我吗？

这种情况下服务端会永远处于 ESTABLISHED 吗？

![](https://user-gold-cdn.xitu.io/2019/4/10/16a047d01a97dbcd)

TCP 的 half open
---------------

上面所说的情况就是典型的 TCP「半打开 half open」

> 这一个情况就是如果在未告知另一端的情况下通信的一端关闭或终止连接，那么就认为该条TCP连接处于半打开状态。 这种情况发现在通信的一方的主机崩溃、电源断掉的情况下。 只要不尝试通过半开连接来传输数据，正常工作的一端将不会检测出另外一端已经崩溃。

模拟客户端网络故障
---------

准备两台虚拟机 c1（服务器），c2（客户端）。在 c1 上执行 `nc -l 8080` 启动一个 TCP 服务器监听 8080 端口，同时在服务器 c1 上执行 tcpdump 查看包发送的情况。 在 c2 上用 `nc c1 8080`创建一条 TCP 连接 在 c1 上执行 netstat 查看连接状态，可以看到服务端已处于 ESTABLISHED 状态

    sudo netstat -lnpa | grep -i 8080
    tcp        0      0 10.211.55.5:8080        10.211.55.10:60492      ESTABLISHED 2787/nc


这时断掉 c1 的网络连接，可以看到 tcpdump 抓包没有任何包交互。此时再用 netstat 查看，发现连接还是处于 ESTABLISHED 状态。

过了几个小时以后再来查看，依旧是 ESTABLISHED 状态，且 tcpdump 输出显示没有任何包传输。

TCP 的 keepalive
---------------

TCP 协议的设计者考虑到了这种检测长时间死连接的需求，于是乎设计了 keepalive 机制。 在我的 CentOS 机器上，keepalive 探测包发送数据 7200s，探测 9 次，每次探测间隔 75s，这些值都有对应的参数可以配置。

为了能更快的演示，修改 centos 机器上 keepalive 相关的参数如下

    // 30s没有数据包交互发送 keepalive 探测包
    echo 30 > /proc/sys/net/ipv4/tcp_keepalive_time
    // 每次探测TCP 包间隔
    echo 10 > /proc/sys/net/ipv4/tcp_keepalive_intvl
    // 探测多少次
    echo 5 > /proc/sys/net/ipv4/tcp_keepalive_probes


默认情况下 nc 是没有开启 keepalive 的，怎么样在不修改 nc 源码的情况下，让它拥有 keepalive 的功能呢？

正常情况下，我们设置 tcp 的 keepalive 选项的代码如下：

    int flags = 1;
    setsockopt(socket_fd, SOL_TCP, TCP_KEEPALIVE, (void *)&flags, sizeof(flags)


我们可以用 strace 看下 `nc -l 8080`背后的系统调用

    socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 4
    setsockopt(4, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
    bind(4, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("0.0.0.0")}, 128) = 0
    listen(4, 10)


可以看到 nc 只调用 setsockopt 设置了 SO\_REUSEADDR 允许端口复用，并没有设置 TCP\_KEEPALIVE，那我们 hook 一下 setsockopt 函数调用，让它在设置端口复用的同时设置 TCP\_KEEPALIVE。那怎么样来做 hook 呢？

偷梁换柱之 LD\_PRELOAD
-----------------

LD\_PRELOAD 是一个 Linux 的环境变量，运行在程序运行前优先加载动态链接库，类似于 Java 的字节码改写 instrument。通过这个环境变量，我们可以修改覆盖真正的系统调用，达到我们的目的。 这个过程如下：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a047d01b7d0273)

新建文件 setkeepalive.c，全部代码如下：

    #include <sys/socket.h>
    #include <stdio.h>
    #include <dlfcn.h>
    #include <string.h>
    
    static int (*real_setsockopt)(int , int , int , void *, socklen_t) = NULL;
    
    __attribute__((constructor)) void init() {
        real_setsockopt = dlsym(RTLD_NEXT, "setsockopt");
    }
    
    int setsockopt(int sockfd, int level, int optname,
                   const void *optval, socklen_t optlen) {
            printf("SETSOCKOPT: %d: level: %d %d=%d (%d)\r\n",
     sockfd, level, optname, *(int*)optval, optlen);
            
            real_setsockopt(sockfd, level, optname, &optval, optlen);
            
            if (level == SOL_SOCKET && optname == SO_REUSEADDR) {
                    int val = 1;
                    
                    real_setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &val, optlen);
                    return 0;
            }
      return 0;
    }


编译上面的 setkeepalive.c 文件为 .so 文件： `gcc setkeepalive.c -fPIC -D_GNU_SOURCE -shared -ldl -o setkeepalive.so`

替换并测试运行

    LD_PRELOAD=./setkeepalive.so nc -l 8080


再来重复上面的测试流程，抓包如下：

![](https://user-gold-cdn.xitu.io/2019/4/10/16a047d01e58b593)

完美的展现了 keepalive 包的探测的过程: 1 ~ 3：三次握手，随后模拟客户端断网 4：30s 以后服务端发送第一个探测包（对应 tcp\_keepalive\_time） 5 ~ 8：因探测包一直没有回应，每隔 10s 发出剩下的 4 次探测包 9：5 次探测包以后，服务端觉得没有希望了，发出 RST 包，断掉这个连接

为什么大部分应用程序都没有开启 keepalive 选项
----------------------------

现在大部分应用程序（比如我们刚用的 nc）都没有开启 keepalive 选项，一个很大的原因就是默认的超时时间太长了，从没有数据交互到最终判断连接失效，需要花 2.1875 小时（7200 + 75 \* 9），显然太长了。但如果修改这个值到比较小，又违背了 keepalive 的设计初衷（为了检查长时间死连接）

对我们的启示
------

在应用层做连接的有效性检测是一个比较好的实践，也就是我们常说的心跳包。

小结
--

这篇文章我们介绍了 TCP keepalive 机制的由来，通过定时发送探测包来探测连接的对端是否存活，不过默认情况下需要 7200s 没有数据包交互才会发送 keepalive 探测包，往往这个时间太久了，我们熟知的很多组件都没有开启 keepalive 特性，而是选择在应用层做心跳机制。

思考题
---

TCP 的 keepalive 与 HTTP 的 keep-alive 有什么区别？


[Source](https://juejin.im/book/6844733788681928712/section/6844733788845506567)