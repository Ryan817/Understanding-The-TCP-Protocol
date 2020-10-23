

TCP 的自连接是一个比较有意思的现象，甚至很多人认为是 Linux 内核的 bug。我们先来看看 TCP 的自连接是什么。

TCP 自连接是什么
----------

新建一个脚本 self\_connect.sh，内容如下：

    while true
    do
    	nc 127.0.0.1 50000
    done


执行这段脚本之前先用 netstat 等命令确认 50000 没有进程监听。然后执行脚本，经过一段时间，telnet 居然成功了。

    Trying 127.0.0.1...
    telnet: connect to address 127.0.0.1: Connection refused
    Trying 127.0.0.1...
    telnet: connect to address 127.0.0.1: Connection refused
    Trying 127.0.0.1...
    Connected to 127.0.0.1.
    Escape character is '^]'.


使用 netstat 查看当前的 50000 端口的连接状况，如下所示。

    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 127.0.0.1:50000         127.0.0.1:50000         ESTABLISHED 24786/telnet


可以看到源 IP、源端口是 `127.0.0.1:50000`，目标 ip、目标端口也是 `127.0.0.1:50000`，通过上面的脚本，我们连上了本来没有监听的端口号。

自连接原因分析
-------

自连接成功的抓包结果如下图所示。

![](https://user-gold-cdn.xitu.io/2020/1/31/16ffa47400ab009c)

对于自连接而言，上图中 wireshark 中的每个包的发送接收双方都是自己，所以可以理解为总共是六个包，包的交互过程如下图所示。

![自连接包交互过程](https://user-gold-cdn.xitu.io/2020/1/31/16ffa47403b85491)

这个图是不是似曾相识？前四个包的交互过程就是 TCP 同时打开的过程。

当一方主动发起连接时，操作系统会自动分配一个临时端口号给连接主动发起方。如果刚好分配的临时端口是 50000 端口，过程如下。

*   第一个包是发送 SYN 包给 50000 端口
*   对于发送方而已，它收到了这个 SYN 包，以为对方是想同时打开，会回复 SYN+ACK
*   回复 SYN+ACK 以后，它自己就会收到这个 SYN+ACK，以为是对方回的，对它而已握手成功，进入 ESTABLISHED 状态

自连接的危害
------

设想一个如下的场景：

*   你写的业务系统 B 会访问本机服务 A，服务 A 监听了 50000 端口
*   业务系统 B 的代码写的稍微比较健壮，增加了对服务 A 断开重连的逻辑
*   如果有一天服务 A 挂掉比较长时间没有启动，业务系统 B 开始不断 connect 重连
*   系统 B 经过一段时间的重试就会出现自连接的情况
*   这时服务 A 想启动监听 50000 端口就会出现地址被占用的异常，无法正常启动

如果出现了自连接，至少有两个显而易见的问题：

*   自连接的进程占用了端口，导致真正需要监听端口的服务进程无法监听成功
*   自连接的进程看起来 connect 成功，实际上服务是不正常的，无法正常进行数据通信

如何解决自连接问题
---------

自连接比较罕见，但一旦出现逻辑上就有问题了，因此要尽量避免。解决自连接有两个常见的办法。

*   让服务监听的端口与客户端随机分配的端口不可能相同即可
*   出现自连接的时候，主动关掉连接

对于第一种方法，客户端随机分配的范围由 `/proc/sys/net/ipv4/ip_local_port_range` 文件决定，在我的 Centos 8 上，这个值的范围是 32768~60999，只要服务监听的端口小于 32768 就不会出现客户端与服务端口相同的情况。这种方式比较推荐。

对于第二种方法，我第一次见是在 Golang 的 TCP connect 的代码，代码如下所示。


    func (sd *sysDialer) doDialTCP(ctx context.Context, laddr, raddr *TCPAddr) (*TCPConn, error) {
    	fd, err := internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_STREAM, 0, "dial", sd.Dialer.Control)
    
    	// TCP has a rarely used mechanism called a 'simultaneous connection' in
    	// which Dial("tcp", addr1, addr2) run on the machine at addr1 can
    	// connect to a simultaneous Dial("tcp", addr2, addr1) run on the machine
    	// at addr2, without either machine executing Listen. If laddr == nil,
    	// it means we want the kernel to pick an appropriate originating local
    	// address. Some Linux kernels cycle blindly through a fixed range of
    	// local ports, regardless of destination port. If a kernel happens to
    	// pick local port 50001 as the source for a Dial("tcp", "", "localhost:50001"),
    	// then the Dial will succeed, having simultaneously connected to itself.
    	// This can only happen when we are letting the kernel pick a port (laddr == nil)
    	// and when there is no listener for the destination address.
    	// It's hard to argue this is anything other than a kernel bug. If we
    	// see this happen, rather than expose the buggy effect to users, we
    	// close the fd and try again. If it happens twice more, we relent and
    	// use the result. See also:
    	//	https://golang.org/issue/2690
    	//	https://stackoverflow.com/questions/4949858/
    	//
    	// The opposite can also happen: if we ask the kernel to pick an appropriate
    	// originating local address, sometimes it picks one that is already in use.
    	// So if the error is EADDRNOTAVAIL, we have to try again too, just for
    	// a different reason.
    	//
    	// The kernel socket code is no doubt enjoying watching us squirm.
    	for i := 0; i < 2 && (laddr == nil || laddr.Port == 0) && (selfConnect(fd, err) || spuriousENOTAVAIL(err)); i++ {
    		if err == nil {
    			fd.Close()
    		}
    		fd, err = internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_STREAM, 0, "dial", sd.Dialer.Control)
    	}
    
    	if err != nil {
    		return nil, err
    	}
    	return newTCPConn(fd), nil
    }
    
    func selfConnect(fd *netFD, err error) bool {
    	// If the connect failed, we clearly didn't connect to ourselves.
    	if err != nil {
    		return false
    	}
    
    	// The socket constructor can return an fd with raddr nil under certain
    	// unknown conditions. The errors in the calls there to Getpeername
    	// are discarded, but we can't catch the problem there because those
    	// calls are sometimes legally erroneous with a "socket not connected".
    	// Since this code (selfConnect) is already trying to work around
    	// a problem, we make sure if this happens we recognize trouble and
    	// ask the DialTCP routine to try again.
    	// TODO: try to understand what's really going on.
    	if fd.laddr == nil || fd.raddr == nil {
    		return true
    	}
    	l := fd.laddr.(*TCPAddr)
    	r := fd.raddr.(*TCPAddr)
    	return l.Port == r.Port && l.IP.Equal(r.IP)
    }


这里详细解释了为什么有 selfConnect 方法的判断，判断是否是自连接的逻辑是判断源 IP 和目标 IP 是否相等，源端口号和目标端口号是否相等。

小结
--

到这里，TCP 自连接的知识就介绍完了，在以后写 web 服务监听端口时，记得看下机器上的端口范围，不要胡来。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788824535048)