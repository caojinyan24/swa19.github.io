---
layout: post
title:  "Tomcat-StringManager"
date:   2018-03-20 11:44:15 +0800
categories: 框架
tags: tomcat
---

* TOC
{:toc}


这些天在看tomcat,有一个很有趣的类:`StringManager`,这个类负责将抛出的异常信息包装成指定的语言并输出.

## 获得StringManager实例

获得实例使用了单例模式和工厂模式,每个类所在的包对应不同StringManager实例

~~~
    private static final StringManager sm = StringManager.getManager(PasswdUserDatabase.class);

    public static final synchronized StringManager getManager(String packageName) {
        StringManager mgr = managers.get(packageName);
        if (mgr == null) {
            mgr = new StringManager(packageName);
            managers.put(packageName, mgr);
        }
        return mgr;
    }
~~~

最后看下StringManager类的私有构造方法,这里的ResourceBundle和Local早在JDK1.1就已经存在了,以下就是Tomcat源代码中对ResourceBundle的用法

~~~
    private StringManager(String packageName) {
        String bundleName = packageName + ".LocalStrings";
        ResourceBundle tempBundle = null;
        try {
            tempBundle = ResourceBundle.getBundle(bundleName, Locale.getDefault());
        } catch( MissingResourceException ex ) {
            // Try from the current loader (that's the case for trusted apps)
            // Should only be required if using a TC5 style classloader structure
            // where common != shared != server
            ClassLoader cl = Thread.currentThread().getContextClassLoader();
            if( cl != null ) {
                try {
                    tempBundle = ResourceBundle.getBundle(
                            bundleName, Locale.getDefault(), cl);
                } catch(MissingResourceException ex2) {
                    // Ignore
                }
            }
        }
        // Get the actual locale, which may be different from the requested one
        if (tempBundle != null) {
            locale = tempBundle.getLocale();
        } else {
            locale = null;
        }
        bundle = tempBundle;
    }
~~~

`ResourceBundle`类中的成员变量包括:
private static final ConcurrentMap<CacheKey, BundleReference> cacheList = new ConcurrentHashMap<>(INITIAL_CACHE_SIZE);
通过key来获取资源的类加载器
private static final ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();
当前bundle中的类加载器的引用

ResourceBundle的说明文档中,有一段说明:

~~~
You do not have to restrict yourself to using a single family of
`ResourceBundle`s. For example, you could have a set of bundles for
exception messages, `ExceptionResources`
(`ExceptionResources_fr`, `ExceptionResources_de`, ...),
and one for widgets, <code>WidgetResource</code> (`WidgetResources_fr`,
`WidgetResources_de`, ...); breaking up the resources however you like.
~~~
而ResourceBundle的子类中,也确实有很多有意思的子类.后边如果需要多语言的功能,可以使用这个类来实现,避免重复造轮子了


## 创建Resource包
在IDEA中,在要建立Resource包的位置右键:New->Resource Bundle,创建一个Resource包,添加语言包,IDEA中默认有5种,但可以自定义添加所需要的语言.
![](/_pic/201803/createbundle.png)

添加完成后,在IDEA可以方便地添加key-value属性
![](/_pic/201803/bundle.png)



