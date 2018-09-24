---
layout: post
title:  "SpringCloud基础教程"
date:   2018-04-26 19:23:50 +0800
categories: 框架
tags: springCloud
---

* TOC
{:toc}

借用一篇博客中的目录理下springcloud涉及的内容，整理自`Finchley.RC1`的SpringCloud文档

# 服务注册与发现
SpringCloudNetflix整合了Netflix的部分模块，包括：Eureka(服务发现),Hystrix(断流),Zuul(路由),Ribbon(客户端负载均衡).

## eureka
使用Eureka做服务发现    
服务端：负责管理注册在服务端的所有的client，以下为相关配置

1. 启动类添加`@EnableEurekaServer`注解
2.  配置文件中添加服务注册地址和端口号


        spring.application.name=eureka-server
        server.port=7001
        eureka.client.register-with-eureka=false
        eureka.client.fetch-registry=false
        #server,作为服务管理中心管理http://localhost:7001/eureka/地址的服务
        eureka.client.service-url.defaultZone=http://localhost:${server.port}/eureka/


3. 添加maven依赖

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>


客户端：


2.  配置文件中添加服务注册地址和端口号

        eureka.client.service-url.defaultZone=http://localhost:7001/eureka/

3. 添加maven依赖

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>



## consul
# 消费者
## ribbon
## feign
# 分布式配置中心
# 服务容错
# 断路器

SpringCloud采用了Netflix的Hystrix来实现断路器，为了解决服务中的雪崩问题，它采用的机制有：

## 资源隔离：线程池隔离和信号量隔离

每个服务分配独立的线程池，每个请求设置超时时间，堆积的请求进入线程池队列，从而保证了各个接口的请求不会因为抢占系统资源而互相影响。

每个线程池存在一个计数器，统计当前有多少个线程在运行，当超过设置的最大线程数则丢弃新的请求，调用回调函数返回。避免请求量大的时候新的请求不断堆积在内存中等待。

## 降级

当出现超时、资源不足时，会触发降级机制，此时不会请求接口，而是直接调用回调接口返回

## 熔断

当失败率达到设置的阈值，熔断器触发的快速失败会通过尝试连接来判断服务是否恢复

## 缓存




# 服务网关
# 分布式消息
# 服务跟踪
## 跟踪原理
## logstash
## zipkin
## 收集
## 抽样

# 参考

[SpringCloud](http://cloud.spring.io/spring-cloud-static/Finchley.RC1/single/spring-cloud.html)
