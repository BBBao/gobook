安装：
yum -y install httpd-tools

有关ab命令的使用，我们可以通过帮助命令进行查看。如下：
ab --help

#### 常用参数
下面我们对这些参数，进行相关说明。如下：
-n 在测试会话中所执行的请求个数。默认时，仅执行一个请求。
-c 一次产生的请求个数。默认是一次一个。
-t 测试所进行的最大秒数。其内部隐含值是-n 50000，它可以使对服务器的测试限制在一个固定的总时间以内。默认时，没有时间限制。
-k启用HTTP KeepAlive功能，即在一个HTTP会话中执行多个请求。默认时，不启用KeepAlive功能。

#### ab性能指标
在进行性能测试过程中有几个指标比较重要：
1、吞吐率（Requests per second）
服务器并发处理能力的量化描述，单位是reqs/s，指的是在某个并发用户数下单位时间内处理的请求数。某个并发用户数下单位时间内能处理的最大请求数，称之为最大吞吐率。
记住：吞吐率是基于并发用户数的。这句话代表了两个含义：
a、吞吐率和并发用户数相关
b、不同的并发用户数下，吞吐率一般是不同的
计算公式：总请求数/处理完成这些请求数所花费的时间，即
Request per second=Complete requests/Time taken for tests
必须要说明的是，这个数值表示当前机器的整体性能，值越大越好。
2、并发连接数（The number of concurrent connections）
并发连接数指的是某个时刻服务器所接受的请求数目，简单的讲，就是一个会话。
3、并发用户数（Concurrency Level）
要注意区分这个概念和并发连接数之间的区别，一个用户可能同时会产生多个会话，也即连接数。在HTTP/1.1下，IE7支持两个并发连接，IE8支持6个并发连接，FireFox3支持4个并发连接，所以相应的，我们的并发用户数就得除以这个基数。

4、用户平均请求等待时间（Time per request）
计算公式：处理完成所有请求数所花费的时间/（总请求数/并发用户数），即：
Time per request=Time taken for tests/（Complete requests/Concurrency Level）
5、服务器平均请求等待时间（Time per request:across all concurrent requests）
计算公式：处理完成所有请求数所花费的时间/总请求数，即：
Time taken for/testsComplete requests
可以看到，它是吞吐率的倒数。
同时，它也等于用户平均请求等待时间/并发用户数，即
Time per request/Concurrency Level
6、Transfer rate 表示这些请求在单位时间内从服务器获取的数据长度，
计算公式：Total trnasferred/ Time taken for tests，这个统计很好的说明服务器的处理能力达到极限时，其出口宽带的需求量。
7、Percentage of the requests served within a certain time (ms) 这部分数据用于描述每个请求处理时间的分布情况，所有请求的平均速度。
比如以上测试，这个处理时间是指前面的Time per request，即对于单个用户而言，平均每个请求的处理时间。

#### 实际应用
ab的命令参数比较多，我们经常使用的是-c和-n参数。
