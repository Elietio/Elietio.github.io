---
title: '对于含有泛型的类,如何在类中获取泛型的class对象？'
categories:
  - Java
mathjax: true
copyright: true
comment: true
date: 2018-5-5
tags: Java
abbrlink: eb59d595
---


对于含有泛型的类,如何在类中获取泛型的class对象？

如果子类继承该类的时候传递了泛型，所以编译期该类的泛型其实已经指定了。那么在类中定义一个Class对象，然后通过构造代码块，this指向的是当前调用的子类

<!-- more -->
``` java
 private Class<T> clazz;
 {
    // 获取当前类上的泛型(this指向当前子类)
    ParameterizedType type =(ParameterizedType)this.getClass().getGenericSuperclass();
    clazz = (Class<T>) type.getActualTypeArguments()[0];
  }
```

对于没有继承该类的子类，可以采用new的时候有参构造函数传递

``` java
private Class<T> clazz;
//通过构造函数指定元素类型
public classname(Class<T> clazz) {
      super();
      this.clazz = clazz;
 }
```
