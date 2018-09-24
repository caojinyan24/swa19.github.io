---
layout: post
title:  "Jenkins部署配置"
date:   2018-03-01 16:05:31 +0800
categories: 基础
tags: ops
---

* TOC
{:toc}

# 使用Jenkin部署web服务的配置

1. 首先下载Jenkins的jar包，并上传到服务器的指定目录下
然后通过java命令运行：`java -jar jenkins.jar`
指定端口号：`java -jar jenkins.war --httpPort=8080`
2. 在浏览器中打开：`http://host:8080`
根据引导做配置。

3. 添加部署项目
New Item->Multi-configuration project

4. 使用tomcat运行web服务

* Build：

~~~
BUILD_ID=STOPTOMCAT
${projectPath}/bin/shutdown.sh
sleep 3
rm -rf ${projectPath}/webapps/*
sleep 3
~~~

使用tomcat的shutdown.sh脚本关闭运行tomcat进程的时候，经常会无法关闭，可以写一个自定义的脚本来强制关闭进程。

* 使用maven指定profile打包：

~~~
clean install -PdevB -Dmaven.test.skip=true -U
~~~

* 启动：

~~~
BUILD_ID=STARTTOMCAT
cp /root/.jenkins/workspace/${project}/target/${projectName}.war ${projectPath}/webapps/
${projectPath}/bin/startup.sh
sleep 10
~~~

# tips
* 使用tomcat启动web服务时去除项目路径
在使用tomcat启动打包好的war包之后，请求的url路径中会包含项目名称，如果需要去掉项目名称，可以把打包好的war包重命名为ROOT.war，然后再通过tomcat的startup.sh脚本启动。由于Tomcat默认的启动路径是ROOT，所以启动后的web服务的请求路径中就不会再包含项目名称

* 修改端口号
当同一台服务器上部署多台web服务的时候，为了避免端口号冲突，需要修改tomcat的启动端口号：修改conf/server.xml中的端口号配置

* 页面中文乱码
在conf/server.xml中添加编码格式：

~~~
<Connector executor="tomcatThreadPool"
        port="8080" protocol="HTTP/1.1"
        connectionTimeout="20000"
        redirectPort="8443"
        URIEncoding="GBK" />
~~~

* 部署git项目时出现权限错误

~~~
stderr: error: The requested URL returned error: 401 Unauthorized while accessing
~~~
原因是git版本太低了（使用了1.7.0），需要升级git版本

参考：http://blog.csdn.net/aeolus_pu/article/details/71628363 