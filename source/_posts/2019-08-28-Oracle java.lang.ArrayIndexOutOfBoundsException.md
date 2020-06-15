---
title: Oracle批量插入数据异常 java.lang.ArrayIndexOutOfBoundsException
tags:
  - Java
  - Oracle
  - 异常
categories:
  - Java
mathjax: true
copyright: true
comment: true
abbrlink: e6c1ebcb
date: 2019-08-28 20:14:49
photo:
---
今天测试过程中一个Oracle mybatis批量插入数据的代码报出了一个异常
 ` Caused by: java.lang.ArrayIndexOutOfBoundsException: -32768`
<!-- more -->
具体异常堆栈信息如下：
```java
Caused by: java.lang.ArrayIndexOutOfBoundsException: -32768
        at oracle.jdbc.driver.OraclePreparedStatement.setupBindBuffers(OraclePreparedStatement.java:2673)
        at oracle.jdbc.driver.OraclePreparedStatement.processCompletedBindRow(OraclePreparedStatement.java:2206)
        at oracle.jdbc.driver.OraclePreparedStatement.executeInternal(OraclePreparedStatement.java:3365)
        at oracle.jdbc.driver.OraclePreparedStatement.execute(OraclePreparedStatement.java:3476)
        at com.alibaba.druid.filter.FilterChainImpl.preparedStatement_execute(FilterChainImpl.java:3409)
        at com.alibaba.druid.wall.WallFilter.preparedStatement_execute(WallFilter.java:619)
        at com.alibaba.druid.filter.FilterChainImpl.preparedStatement_execute(FilterChainImpl.java:3407)
        at com.alibaba.druid.filter.FilterAdapter.preparedStatement_execute(FilterAdapter.java:1080)
        at com.alibaba.druid.filter.FilterChainImpl.preparedStatement_execute(FilterChainImpl.java:3407)
        at com.alibaba.druid.filter.FilterEventAdapter.preparedStatement_execute(FilterEventAdapter.java:440)
        at com.alibaba.druid.filter.FilterChainImpl.preparedStatement_execute(FilterChainImpl.java:3407)
        at com.alibaba.druid.proxy.jdbc.PreparedStatementProxyImpl.execute(PreparedStatementProxyImpl.java:167)
        at com.alibaba.druid.pool.DruidPooledPreparedStatement.execute(DruidPooledPreparedStatement.java:498)
```
感觉很奇怪，查看日志SQL打印，这个方也就拼接了400条SQL，参数也不是很多，于是从日志里面copy了一下接口入参在本地用Postman debug了一下，结果第一次居然入库成功，再执行一次，出现了同样的错误，再执行，又成功......反复如此，顿时懵逼。看异常信息，执行ojdbc包内的`oracle.jdbc.driver.OraclePreparedStatement.setupBindBuffers`方法数组下标越界，好吧，放Google搜一下，发现一段这样描述
> The 10g driver apparently keeps a global serialnumber for all parameters in the entire batch, with a "short"variable. So you can have at most 32768 parameters in the batch. I was havingthe same exception because I have a INSERT statement with 42 parameters and mybatches can be as big as 1000 records, so 42000 > 32768 and this overflowsto a negative index. I reduced the batch factor to 100 to be safe, and all iswell. I guess your update DML should have a larger number of parameters perrecord, right? (My diagnostic of the bug is just deduction from the symptoms)
>https://community.oracle.com/thread/599441?start=15&tstart=0>

说是10g driver statement最大允许参数个数为32768，超过会报错。似乎有点类似，但是我只插入了400条啊，而且每个SQL参数只有9个，也就是3600个参数，远小于32768。
还有另外一个说法
> In Oracle Metalink (Oracle's support site - Note ID 736273.1) I found that this is a bug in JDBC adapter (version 10.2.0.0.0 to 11.1.0.7.0) that when you call preparedStatement with more than 7 positional parameters then JDBC will throw this error.
> <https://stackoverflow.com/questions/277744/jdbc-oracle-arrayindexoutofboundsexception>

感觉也不符合，但是从搜索的结果看，10g 的 ojdbc似乎确实有些问题，于是看了下pom，乖乖
```xml
<dependency>
    <groupId>ojdbc14</groupId>
    <artifactId>ojdbc14</artifactId>
    <version>10.2.0.4.0</version>
</dependency>
```
那么换个版本吧，我们数据库是11.2g的，于是换了个ojdbc6
```xml
<dependency>
    <groupId>com.oracle</groupId>
    <artifactId>ojdbc6</artifactId>
    <version>11.2.0.4.0</version>
</dependency>
```
嗯，这次很顺利，没有再出现异常，看来ojdbc14确实有些问题，但是还是比较疑惑，单单400条数据，每条9个参数就已经超过限制了吗？
```xml
<insert id="insertBatch" parameterType="java.util.List">
        INSERT INTO table (a, b, c, d ......
        )
        SELECT SEQ.nextval,A.* FROM (
        <foreach collection="list" item="item" index="index" separator="union all">
            SELECT
            #{item.a},#{item.b},#{item.c},#{item.d}......
            FROM dual
        </foreach>) A
    </insert>
```
拼接下来实际SQL如下，类似于insert  into tableA select * from tableB
```sql
INSERT INTO table (a, b, c, d ......)
  SELECT
    SEQ.nextval,
    A.*
  FROM
	( SELECT ?, ?, ?, ?, ?, ?, ?, ? FROM dual 
 		union all 
 	  SELECT ?, ?, ?, ?, ?, ?, ?, ? FROM dual 
 		union all 
      SELECT ?, ?, ?, ?, ?, ?, ?, ? FROM dual
 		union all 
 		......
    ) A 
```
实际debug了一下，确实出异常的时候OraclePreparedStatement.setupBindBuffers方法short数组 bindIndicators大小超过了32768，换成ojdbc6的时候该方法未调用。