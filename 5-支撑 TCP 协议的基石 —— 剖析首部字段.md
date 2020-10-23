

这篇文章来讲讲 TCP 报文首部相关的概念，这些头部是支撑 TCP 复杂功能的基石。 完整的 TCP 头部如下图所示

![](https://user-gold-cdn.xitu.io/2019/9/27/16d702629b61cbcc)

我们用一次访问百度网页抓包的例子来开始。

    curl -v www.baidu.com


完整的抓包文件可以来 github 下载：[curl\_baidu.pcapng](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_header/curl_baidu.pcapng)

![](https://user-gold-cdn.xitu.io/2019/9/27/16d702629babb3f6)

源端口号、目标端口号
----------

在第一个包的详情中，首先看到的高亮部分的源端口号（Src Port）和目标端口号（Dst Port)，这个例子中本地源端口号为 61024，百度目标端口号是 80。

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70262f5e8a192)

TCP 报文头部里没有源 ip 和目标 ip 地址，只有源端口号和目标端口号

这也是初学 wireshark 抓包时很多人会有的一个疑问：过滤 ip 地址为 172.19.214.24 包的条件为什么不是 "tcp.addr == 172.19.214.24"，而是 "ip.addr == 172.19.214.24"

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70262f65563ce)

TCP 的报文里是没有源 ip 和目标 ip 的，因为那是 IP 层协议的事情，TCP 层只有源端口和目标端口。

源 IP、源端口、目标 IP、目标端口构成了 TCP 连接的「四元组」。一个四元组可以唯一标识一个连接。

后面文章中专门有一节是用来介绍端口号相关的知识。

* * *

接下来，我们看到的是序列号，如截图中 2 的标识。

序列号（Sequence number）
--------------------

TCP 是面向字节流的协议，通过 TCP 传输的字节流的每个字节都分配了序列号，序列号（Sequence number）指的是本报文段第一个字节的序列号。

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70262f7fb618a)

序列号加上报文的长度，就可以确定传输的是哪一段数据。序列号是一个 32 位的无符号整数，达到 2^32-1 后循环到 0。

在 SYN 报文中，序列号用于交换彼此的初始序列号，在其它报文中，序列号用于保证包的顺序。

因为网络层（IP 层）不保证包的顺序，TCP 协议利用序列号来解决网络包乱序、重复的问题，以保证数据包以正确的顺序组装传递给上层应用。

如果发送方发送的是四个报文序列号分别是1、2、3、4，但到达接收方的顺序是 2、4、3、1，接收方就可以通过序列号的大小顺序组装出原始的数据。

### 初始序列号（Initial Sequence Number, ISN）

在建立连接之初，通信双方都会各自选择一个序列号，称之为初始序列号。在建立连接时，通信双方通过 SYN 报文交换彼此的 ISN，如下图所示

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70264eaa6828c)

初始建立连接的过程中 SYN 报文交换过程如下图所示

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70264ef144241)

其中第 2 步和第 3 步可以合并一起，这就是三次握手的过程

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70264f4692792)

### 初始序列号是如何生成的

    __u32 secure_tcp_sequence_number(__be32 saddr, __be32 daddr,
    				 __be16 sport, __be16 dport)
    {
    	u32 hash[MD5_DIGEST_WORDS];
    
    	net_secret_init();
    	hash[0] = (__force u32)saddr;
    	hash[1] = (__force u32)daddr;
    	hash[2] = ((__force u16)sport << 16) + (__force u16)dport;
    	hash[3] = net_secret[15];
    	
    	md5_transform(hash, net_secret);
    
    	return seq_scale(hash[0]);
    }
    
    static u32 seq_scale(u32 seq)
    {
    	return seq + (ktime_to_ns(ktime_get_real()) >> 6);
    }


代码中的 net\_secret 是一个长度为 16 的 int 数组，只有在第一次调用 net\_secret\_init 的时时候会将将这个数组的值初始化为随机值。在系统重启前保持不变。

可以看到初始序列号的计算函数 secure\_tcp\_sequence\_number() 的逻辑是通过源地址、目标地址、源端口、目标端口和随机因子通过 MD5 进行进行计算。如果仅有这几个因子，对于四元组相同的请求，计算出的初始序列号总是相同，这必然有很大的安全风险，所以函数的最后将计算出的序列号通过 seq\_scale 函数再次计算。

seq\_scale 函数加入了时间因子，对于四元组相同的连接，序列号也不会重复了。

### 序列号回绕了怎么处理

序列号是一个 32 位的无符号整数，从前面介绍的初始序列号计算算法可以知道，ISN 并不是从 0 开始，所以同一个连接的序列号是有可能溢出回绕（sequence wraparound）的。TCP 的很多校验比如丢包、乱序判断都是通过比较包的序号来实现的，我们来看看 linux 内核是如何处理的，代码如下所示。

    static inline bool before(__u32 seq1, __u32 seq2)
    {
            return (__s32)(seq1-seq2) < 0;
    }


其中 `__u32` 表示无符号的 32 位整数，`__s32` 表示有符号的 32 位整数。为什么 seq1 - seq2 转为有符号的 32 位整数就可以判断 seq1 和 seq2 的大小了呢？

以 seq1 为 0xFFFFFFFF、seq2 为 0x02（回绕）为例，它们相减的结果如下。

    seq1 - seq2 = 0xFFFFFFFF - 0x02 = 0xFFFFFFFD


0xFFFFFFFD 最高位为 1，表示为负数，实际值为 -(0x00000002 + 1) = -3，这样即使 seq2 回绕了，也可以知道 seq1<seq2。

* * *

确认号
---

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70265e2c9f7c0)

TCP 使用确认号（Acknowledgment number, ACK）来告知对方下一个期望接收的序列号，小于此确认号的所有字节都已经收到。

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70265e3d247b3)

关于确认号有几个注意点：

*   不是所有的包都需要确认的
*   不是收到了数据包就立马需要确认的，可以延迟一会再确认
*   ACK 包本身不需要被确认，否则就会无穷无尽死循环了
*   确认号永远是表示小于此确认号的字节都已经收到

TCP Flags
---------

TCP 有很多种标记，有些用来发起连接同步初始序列号，有些用来确认数据包，还有些用来结束连接。TCP 定义了一个 8 位的字段用来表示 flags，大部分都只用到了后 6 个，如下图所示

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70265e3ed0a23)

下面这个是 wireshark 第一个 SYN 包的 flags 截图

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70266d81f6e83)

我们通常所说的 SYN、ACK、FIN、RST 其实只是把 flags 对应的 bit 位置为 1 而已，这些标记可以组合使用，比如 SYN+ACK，FIN+ACK 等

最常见的有下面这几个：

*   SYN（Synchronize）：用于发起连接数据包同步双方的初始序列号
*   ACK（Acknowledge）：确认数据包
*   RST（Reset）：这个标记用来强制断开连接，通常是之前建立的连接已经不在了、包不合法、或者实在无能为力处理
*   FIN（Finish）：通知对方我发完了所有数据，准备断开连接，后面我不会再发数据包给你了。
*   PSH（Push）：告知对方这些数据包收到以后应该马上交给上层应用，不能缓存起来

窗口大小
----

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70266d82aaebd)

可以看到用于表示窗口大小的"Window Size" 只有 16 位，可能 TCP 协议设计者们认为 16 位的窗口大小已经够用了，也就是最大窗口大小是 65535 字节（64KB）。就像网传盖茨曾经说过：“640K内存对于任何人来说都足够了”一样。

自己挖的坑当然要自己填，因此TCP 协议引入了「TCP 窗口缩放」选项 作为窗口缩放的比例因子，比例因子值的范围是 0 ~ 14，其中最小值 0 表示不缩放，最大值 14。比例因子可以将窗口扩大到原来的 2 的 n 次方，比如窗口大小缩放前为 1050，缩放因子为 7，则真正的窗口大小为 1050 \* 128 = 134400，如下图所示

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70266db317004)

在 wireshark 中最终的窗口大小会自动计算出来，如下图中的 Calculated window size。以本文中抓包的例子为例

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70266f7f45c6b)

值得注意的是，窗口缩放值在三次握手的时候指定，如果抓包的时候没有抓到 SYN 包，wireshark 是不知道真正的窗口缩放值是多少的。

可选项
---

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70266f7d32a0f)

可选项的格式入下所示

![](https://user-gold-cdn.xitu.io/2019/9/27/16d70266f820427e)

以 MSS 为例，kind=2，length=4，value=1460

![](https://user-gold-cdn.xitu.io/2019/9/27/16d7026733bbfa60)

常用的选项有以下几个：

*   MSS：最大段大小选项，是 TCP 允许的从对方接收的最大报文段
*   SACK：选择确认选项
*   Window Scale：窗口缩放选项

作业题
---

1、如果一个 TCP 连接正在传送 5000 字节的数据，第一个字节的序号是 10001，数据被分为 5 段，每个段携带 1000 字节，请问每个段的序号是什么？

2、A B 两个主机之间建立了一个 TCP 连接，A 主机发给 B 主机两个 TCP 报文，大小分别是 500 和 300，第一个报文的序列号是 200，那么 B 主机接收两个报文后，返回的确认号是（）

*   A、200
*   B、700
*   C、800
*   D、1000

3、客户端的使用 ISN=2000 打开一个连接，服务器端使用 ISN=3000 打开一个连接，经过 3 次握手建立连接。连接建立起来以后，假定客户端向服务器发送一段数据`Welcome the server!`（长度 20 Bytes），而服务器的回答数据`Thank you!`（长度 10 Bytes ），试画出三次握手和数据传输阶段报文段序列号、确认号的情况。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788816162823)