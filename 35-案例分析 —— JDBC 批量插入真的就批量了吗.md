

这篇文章我们以 JDBC 批量插入的问题来看看网络分析在实际工作用的最简单的应用。

几年前遇到过一个问题，使用 jdbc 批量插入，插入的性能总是上不去，看代码又查不出什么结果。代码简化以后如下：

    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
    
        String url = "jdbc:mysql://localhost:3306/test?useSSL=false";
        Connection connection = DriverManager.getConnection(url, "root", "");
        PreparedStatement statement = connection.prepareStatement("insert into batch_insert_test(name)values(?)");
    
        for (int i = 0; i < 10; i++) {
            statement.setString(1, "name#" + System.currentTimeMillis() + "#" + i);
            statement.addBatch();
        }
        statement.executeBatch();
    }


通过 wireshark 抓包，结果如下

![](https://user-gold-cdn.xitu.io/2019/4/2/169dc5d37554c47f)

可以看到 jdbc 实际上是发送了 10 次 insert 请求，既不能降低网络通信的成本，也不能在服务器上批量执行。

单步调试，发现调用到了`executeBatchSerially`

![](https://user-gold-cdn.xitu.io/2019/4/2/169dc5d3748f9d16)


    protected long[] executeBatchSerially(int batchTimeout) throws SQLException


看源码发现跟`connection.getRewriteBatchedStatements()`有关，当等于 true 时，会进入批量插入的流程，等于 false 时，进入逐条插入的流程。

修改 sql 连接的参数，增加`rewriteBatchedStatements=true`

    // String url = "jdbc:mysql://localhost:3306/test?useSSL=false";
    String url = "jdbc:mysql://localhost:3306/test?useSSL=false&rewriteBatchedStatements=true";


单步调试，可以看到这下进入到批量插入的逻辑了。

![](https://user-gold-cdn.xitu.io/2019/4/2/169dc5d36f57e24d)

wireshark 抓包情况如下，可以确认批量插入生效了

![](https://user-gold-cdn.xitu.io/2019/4/2/169dc5d37d90af87)

rewriteBatchedStatements 参数将

    insert into batch_insert_test(name)values('name#1554175696958#0')
    insert into batch_insert_test(name)values('name#1554175696958#1')
    insert into batch_insert_test(name)values('name#1554175696958#2')
    insert into batch_insert_test(name)values('name#1554175696958#3')
    insert into batch_insert_test(name)values('name#1554175696958#4')
    insert into batch_insert_test(name)values('name#1554175696958#5')
    insert into batch_insert_test(name)values('name#1554175696958#6')
    insert into batch_insert_test(name)values('name#1554175696958#7')
    insert into batch_insert_test(name)values('name#1554175696958#8')
    insert into batch_insert_test(name)values('name#1554175696958#9')


改写为真正的批量插入

    insert into batch_insert_test(name)values
    ('name#1554175696958#0'),('name#1554175696958#1'),
    ('name#1554175696958#2'),('name#1554175696958#3'),
    ('name#1554175696958#4'),('name#1554175696958#5'),
    ('name#1554175696958#6'),('name#1554175696958#7'),
    ('name#1554175696958#8'),('name#1554175696958#9')


小结与思考
-----

这篇文章以一个非常简单的例子讲述了在用抓包工具来解决在 JDBC 上批量插入效率低下的问题。我们经常会用很多第三方的库，这些库我们一般没有精力把每行代码都读通读透，遇到问题时，抓一些包就可以很快确定问题的所在，这就是抓包网络分析的魅力所在。


[Source](https://juejin.im/book/6844733788681928712/section/6844733788853911560)