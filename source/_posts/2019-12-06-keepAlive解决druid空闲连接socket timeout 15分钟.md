---
title: keepAlive解决druid空闲连接socket timeout 15分钟
categories: Java
tags:
  - keepAlive
  - druid
  - socketRead0
  - socket timeout
copyright: true
comment: true
abbrlink: fd295ff1
date: 2019-12-06 20:26:54
description:
---

### 问题描述：

测试环境发现了一个很奇怪的现象，一台服务器出现了请求卡顿15分钟然后才执行SQL返回结果的现象
<!-- more -->

### 问题排查：

最开始我们认为是网络问题，因为这台服务器网络环境比较特殊，正常我们环境的应用服务器和数据库服务器是同一网段的，而这台服务器和数据库服务器并不在同一网段，存在防火墙策略，但是后来发现能够稳定复现，并且每次都是发生在服务空闲一段时间后第一次请求，由于我们使用的是druid，于是查询了相关问题，果然网络上已经有了很多相关的描述

[druid下莫名其妙hold15分钟+。疑是socket timeout超时15分钟后，重建了新连接导致](https://my.oschina.net/u/3434392/blog/3017866)

[从连接池中获取到失效连接，在检验连接有效性时出现长时间等待，大概15分钟 #2905](https://github.com/alibaba/druid/issues/2905)

大致原因是这样，当应用服务器和数据库服务器直接存在防火墙策略时，如果服务空闲时间过长，会被防火墙主动断开数据库连接，但是此时druid并未感知，此时当有请求过来时，如果druid配置了`testWhileIdle`（申请连接的时候检测，如果空闲时间大于`timeBetweenEvictionRunsMillis`，执行`validationQuery`检测连接是否有效。），那么将进行一次检测，检测的方式也是根据配置的策略，一般是`select 1 from dual` ，由于这个连接已经被防火墙断掉，根本到达不了数据库，而druid这边则在一直等待，而这个等待的超时受到`socket timeout`限制，而我们服务器本身的系统`socket timeout`设置就是15分钟，所以druid会一直hold15分钟，直到触发超时重新建立连接。所以问题就出现了druid无法主动恢复防火墙主动断开的连接，只有当触发超时才能进行重建连接。

### 问题解决：

问题解决的思路有两种，第一钟思路，更改jdbc的`socke timeout`，但是不建议，因为如果要能够快速重连，那么这个`socket timeout`就需要配置很短，但是这个时间很短会导致执行过长的SQL无法返回结果，`socket timeout`必须大于`statement timeout`，否则`socket timeout`先生效则`statement timeout`毫无意义，所以即使配置几分钟还是会出现请求hold的现象，因此不合适。

第二种思路，既然长时间空闲后连接会被防火墙断开，那么维持一个心跳，不让连接被防火墙断开即可，因此，需要引入druid的`keepAlive`

引入[druid GitHub](https://github.com/alibaba/druid/wiki/KeepAlive_cn)上对此配置的解释

> 在Druid-1.0.27之前的版本，DruidDataSource建议使用TestWhileIdle来保证连接的有效性，但仍有很多场景需要对连接进行保活处理。在1.0.28版本之后，新加入keepAlive配置，缺省关闭。使用keepAlive功能，建议使用1.1.16或者更高版本
>
> # 打开KeepAlive之后的效果
>
> 1. 初始化连接池时会填充到minIdle数量。
> 2. 连接池中的minIdle数量以内的连接，空闲时间超过minEvictableIdleTimeMillis，则会执行keepAlive操作。
> 3. 当网络断开等原因产生的由ExceptionSorter检测出来的死连接被清除后，自动补充连接到minIdle数量。

开启连接保活配置`keepAlive`,对于`minIdle`以内的连接，按照`timeBetweenEvictionRunsMillis`间隔进行保活检测，当空闲时间大于`minEvictableIdleTimeMillis`，发送心跳保持连接活跃，只要发送保活心跳的时间间隔小于防火墙断开空闲连接的时间即可。对于下面配置，只有1个空闲连接在空闲状态下会在5分钟左右进行心跳，保持长时间存活。

```yaml
       minIdle: 1
       timeBetweenEvictionRunsMillis: 60000
       minEvictableIdleTimeMillis: 300000
       validationQuery: SELECT 1 FROM DUAL
       validationQueryTimeout: 10
       keepAlive: true
       testWhileIdle: true
       ....
```

