---
layout: post
title:  "Freemarker配置与使用"
date:   2017-12-09 15:56:20 +0800
categories: 基础
tags: web
---

* TOC
{:toc}

又要做页面了，这次决定好好学习前端相关的东西，认真做个好看的页面。上次做会议预定系统（manage）的时候用过的Freemarker现在已经忘的差不多了，这次从头到尾再总结一次。
这次的页面使用了Bootstrap+Freemarker，示例代码在job-scheduler的工程中。

# 准备工作
要使用Freemarker做模板引擎，需要首先在项目中添加依赖：
~~~
            <dependency>
                <groupId>org.freemarker</groupId>
                <artifactId>freemarker</artifactId>
                <version>2.3.23</version>
            </dependency>
~~~
然后配置试图解析器
~~~
    <bean  id="viewResolver" class="org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver">
        <property name="suffix" value=".ftl" />
        <property name="contentType" value="text/html;charset=UTF-8" />
        <property name="requestContextAttribute" value="rc" />
    </bean>

    <bean id="freeMarkerConfigurer" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
        <property name="configuration"  ref="configuration"/>
    </bean>
    <bean id="configuration" class="freemarker.template.Configuration">
        <property name="directoryForTemplateLoading" value="WEB-INF/view"/>
        <property name="defaultEncoding" value="UTF-8"/>
        <property name="templateExceptionHandler" value="RETHROW_HANDLER"/>
        <property name="logTemplateExceptions" value="false"/>
        <property name="objectWrapper" value="BEANS_WRAPPER"/>
    </bean>
~~~

# 配置模板
1. 通过宏变量创建一个默认布局
参考文档：http://freemarker.foofun.cn/ref_directive_macro.html

宏变量
示例：
~~~
<#macro test>
  Test text
</#macro>
<#-- call the macro: -->
<@test/>
~~~
输出内容为：`Test text`

~~~
<#macro test foo bar baaz>
  Test text, and the params: ${foo}, ${bar}, ${baaz}
</#macro>
<#-- call the macro: -->
<@test foo="a" bar="b" baaz=5*5-2/>
~~~
输出内容为： ` Test text, and the params: a, b, 23`
nested
nested 指令执行自定义指令开始和结束标签中间的模板片段
根据Bootstrap的示例代码，我选择了适合我的一个示例代码，然后选定header和footer公共的代码，分别放置在两个文件中：header.ftl,footer.ftl
后续写页面的时候，对公用的代码就可以直接引入这两个文件即可。



