---
title: Jedis Unexpected end of stream 异常
categories: Java
tags:
  - Redis
copyright: true
comment: true
abbrlink: ab6869b3
date: 2019-05-12 19:37:06
description:
---

这周末版本升级遇到一个Jedis异常，其中一步是从Mysql中的临时表查找数据然后拼装key从Redis中查找对应的缓存数据并修改。然而升级过程中修数程序却抛出一个异常`Unexpected end of stream`意外停止。
<!-- more -->

```java
redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.
    at redis.clients.util.RedisInputStream.ensureFill(RedisInputStream.java:199)
    at redis.clients.util.RedisInputStream.readByte(RedisInputStream.java:40)
    at redis.clients.jedis.Protocol.process(Protocol.java:151)
......
```
Redis采用的是codis搭建的集群，我们立即Telnet访问的端口，并在服务器使用Redis-cli直接连接，结果都未发现异常。于是立即去网上查找相关资料，网上都这样描述此异常。

客户端缓冲区异常
这个异常是客户端缓冲区异常，产生这个问题可能有三个原因：

1. 多个线程使用一个Jedis连接。

2. 客户端缓冲区满了,Redis有三种客户端缓冲区：
	普通客户端缓冲区(normal)：用于接受普通的命令，例如get、set、mset、hgetall、zrange等。
	slave客户端缓冲区(slave)：用于同步master节点的写命令，完成复制。
	发布订阅缓冲区(pubsub)：pubsub不是普通的命令，因此有单独的缓冲区。
	Redis客户端缓冲区配置的格式是：

	```yaml
	client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
	```
     class: 客户端类型：可选值为normal、slave和pubsub。
     hard limit: 如果客户端使用的输出缓冲区大于hard limit，客户端会被立即关闭，单位为秒。
     soft limit和soft seconds: 如果客户端使用的输出缓冲区超过了soft limit并且持续了soft limit秒，客户端会被立即关闭，单位为秒。
   
3. 长时间闲置连接会被服务端主动断开，可以查询timeout配置的设置以及自身连接池配置确定是否需要做空闲检测。

于是我们立即对配置进行了检查，并未发现有相关问题，timeout默认设置为0，客户端缓冲区临时修改为不限制也未见生效。

这下犯愁了，由于这个程序几乎没有日志，代码也不是我们编写，而是一位外地的同事提供的。只能一波人紧急分析代码，一波人继续查询错误日志，从异常的堆栈中我们发现异常是从执行redis的一个get方法抛出的，难道是get某个key的时候出现了异常？由于日志中没有打印具体的key信息，所以也不清楚具体情况，难道是某个key的体积过大，导致查询的时候超过了限制？于是立即使用bigkeys查询了一下Redis的大体积key，结果最大的也只有十几kb，很显然这也不是原因。大家正在一筹莫展，准备把程序加上详细的日志再具体分析，但是生产环境做紧急变更交付物是很困难的，之前我们已经在模拟环境同步生产数据测了好几轮，都未发现问题，为何生产就出现问题了呢？

继续看codis porxy的日志，发现了一个特殊的地方，客户端建立的连接每次都是经过60s后被断开，显示EOF错误，代表客户端客户端主动断开，于是我们立即查找相关配置，是否存在60s的配置，这时候运维同学提到了一件事，codis-porxy使用了nginx做负载均衡代理，nginx应该做了超时配置，于是我们翻看了nginx配置，果然存在一个60s的超时配置，而这时候我们再去翻看代码逻辑，发现了一个问题，首先程序使用jedis建立一个redis连接，然后从MySQL从查找临时表所有数据，然后修改redis缓存。而我们的临时表数据过于庞大，而且缺乏索引，所以查询这一步花费了很长时间，已经超过60s，等数据查询完毕，再执行redis操作，此时一直空闲的连接已经被nginx当作超时给断开了。为何之前模拟环境并未出现这个问题？应该是最近待修的数据又增加了很多，刚好超过60s，导致这个问题并未在模拟阶段发现。

于是我们在模拟环境复现该问题，并临时把nginx配置修改为600s，于是修数程序正常执行，没有异常，问题到此解决，于是生产同步操作，最终完成了升级。