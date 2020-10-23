

今天我们来介绍三个常用的命令：telnet、nc 和 netstat

命令一：telnet
----------

现在 telnet server 几乎没有人在用了，但是 telnet client 却被广泛的使用着。它的功能已经比较强大，有较多巧妙的用法。下面选取几个用的比较多的来介绍一下。

### 0x01 检查端口是否打开

telnet 的一个最大作用就是检查一个端口是否处于打开，使用的命令是 `telnet [domainname or ip] [port]`，这条命令能告诉我们到远端 server 指定端口的网连接是否可达。

> telnet \[domainname or ip\] \[port\]

telnet 第一个参数是要连接的域名或者 ip，第二个参数是要连接的端口。

比如你要连接 220.181.57.216（百度) 服务器上的 80 端口，可以使用如下的命令：`telnet 220.181.57.216 80`

如果这个网络连接可达，则会提示你`Connected to 220.181.57.216`，输入`control ]`可以给这个端口发送数据包了

![-w349](https://user-gold-cdn.xitu.io/2019/5/29/16b04184c7883850)

如果网路不可达，则会提示`telnet: Unable to connect to remote host`和具体不能连上的原因，常见的有 Operation timed out、Connection refused。

比如我本机没有进程监听 90 端口，`telnet 127.0.0.1 90`的信息如下

![-w549](https://user-gold-cdn.xitu.io/2019/5/29/16b04184cf017730)

### 0x02 telnet 还能发 http 请求？

我们知道 curl 可以方便的发送 http 请求，telnet 也是可以方便的发送 http 请求的

执行 `telnet www.baidu.com 80`，粘贴下面的文本（注意总共有四行，最后两行为两个空行）

    GET / HTTP/1.1
    Host: www.baidu.com


​    
​    

可以看到返回了百度的首页

    ➜ telnet www.baidu.com 80
    Trying 14.215.177.38...
    Connected to www.a.shifen.com.
    Escape character is '^]'.
    GET / HTTP/1.1
    Host: www.baidu.com
    
    HTTP/1.1 200 OK
    Accept-Ranges: bytes
    Cache-Control: no-cache
    Connection: Keep-Alive
    Content-Length: 14615
    ...


### 0x03 telnet 还可以连接 Redis

假设 redis 服务器跑在本地，监听 6379端口，用 `telnet 6379` 命令可以连接上。接下来就可以调用 redis 的命令。

调用"set hello world"，给 key 为 hello 设置值为 "world"，随后调用 get hello 获取值

![render1548074308853](https://user-gold-cdn.xitu.io/2019/5/29/16b04184ccd3d71d?imageslim)

Redis 客户端和 Redis 服务器使用 RESP 协议通信，RESP 是 REdis Serialization Protocol 的简称。在 RESP 中，通过检查服务器返回数据的第一个字节来确定这个回复是什么类型：

*   对于 Simple Strings 来说，第一个字节是 "+"
*   对于 Errors 来说，第一个字节是 "-"
*   对于 Integers 来说，第一个字节是 ":"
*   对于 Bulk Strings 来说，首字节是 "$"
*   对于 Arrays 来说，首字节是 "\*"

> RESP Simple Strings

Simple Strings 被用来传输非二进制安全的字符串，是按下面的方式进行编码: 一个加号，紧接着是不包含 CR 或者 LF 的字符串(不允许换行)，最后以CRLF("\\r\\n")结尾。

执行 "set hello world" 命令成功，服务器会响应一个 "OK"，这是 RESP 一种 Simple Strings 的场景，这种情况下，OK 被编码为五个字节：`+OK\r\n`

> RESP Bulk Strings

get 命令读取 hello 的值，redis 服务器返回 `$5\r\nworld\r\n`，这种类型属于是 Bulk Strings 被用来表示二进制安全的字符串。

Bulk Strings 的编码方式是下面这种方式：以 "$" 开头，后跟实际要发送的字节数，随后是 CRLF，然后是实际的字符串数据，最后以 CRLF 结束。

所以 "world" 这个 string 会被编码成这样：`$5\r\nworld\r\n`

命令二：netcat
----------

netcat 因为功能强大，被称为网络工具中的瑞士军刀，nc 是 netcat 的简称。这篇文章将介绍 nc 常用的几个场景。

### 0x01 用 nc 来当聊天服务器

实验步骤

1.  在服务器（10.211.55.5）命令行输入 `nc -l 9090`
    
    ![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcb1c2c573)
    
    这里的 `-l` 参数表示 nc 将监听某个端口，`l`的含义是 listen。后面紧跟的 9090 表示要监听的端口号为 9090。
2.  在另外客户端机器的终端中输入`nc 10.211.55.5 9090`
    
    ![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcb1d7091c)
    
    此时两台机器建立了一条 tcp 连接
3.  在客户端终端中输入 "Hello, this is a message from client"
    
    ![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcb2645f30)
    
    可以看到服务器终端显示出了客户端输入的消息
    
    ![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcb306b7c8)
    
4.  在服务器终端输入 "Hello, this is a message from server"
    
    ![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcb3f8e380)
    
    可以看到客户端终端显示了刚刚服务器端输入的消息
    
    ![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcb44ec564)
    

如果不想继续聊下去，在任意一端输入"Ctrl c"都会终止这个连接。

当然，真正在现实场景中用 nc 来聊天用的非常少。`nc -l`命令一个有价值的地方是可以快速的启动一个 tcp server 监听某个端口。

### 0x02 发送 http 请求

先访问一次 www.baidu.com 拿到百度服务器的 ip（183.232.231.172）

输入 "nc 183.232.231.172 80"，然后输入enter，

    nc 183.232.231.172 80
    <enter>
    <enter>


百度的服务器返回了一个 http 的报文 `HTTP/1.1 400 Bad Request`

![](https://user-gold-cdn.xitu.io/2019/3/7/169580bcea03aa7e)

来回忆一下 HTTP 请求报文的组成：

1.  起始行（start line）
2.  首部（header）
3.  可选的内容主体（body）

    nc 183.232.231.172 80
    GET / HTTP/1.1
    host: www.baidu.com
    <enter>
    <enter> 
    

![](https://user-gold-cdn.xitu.io/2019/3/7/169580be833e11cc)

除了狂按 enter，你也可以采用 unix 管道的方式，把 HTTP 请求报文传输过去

    echo -ne "GET / HTTP/1.1\r\nhost:www.baidu.com\r\n\r\n" | nc 183.232.231.172 80


echo 的 -n 参数很关键，echo 默认会在输出的最后增加一个换行，加上 -n 参数以后就不会在最后自动换行了。

执行上面的命令，可以看到也返回了百度的首页 html

### 0x03 查看远程端口是否打开

前面介绍过 telnet 命令也可以检查远程端口是否打开，既然 nc 被称为瑞士军刀，这个小功能不能说不行。

> nc -zv \[host or ip\] \[port\]

其中 -z 参数表示不发送任何数据包，tcp 三次握手完后自动退出进程。有了 -v 参数则会输出更多详细信息（verbose）。

![](https://user-gold-cdn.xitu.io/2019/3/7/169580bd23a6dcae)

### 0x04 访问 redis

nc 为 在没有 redis-cli 的情况下访问 redis 又新增了一种方法

    nc localhost 6379
    ping
    +PONG
    get hello
    $5
    world


同样可以把命令通过管道的方式传给 redis 服务器。

    echo ping  | nc localhost 6379
    +PONG


命令三：netstat
-----------

netstat 很强大的网络工具，可以用来显示套接字的状态。下面来介绍一下常用的命令选项

### 列出所有套接字

    netstat -a


`-a`命令可以输出所有的套接字，包括监听的和未监听的套接字。 示例输出：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b04184cf1da045)

### 只列出 TCP 套接字

    netstat -at


`-t` 选项可以只列出 TCP 的套接字，也可也用`--tcp`

示例输出

![](https://user-gold-cdn.xitu.io/2019/5/29/16b04184cf2ce909)

### 只列出 UDP 连接

    netstat -au


`-u` 选项用来指定显示 UDP 的连接，也可也用`--udp` 示例输出：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b04184d021165b)

### 只列出处于监听状态的连接

    netstat -l


`-l` 选项用来指定处于 LISTEN 状态的连接，也可以用`--listening` 示例输出：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b0418533dc022b)

与`-a`一样，可以组合`-t`来过滤处于 listen 状态的 TCP 连接

    netstat -lt


示例输出

![](https://user-gold-cdn.xitu.io/2019/5/29/16b0418538632d79)

### 禁用端口 和 IP 映射

    netstat -ltn


上面的例子中，常用端口都被映射为了名字，比如 22 端口输出显示为 ssh，8080 端口被映射为 webcache。大部分情况下，我们并不想 netstat 帮我们做这样的事情，可以加上`-n`禁用

![](https://user-gold-cdn.xitu.io/2019/5/29/16b04185f51f091f)

### 显示进程

    netstat -ltnp


使用 `-p`命令可以显示连接归属的进程信息，在查看端口被哪个进程占用时非常有用 示例输出如下：

![](https://user-gold-cdn.xitu.io/2019/5/29/16b04185b89d751d)

### 显示所有的网卡信息

    netstat -i


用 `-i` 命令可以列出网卡信息，比如 MTU 等

示例输出

![](https://user-gold-cdn.xitu.io/2019/5/29/16b041861f153178)

到此，netstat 基本命令选项都介绍完了，可以管道操作进行进一步的过滤。

### 显示 8080 端口所有处于 ESTABLISHED 状态的连接

    netstat -atnp | grep ":8080" | grep ESTABLISHED
    tcp        0      0 10.211.55.10:8080       10.211.55.5:45438       ESTABLISHED 24972/nc


### 统计处于各个状态的连接个数

    netstat -ant | awk '{print $6}' | sort | uniq -c | sort -n
          1 established)
          1 Foreign
          2 LISTEN
          3 TIME_WAIT
         30 ESTABLISHED


使用 awk 截取出状态行，然后用 sort、uniq 进行去重和计数即可

小结与思考题
------

这篇文章我们首先讲解了 telnet 的妙用，来回顾一下重点：第一， telnet 可以检查指定端口是否存在，用来判断指定的网络连接是否可达。第二 telnet 可以用来发送 HTTP 请求，HTTP 是基于 TCP 的应用层协议，可以认为 telnet 是 TCP 包的一个构造工具，只要构造出的包符合 HTTP 协议的格式，就可以得到正确的返回。第三，介绍了如何用 telnet 访问 redis 服务器，在没有安装 redis-cli 的情况下，也可以通过 telnet 的方式来快速进行访问，然后结合实际场景介绍了 Redis 的通信协议 RESP。

然后介绍了 nc 在诸多类似场景下的应用，最后介绍了 netstat 命令的的用法。

留一道作业题：

*   怎么样用 nc 发送 UDP 数据

欢迎你在留言区留言，和我一起讨论。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788849700878)