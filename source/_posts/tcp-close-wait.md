---
title: TCP连接状态异常记录
date: 2018-7-26
categories: TCP
tags:
  - TCP
  - Linux
abbrlink: 63769
---

## 问题描述
分布式事务Lottor在测试环境中运行一段时间之后，出现Lottor客户端连接不上Lottor Server的情况。经过排查，发现根源问题是Lottor客户端获取不到Lottor Server的集群信息。

Lottor Server启动了两个端口：9666为Tomcat容器的端口、9888为netty 服务器的端口。通过如下命令查看端口的状态：

```bash
netstat -apn
```

### netty服务的端口

```bash
netstat -apn|grep 9888
```
首先查看了netty服务器的端口，连接很正常，和我们上面描述的问题没什么联系。排除netty服务连接的因素。

### Tomcat容器的端口

其次查看Tomcat容器的端口的状态：

```bash
netstat -apn|grep 9666
```
发现有大量的FIN_WAIT2 和 CLOSE_WAIT状态的连接。
![image](http://image.blueskykong.com/netstate-lottor.jpg)

另外还可以通过如下的命令查看当前的连接数：

```bash
netstat -apn|grep 9666 | wc -l
```

更加细化可以发现FIN_WAIT2 和 CLOSE_WAIT状态的连接数量相当。

![](http://image.blueskykong.com/netstate-details.jpg)

## TCP四次挥手
通信的客户端和服务端通过三次握手建立TCP连接。当数据传输完毕后，双方都可释放连接。最开始的时候，客户端和服务器都是处于ESTABLISHED状态，然后客户端主动关闭，服务器被动关闭。

![image](http://image.blueskykong.com/tcp四次挥手.jpg)

连接的主动断开是可以发生在客户端，也同样可以发生在服务端。常用的三个状态是：ESTABLISHED 表示正在通信，TIME_WAIT 表示主动关闭，CLOSE_WAIT 表示被动关闭。
### ESTABLISHED
最开始的时候，客户端和服务器都是处于ESTABLISHED状态，然后客户端或服务端发起主动关闭，另一端则是被动关闭。

### FIN_WAIT1
当一方接受到来自应用断开连接的信号时候，就发送 FIN 数据报来进行主动断开，并且该连接进入 FIN_WAIT1 状态，连接处于半段开状态(可以接受、应答数据，当不能发送数据)，并将连接的控制权托管给 Kernel，程序就不再进行处理。一般情况下，连接处理 FIN_WAIT1 的状态只是持续很短的一段时间。

### CLOSE_WAIT
被动关闭的一端在收到FIN包文之后，其所处的状态为CLOSE_WAIT，表示被动关闭。
### FIN_WAIT2
当主动断开一端的 FIN 请求发送出去后，并且成功够接受到相应的 ACK 请求后，就进入了 FIN_WAIT2 状态。其实 FIN_WAIT1 和 FIN_WAIT2 状态都是在等待对方的 FIN 数据报。当 TCP 一直保持这个状态的时候，对方就有可能永远都不断开连接，导致该连接一直保持着。

### TIME_WAIT
从以上TCP连接关闭的状态转换图可以看出，主动关闭的一方在发送完对对方FIN报文的确认(ACK)报文后，会进入TIME_WAIT状态。TIME_WAIT状态也称为2MSL状态。MSL值得是数据包在网络中的最大生存时间。产生这种结果使得这个TCP连接在2MSL连接等待期间，定义这个连接的四元组（客户端IP地址和端口，服务端IP地址和端口号）不能被使用。

## 原因分析
在上一小节介绍了TCP四次挥手的相关状态之后，我们将会分析在什么情况下，连接处于CLOSE_WAIT状态呢？
在被动关闭连接情况下，已经接收到FIN，但是还没有发送自己的FIN的时刻，连接处于CLOSE_WAIT状态。
通常来讲，CLOSE_WAIT状态的持续时间应该很短，正如SYN_RCVD状态。但是在一些特殊情况下，就会出现连接长时间处于CLOSE_WAIT状态的情况。

出现大量close_wait的现象，主要原因是某种情况下对方关闭了socket链接，但是另一端由于正在读写，没有关闭连接。代码需要判断socket，一旦读到0，断开连接，read返回负，检查一下errno，如果不是AGAIN，就断开连接。

Linux分配给一个用户的文件句柄是有限的，而TIME_WAIT和CLOSE_WAIT两种状态如果一直被保持，那么意味着对应数目的通道就一直被占着，一旦达到句柄数上限，新的请求就无法被处理了，接着就是大量Too Many Open Files异常，导致tomcat崩溃。关于TIME_WAIT过多的解决方案参见[TIME_WAIT数量太多](https://blog.csdn.net/zf766045962/article/details/64905587?locationNum=15&fps=1)。

### 常见错误原因
从原理上来讲，由于Server的Socket在客户端已经关闭时而没有调用关闭，造成服务器端的连接处在“挂起”状态，而客户端则处在等待应答的状态上。此问题的典型特征是：一端处于FIN_WAIT2 ，而另一端处于CLOSE_WAIT。具体来说：

1.代码层面上未对连接进行关闭，比如关闭代码未写在 finally 块关闭，如果程序中发生异常就会跳过关闭代码，自然未发出指令关闭，连接一直由程序托管，内核也无权处理，自然不会发出 FIN 请求，导致连接一直在 CLOSE_WAIT 。

2.程序响应过慢，比如双方进行通讯，当客户端请求服务端迟迟得不到响应，就断开连接，重新发起请求，导致服务端一直忙于业务处理，没空去关闭连接。这种情况也会导致这个问题。

### Lottor中的问题
Lottor中，客户端定时刷新本地存储的Lottor Server的地址信息，具体来说是每60秒刷新一次，通过HttpClient请求获取结果。笔者根据网上查找的资料和如上的分析，首先尝试将周期性刷新关闭，观察Lottor Server的9666端口的连接数。之前线性增长的连接数停了下来，初步定位到问题的根源。

> 服务器A是一台爬虫服务器，它使用简单的HttpClient去请求资源服务器B上面的apache获取文件资源，正常情况下，如果请求成功，那么在抓取完 资源后，服务器A会主动发出关闭连接的请求，这个时候就是主动关闭连接，服务器A的连接状态我们可以看到是TIME_WAIT。如果一旦发生异常呢？假设 请求的资源服务器B上并不存在，那么这个时候就会由服务器B发出关闭连接的请求，服务器A就是被动的关闭了连接，如果服务器A被动关闭连接之后程序员忘了 让HttpClient释放连接，那就会造成CLOSE_WAIT的状态了。

笔者的场景如上面的情况一般，该问题的解决方法是我们在遇到异常的时候应该直接调用中止本次连接，以防CLOSE_WAIT状态的持续。案例可以参见[HttpClient连接池抛出大量ConnectionPoolTimeoutException: Timeout waiting for connection异常排查](https://blog.csdn.net/shootyou/article/details/6615051)。笔者后来的改进方法是重写了这部分的逻辑，由Spring Cloud FeignClient调用执行，避免之前的负载均衡查询等繁琐的操作。

#### 参考
1. [HttpClient连接池抛出大量ConnectionPoolTimeoutException: Timeout waiting for connection异常排查](https://blog.csdn.net/shootyou/article/details/6615051)
2. [网络连接无法释放—— CLOSE_WAIT](https://blog.csdn.net/ruixj/article/details/1871979)
3. [TCP三次握手四次挥手详解](https://www.cnblogs.com/zmlctt/p/3690998.html)

