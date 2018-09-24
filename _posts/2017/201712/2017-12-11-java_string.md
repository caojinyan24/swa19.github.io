---
layout: post
title:  "String，StringBuilder和StringBuffer比较"
date:   2017-12-11 18:42:58 +0800
categories: 基础
tags: java
---

* TOC
{:toc}

今天用idea的时候，注意到生成toString代码的时候，提供了两种方式：StringBuilder和StringBuffer，之前一直用的StringBuilder，避免在修改字符串的时候频繁申请内存空间，StringBuffer倒是很少用，刚好有时间，理下这三个类的区别。

# String
String变量是不可更改的，每次初始化一个String变量：`String parameter="test";` 在堆内存中都会保存一个常量字符串，每次对String变量做修改都会新建一个字符串常量

~~~
    public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
    public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
    public String replace(char oldChar, char newChar) {
        if (oldChar != newChar) {
            int len = value.length;
            int i = -1;
            char[] val = value; /* avoid getfield opcode */

            while (++i < len) {
                if (val[i] == oldChar) {
                    break;
                }
            }
            if (i < len) {
                char buf[] = new char[len];
                for (int j = 0; j < i; j++) {
                    buf[j] = val[j];
                }
                while (i < len) {
                    char c = val[i];
                    buf[i] = (c == oldChar) ? newChar : c;
                    i++;
                }
                return new String(buf, true);
            }
        }
        return this;
    }
~~~

可以看到，由于String实例只是堆内存中一个常量字符串的引用，对这个String实例指向的字符串常量是不可修改的，唯一能修改的是这个实例指向的内存地址，所以每次对一个String实例做修改的时候，只会重新分配一个内存空间，赋值后将这个String实例指向修改后的内存空间。


多说一句关于String的方法`public native String intern();`看网上分析说1.6和1.7的intern方法实现不同，我粘一段1.8的解释,这个方法用于返回一个字符串，它的内容与这个字符串相同，但它来自于字符串常量池。

~~~
返回字符串对象的规范表示。 一组字符串，最初是空的，由类String私下维护。 当调用intern()方法时，如果池中已经包含了一个字符串，该字符串等于equals(object)方法所确定的string对象，那么将返回池中的字符串。否则，这个String对象被添加到池中，并返回这个String对象的引用。
它遵循了任何两个字符串s和t,只有当s.equals(t)的时候，才会有s.intern()==t.intern()。 所有的字符串和字符串值常量表达式都被interned。String 3.10.5节中定义的Java™语言规范。
~~~

# StringBuffer
StringBuffer是线程安全的，这个类中需要做多线程同步的方法都加了synchronized关键字做线程同步。

# StringBuilder
StringBuilder是StringBufferr的补充版，两个类提供了一样的接口，但是由于StringBuilder不需要做同步处理，所以处理速度要比StringBuffer快。
