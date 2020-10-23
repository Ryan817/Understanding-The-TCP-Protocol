

在开发过程中，你一定遇到过这个异常：`java.net.SocketException: Connection reset`，在这个异常的产生的原因就是因为 RST 包，这篇文章会解释 RST 包产生的原因和几个典型的出现场景。

> RST（Reset）表示复位，用来强制关闭连接

场景一：对端主机端口不存在
-------------

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d36d895c715)

服务器 10.211.55.5 上执行 netstat 命令可以查看当前机器监听的端口信息，`-l`表示只列出 listen 状态的 socket。

    sudo netstat -lnp  | grep tcp
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1365/sshd             


可以看到目前服务器上只监听了 22 端口

这个时候客户端想连接服务端的 80 端口会发生什么呢？在客户端（10.211.55.10）开启 tcpdump 抓包，然后尝试连接服务器的 80 端口（nc 10.211.55.5 80）。

可以看到客户端发了一个 SYN 包到服务器，服务器马上回了一个 RST 包，表示拒绝

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d36d90e3b8c)

场景二：Nginx 502（Bad Gateway）
--------------------------

Nginx 的 upstream server 没有启动或者进程挂掉是绝大多数 502 状态码的根源，先来复现一下

*   准备两台虚拟机 A（10.211.55.5） 和 B（10.211.55.10），A 装好 Nginx，B 启动一个 web 服务器监听 8080 端口（Java、Node.js、Go 什么都可以） A 机器 Nginx 配置文件如下

    upstream web_server {
            server 10.211.55.10:8080;
            keepalive 16;
    }
    server {
            listen 80;
            server_name test.foo.com;
            location /test {
                    proxy_http_version 1.1;
                    proxy_pass http://web_server/;
            }
    }
    

此时请求 [test.foo.com/test](http://test.foo.com/test) 就返回正确的 Node.js 页面

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d36d92f6658)

下一步，kill 掉 B 机器上的 Node 进程，这时客户端请求返回了 502

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d36d91aba26)

整个过程如下：

1.  客户端发起一个 http 请求到 nginx
2.  Nginx 收到请求，根据配置文件的信息将请求转发到对应的下游 server 的 8080 端口处理，如果还没有建立连接，会发送 SYN 包准备三次握手建连，如果已经建立了连接，会发送数据包。
3.  下游服务器发现并没有进程监听 8080 端口，于是返回 RST 包 Nginx
4.  Nginx 拿到 RST 包以后，认为后端已经挂掉，于是返回 502 状态码给客户端

简略图如下：

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d36d94f8794)

场景三：从一次 OKHttp 请求失败惨案看 RST
--------------------------

这个场景是使用 okhttp 发送 http 请求，发现偶发性出现请求失败的情况

    Exception in thread "main" java.io.IOException: unexpected end of stream on Connection{test.foo.com:80, proxy=DIRECT hostAddress=test.foo.com/10.211.55.5:80 cipherSuite=none protocol=http/1.1}
    	at okhttp3.internal.http1.Http1Codec.readResponseHeaders(Http1Codec.java:208)
    	at okhttp3.internal.http.CallServerInterceptor.intercept(CallServerInterceptor.java:88)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:147)
    	at okhttp3.internal.connection.ConnectInterceptor.intercept(ConnectInterceptor.java:45)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:147)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:121)
    	at okhttp3.internal.cache.CacheInterceptor.intercept(CacheInterceptor.java:93)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:147)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:121)
    	at okhttp3.internal.http.BridgeInterceptor.intercept(BridgeInterceptor.java:93)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:147)
    	at okhttp3.internal.http.RetryAndFollowUpInterceptor.intercept(RetryAndFollowUpInterceptor.java:126)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:147)
    	at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:121)
    	at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:254)
    	at okhttp3.RealCall.execute(RealCall.java:92)
    	at MyOkHttpKeepAliveKt.sendHttpRequest(MyOkHttpKeepAlive.kt:36)
    	at MyOkHttpKeepAliveKt.main(MyOkHttpKeepAlive.kt:25)
    Caused by: java.io.EOFException: \n not found: limit=0 content=…
    	at okio.RealBufferedSource.readUtf8LineStrict(RealBufferedSource.java:236)


因为 okhttp 开启了连接池，默认启用了 HTTP/1.1 keepalive，如果拿到一个过期的连接去发起 http 请求，就一定会出现请求失败的情况。Nginx 默认的 keepalive 超时时间是 65s，为了能更快的复现，我把 Nginx 的超时时间调整为了 5s

    http {
        ...
        keepalive_timeout  5s;
        ...
    }


客户端请求代码简化如下

    private val okHttpClient = OkHttpClient.Builder()
            .retryOnConnectionFailure(false)
            .connectTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(10, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    
    fun main(args: Array<String>) {
        // 发起第一次 http 请求
        sendHttpRequest()
        TimeUnit.SECONDS.sleep(6)
        // 发起第二次 http 请求，因为第一个连接已经释放，第二次会拿到同一条连接
        sendHttpRequest()
        System.`in`.read()
    }
    
    private fun sendHttpRequest() {
        val request = Request.Builder().url("http://test.foo.com/test").get().build()
        val response = okHttpClient.newCall(request).execute()
        println("http status: " + response.code())
        response.close()
    }


运行以后，马上出现了上面请求失败的现象，出现的原因是什么呢？

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d36d967081a)

Nginx的 keepalive 时间是 65s，客户端请求了第一次以后，开始闲下来，65s 倒计时到了以后 Nginx 主动发起连接要求正常分手断掉连接，客户端操作系统马上回了一个，好的，我收到了你的消息。但是连接池并不知道这个情况，没有关闭这个 socket，而是继续用这个断掉的连接发起 http 请求。就出现问题了。

tcpdump 抓包结果如下

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d3800e2cf4e)

记客户端 10.211.55.10 为 A，服务器 10.211.55.5 为 B，逐行分析结果如下：

*   1 ~ 3：A 与 B 三次握手过程，SYN -> SYN+ACK -> ACK
*   4 ~ 5：A 向 B 发起 HTTP 请求报文，服务器 B 回了 ACK
*   6 ~ 7：B 向 A 发送 HTTP 响应报文，客户端 A 收到报文以后回了 ACK
*   8 ~ 9：经过漫长的65s，客户端 A 没有任何后续请求，Nginx 决定断掉这个连接，于是发送了一个 FIN 给客户端 A，然后进入 FIN\_WAIT2 状态，A 收到 FIN 以后进入 CLOSE\_WAIT 状态
*   10：客户端 A 继续发送 HTTP 请求报文到 B
*   11：因为此时 B 已经不能发送任何报文到 A，于是发送了一个 RST 包给 A，让它可以尽早断开这条连接。

这个有两个解决的方案：

第一，把 okhttp 连接池的 keepAlive 超时时间设置短于 Nginx 的超时时间 65s，比如设置成 30s `builder.connectionPool(ConnectionPool(5, 30, TimeUnit.SECONDS))` 在这种情况下，okhttp 会在连接空闲 30s 以后主动要求断掉连接，这是一种主动出击的解决方案

这种情况抓包结果如下

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d38011fb40f)

*   1 ~ 7：完成第一次 HTTP 请求
*   8：过了 30s，客户端 A 发送 FIN 给服务器 B，要求断开连接
*   9：服务器 B，收到以后也回了 FIN + ACK
*   10：客户端 A 对服务器 B 发过来的 FIN 做确认，回复 ACK，至此四次挥手结束
*   11 ~ 13：客户端 A 使用新的端口 58604 与服务器 B 进行三次握手建连
*   13 ~ 20：剩余的过程与第一次请求相同

第二，把 `retryOnConnectionFailure` 属性设置为 true。这种做法的原理是等对方 RST 掉以后重新发起请求，这是一种被动的处理方案

![](https://user-gold-cdn.xitu.io/2019/4/9/16a02d38011422ba)

retryOnConnectionFailure 这个属性会在请求被远端 connection reset 掉以后进行重试。可以看到 10 ~ 11 行，拿一个过期的连接发起请求，服务器 B 返回了 RST，紧接着客户端就进行了重试，完成了剩下的请求，对上层调用完全无感。

小结
--

这篇文章用三个简单例子讲解了 RST 包在真实场景中的案例。

*   第 1 个例子：对端主机端口不存在或者进程崩溃的时候建连或者发请求会收到 RST 包
*   第 2 个例子：后端 upstream 挂掉的时候，Nginx 返回 502，这个例子不过是前面第 1 个例子在另一个场景的应用
*   第 3 个例子：okhttp 参数设置不合理导致的 Connection Reset，主要原因是因为对端已经关掉连接，用一条过期的连接发送数据对端会返回 RST 包

平时工作中你有遇到到 RST 导致的连接问题吗？


[Source](https://juejin.im/book/6844733788681928712/section/6844733788853895181)