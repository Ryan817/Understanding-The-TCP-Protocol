

![](https://user-gold-cdn.xitu.io/2019/6/24/16b86ce0f3dfb74b)

之前有一个组员碰到了一个代码死活连不上 Zookeeper 的问题，我帮忙分析了一下，过程记录了在下面。

他那边包的错误堆栈是这样的：

    java.io.IOException: Connection reset by peer
            at sun.nio.ch.FileDispatcher.read0(Native Method)
            at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:21)
            at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:233)
            at sun.nio.ch.IOUtil.read(IOUtil.java:200)
            at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:236)
            at org.apache.zookeeper.ClientCnxnSocketNIO.doIO(ClientCnxnSocketNIO.java:68)
            at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:355)
            at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1068)


其它组员没有遇到这个问题，他换成无线网络也可以恢复正常，从抓包文件也看到服务端发送了 RST 包给他这台机器，这就比较有意思了。

基于上面的现象，首先排除了 Zookeeper 本身服务的问题，一定是跟客户端的某些特征有关。

当时没有登录部署 ZooKeeper 机器的权限，没有去看 ZooKeeper 的日志，先从客户端这边来排查。

首先用 netstat 查看 ZooKeeper 2181 端口的连接状态，发现密密麻麻，一屏还显示不下，使用 wc -l 统计了一下，发现有 60 个，当时对 ZooKeeper 的原理并不是很了解，看到这个数字没有觉得有什么特别。

但是经过一些实验，发现小于 60 个连接的时候，客户端使用一切正常，达到 60 个的时候，就会出现 Connection Reset 异常。

直觉告诉我，可能是 ZooKeeper 对客户端连接有限制，于是去翻了一下文档，真有一个配置项`maxClientCnxns`是与客户端连接个数有关的。

> maxClientCnxns: Limits the number of concurrent connections (at the socket level) that a single client, identified by IP address, may make to a single member of the ZooKeeper ensemble. This is used to prevent certain classes of DoS attacks, including file descriptor exhaustion. Setting this to 0 or omitting it entirely removes the limit on concurrent connections.

这个参数的含义是，限制客户端与 ZooKeeper 的连接个数，通过 IP 地址来区分是不是一个客户端。如果设置为 0 表示不限制连接个数。

这个值可以通过 ZooKeeper 的配置文件`zoo.cfg` 进行修改，这个值默认是 60。

知道这一点以后重新做一下实验，将远程虚拟机中 ZooKeeper 的配置 `maxClientCnxns`改为 1

    zoo.cfg


​    
​    
    maxClientCnxns=1


在本地`zkCli.sh`连接 ZooKeeper

    zkCli.sh -server c2:2181


发现一切正常成功

![](https://user-gold-cdn.xitu.io/2019/6/24/16b87cb3f6189b51)

在本地再次用`zkCli.sh`连接 ZooKeeper，发现连接成功，随后出现 `Connection Reset` 错误

![](https://user-gold-cdn.xitu.io/2019/6/24/16b87cb3f630a2ea)

通过抓包文件也可以看到，ZooKeeper 发出了 RST 包

![](https://user-gold-cdn.xitu.io/2019/6/24/16b87cb3f6260d68)

完整的包见：[zk\_rst.pcapng](https://github.com/arthur-zhang/tcp_ebook/blob/master/tcp_case/zk_rst.pcapng)

同时在 ZooKeeper 那一端也出现了异常提示

    2019-06-23 05:22:25,892 [myid:] - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@188] - Too many connections from /10.211.55.2 - max is 1


问题基本上就定位和复现成功了，我们来看一下 ZooKeeper 的源码，看下这部分是如何处理的，这部分逻辑在`NIOServerCnxnFactory.java`的 run 方法。

![](https://user-gold-cdn.xitu.io/2019/6/24/16b87cb3f7d7ce8b)

这部分逻辑是如果 maxClientCnxns 大于 0，且当前 IP 的连接数大于 maxClientCnxns 的话，就会主动关闭 socket，同时打印日志。

后面发现是因为同事有一个操作 ZooKeeper 的代码有 bug，导致建连非常多，后面解决以后问题就再也没有出现了。

这个案例比较简单，给我们的启示是对于黑盒的应用，通过抓包等方式可以定位出大概的方向，然后进行分析，最终找到问题的根因。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788858089486)