---
layout: post
title:  "Zookeeper学习总结"
date:   2017-12-09 15:56:20 +0800
categories: 框架
tags: zookeeper
---

* TOC
{:toc}

# ZooKeeper可以做什么

Zookeeper是分布式服务框架，主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等等。

首先ZooKeeper用于分布式应用管理，要了解ZooKeeper的特性，首先要考虑分布式系统设计的痛点在哪里？分布式系统就是将一个应用部署在多个物理机器上，对外提供统一的服务。这种系统设计会遇到的问题有：

* 扩展问题
当需要添加或减少服务器时，需要修改相关配置
* 故障处理
当一台机器出现问题的时候，能及时下线该服务，避免请求继续指向这台已经挂掉的服务器上

ZooKeeper提供了动态管理机器的功能，Zookeeper的系统架构为

![](/_pic/201711/zookeeper-struct.jpeg)

当服务启动时，自动向注册中心注册节点，client端则从注册中心获取可用的服务器信息；同时注册中心会定时向服务器发送心跳消息，及时删除无响应的服务节点信息．

分布式系统设计中广泛采用的系统架构：主从架构。

主从架构需要考虑的问题：主节点崩溃；从节点崩溃；主从节点间发生通信故障

# ZooKeeper相关  

## ZooKeeper数据的存储结构  

Zookeeper采用了树形的文件存储结构

## ZooKeeper节点类型    

ZooKeeper的节点包括四种，持久节点；临时节点；持久有序节点；临时有序节点；   

基于通知的访问机制，客户端每次访问节点的时候，都会放置一个监控点；当节点发生变更时，先发送通知给客户端，然后变更数据。

解决并行更新节点数据的问题：引入版本控制，当版本不一致时，更新失败    

ZooKeeper的服务器端运行模式：

独立模式和仲裁模式，对于仲裁模式，需要设定法定人数，这对ZooKeeper的服务器数量提出了要求。法定人数过多，过少都不好，最好的选择是服务器数量为奇数，法定数量为大于一半的最小数。    

## ZooKeeper的集群管理

服务器节点有三种：leader，follower，observer

leader负责所有状态变更的请求，在处理的时候会对事件进行排序和编号，保证集群内部消息处理的先后顺序。

使用ZooKeeper服务，首先需要实例化一个ZooKeeper的实例client，一旦获得一个server的连接，一个sessionId会被分配给client，之后，client和server之间维持长连接，并持续发送心跳给server，如果超过一定时间没有收到心跳消息，则session过期。    

只要这个连接有效，应用都可以通过client获得Zookeeper的API。若client在一定时间内未发送心跳，server会将这个session置为失效。若client当前连接的server无响应，若session仍旧有效，client会自动尝试重连另一个server。    

当集群中半数以上的机器同时挂掉，则，整个集群不可用；当leader挂掉，当半数以上的follower选举同意时，某server被选举为leader。   

由于ZooKeeper的写入首先需要通过Leader，然后这个写入的消息需要传播到半数以上的Fellower通过才能完成整个写入。所以整个集群写入的性能无法通过增加服务器的数量达到目的，相反，整个集群中Fellower数量越多，整个集群写入的性能越差；ZooKeeper集群中的每一台服务器都可以提供数据的读取服务，所以整个集群中服务器的数量越多，读取的性能就越好。但是Fellower增加又会降低整个集群的写入性能。  

为了避免这个问题，可以将ZooKeeper集群中部分服务器指定为Observer。observers不参与投票，只监听投票结果。除了这个区别之外，其他的功能和followers一样：客户端可以连接服务器，病向服务器发送、读取、写入请求。由于observers不参与投票，所以observers挂掉或者跟集群连接中断，不会影响到ZooKeeper服务。  

observer可用于：

* 跨数据中心部署，由于observer可以的部署对网络要求不高，可以跟client部署在一个数据中心，从而提高读取速度。observer不参与投票，降低了写入的网络开销
* 对读进行扩展，同时不影响写入性能。但observer的使用不能消除数据中心之间的网络延时，因为observer需要把更新请求转发到另一个数据中心的leader，处理inform消息，当网络速度极慢的话也会影响数据的同步。    

设置observer

在要充当observer节点的机器配置中添加：peerType=observer。然后，在每一个服务器配置文件中，添加observer，如：server.1:localhost:2181:3181:observer。这样用来告知ZooKeeper集群，这台server1是observer。

## ZooKeeper的watcher机制

ZooKeeper通过事件机制来管理各个服务器和客户端的连接，每一个ZooKeeper实例都会有一个watchermanager，用来管理当前已被注册的watcher列表，并负责触发它们。DataTree 中有一个WatchManager对象来维护所有类型的watch。    

当处理一个事务时，DataTree这个类会查明是否需要触发相应的Watch，如果发现有Watch需要触发，这个类就调用manager的触发方法。

ServerCnxn用来管理server和client之间的连接。在server端触发的Watch，这个类中的process方法会将这个Watch序列化并传播到client。

client实现监控需要的四个条件    

* 传入参数    
* 获取跟节点关联的数据，并开始处理
* 如果节点发生变化，client获取变化内容并处理
* 如果节点消失，client终止处理过程

ZooKeeper应用划分为两部分，一个维持连接，一个监控数据。

Watcher接口用来和container进行交流，只包含一个process方法，ZooKeeper用watcher告诉主线程关心的事件，如ZooKeeper 连接的状态的变化，或者session变化。Executor将事件交给dataMonitor来决定怎么处理。

## ZooKeeper做数据管理

每个节点上保存了我们要进行管理和同步的数据，添加节点时写入数据，读取的时候可以从不同的server中读取出保存的数据。（zk通过监听节点变化的事件来做节点数据的管理和同步），（实现服务管理的基本原理）

数据一致性

对节点的操作都会添加版本号，如果版本号和服务器的版本号不一致，会导致操作失败，所以每次在对节点做操作之前需要先从zookeeper server读取节点版本号，由此保证对同一个节点做更新操作时，不会被覆盖。

每个客户端同一时刻只和一个zookeeperserver保持一个会话信息，并通过定时发送心跳消息保持会话，通过这中方式保证事件处理的有序性。

## 获得一个ZooKeeper连接的过程：

解析server地址，并保存在server端。创建ClientCnxn实例（），并触发两个线程的开启：SendThread，EventThread。根据这两个线程的名字，一个是处理读写数据，一个处理watch事件。

EventThread：触发之后，遍历clientCnxn中管理的waitingevents列表，依次处理（触发watchEvent的process方法）。
SendEvent：负责发送请求队列，并生成心跳；可以生成ReadThread。

## ZooKeeper运行模式

ZooKeeper的standalone模式和quorum模式。standalone模式：只有一台机器来运行server进程；quorum模式：使用多台机器，每台机器运行一个server进程，或使用一台机器，在该机器上运行多个server进程

## sessionId

当新建一个client连接时，clientCnxn会生成一个sessionId，同时更新【最近一次发送心跳时间】。session由server端维护，如果standalone模式，由单个服务器维护，quorum模式下则是由leader来维护，follower会将client提交的sessionId信息转发给leader。

client端的SendThread会向server发送心跳消息，leader每隔半个tick时间向leader发送一个ping消息，learner返回自从上一个ping之后的session列表。leader中维护一个成为ExpiryQueue的数据结构来处理过期的session。
（SessionTrackerImpl）生成一个sessionId来关联session。通过session来判断当前client跟server的连接状态。管理过期session通过bucket，每个bucket中包含在指定expirationInterval要过期的session



# 配置与使用

* 下载ZooKeeper源码
* 在conf文件夹下添加配置文件，文件内容如下:

~~~
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
~~~

其中，tickTime：心跳时间，时间单位为ms，最小的session超时时间为2倍的tickTime
dataDir：内存数据保存位置
clientPort：监听client连接的端口

* 启动server端
`bin/zkServer.sh start`
* 将client连接到server
`bin/zkCli.sh -server 127.0.0.1:2181`
* client成功连接之后，就可以通过控制台输入一些命令完成相关操作：
创建节点：`create /zk_node my_data`
其中，zk_node是节点名称，my_data是该节点关联的数据
其他操作命令可以通过键入help来进行了解

~~~
jinyan@jinyan-latitude-e7470:~/Documents/applications/zookeeper/bin$ ./zkCli.sh  -server 127.0.0.1:2181
Connecting to 127.0.0.1:2181
Welcome to ZooKeeper!
[zk: 127.0.0.1:2181(CONNECTED) 0] ls /
[zookeeper, zk_node]
[zk: 127.0.0.1:2181(CONNECTED) 1] create /zk_node1 data
Created /zk_node1
[zk: 127.0.0.1:2181(CONNECTED) 2] ls /
[zk_node1, zookeeper, zk_node]
[zk: 127.0.0.1:2181(CONNECTED) 4] get /zk_node1
data
cZxid = 0x9
ctime = Sat Jun 17 16:19:34 CST 2017
mZxid = 0x9
mtime = Sat Jun 17 16:19:34 CST 2017
pZxid = 0x9
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
[zk: 127.0.0.1:2181(CONNECTED) 5] delete /zk_node
zk_node1   zk_node
[zk: 127.0.0.1:2181(CONNECTED) 5] delete /zk_node
[zk: 127.0.0.1:2181(CONNECTED) 6] ls /
[zk_node1, zookeeper]
~~~

在开发测试时可以以单机模式使用ZooKeeper，但是在线上环境，需要以复制模式来运行
复制模式下的配置文件

~~~
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
#监听Client端请求的端口号
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
#2888：监听同ZooKeeper集群内其他ZooKeeper Server进程通信请求的端口号
#3888：监听ZooKeeper集群内“leader”选举请求的端口号
~~~

建立一个client连接时，需要同时向这多个服务器地址做注册。


# 使用ZooKeeper开发

## 创建节点

ZooKeeper创建节点时，有四种类型可以选择：

~~~
CreateMode:
PERSISTENT (0, false, false),//当client失去连接时，该节点不会自动删除
PERSISTENT_SEQUENTIAL (2, false, true),//当client失去连接时，该节点不会被自动删除，并且创建节点时的名字后会添加一个自增的数字
EPHEMERAL (1, true, false),//当client失去连接时，该节点会自动删除
EPHEMERAL_SEQUENTIAL (3, true, true);//当client失去连接时，该节点会被自动删除，并且创建节点时的名字后会添加一个自增的数字
~~~

## 对节点做监听

ZooKeeper通过事件机制来对节点的变化做管理，当调用ZooKeeper 的exist接口检查节点是否存在的时候，可以设置在检查的节点上添加一个watcher。
源码如下：

~~~
/**
 * Return the stat of the node of the given path. Return null if no such a
 * node exists.
 * <p>
 * If the watch is non-null and the call is successful (no exception is thrown),
 * a watch will be left on the node with the given path. The watch will be
 * triggered by a successful operation that creates/delete the node or sets
 * the data on the node.
 *
 * @param path the node path
 * @param watcher explicit watcher
 * @return the stat of the node of the given path; return null if no such a
 *         node exists.
 * @throws KeeperException If the server signals an error
 * @throws InterruptedException If the server transaction is interrupted.
 * @throws IllegalArgumentException if an invalid path is specified
 */
public Stat exists(final String path, Watcher watcher)
    throws KeeperException, InterruptedException
{
    final String clientPath = path;
    PathUtils.validatePath(clientPath);

    // the watch contains the un-chroot path
    WatchRegistration wcb = null;
    if (watcher != null) {
        wcb = new ExistsWatchRegistration(watcher, clientPath);
    }

    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(ZooDefs.OpCode.exists);
    ExistsRequest request = new ExistsRequest();
    request.setPath(serverPath);
    request.setWatch(watcher != null);
    SetDataResponse response = new SetDataResponse();
    ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);
    if (r.getErr() != 0) {
        if (r.getErr() == KeeperException.Code.NONODE.intValue()) {
            return null;
        }
        throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                clientPath);
    }

    return response.getStat().getCzxid() == -1 ? null : response.getStat();
}
~~~

节点注册watcher：

~~~
 zk.exists("/registry", this);
            List<String> nodeList = zk.getChildren("/registry", this);
            for (String node : nodeList) {
                zk.exists("/registry"+ '/' + node, this);
            }
~~~

在/registry下创建节点，触发的eventType为：NodeChildrenChanged，path为：/registry
删除/reigistry下的节点/app10000000015，触发的eventType为：NodeDeleted，path为：/registry/app10000000015

# 其他

项目[hot-deploy](https://github.com/swa19/hot-deploy)中使用了ZooKeeper做分布式管理






# 参考：

[一步到位分布式开发Zookeeper实现集群管理](http://www.cnblogs.com/zhangs1986/p/6564839.html)    

[ZooKeeper安装与运行](http://blog.csdn.net/dslztx/article/details/51079457)    
