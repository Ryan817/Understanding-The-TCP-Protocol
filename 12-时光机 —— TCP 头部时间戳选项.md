

TCP 头部时间戳选项（TCP Timestamps Option，TSopt）
----------------------------------------

### Timestamps 选项是什么

除了我们之前介绍的 MSS、Window Scale 还有以一个非常重要的选项：时间戳（TCP Timestamps Option，TSopt）。这个选项在 TCP 头部的位置如下所示。

![](https://user-gold-cdn.xitu.io/2019/6/14/16b54c4c5c635f86)

Timestamps 选项最初是在 RFC 1323 中引入的，这个 RFC 的标题是 "TCP Extensions for High Performance"，在这个 RFC 中同时提出的还有 Window Scale、PAWS 等机制。

### Timestamps 选项的组成部分

在 Wireshark 抓包中，常常会看到 TSval 和 TSecr 两个选项，值得注意的是第二个选项 TSecr 不是 secrets 的意思，而是 "TS Echo Reply" 的缩写，TSval 和 TSecr 是 TCP 选项时间戳的一部分。

TCP Timestamps Option 由四部分构成：类别（kind）、长度（Length）、发送方时间戳（TS value）、回显时间戳（TS Echo Reply）。时间戳选项类别（kind）的值等于 8，用来与其它类型的选项区分。长度（length）等于 10。两个时间戳相关的选项都是 4 字节。

如下图所示：

![](https://user-gold-cdn.xitu.io/2019/6/14/16b54c4be8611658)

是否使用时间戳选项是在三次握手里面的 SYN 报文里面确定的。下面的包是`curl github.com`抓包得到的结果。

![](https://user-gold-cdn.xitu.io/2019/6/14/16b54c4be8843d80)

*   发送方发送数据时，将一个发送时间戳 1734581141 放在发送方时间戳`TSval`中
*   接收方收到数据包以后，将收到的时间戳 1734581141 原封不动的返回给发送方，放在`TSecr`字段中，同时把自己的时间戳 3303928779 放在`TSval`中
*   后面的包以此类推

![](https://user-gold-cdn.xitu.io/2019/6/14/16b54c4c5c7ae349)

Timestamps 选项的作用
----------------

Timestamps 选项的提出初衷是为了解决两个问题：

1、两端往返时延测量（RTTM）

2、序列号回绕（PAWS），接下来我们来进行介绍。

### 测量 RTTM

发送端在收到接收方发出的 ACK 报文以后，就可以通过这个响应报文的 TSecr

在启用 timestamp 选项之前，测量 RTT 的过程如下。

![timestamps_rttm2](https://user-gold-cdn.xitu.io/2020/3/22/17102ef66fd1e657)

TCP 在发送一个包时，会记录这个包的发送的时间 t1，用收到这个包的确认包时 t2 减去 t1 就可以得到这次的 RTT。这里有一个问题，如果发出的包出现重传，计算就变得复杂起来，如下所示。

![timestamps_rttm](https://user-gold-cdn.xitu.io/2020/3/22/17102ef66de821b5)

这里的 RTT 到底是 t3 - t1 还是 t3 - t2 呢？这两种方式无论选择哪一种都不太合适，无法得知收到的确认 ACK 是对第一次包还是重传包的的确认。TCP RFC6298 对这种行为的处理是不对重传包进行 RTT 计算，这样计算不会带来错误，但当所有包都出现重传的情况下，将没有包可用来计算 RTT。

在启用 Timestamps 选项以后，因为 ACK 包里包含了 TSval 和 TSecr，这样无论是正常确认包，还是重传确认包，都可以通过这两个值计算出 RTT。

### PAWS

Timestamps 选项带来的第二个作用是帮助判断 PAWS，TCP 的序列号用 32bit 来表示，因此在 2^32 字节的数据传输后序列号就会溢出回绕。TCP 的窗口经过窗口缩放可以最高到 1GB（2^30)，在高速网络中，序列号在很短的时间内就会被重复使用。

下面以一个实际的例子来说明，如下图所示。

![paws](https://user-gold-cdn.xitu.io/2020/3/22/17102ef66f71cbd6)

假设发送了 6 个数据包，每个数据包的大小为 1GB，第 5 个包序列号发生回绕。第 2 个包因为某些原因延迟导致重传，但没有丢失到时间 t7 才到达。这个迷途数据包与后面要发送的第 6 个包序列号完全相同，如果没有一些措施进行区分，将会造成数据的紊乱。

如果有 Timestamps 的存在，内核会维护一个为每个连接维护一个 ts\_recent 值，记录最后一次通信的的 timestamps 值，在 t7 时间点收到迷途数据包 2 时，由于数据包 2 的 timestamps 值小于 ts\_recent 值，就会丢弃掉这个数据包。等 t8 时间点真正的数据包 6 到达以后，由于数据包 6 的 timestamps 值大于 ts\_recent，这个包可以被正常接收。

补充说明
----

有几个需要说明的点

*   timestamps 值是一个单调递增的值，与我们所知的 epoch 时间戳不是一回事，这个选项不要求两台主机进行时钟同步。两端 timestamps 值增加的间隔也可能步调不一致，比如一条主机以每 1ms 加一的方式递增，另外一条主机可以以每 1s 加一的方式递增。
    
*   与序列号一样，既然是递增 timestamps 值也是会溢出回绕的。
    
*   timestamps 是一个双向的选项，如果只要有一方不开启，双方都将停用 timestamps。比如下面是`curl www.baidu.com`得到的包。
    

![](https://user-gold-cdn.xitu.io/2019/6/14/16b54c4c6e0a8f69)

可以看到客户端发起 SYN 包时带上了自己的 TSval，服务器回复的 SYN+ACK 包没有 TSval和TSecr，从此之后的包都没有带上时间戳选项了。

Timestamps 选项造成的 RST
--------------------

三次握手中的第二步，如果服务端回复 SYN+ACK 包中的 TSecr 不等于握手第一步客户端发送 SYN 包中的 TSval，客户端在对 SYN+ACK 回复 RST。示例包如下所示。

![](https://user-gold-cdn.xitu.io/2020/3/22/17102ef678eeb40a)

待补充内容
-----

随着 Timestamps 选项的引入，带来了一些安全性相关的问题，因为比较冷门，如果有读者感兴趣，可以留言，后面我再补充。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788824551438)