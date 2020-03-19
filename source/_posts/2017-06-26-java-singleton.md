---
title: JAVA单例模式的七种写法
categories:
  - Java
mathjax: true
copyright: true
comment: true
url: 1.html
id: 1
tags: Java
abbrlink: 7a9cc514
date: 2017-06-26 00:29:32
photo:
---

JAVA单例模式的七种写法
<!-- more -->

- 第一种（懒汉，线程不安全）：

```java
public class Singleton { 
    
    private static Singleton instance;  
    private Singleton (){} 
    
	public static Singleton getInstance() {  
		if (instance == null) {  
    		instance = new Singleton();  
		}  
	return instance;  
	}  
}
```

懒加载，但是多线程单例会失败。

- 第二种（懒汉，线程安全）：

```java
public class Singleton {  
    private static Singleton instance;  
    private Singleton (){}  
    public static synchronized Singleton getInstance() {  
    if (instance == null) {  
        instance = new Singleton();  
    }  
    return instance;  
    }  
}
```

线程安全，效率低，大多数情况下不需要同步。

- 第三种（饿汉）：

```java
public class Singleton {  
    private static Singleton instance = new Singleton();  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return instance;  
    }  
}
```

避免了线程安全问题，但是instance在类加载的时候就实例化了，不能达到懒加载的效果。

- 第四种（饿汉，变种）：

```java
public class Singleton {  
    private Singleton instance = null;  
    static {  
    instance = new Singleton();  
    }  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return this.instance;  
    }  
}
```

和第三种实际上差不多。类初始化的时候实例了instance

- 第五种（静态内部类）：

```java
public class Singleton {  
    private static class SingletonHolder {  
    private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
    return SingletonHolder.INSTANCE;  
    }  
}
```

这种方式同样利用了classloder的机制来保证初始化instance时只有一个线程，它跟第三种和第四种方式不同的是：第三种和第四种方式是只要Singleton类被装载了，那么instance就会被实例化（没有达到lazy loading效果），而这种方式是Singleton类被装载了，instance不一定被初始化。因为SingletonHolder类没有被主动使用，只有显示通过调用getInstance方法时，才会显示装载SingletonHolder类，从而实例化instance。

静态内部类和非静态内部类一样，都是在被调用时才会被加载。当只调用外部类的静态变量，静态方法时，是不会让静态内部类的被加载。

不过在加载静态内部类的过程中也会加载外部类。

- 第六种（枚举）：

```java
public enum Singleton {  
    INSTANCE;  
    public void whateverMethod() {  
    }  
}
```

它不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象，jdk1.5以上支持

- 第七种（双重校验锁）：

```java
public class Singleton {  
    private volatile static Singleton singleton;  
    private Singleton (){}  
    public static Singleton getSingleton() {  
    if (singleton == null) {  
        synchronized (Singleton.class) {  
        if (singleton == null) {  
            singleton = new Singleton();  
        }  
        }  
    }  
    return singleton;  
    }  
}
```

 只锁定singleton==null的情况，一旦实例化后就不再同步。但是在singleton=new Singleton();这一步jvm执行是无序的，可能先设置了singleton为非空，再构造了Singleton，若此时其它线程进入了判断，singleton！=null，返回的singleton并不是Singleton的实例。 

在这里用到了volatile关键字，一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：

*   保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
*   禁止进行指令重排序

