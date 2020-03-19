---
title: Logback totalSizeCap不生效的两个bug
categories: Java
tags:
  - Logback
  - totalSizeCap
  - 999
  - 排错
copyright: true
comment: true
abbrlink: f6b1711c
date: 2020-02-26 19:11:44
description:
---

### 问题描述：

线上的一个数据接收服务最近数据量比较大，日志也打的很频繁，但是却出现了totalSizeCap配置不生效，无法删除日志文件的问题，每天日志一度回滚累计到几千个 。
<!-- more -->

日志设置maxHistory 30,maxFileSize 20MB,totalSizeCap 5GB

```xml
<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
    <fileNamePattern>
        ${logger.path}/all.%d{yy-MM-dd}.%i.log
    </fileNamePattern>
    <maxHistory>30</maxHistory>
    <maxFileSize>20MB</maxFileSize>
    <totalSizeCap>5GB</totalSizeCap>
</rollingPolicy>
```

### 问题排查：

面向Google查询相关问题，看到一个logback无法处理totalSizeCap 超过2GB的bug

```java
void capTotalSize(Date now) {
    int totalSize = 0;
    int totalRemoved = 0;
    for (int offset = 0; offset < maxHistory; offset++) {
        Date date = rc.getEndOfNextNthPeriod(now, -offset);
        File[] matchingFileArray = getFilesInPeriod(date);
        descendingSortByLastModified(matchingFileArray);
        for (File f : matchingFileArray) {
            long size = f.length();
            if (totalSize + size > totalSizeCap) {
                addInfo("Deleting [" + f + "]" + " of size " + new FileSize(size));
                totalRemoved += size;
                f.delete();
            }
            totalSize += size;
        }
    }
    addInfo("Removed  " + new FileSize(totalRemoved) + " of files");
}
```

这段代码定义了totalSize来累加当前日志文件大小，但是数据类型是int，我们知道int类型最大值是 2147483647，而如果totalSize超过了2GB `totalSize + size > totalSizeCap`会不成立。。。。

所以一旦totalSizeCap超过2GB，那么就会导致日志清除失效。

这个问题有人反馈过 [issue](https://jira.qos.ch/browse/LOGBACK-1231)，而logback也在1.2.0版本修复了这个问题，[解决方案是修改int为long类型](https://github.com/qos-ch/logback/commit/2240d4253c9f3fa28497e74f34be3bb1edc73078#diff-cb428af44f615094fb61e87a9ca3411a)。

![commit_2240d42](/img/logback-totalSizeCap/commit_2240d42.png)

但是这个并不适用于线上遇到的情况，因为服务中使用的是1.2.3版本，于是继续查询分析。又发现了一个[类似报告](https://jira.qos.ch/browse/LOGBACK-1500)，而且报告的版本也是1.2.3

>  I'm using SizeAndTimeBasedRollingPolicy.
>  When '%i' file index reaches 999 it stops deleting the old files and totalSizeCap is not respected any more.
>  This soon leads to disk full issues (as logging in my case was fast enough) 

这个问题有点相似了，看一下生产环境的日志文件，前几天都是只保留了后缀1000以上的，当天的保留了700以上的，也就是删除了一部分，后面未删除，而且感觉和这个999又很大关系，这个issue里面没有相关回复，那我们自己一边查看源码一般继续Google吧，上面的源码我们已经看到是日志大小回滚的实现，`File[] matchingFileArray = getFilesInPeriod(date)`，这一步应该是获取相关的文件数组，我们进去继续查看

```java
protected File[] getFilesInPeriod(Date dateOfPeriodToClean) {
        File archive0 = new File(fileNamePattern.convertMultipleArguments(dateOfPeriodToClean, 0));
        File parentDir = getParentDir(archive0);
        String stemRegex = createStemRegex(dateOfPeriodToClean);
        File[] matchingFileArray = FileFilterUtil.filesInFolderMatchingStemRegex(parentDir, stemRegex);
        return matchingFileArray;
    }
 private String createStemRegex(final Date dateOfPeriodToClean) {
        String regex = fileNamePattern.toRegexForFixedDate(dateOfPeriodToClean);
        return FileFilterUtil.afterLastSlash(regex);
    }
```
上面这一块的逻辑是生成一个正则去和日志目录下的日志匹配获取日志文件，下面的代码就是具体拼接正则的实现
```java
/**
     * Given date, convert this instance to a regular expression.
     *
     * Used to compute sub-regex when the pattern has both %d and %i, and the
     * date is known.
     * 
     * @param date - known date
     */
    public String toRegexForFixedDate(Date date) {
        StringBuilder buf = new StringBuilder();
        Converter<Object> p = headTokenConverter;
        while (p != null) {
            if (p instanceof LiteralConverter) {
                buf.append(p.convert(null));
            } else if (p instanceof IntegerTokenConverter) {
                buf.append("(\\d{1,3})");
            } else if (p instanceof DateTokenConverter) {
                buf.append(p.convert(date));
            }
            p = p.getNext();
        }
        return buf.toString();
    }
```

看到这一块果然发现一个问题`buf.append("(\\d{1,3})")`，这个正则是匹配1位到3位的数字，这不刚好，只能识别日志文件后缀1-999，999以上就无法匹配了。

同样我们也找到了相关的[issue](https://jira.qos.ch/browse/LOGBACK-1297)

> I found cause.
>
> Check the **toRegexForFixedDate()** method in **ch.qos.logback.core.rolling.helper.FileNamePattern.java**
>
> Regular expression hardcoded like this:
>
> ***buf.append("(\ \d{1,3})");***
>
> So, files indexed more than 3-digit number are not visible to delete...
>
> I don't know why the expression hardcoded.
>
> Anyway, you'd better modify the source.

有人提到了现在已经修复
> this issue fixed in 1.3.0-alpha1
>
> https://jira.qos.ch/browse/LOGBACK-1175

果然这里存在问题，[1.3.0-alpha1版本修复这个](https://github.com/qos-ch/logback/commit/f264607fb450)

![commit_f264607](/img/logback-totalSizeCap/commit_f264607.png)

但是为什么出现了前几天日志文件保留了后缀999以上，当天存在部分低于999的呢？其实这个也很好理解，还是上面源码，先正则匹配到文件之后循环累加大小，maxFileSize =20MB,totalSizeCap =5GB= 5120MB，等于256个文件，256不能被999整除，还会剩余231，也就是说第一天清理了768个文件后，剩余  231 * 20MB<5120MB ,所以不会清理掉，第二天才会继续累加这部分，然后删除上一天剩余的后缀小于1000的日志文件，这样就导致每天可能剩余一定量的后缀小于1000的日志文件，直到第二天被清理，这和我们的实际情况是相符的。

顺便吐槽一下，从GitHub的文件history来看，2012年作者已经改过一次这个正则了，把[d{1,2}改成 d{1,3}](https://github.com/qos-ch/logback/commit/608ed4c58717296b8182006e35aff52ca2ecb598#diff-80c6c7b6b5e9e46b2e5afd6ec7eb07fd) ，真是醉了┑(￣Д ￣)┍

![commit_608ed4c](/img/logback-totalSizeCap/commit_608ed4c.png)


### 问题解决：

问题已经找出，那么解决就很简单，升级logback版本就行，当然我觉得首先这个日志打印问题就很大，一天打印上千个日志文件，大小几十G，这根本没有日志的意义了，纯属浪费资源。

<blockquote class="question">
参考📚：
<i class="fa fa-hand-o-right" aria-hidden="true"></i>   https://tidyko.com/posts/589711b0.html
 <i class="fa fa-hand-o-right" aria-hidden="true"></i>   https://jira.qos.ch/browse/LOGBACK-1500
 <i class="fa fa-hand-o-right" aria-hidden="true"></i>   https://jira.qos.ch/browse/LOGBACK-1297
</blockquote>