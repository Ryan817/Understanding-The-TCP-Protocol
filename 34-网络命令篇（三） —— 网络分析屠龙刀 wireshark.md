

这篇文章我们讲解 wireshark。前面我们介绍了 tcpdump，它是命令行程序，对 linux 服务器比较友好，简单快速适合简单的文本协议的分析和处理。wireshark 有图形化的界面，分析功能非常强大，不仅仅是一个抓包工具，且支持众多的协议。它也有命令行版本的叫做 tshark，不过用的比较少一点。

抓包过滤
----

抓包的过程很耗 CPU 和内存资源而且大部分情况下我们不是对所有的包都感兴趣，因此可以只抓取满足特定条件的包，丢弃不感兴趣的包，比如只想抓取 ip 为172.18.80.49 端口号为 3306 的包，可以输入`host 172.18.80.49 and port 3306`

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8ea8ed884dd)

显示过滤（Display filter）
--------------------

显示过滤可以算是 wireshark 最常用的功能了，与抓包过滤不一样的是，显示过滤不会丢弃包的内容，不符合过滤条件的包被隐藏起来，方便我们阅读。

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8ea8f77b625)

过滤的方式常见的有以下几种：

*   协议、应用过滤器（ip/tcp/udp/arp/icmp/ dns/ftp/nfs/http/mysql)
*   字段过滤器（http.host/dns.qry.name）

比如我们只想看 http 协议报文，在过滤器中输入 http 即可

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8ea982fb47d)

字段过滤器可以更加精确的过滤出想要的包，比如我们只想看锤科网站`t.tt`域名的 dns 解析，可以输入`dns.qry.name == t.tt`

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8ea986c9baa)

再比如，我只想看访问锤科的 http 请求，可以输入`http.host == t.tt`

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8ea987c3d41)

要想记住这些很难，有一个小技巧，比如怎么知道 域名为`t.tt` 的 dns 查询要用`dns.qry.name`呢？

可以随便找一个 dns 的查询，找到查询报文，展开详情里面的内容，然后鼠标选中想过滤的字段，最下面的状态码就会出现当前 wireshark 对应的查看条件，比如下图中的`dns.qry.name`

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8eab690a616)

常用的查询条件有：

tcp 相关过滤器

*   tcp.flags.syn==1：过滤 SYN 包
*   tcp.flags.reset==1：过滤 RST 包
*   tcp.analysis.retransmission：过滤重传包
*   tcp.analysis.zero\_window：零窗口

http 相关过滤器

*   http.host==t.tt：过滤指定域名的 http 包
*   http.response.code==302：过滤http响应状态码为302的数据包
*   http.request.method==POST：过滤所有请求方式为 POST 的 http 请求包
*   http.transfer\_encoding == "chunked" 根据transfer\_encoding过滤
*   http.request.uri contains "/appstock/app/minute/query"：过滤 http 请求 url 中包含指定路径的请求

通信延迟常用的过滤器

*   `http.time>0.5`：请求发出到收到第一个响应包的时间间隔，可以用这个条件来过滤 http 的时延
*   tcp.time\_delta>0.3：tcp 某连接中两次包的数据间隔，可以用这个来分析 TCP 的时延
*   dns.time>0.5：dns 的查询耗时

wireshakr 所有的查询条件在这里可以查到：https:/ /www.wireshark.org/docs/dfref/

比较运算符
-----

wireshark 支持比较运算符和逻辑运算符。这些运算符可以灵活的组合出强大的过滤表达式。

*   等于：== 或者 eq
*   不等于：!= 或者 ne
*   大于：> 或者 gt
*   小于：< 或者 lt
*   包含 contains
*   匹配 matches
*   与操作：AND 或者 &&
*   或操作：OR 或者 ||
*   取反：NOT 或者 !

比如想过滤 ip 来自 192.168.1.1 且是 TCP 协议的数据包：

    ip.addr == 10.0.0.10 and tcp


从 wireshark 看协议分层
-----------------

下图是抓取的一次 http 请求的包`curl http://www.baidu.com`：

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8eac9b0de7a)

可以看到协议的分层，从上往下依次是

*   Frame：物理层的数据帧
*   Ethernet II：数据链路层以太网帧头部信息
*   Internet Protocol Version 4：互联网层IP包头部信息
*   Transmission Control Protocol：传输层的数据段头部信息，此处是TCP协议
*   Hypertext Transfer Protocol：应用层 HTTP 的信息

跟踪 TCP 数据流（Follow TCP Stream）
-----------------------------

在实际使用过程中，跟踪 TCP 数据流是一个很高频的使用。我们通过前面介绍的那些过滤条件找到了一些包，大多数情况下都需要查看这个 TCP 连接所有的包来查看上下文。

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8eac9e95d60)

这样就可以查看整个连接的所有包交互情况了，如下图所示，三次握手、数据传输、四次挥手的过程一目了然

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8eacdb81ff6)

解密HTTPS包
--------

随着 https 和 http2.0 的流行，https 正全面取代 http，这给我们抓包带来了一点点小困难。Wireshark 的抓包原理是直接读取并分析网卡数据。 下图是访问 [www.baidu.com](https://www.baidu.com/) 的部分包截图，传输包的内容被加密了。

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8ead6c9dea3)

要想让它解密 HTTPS 流量，要么拥有 HTTPS 网站的加密私钥，可以用来解密这个网站的加密流量，但这种一般没有可能拿到。要么某些浏览器支持将 TLS 会话中使用的对称加密密钥保存在外部文件中，可供 Wireshark 解密流量。 在启动 Chrome 时加上环境变量 SSLKEYLOGFILE 时，chrome 会把会话密钥输出到文件。

    SSLKEYLOGFILE=/tmp/SSLKEYLOGFILE.log /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome


wireshark 可以在`Wireshark -> Preferences... -> Protocols -> SSL`打开Wireshark 的 SSL 配置面板，在`(Pre)-Master-Secret log filename`选项中输入 SSLKEYLOGFILE 文件路径。

![](https://user-gold-cdn.xitu.io/2019/3/30/169cd8eae8cef115)

这样就可以查看加密前的 https 流量了

书籍推荐
----

上面仅列举出了部分常用的选项，关于 wireshark 可以写的东西非常多，推荐林沛满写的 wireshark 系列，我从中受益匪浅。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788853895182)