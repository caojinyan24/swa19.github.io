---
layout: post
title:  "Html页面中字符的转义"
date:   2018-03-01 16:36:27 +0800
categories: 基础
tags: web
---

* TOC
{:toc}

最近开发任务调度的页面，遇到个问题涉及到Html页面中字符的转义，记录下解决方案
首先后端和页面交互的变量是json格式，可能包含单引号，双引号，也包含大括号，如`{"key":"value"}`,`{'key':'value'}`

# 页面展示
使用jstl函数：`<c:out value="${executeParam}"></c:out>`

传参：`<c:set var="param11" value="${fn:escapeXml(executeParam)}"/>`

通过`escapeXml`函数可以把参数中的Http请求头中不支持的字符做转义，从而使得变量可以通过url做传递

对特殊字符做转义：`url = "url?param=" + encodeURIComponent(param);`

# 代码示例

~~~
<td>
    <input style="height:360px ;width: 360px; " size=65 type="text" name="param"
         id="param"
         value="<c:out value="${param}"></c:out>"/>
</td>
<script>
    function func() {
       var param = document.getElementById("param").value;
        $.ajax({
            type: "POST",
            url: "/url",
            data: {param: param},
            dataType: "text",
            async: true,
            success: function (data) {
                alert(data);
                window.close();
            },
            error: function (data) {
                alert(data);
            }
        })
    }
</script>

~~~

~~~
<c:set var="param11" value="${fn:escapeXml(param)}"/>
<input type="button" onclick="execute('${param11}')">
<script type="text/javascript">
    function execute(param) {
        url = "url?param=" +param;
        WinOpen(url, 14, 'execute');
    }
</script>
~~~



