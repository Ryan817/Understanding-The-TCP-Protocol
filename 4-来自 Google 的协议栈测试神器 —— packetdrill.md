

从大学开始懵懵懂懂粗略学习（死记硬背）了一些 TCP 协议的内容，到工作多年以后，一直没有找到顺手的网络协议栈调试工具，对于纷繁复杂 TCP 协议。业界流行的 scapy 不是很好用，有很多局限性。直到前段时间看到了 Google 开源的 packetdrill，真有一种相见恨晚的感觉。这篇文章讲介绍 packetdrill 的基本原理和用法。

![](https://user-gold-cdn.xitu.io/2019/2/23/16918dad21e0af9f)

packetdrill 在 2013 年开源，在 Google 内部久经考验，Google 用它发现了 10 余个 Linux 内核 bug，同时用测试驱动开发的方式开发新的网络特性和进行回归测试，确保新功能的添加不影响网络协议栈的可用性。

0x01 安装
-------

以 centos7 为例

1.  首先从 github 上 clone 最新的源码 [github.com/google/pack…](https://github.com/google/packetdrill)
2.  进入源码目录`cd gtests/net/packetdrill`
3.  安装 bison和 flex 库：`sudo yum install -y bison flex`
4.  为避免 offload 机制对包大小的影响，修改 netdev.c 注释掉 set\_device\_offload\_flags 函数所有内容
5.  执行 `./configure`
6.  修改 `Makefile`，去掉第一行的末尾的 `-static`
7.  执行 make 命令编译
8.  确认编译无误地生成了 packetdrill 可执行文件

0x02 初体验
--------

packetdrill 脚本采用 c 语言和 tcpdump 混合的语法。脚本文件名一般以 .pkt 为后缀，执行脚本的方式为`sudo ./packetdrill test.pkt`

脚本的每一行可以由以下几种类型的语句构成：

*   执行系统调用（system call），对比返回值是否符合预期
*   把数据包（packet）注入到内核协议栈，模拟协议栈收到包
*   比较内核协议栈发出的包与预期是否相符
*   执行 shell 命令
*   执行 python 命令

脚本每一行都有一个时间参数用来表明执行的时间或者预期事件发生的时间，packetdrill 支持绝对时间和相对时间。绝对时间就是一个简单的数字，相对时间会在数字前面添加一个`+`号。比如下面这两个例子

    // 300ms 时执行 accept 调用
    0.300 accept(3, ..., ...) = 4
    
    // 在上一行语句执行结束 10ms 以后执行
    +.010 write(4, ..., 1000) = 1000`


如果预期的事件在指定的时间没有发生，脚本执行会抛出异常，由于不同机器的响应时间不同，所以 packetdrill 提供了参数（--tolerance\_usecs）用来设置误差范围，默认值是 4000us（微秒），也即 4ms。这个参数默认值在 config.c 的 set\_default\_config 函数里进行设置`config->tolerance_usecs = 4000;`

我们以一个最简单的 demo 来演示 packetdrill 的用法。乍一看很懵，容我慢慢道来

      1 0   socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
      2 +0  setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
      3 +0  bind(3, ..., ...) = 0
      4 +0  listen(3, 1) = 0
      5
      6 
      7 +0  < S 0:0(0) win 4000 <mss 1000>
      8 +0  > S. 0:0(0) ack 1 <...>
      9 +.1 < . 1:1(0) ack 1 win 1000
     10
     11 +0 accept(3, ..., ...) = 4
     12 +0 < P. 1:201(200) win 4000
     13 +0 > . 1:1(0) ack 201


第 1 行：`0 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3`

在脚本执行的第 0s 创建一个 socket，使用的是系统调用的方式，socket 函数的签名和用法如下

    #include <sys/socket.h>
    int socket(int domain, int type, int protocol);
    
    成功时返回文件描述符，失败时返回 -1
    int socket_fd = socket(AF_INET, SOCK_STREAM, 0);


*   domain 表示套接字使用的协议族信息，IPv4、IPv6等。AF\_INET 表示 IPv4 协议族，AF\_INET6 表示 IPv6 协议族。绝大部分使用场景下都是用 AF\_INET，即 IPv4 协议族
*   type 表示套接字数据传输类型信息，主要分为两种：面向连接的套接字（SOCK\_STREAM）和面向无连接报文的套接字（SOCK\_DGRAM）。众所周知，SOCK\_STREAM 默认协议是 TCP，SOCK\_DGRAM 的默认协议是 UDP。
*   protocol 这个参数通常是 0，表示为给定的协议族和套接字类型选择默认协议。

在 packetdrill 脚本中用 `...` 来表示当前参数省略不相关的细节信息，使用 packetdrill 程序的默认值。

脚本返回新建的 socket 文件句柄，这里用`=`来断言会返回`3`，因为linux 在每个程序开始的时刻，都会有 3 个已经打开的文件句柄，分别是：标准输入stdin(0)、标准输出stdout(1)、错误输出stderr(2) 默认的，其它新建的文件句柄则排在之后，从 3 开始。

![](https://user-gold-cdn.xitu.io/2019/2/23/16918dad1a156295)

    2 +0  setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
    3 +0  bind(3, ..., ...) = 0
    4 +0  listen(3, 1) = 0


*   第 2 行：调用 setsockopt 函数设置端口重用。
*   第 3 行：调用 bind 函数，这里的 socket 地址省略会使用默认的端口 8080，第一个参数 3 是套接字的 fd
*   第 4 行：调用 listen 函数，第一个参数 3 也是套接字 fd 到此为止，socket 已经可以接受客户端的 tcp 连接了。

第 7 ~ 9 行是经典的三次握手，packetdrill 的语法非常类似 tcpdump 的语法

`<` 表示输入的数据包（input packets)， packetdrill 会构造一个真实的数据包，注入到内核协议栈。比如：

    // 构造 SYN 包注入到协议栈
    +0  < S 0:0(0) win 32792 <mss 1000,sackOK,nop,nop,nop,wscale 7>
    
    // 构造 icmp echo_reply 包注入到协议栈
    0.400 < icmp echo_reply


`>` 表示预期协议栈会响应的包（outbound packets），这个包不是 packetdrill 构造的，是由协议栈发出的，packetdrill 会检查协议栈是不是真的发出了这个包，如果没有，则脚本报错停止执行。比如

    // 调用 write 函数调用以后，检查协议栈是否真正发出了 PSH+ACK 包
    +0  write(4, ..., 1000) = 1000
    +0  > P. 1:1001(1000) ack 1
    
    // 三次握手中过程向协议栈注入 SYN 包以后，检查协议栈是否发出了 SYN+ACK 包以及 ack 是否等于 1
    0.100 < S 0:0(0) win 32792 <mss 1000,nop,wscale 7>
    0.100 > S. 0:0(0) ack 1 <mss 1460,nop,wscale 6>


![](https://user-gold-cdn.xitu.io/2019/2/23/16918dad15d16381)

第 7 行：`+0 < S 0:0(0) win 1000 <mss 1000>`

packetdrill 构造一个 SYN 包发送到协议栈，它使用与 tcpdump 类似的相对 sequence 序号，S 后面的三个 0 ，分别表示发送包的起始 seq、结束 seq、包的长度。比如`P. 1:1001(1000)`表示发送的包起始序号为 1，结束 seq 为 1001，长度为1000。紧随其后的 win 表示发送端的接收窗口大小 1000。依据 TCP 协议，SYN 包也必须带上自身的 MSS 选项，这里的 MSS 大小为 1000

第 8 行：`+0 > S. 0:0(0) ack 1 <...>`

预期协议栈会立刻回复 SYN+ACK 包，因为还没有发送数据，所以包的 seq开始值、结束值、长度都为 0，ack 为上次 seq + 1，表示第一个 SYN 包已收到。

> 第 9 行：`+.1 < . 1:1(0) ack 1 win 1000`

0.1s 以后注入一个 ACK 包到协议栈，没有携带数据，包的长度为 0，至此三次握手完成，过程如下图

![](https://user-gold-cdn.xitu.io/2019/2/23/16918dad15f09bbf)

`+0 accept(3, ..., ...) = 4` accept 系统调用返回了一个值为 4 的新的文件 fd，这时 packetdrill 可以往这个 fd 里面写数据了

    +0 write(4, ..., 10)=10
    +0 > P. 1:11(10) ack 1
    +.1 < . 1:1(0) ack 11 win 1000


packetdrill 调用 write 函数往 socket 里写了 10 字节的数据，协议栈立刻发出这 10 个字节数据包，同时把 PSH 标记置为 1。这个包的起始 seq 为 1，结束 seq 为 10，长度为 10。100ms 以后注入 ACK 包，模拟协议栈收到 ACK 包。

整个过程如下

![](https://user-gold-cdn.xitu.io/2019/2/23/16918dad1855b751)

采用 tcpdump 对 8080 端口进行抓包，结果如下

    sudo tcpdump -i any port 8080 -nn                                                                                                                                                                   
    10:02:36.591911 IP 192.0.2.1.37786 > 192.168.31.139.8080: Flags [S], seq 0, win 4000, options [mss 1000], length 0
    10:02:36.591961 IP 192.168.31.139.8080 > 192.0.2.1.37786: Flags [S.], seq 2327356581, ack 1, win 29200, options [mss 1460], length 0
    10:02:36.693785 IP 192.0.2.1.37786 > 192.168.31.139.8080: Flags [.], ack 1, win 1000, length 0
    10:02:36.693926 IP 192.168.31.139.8080 > 192.0.2.1.37786: Flags [P.], seq 1:11, ack 1, win 29200, length 10
    10:02:36.801092 IP 192.0.2.1.37786 > 192.168.31.139.8080: Flags [.], ack 11, win 1000, length 0


0x03 packetdrill 原理简述
---------------------

在脚本的最后一行，加上

    +0 `sleep 1000000`


让脚本执行完不要退出，执行 ifconfig 可以看到，比没有执行脚本之前多了一个虚拟的网卡 tun0。

![](https://user-gold-cdn.xitu.io/2019/2/23/16918dad1a9ae238)

packetdrill 就是在执行脚本前创建了一个名为 tun0 的虚拟网卡，脚本执行完，tun0 会被销毁。该虚拟网卡对应于操作系统中`/dev/net/tun`文件，每次程序通过 write 等系统调用将数据写入到这个文件 fd 时，这些数据会经过 tun0 这个虚拟网卡，将数据写入到内核协议栈，read 系统调用读取数据的过程类似。协议栈可以向操作普通网卡一样操作虚拟网卡 tun0。

关于 linux 下 tun 的详细使用介绍，可以参考 IBM 的文章 [www.ibm.com/developerwo…](https://www.ibm.com/developerworks/cn/linux/l-tuntap/index.html)

0x04 把 packetdrill 命令加到环境变量里
----------------------------

把 packetdrill 加入到环境变量里以便于可以在任意目录可以执行。第一步是修改`/etc/profile`或者`.zshrc`（如果你用的是最好用的 zsh 的话）等可以修改环境变量的文件。

    export PATH=/path_to_packetdrill/:$PATH
    
    source ~/.zshrc


在命令行中输入 packetdrill 如果有输出 packetdrill 的 usage 文档说明第一步成功啦。

但是 packetdrill 命令是需要 sudo 权限执行的，如果现在我们在命令行中输入`sudo packetdrill`，会提示找不到 packetdrill 命令

    sudo：packetdrill：找不到命令


这是因为 sudo 命令为了安全性的考虑，覆盖了用户自己 PATH 环境变量，我们可以用`sudo sudo -V | grep PATH` 来看

    sudo sudo -V | grep  PATH                                                                                                                                  
    覆盖用户的 $PATH 变量的值：/sbin:/bin:/usr/sbin:/usr/bin


可以看到 sudo 命令覆盖了用户的 PATH 变量。这些初始值是在`/etc/sudoers`中定义的

    sudo cat /etc/sudoers | grep -i PATH                                                                                                                          
    Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin


一个最简单的办法是在sudo 启动时重新赋值它的 PATH 变量：`sudo env PATH="$PATH" cmd_x`，可以用`sudo env PATH="$PATH" env | grep PATH`与`sudo env | grep PATH`做前后对比

![](https://user-gold-cdn.xitu.io/2019/5/7/16a8fc67c4be0c8b)

对于本文中的 packetdrill，可以用`sudo env PATH=$PATH packetdrill delay_ack.pkt`来执行，当然你可以做一个 sudo 的 alias

    alias sudo='sudo env PATH="$PATH"'


这样就可以在任意地方执行`sudo packetdrill`了

0x05 小结
-------

packetdrill 上手的难度有一点大，但是熟悉了以后用起来特别顺手，后面很多 TCP 包超时重传、快速重传、滑动窗口、nagle 算法都是会用这个工具来进行测试，希望你可以熟练掌握。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788816146445)