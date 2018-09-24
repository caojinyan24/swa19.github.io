---
layout: post
title:  "Http请求和响应"
date:   2018-01-14 17:28:11 +0800
categories: 基础
tags: java
---

* TOC
{:toc}

前几天遇到个问题,在一个需求中,需要通过Http调用第三方的接口,这个接口指定了请求类型为POST请求,请求接口为
![](/_pic/201801/httpReq.png)

由于之前已经写过类似的Http请求,这次可以直接服用原有的HttpClient的配置,代码如下:

~~~
@PostConstruct
public void setUp() {
   AsyncHttpClientConfig.Builder builder = new AsyncHttpClientConfig.Builder();
   builder.setConnectTimeout(50000);
   builder.setRequestTimeout(50000);
   builder.setAllowPoolingConnections(true);
   builder.setCompressionEnforced(true);
   builder.setPooledConnectionIdleTimeout(3 * 60 * 1000);
   client = new AsyncHttpClient(builder.build());

}
@Override
public RespDto request(ReqDto reqDto) {
   RespDto respDto = new RespDto();
   try {
       String url = postLoanAddress + CASE_SYNC_API;
       AsyncHttpClient.BoundRequestBuilder requestBuilder = client.preparePost(url);
       requestBuilder.addQueryParam("time", DateUtil.date2Str(reqDto.getSyncTime(), "yyyy-MM-dd HH:mm:ss"));
       requestBuilder.addQueryParam("total", reqDto.getSyncCount().toString());
       requestBuilder.addQueryParam("data", JSON.toJSONString(reqDto.getSyncData()));
       requestBuilder.setMethod("POST");
       Request request = requestBuilder.build();
       ListenableFuture<Response> responseFuture = client.executeRequest(request);
       Response response = responseFuture.get();
       if (!Strings.isNullOrEmpty(response.getResponseBody())) {
           caseSyncRespDto = JSON.parseObject(response.getResponseBody(), RespDto.class);
       }
   } catch (Exception e) {
       logger.error("request:", e);
       throw new Exception("处理失败,请稍后再试");
   }
   return respDto;
}
~~~

这里就出现了个问题,当data的长度比较短的时候,请求可以发送,且接收者能接收到请求;但长度再长(数组长度为100)的话,接收者无法接收到请求:
![](/_pic/201801/untitled.png)

如图,返回错误码为400.
首先怀疑是发送的请求参数的问题,然后查看BoundRequestBuilder的API,发现有个方法`setBody()`,抱着试试的想法,把200条数据放在了body里并发送,得到了正确的响应结果:200.网上查这两个API有什么区别,没查想要的结果,结果在翻看搜索结果的时候看到GET和POST请求的区别,突然想到,GET请求是把请求参数直接拼在请求路径后,而POST则是把数据放入body中,而且这个body的大小不受限制.这样看来,这个QueryParam就是拼接在路径后的参数,而body就是放在请求体的数据了.

正常来说,在SpringMVC中,可以通过@RequestParam注解,直接获取路径后的key-value参数,那么POST请求的请求数据如何获取呢?
我做了几个测试方法

1. key-value形式的请求可通过springmvc直接映射到实体类中:
通过浏览器访问:`http://127.0.0.1:8082/test?syncTime=20170101120000&syncCount=1&syncData=3`
~~~
@RequestMapping("/test")
    @ResponseBody
    public String index(CaseSyncReqDto body) {
        logger.info("body:{}", body);
        Map<String, Object> result = Maps.newHashMap();
        result.put("code", 1);
        result.put("msg", "c成功");
        return JSON.toJSONString(result);
    }
~~~
日志显示:`19:40:34.254 [http-nio-8082-exec-1] INFO swa.controller.Index - body:CaseSyncReqDto{syncTime='20170101120000', syncCount=1, syncData='3'}`
也就是说,对于key-value的格式,spring可以自转成对象,但是这里的syncTime是String类型的,通过Spring注解`@DateTimeFormat`可以将String类型自动转化为Date类型

2. 如果通过POST请求发送的数据并不是key-value形式的,此时Spring将无法完成自动转换.

~~~
@RequestMapping("/../syncCollector")
@ResponseBody
public String index(@RequestBody CaseSyncReqDto body) {
    logger.info("param:{},{},{}", time, total, data);
    logger.info("body:{}", body);
    Map<String, Object> result = Maps.newHashMap();
    result.put("code", 1);
    result.put("msg", "c成功");
    return JSON.toJSONString(result);
}
~~~
此时发送方收到错误响应码415,将doby的类型改为String类型,此时发送方可得到正确的响应码200.
也就是说,对于非key-value的请求,SpringMVC无法对接收到的请求做自动转换.

网上找到的一个POST请求的content-type:
1. application/x-www-form-urlencoded
POST最常见的提交数据的方式,浏览器的原生 form 表单，如果不设置 enctype 属性，那么最终就会以 application/x-www-form-urlencoded 方式提交数据.提交的数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行了 URL 转码.用 Ajax 提交数据时，也是使用这种方式。例如 JQuery 和 QWrap 的 Ajax，Content-Type 默认值都是「application/x-www-form-urlencoded;charset=utf-8」

2. multipart/form-data
一般用来上传文件

3. application/json
4. text/xml

# 参考
[post接口提交参数方式](http://blog.csdn.net/muier/article/details/50946848)
