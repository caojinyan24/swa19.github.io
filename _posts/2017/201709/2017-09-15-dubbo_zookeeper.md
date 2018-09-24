---
layout: post
title:  "Dubbo之服务注册"
date:   2017-09-15 +0800
categories: 框架
tags: dubbo
---

* TOC
{:toc}

Dubbo作为一个分布式的服务框架，注册中心除了zookeeper之外，还包括Redis和dubbo默认注册中心（当不配置注册中心的时候做默认选项，在这个默认的实现中，注册的服务被保存在内存中，Dubbo通过定时重试来监测注册服务是否可用，当注册服务发生变更时，DubboRegistry类负责通知服务变更信息给注册的Listener）。

![](/_pic/201709/FailbackRegistry.png)

之前写过一个简单的服务，其中的服务注册和管理使用了zookeeper：调用方通过zookeeper来获取服务提供方的URL地址，然后到对应的URL地址请求服务。所以涉及服务注册的代码就着重看下Dubbo中对zookeeper的使用和封装，然后跟自己的实现做个对比

# 代码结构  

分析源码，首先要搞清楚各个类之间的继承关系和职责。 既然是服务注册，首先要有提供服务注册的类：RegisterService
ZookeeperRegistry类的依赖关系：

![](/_pic/201709/zookeeperRegistry.png)

~~~
public interface Node {

    /**
     * get url.
     *
     * @return url.
     */
    URL getUrl();

    /**
     * is available.
     *
     * @return available.
     */
    boolean isAvailable();

    /**
     * destroy.
     */
    void destroy();

}
~~~
~~~
public interface RegistryService {

    /**
     * 注册数据，比如：提供者地址，消费者地址，路由规则，覆盖规则，等数据。
     * <p>
     * 注册需处理契约：<br>
     * 1. 当URL设置了check=false时，注册失败后不报错，在后台定时重试，否则抛出异常。<br>
     * 2. 当URL设置了dynamic=false参数，则需持久存储，否则，当注册者出现断电等情况异常退出时，需自动删除。<br>
     * 3. 当URL设置了category=routers时，表示分类存储，缺省类别为providers，可按分类部分通知数据。<br>
     * 4. 当注册中心重启，网络抖动，不能丢失数据，包括断线自动删除数据。<br>
     * 5. 允许URI相同但参数不同的URL并存，不能覆盖。<br>
     *
     * @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void register(URL url);

    /**
     * 取消注册.
     * <p>
     * 取消注册需处理契约：<br>
     * 1. 如果是dynamic=false的持久存储数据，找不到注册数据，则抛IllegalStateException，否则忽略。<br>
     * 2. 按全URL匹配取消注册。<br>
     *
     * @param url 注册信息，不允许为空，如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void unregister(URL url);

    /**
     * 订阅符合条件的已注册数据，当有注册数据变更时自动推送.
     * <p>
     * 订阅需处理契约：<br>
     * 1. 当URL设置了check=false时，订阅失败后不报错，在后台定时重试。<br>
     * 2. 当URL设置了category=routers，只通知指定分类的数据，多个分类用逗号分隔，并允许星号通配，表示订阅所有分类数据。<br>
     * 3. 允许以interface,group,version,classifier作为条件查询，如：interface=com.alibaba.foo.BarService&version=1.0.0<br>
     * 4. 并且查询条件允许星号通配，订阅所有接口的所有分组的所有版本，或：interface=*&group=*&version=*&classifier=*<br>
     * 5. 当注册中心重启，网络抖动，需自动恢复订阅请求。<br>
     * 6. 允许URI相同但参数不同的URL并存，不能覆盖。<br>
     * 7. 必须阻塞订阅过程，等第一次通知完后再返回。<br>
     *
     * @param url      订阅条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件监听器，不允许为空
     */
    void subscribe(URL url, NotifyListener listener);

    /**
     * 取消订阅.
     * <p>
     * 取消订阅需处理契约：<br>
     * 1. 如果没有订阅，直接忽略。<br>
     * 2. 按全URL匹配取消订阅。<br>
     *
     * @param url      订阅条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件监听器，不允许为空
     */
    void unsubscribe(URL url, NotifyListener listener);

    /**
     * 查询符合条件的已注册数据，与订阅的推模式相对应，这里为拉模式，只返回一次结果。
     *
     * @param url 查询条件，不允许为空，如：consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @return 已注册信息列表，可能为空，含义同{@link com.alibaba.dubbo.registry.NotifyListener#notify(List<URL>)}的参数。
     * @see com.alibaba.dubbo.registry.NotifyListener#notify(List)
     */
    List<URL> lookup(URL url);

}
~~~

zookeeper通过不断发送心跳信息来感知服务是否可用，总的来说它包含了两个功能，一个是存储功能，供服务调用者获取可用的服务信息；另一个是服务的可用管理，对于不可用或新增的服务信息，及时更新存储信息，一方面避免服务调用方继续调取不可用的服务，另一方面利于服务的横向扩展。在Dubbo中ZookeeperRegister提供了zookeeper注册中心服务的注册和管理服务。

# 过程调用

既然Dubbo提供了多种服务注册方式，那么对于Dubbo使用者来说，怎么选择注册中心呢？这就涉及到java的扩展机制了。

注册Dubbo服务时的zookeeper配置如下：
集群：

~~~
<dubbo:registry protocol="zookeeper" address="172.16.3.57:2181,172.16.3.58:2181" />
~~~

单机：

~~~
<dubbo:registry protocol="zookeeper" address="172.16.3.57:2181" />
~~~


在使用了Dubbo服务的项目启动时，DubboBeanDefinitionParser中的parse方法负责解析以上配置信息，其中关于protocol的解析代码如下：

~~~
                                } else if ("protocol".equals(property)
                                        && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                        && (!parserContext.getRegistry().containsBeanDefinition(value)
                                        || !ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                    if ("dubbo:provider".equals(element.getTagName())) {
                                        logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                    }
                                    // 兼容旧版本配置
                                    ProtocolConfig protocol = new ProtocolConfig();
                                    protocol.setName(value);
                                    reference = protocol;
                                }
~~~
ExtensionLoader类负责获得外部扩展类，在源码中，loadFile方法从资源文件中把对应的处理类加载进来，那么对应于zookeeper注册中心的资源文件名为：com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory，内容如下：

~~~
zookeeper=com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory
~~~

后续通过注解Adaptive的处理之后，就可以通过`RegistryFactory.getRegistry(URL url);`方法连接注册中心

~~~
@SPI("dubbo")
public interface RegistryFactory {

    /**
     * 连接注册中心.
     * <p>
     * 连接注册中心需处理契约：<br>
     * 1. 当设置check=false时表示不检查连接，否则在连接不上时抛出异常。<br>
     * 2. 支持URL上的username:password权限认证。<br>
     * 3. 支持backup=10.20.153.10备选注册中心集群地址。<br>
     * 4. 支持file=registry.cache本地磁盘文件缓存。<br>
     * 5. 支持timeout=1000请求超时设置。<br>
     * 6. 支持session=60000会话超时或过期设置。<br>
     *
     * @param url 注册中心地址，不允许为空
     * @return 注册中心引用，总不返回空
     */
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);

}
~~~

实现如下：

~~~
    public Registry getRegistry(URL url) {
        url = url.setPath(RegistryService.class.getName())
                .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
        String key = url.toServiceString();
        // 锁定注册中心获取过程，保证注册中心单一实例
        LOCK.lock();
        try {
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // 释放锁
            LOCK.unlock();
        }
    }
~~~

最后看下`ZookeeperRegistry`中的注册方法:

~~~
    @Override
    public void register(URL url) {
        if (destroyed.get()){
            return;
        }
        super.register(url);
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            // Sending a registration request to the server side
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;

            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }

            // Record a failed registration request to a failed list, retry regularly
            failedRegistered.add(url);
        }
    }
~~~

其中的doRegister将url中包含的接口和地址存放在zk节点上.
url长类似这样:`url=consumer://ip/interface?application=..-provider&category=consumers&check=false&dubbo=2.5.3&group=..&interface=interface&methods=method1,method2&organization=..&owner=bing&pid=28961&reference.filter=cbsChannelFilter&retries=0&revision=1.2.2-SNAPSHOT&side=consumer&timeout=8000&timestamp=1518157342897&version=1.0.0
`



最后理下Zookeeper服务的调用链：

1. ServiceBean:afterPropertiesSet()
服务启动后，做Service的初始化工作
2. ServiceConfig:export()->doExport()->doExportUrls()->doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs)
服务启动后，为配置的Dubbo服务寻找host和port，并放入到本地内存中
3. ProtocolFilterWrapper:export(Invoker<T> invoker)
4. RegistryProtocol:export(final Invoker<T> originInvoker)  ~(这个方法中同时还做了服务订阅的动作，这个稍后再说) ~
5. FailBackRegistry.register(URL url)->doRegister(URL url)
6. ZookeeperRegistry.doRegister(URL url)

# 总结

Dubbo对于不同的注册中心实现，只需要实现RegistryFactory的相关接口即可，对zookeeper的实现做了再次封装，如定义了一个统一的接口NotifyListener来做服务变更的通知等。在系统设计中也可以参考这种实现方式，通过对第三方接口的再次封装，为上层服务提供了统一的接口，从而为系统扩展提供了最大的便利。
