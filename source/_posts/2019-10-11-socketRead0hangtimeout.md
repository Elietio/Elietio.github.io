---
title: socketRead0阻塞导致线程卡死问题分析排查
categories: Java
tags:
  - 排错
  - 异常
  - socketRead0
copyright: true
comment: true
abbrlink: 774f1877
date: 2019-10-11 20:26:54
description:
---

### 问题描述：Consumer进程无法处理消息

模拟环境中我们发现一台主备的Consumer进程无法处理消息，首先我们查看服务订阅情况，发现服务订阅正常，日志显示正常收到消息，但是Listener收到消息后业务代码未执行，导致触发消息超时反馈。
<!-- more -->
### 问题排查：

为什么业务代码不执行呢？查看业务代码日志，发现在一次处理中，执行了一次Rest请求后，剩余代码未执行，整个线程hold住了，是否由于该线程阻塞导致Listener收到消息后无法处理呢？梳理了一下代码逻辑，我们采用的是Guava EventBus的事件监听和发布订阅模式，业务方法使用@Subscribe订阅到消息后进行处理，而订阅者对象在处理事件时是使用了synchronized同步锁，所以是因为锁一直未释放导致其他消息无法订阅处理吗？为什么线程会卡住呢？

接下来我们使用JDK的jstack工具打印出服务的线程堆栈信息，确实发现有线程Blocked并且等待锁的情况。

```java
"IMT_328b84d5-3364-46cf-acfe-5e78de9f9cce_BL" prio=10 tid=0xada04400 nid=0x7816 waiting for monitor entry [0x9d1fe000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.google.common.eventbus.Subscriber$SynchronizedSubscriber.invokeSubscriberMethod(Subscriber.java:150)
	- waiting to lock <0xb73e5c58> (a com.google.common.eventbus.Subscriber$SynchronizedSubscriber)
	at com.google.common.eventbus.Subscriber$1.run(Subscriber.java:76)
    ......
```

`waiting to lock <0xb73e5c58> (a com.google.common.eventbus.Subscriber$SynchronizedSubscriber)` 
从这个信息看确实是Subscribe的同步锁，那么继续寻找当前占有锁的线程，发现如下

```java
"IMT_0945732d-7dc7-4d12-8e5f-c4f7e9f9f295_BL" prio=10 tid=0xa7d02c00 nid=0x68ce runnable [0x9e9fc000]
   java.lang.Thread.State: RUNNABLE
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.read(SocketInputStream.java:152)
	at java.net.SocketInputStream.read(SocketInputStream.java:122)
	at org.apache.http.impl.io.AbstractSessionInputBuffer.fillBuffer(AbstractSessionInputBuffer.java:158)
	at org.apache.http.impl.io.SocketInputBuffer.fillBuffer(SocketInputBuffer.java:82)
	at org.apache.http.impl.io.AbstractSessionInputBuffer.readLine(AbstractSessionInputBuffer.java:271)
	at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:140)
	at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:57)
	at org.apache.http.impl.io.AbstractMessageParser.parse(AbstractMessageParser.java:259)
	at org.apache.http.impl.AbstractHttpClientConnection.receiveResponseHeader(AbstractHttpClientConnection.java:281)
	at org.apache.http.impl.conn.DefaultClientConnection.receiveResponseHeader(DefaultClientConnection.java:259)
	at org.apache.http.impl.conn.ManagedClientConnectionImpl.receiveResponseHeader(ManagedClientConnectionImpl.java:209)
	at org.apache.http.protocol.HttpRequestExecutor.doReceiveResponse(HttpRequestExecutor.java:273)
	at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:125)
	at org.apache.http.impl.client.DefaultRequestDirector.tryExecute(DefaultRequestDirector.java:686)
	at org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:488)
	at org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java:884)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:82)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:107)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:55)
	at com.netflix.loadbalancer.PingUrl.isAlive(PingUrl.java:126)
	at com.netflix.loadbalancer.BaseLoadBalancer$SerialPingStrategy.pingServers(BaseLoadBalancer.java:902)
	at com.netflix.loadbalancer.BaseLoadBalancer$Pinger.runPinger(BaseLoadBalancer.java:672)
	at com.netflix.loadbalancer.BaseLoadBalancer.forceQuickPing(BaseLoadBalancer.java:814)
	at com.netflix.loadbalancer.DynamicServerListLoadBalancer.updateAllServerList(DynamicServerListLoadBalancer.java:268)
	at com.netflix.loadbalancer.DynamicServerListLoadBalancer.updateListOfServers(DynamicServerListLoadBalancer.java:250)
	at com.netflix.loadbalancer.DynamicServerListLoadBalancer.restOfInit(DynamicServerListLoadBalancer.java:144)
	at com.netflix.loadbalancer.DynamicServerListLoadBalancer.<init>(DynamicServerListLoadBalancer.java:95)
	at com.netflix.loadbalancer.ZoneAwareLoadBalancer.<init>(ZoneAwareLoadBalancer.java:82)
	at org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration.ribbonLoadBalancer(RibbonClientConfiguration.java:140)
	......
```

从以上信息可以看到，线程一直runnable状态，锁释放不了，代码最后执行到`java.net.SocketInputStream.socketRead0` 发生阻塞，继续向下看，由Apache Httpclient调用，再往后是Ribbon。而我们的服务确实是通过Ribbon管理目标服务IP发送Rest请求，由于本服务器网络特殊，并未注册到Eureka管理，而是直接把多活的目标服务列表写在配置中维护，加上之前日志的分析，线程确实是卡在Rest请求上，这就很奇怪了，Connection Timeout和Read Timeout我们都有设置，为何未生效？而且这个问题只在本环境发生，并且重启后依然稳定复现。首先对访问的目标服务器端口telnet，一切正常，而执行netstat查看服务器的网络连接，发现本机与一个目标服务端口一直存在一个ESTABLISHED的tcp连接，是否这个一直阻塞着？

查阅了一下socketRead0阻塞的问题，从网上资料看，这个问题确实有存在，并且从一篇博客[从锁死的RUNNABLE线程谈UNIX的I/O模型](https://suclogger.me/a-not-running-runnable/)中发现这样写道，JAVA IO库（java version “1.8.0_131”）的一个坑。

> 由于某些请求的TCP包传输过程中出现异常导致`poll`在没有真实可读数据情况下返回可读标识，使得阻塞的`recv`方法永远阻塞下去，从而使得当前线程一直处于`RUNNABLE`，当线程池的核心线程都被这种线程占据之后，就再也无法处理新提交的任务了。

现在[open jdk](https://bugs.openjdk.java.net/browse/JDK-8075484)已经修复了这一bug `SocketInputStream.socketRead0 can hang even with soTimeout set`

但是从这个来看，发生的概率应该很低的，而我们服务却是百分百发生，重启后依旧，所以继续从堆栈信息上分析。

从堆栈信息上分析可知Ribbon在更新服务列表updateAllServerList，然后执行了PingUrl的isAlive，调用了HttpClient发生了阻塞。默认情况下服务启动后Ribbon并不会直接加载服务列表，而是当第一次Rest请求调用时，Ribbon会去加载服务列表，并且执行设置的PingUrl方法判断服务节点是否存在，加载好服务列表之后根据设置的Loadbalance策略调用服务节点，该线程应该是hold在PingUrl这一步上。

PingUrl是服务中Ribbon默认设置的ribbonPing实现，用于检测服务IP列表。翻看PingUrl中isAlive方法的源码，大致如下：

```java
String urlStr   = "";
if (isSecure){
       urlStr = "https://";
}else{
       urlStr = "http://";
}
urlStr += server.getId();
urlStr += getPingAppendString();
boolean isAlive = false;
HttpClient httpClient = new DefaultHttpClient();
HttpUriRequest getRequest = new HttpGet(urlStr);
String content=null;
httpResponse response = httpClient.execute(getRequest);
```
可以看出实际上就是调用httpClinet发送了一次GET请求，请求的URL就是服务列表的每一个服务端口+指定的URL。既然如此，我们可以直接模拟同样的GET请求，执行curl命令

```shell
curl -X GET 'http://server:port'
```

果然，有一台目标服务器没有响应，一直卡住，这台服务器正是上面tcp一直处于ESTABLISHED状态的服务器。看来问题应该是由于这台服务器导致的，登陆这台服务器之后，df -h直接卡住不显示结果，怀疑用户目录下共享存储出问题了，cd到用户目录下，执行ls依然卡住未返回，找运维同学处理后，再次执行curl后正常，而服务还是阻塞状态，重启后ESTABLISHED状态tcp连接释放，服务恢复，正常处理消息。

### 问题还原：

由于一台服务器共享存储挂了，导致服务器上tomcat服务无法处理远程的请求，没有任何返回，连接一直处于ESTABLISHED状态未释放,虽然telnet服务端口是正常的，但是http请求会hold住无响应。而consumer服务使用Ribbon管理目标服务节点，当服务重启后，第一次执行Rest请求，Ribbon会去加载服务列表并进行ribbonPing检测所有的目标节点是否异常，默认是通过一个GET请求，因此当检测到故障的目标节点后，连接释放不了，线程一直hold，而Subscribe的同步锁也一直被线程占用无法释放，导致其他消息过来时无法处理从而超时，而重启后依然重现这一过程导致问题依旧。