---
title: Java Web——Servlet开发入门
tags:
  - servet
categories:
  - Java
mathjax: true
copyright: true
comment: true
abbrlink: 3d37fb53
date: 2017-08-11 16:07:18
photo:
---
### Servlet是什么？

Servlet，Server Applet,是一个Java编写的小程序，可以在Web服务器上运行（如Tomcat），它通过HTTP协议接受和响应客户端请求。客户端发送请求给Web服务器，服务器将请求信息发送给Servlet，Servlet生成响应，传递给服务器，服务器再响应至客户端。Servlet在Java中是一个接口，用于定义规则和拓展功能。
<!-- more -->

### Servlet生命周期

1. 初始化init
客户端发送请求访问servlet程序，Tomcat会在内存中查找是否存在该servlet的对象，如不存在，则调用init方法创建servlet对象，如已经存在，则直接使用。

2. 运行service
Tomcat为请求创建ServletRequest对象和ServletResponse对象，每执行一次service方法都从ServletRequest中获取请求参数，通过ServletResponse发送响应给客户端。

3. 销毁destory
当服务器关闭或者Web项目移除时，servlet会关闭，Tomat调用servlet的destory方法进行销毁，释放资源。

### HttpServlet

Servlet是一个无协议的接口，定义了服务器和java程序之间规则。HttpServlet是它的一个子类，具有Http协议，它的service方法会对请求判断是执行doGet方法还是doPost方法，因此书写Servlet程序需要继承HttpServlet类，并复写doGet方法和doPost方法。

```java
package ${enclosing_package};
import java.io.IOException;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class ${primary\_type\_name} extends HttpServlet {

 	protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {

    }

	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
     }

}
```

### web.xml配置Servlet

通过web.xml配置，让url路径访问对应的servlet程序

```xml
<servlet>

	<servlet-name>servlet名称</servlet-name>

	<servlet-class>servlet程序的绝对路径</servlet-class>

</servlet>

<servlet-mapping>

	<servlet-name>servlet名称，与上面保持一致建立映射</servlet-name>

	<url-pattern>访问路径，格式为"/具体内容"，表示的URL为localhost:8080/web项目名/具体内容</url-pattern>

</servlet-mapping>
```

### 相对路径和绝对路径

1. 客户端的路径，比如利用页面访问，一般使用绝对路径，后面加上项目名称
  /:指的的是绝对路径，表示http：//ip:port
不带/表示相对路径，表示当前文件所在的父目录的路径

2. 服务端路径，比如web.xml
 /,绝对路径，代表http：/ip:port/项目名称

### ServletConfig

在开发中，servlet创建有时需要给定一些初始化参数，web服务器会把这些初始化信息封装到ServletConfig中，创建servlet时调用的init方法时，会将初始化信息传递给servlet，因此可以看作web程序的局部配置。

初始化信息在web.xml中的servlet标签中，以key和value的形式设置。获取初始化信息通过API getInitParameter(String name) 根据key获取value和 getInitParameterNames() 返回所有key的枚举

### ServletContext

每一个web项目都是运行在java虚拟机中的一个web程序，它拥有一个与之对应的上下文对象，即ServletContext，它可以提供web程序中所有servlet共享的信息，即web程序的全局配置。

1. ServletContext的设置是在web.xml的根目录中,想要获取配置信息需要先获取ServletContext对象。获取ServeltContext可以通过ServletConfig中的getServetContext()获取，或者通过继承的GenericServlet中的getServletContext()方法直接获取.然后通过ServetContext中的getInitParameterNames()或getInitParameterNames()获取全局配置信息.
2. ServletContext可以看作web程序的一个公共容器，一个web程序所有servlet都可以获取其中的数据。
	增加：setAttribute(String name,Object object)
	删除：removeAttribute(String name
	获取：getAttribute(String name)
3. ServeltContext读取文件路径
	getRealPath(String path)    返回的是绝对路径，工程发布后，在电脑中的的实际路径
    getContextPath()        获取项目的相对路径