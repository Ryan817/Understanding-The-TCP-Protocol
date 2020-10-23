

这篇文章我们来介绍 TCP RST 攻击以及如何在不干预通信双方进程的情况下杀掉一条 TCP 连接。

RST 攻击
------

RST 攻击也称为伪造 TCP 重置报文攻击，通过伪造 RST 报文来关闭掉一个正常的连接。

源 IP 地址伪造非常容易，不容易被伪造的是序列号，RST 攻击最重要的一点就是构造的包的序列号要落在对方的滑动窗口内，否则这个 RST 包会被忽略掉，达不到攻击的效果。

下面我们用实验演示不在滑动窗口内的 RST 包会被忽略的情况，完整的代码见：[rst\_out\_of\_window.pkt](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_rst/rst_out_of_window.pkt)

    +0 < S 0:0(0) win 32792 <mss 1460> 
    +0 > S. 0:0(0) ack 1 <...>
    +.1 < . 1:1(0) ack 1 win 65535 
    +0 accept(3, ..., ...) = 4


​    
    +.010 < R. 29202:29202(0) ack 1 win 65535


​    
    +.010 write(4, ..., 1000) = 1000 


​    
    +0 > P. 1:1001(1000) ack 1 <...>


执行上面的脚本，抓包的结果如下，完整的包见：[rst\_out\_of\_window.pcap](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_rst/rst_out_of_window.pcap)

![](https://store-g1.seewo.com/pics/201906221561214405280615612144052280.jpg)

抓包文件中的第 5 个包可以看到，write 调用成功，1000 字节发送成功，write 调用并没有收到 RST 包的影响。

下面来介绍两个工具，利用 RST 攻击的方式来杀掉一条连接。

工具一：tcpkill 工具使用及原理介绍
---------------------

Centos 下安装 tcpkill 命令步骤如下

    yum install epel-release -y
    yum install dsniff -y


实验步骤： 1、机器 c2(10.211.55.10) 启动 nc 命令监听 8080 端口，充当服务器端，记为 B

    nc -l 8080


2、机器 c2 启动 tcpdump 抓包

    sudo tcpdump -i any port 8080 -nn -U -vvv -w test.pcap


3、本地机器终端（10.211.55.2，记为 A）使用 nc 与 B 的 8080 端口建立 TCP 连接

    nc c2 8080


在服务端 B 机器上可以看到这条 TCP 连接

    netstat -nat | grep -i 8080
    tcp        0      0 10.211.55.10:8080       10.211.55.2:60086       ESTABLISHED


4、启动 tcpkill

    sudo tcpkill -i eth0 port 8080


注意这个时候 tcp 连接依旧安然无恙，并没有被杀掉。

5、在本地机器终端 nc 命令行中随便输入一点什么，这里输入`hello`，发现这时服务端和客户端的 nc 进程已经退出了

下面来分析抓包文件，这个文件可以从我的 github 下载 [tcpkill.pcap](https://github.com/arthur-zhang/tcp_ebook/tree/master/kill_tcp_connection)

![](https://user-gold-cdn.xitu.io/2019/6/22/16b7eb9c7490b760)

可以看到，tcpkill 假冒了 A 和 B 的 IP发送了 RST 包给通信的双方，那问题来了，伪造 ip 很简单，它是怎么知道当前会话的序列号的呢？

tcpkill 的原理跟 tcpdump 差不多，会通过 libpcap 库抓取符合条件的包。 因此只有有数据传输的 tcp 连接它才可以拿到当前会话的序列号，通过这个序列号伪造 IP 发送符合条件的 RST 包。

原理如下图所示

![](https://user-gold-cdn.xitu.io/2019/6/22/16b7eb9c74a68a15)

可以看到 tcpkill 对每个端发送了 3 个RST 包，这是因为在高速数据传输的连接上，根据当前抓的包计算的序列号可能已经不再 TCP 连接的窗口内了，这种情况下 RST 包会被忽略，因此默认情况下 tcpkill 未雨绸缪往后计算了几个序列号。还可以指定参数`-n`指定更多的 RST 包，比如`tcpkill -9`

根据上面的分析 tcpkill 的局限还是很明显的，无法杀掉一条僵死连接，下面我们介绍一个新的工具 killcx，看看它是如何来处理这种情况的。

killcx
------

killcx 是一个用 perl 写的在 linux 下可以关闭 TCP 连接的脚本，无论 TCP 连接处于什么状态。

下面来做一下实验，实验的前几步骤跟第一个例子中一模一样

1、机器 c2(10.211.55.10) 启动 nc 命令监听 8080 端口，充当服务器端，记为 B

    nc -l 8080


2、机器 c2 启动 tcpdump 抓包

    sudo tcpdump -i any port 8080 -nn -U -vvv -w test.pcap


3、本地机器终端（10.211.55.2，记为 A）使用 nc 与 B 的 8080 端口建立 TCP 连接

    nc c2 8080


在服务端 B 机器上可以看到这条 TCP 连接

    netstat -nat | grep -i 8080
    tcp        0      0 10.211.55.10:8080       10.211.55.2:61632       ESTABLISHED


4、客户端 A nc 命令行随便输入什么，这一步也完全可以省略，这里输入"hello\\n"

5、执行 killcx 命令，注意 killcx 是在步骤 4 之后执行的

    sudo ./killcx 10.211.55.2:61632


可以看到服务端和客户端的 nc 进程已经退出了。

抓包的结果如下

![](https://user-gold-cdn.xitu.io/2019/6/22/16b7eb9cac8894d7)

前 5 个包都很正常，三次握手加上一次数据传输，有趣的事情从第 6 个包开始

*   第 6 个包是 killcx 伪造 IP 向服务端 B 发送的一个 SYN 包
*   第 7 个包是服务端 B 回复的 ACK 包，里面包含的 SEQ 和 ACK 号
*   第 8 个包是 killcx 伪造 IP 向服务端 B 发送的 RST 包
*   第 9 个包是 killcx 伪造 IP 向客户端 A 发送的 RST 包

整个过程如下图所示

![](https://user-gold-cdn.xitu.io/2019/6/22/16b7eb9c74a1f89a)

小结
--

这篇文章介绍了杀掉 TCP 连接的两个工具 tcpkill 和 killcx：

*   tcpkill 采用了比较保守的方式，抓取流量等有新包到来的时候，获取 SEQ/ACK 号，这种方式只能杀掉有数据传输的连接
*   killcx 采用了更加主动的方式，主动发送 SYN 包获取 SEQ/ACK 号，这种方式活跃和非活跃的连接都可以杀掉

扩展阅读
----

有大神把 tcpkill 源代码魔改了一下，让 tcpkill 也支持了杀掉非活跃连接，原理上就是结合了 killcx 杀掉连接的方式，模拟 SYN 包。有兴趣的读者可以好好读一下：[yq.aliyun.com/articles/59…](https://yq.aliyun.com/articles/59308)


[Source](https://juejin.im/book/6844733788681928712/section/6844733788845522951)