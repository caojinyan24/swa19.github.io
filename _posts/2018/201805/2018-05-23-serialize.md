---
layout: post
title:  "Java序列化"
date:   2018-05-23 11:04:40 +0800
categories: 基础
tags: java
---

面试的时候被问到Java序列化，系统地总结下

# 什么是序列化

对象序列化就是把内存中的对象转化为二进制字节码的过程，反序列化相反

# 为什么要序列化

在程序运行过程中，所有的对象都存储在内存中，通过对象序列化可以实现把内存中的对象保存在文件或进行网络传输的目的。

# 怎么序列化

## 继承Serializable接口

如果需要自定义序列化，在类的定义中添加两个实现方法

~~~
  private void writeObject(java.io.ObjectOutputStream out) throws IOException
  private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException;
  private void readObjectNoData() throws ObjectStreamException;
~~~

## 继承Externalizable

`public interface Externalizable extends java.io.Serializable`

通过实现Externalizable的两个接口，以对序列化和反序列化的属性做个性化的转换。

## 将对象包装成Json字符串

通过转成Json字符串，可以跨语言传输，并且属性的增加或减少不会对解析造成影响，兼容性比较好

# 其他补充

1. 静态变量和成员方法不可以被序列化
2. 如果要序列化一个类对象，这个对象中的所有引用对象也要可以被序列化，否则会导致序列化失败
3. 声明成transient的变量不可以被序列化，反序列化后，这个被声明的变量的值是null

# 序列号

一般继承了`Serializable`的子类都会声明一个`serialVersionUID`变量，指定反序列化的序列号，在反序列化的时候会对序列号和要序列化的对象做比对，如果不相等的话会抛出InvalidClassException异常。
