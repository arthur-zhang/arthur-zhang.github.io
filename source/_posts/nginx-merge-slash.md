---
layout: post
title:  "一次Nginx双斜杠问题排查"
date:   2018-09-11 16:11:18
categories: Nginx
---

# 一次 Nginx 双斜杠问题排查

### 背景
前段时间出现了一个请求在测试环境签名成功，在线上环境签名失败的情况，排查原因是线上`url`中有双斜杠会被合并成一个传给后端，在测试环境中不会出现。这个就比较神奇了，Nginx 版本完全一样。

### 确认问题
方式是抓包确认：在线上`Nginx`和测试`Nginx`抓包，对比
以下例子中
`218.218.218.218`是线上服务器` Nginx`的`ip`
`121.121.121.121`是自己电脑出口`ip`
`10.0.0.1`是线上`Nginx`的局域网`ip`
`10.0.0.2`是Java业务机的局域网ip

```
1. 从自己电脑到线上Nginx的包如下：

17:41:47.110728 IP 121.121.121.121.50935 > 218.218.218.218.80: Flags [P.]
GET /easicare/v1//subCourses/9952078022974031963e5d9a399e9958/text?subCourseId=9952078022974031963e5d9a399e9958 HTTP/1.1
Host: masaike.seewo.com
User-Agent: curl/7.54.0
Accept: */*

2. Nginx到后端的请求如下

17:41:47.113138 IP 10.0.0.1.49610 > 10.0.0.2.40088: Flags [P.]

GET /easicare/v1/subCourses/9952078022974031963e5d9a399e9958/text?subCourseId=9952078022974031963e5d9a399e9958 HTTP/1.1
x-ccloud-pre: 1
X-Forwarded-Url: http://masaike.seewo.com/easicare/v1//subCourses/9952078022974031963e5d9a399e9958/text?subCourseId=9952078022974031963e5d9a399e9958
Host: masaike.seewo.com
X-Real-IP: 121.121.121.121
X-Forwarded-For: 121.121.121.121
X-Forwarded-Proto: http
User-Agent: curl/7.54.0
Accept: */*
```

可以看到`Nginx`转发到后端`Java`这里的时候，`/easicare/v1//subCourses/`已经没有两个斜杠了，但是测试环境转到后端的时候是有的，这里就不贴包内容了。

自己在本地测试了很久，发现都不会合并多余的`/`，`google`了很久也没有答案，只能最后一个方案，`debug`一下`Nginx`的源码

环境：`Mac+Clion`
最终跟进了代码：`src/http/modules/ngx_http_proxy_module.c`的`ngx_http_proxy_create_request`函数

下面这段代码是生成转发给`upstream`的`http`包
```
// 拷贝'GET '
b->last = ngx_copy(b->last, method.data, method.len);
*b->last++ = ' ';

u->uri.data = b->last;

// 拷贝uri，核心差别就在这里
// 如果unparsed_uri=1,url部分就使用unparsed_uri.data,就是没有合并斜杠的url
// 如果unparsed_uri=0,url部分就使用uri.data,就是合并过斜杠的url

if (plcf->proxy_lengths && ctx->vars.uri.len) {
    b->last = ngx_copy(b->last, ctx->vars.uri.data, ctx->vars.uri.len);
} else if (unparsed_uri) {
    // 如果unparsed_uri=1，url使用unparsed_uri.data
    b->last = ngx_copy(b->last, r->unparsed_uri.data, r->unparsed_uri.len);

} else {
    if (r->valid_location) {
        b->last = ngx_copy(b->last, ctx->vars.uri.data, ctx->vars.uri.len);
    }

    if (escape) {
        ngx_escape_uri(b->last, r->uri.data + loc_len,
                       r->uri.len - loc_len, NGX_ESCAPE_URI);
        b->last += r->uri.len - loc_len + escape;

    } else {
        // 如果unparsed_uri=0，url使用uri.data，uri.data是合并过的url
        b->last = ngx_copy(b->last, r->uri.data + loc_len,
                           r->uri.len - loc_len);
    }

    // 这里是拼接querystring
    if (r->args.len > 0) {
        *b->last++ = '?';
        b->last = ngx_copy(b->last, r->args.data, r->args.len);
    }
}

```

那么`unparsed_uri`这个标记位怎么来的?
`ctx->vars.uri.len == 0` 的情况下会置位`1`，`vars.uri`的值的含义是Nginx配置文件中`proxy_pass server`后面那段
比如`proxy_pass http://my-tomcat-server;`那么`vars.uri`值是`NULL`
比如`proxy_pass http://my-tomcat-server/nimei;`那么`vars.uri`值是`/nimei`

![](/images/nginx-merge-slash-1.jpg)

```
unparsed_uri = 0;

if (plcf->proxy_lengths && ctx->vars.uri.len) {
    uri_len = ctx->vars.uri.len;

} else if (ctx->vars.uri.len == 0 && r->valid_unparsed_uri && r == r->main)
{
    // ctx->vars.uri.len == 0 的情况下会置位1
    unparsed_uri = 1;
    uri_len = r->unparsed_uri.len;

} else {
    loc_len = (r->valid_location && ctx->vars.uri.len) ?
                  plcf->location.len : 0;

    if (r->quoted_uri || r->space_in_uri || r->internal) {
        escape = 2 * ngx_escape_uri(NULL, r->uri.data + loc_len,
                                    r->uri.len - loc_len, NGX_ESCAPE_URI);
    }

    uri_len = ctx->vars.uri.len + r->uri.len - loc_len + escape
              + sizeof("?") - 1 + r->args.len;
}
```

回过来看这个问题，就很简单了

```
线上环境配置
location /easicare {
     proxy_pass http://easicare/easicare;
     
测试环境配置
location / {
     proxy_pass http://nginx-ingress;     
```

线上配置`server`后面多了`/easicare`，会走`unparsed_uri=0`的逻辑，会使用合并过/的url，测试环境`server`后面是空的，会走`unparsed_uri=1`的逻辑，会不合并url

还有一个问题，`merge_slashes`这个指令有什么用？merge_slashes这个指令默认是开的，会决定会不会自动合并uri中的/，决定了uri这个基础，会不会有第一步合并这一步


| merge_slashes  | proxy_pass后无url | 最终转发url是否合并/ |
| --- | --- | --- |
| on | 有 | 合并了/ |
| on | 无 | 没有合并/ |
| off |有  | 没有合并/ |
| off |无  |  没有合并/|


这个问题看起来表面上只影响了双斜杠的问题，实际上，很多地方都有影响，比如刚刚好，线上又出现了一起问题
请求是： /easicar/v1/subCourses/{subCourseId}/comments/create
因为前端问题{subCourseId}，没有用值覆盖它，在线上不正常，在测试环境正常
还是因为那个问题
实验结果如下


```
1. server 后面有内容的时候 (模拟线上情况)
proxy_pass http://my-tomcat-server/nimei 

客户端请求到 Nginx
19:03:01.396763 IP 127.0.0.1.61759 > 127.0.0.1.8080: Flags [P.]
POST /apm-demo-server/%7Bfoo%7D//bar HTTP/1.1
Content-Type: text/plain; charset=utf-8

Nginx请求到upstream
19:03:01.398280 IP6 ::1.61760 > ::1.8111: Flags [P.]
POST /nimei/{foo}/bar HTTP/1.1
X-Forwarded-Url: http://ya-dev.test.seewo.com/apm-demo-server/%7Bfoo%7D//bar

可以看到转发到后端服务器那里的时候已经是解码过的{foo}

2. server 后面没有内容的时候 (模拟测试环境)

proxy_pass http://my-tomcat-server;

客户端请求到 Nginx
19:16:37.949701 IP 127.0.0.1.62054 > 127.0.0.1.8080: Flags [P.]
POST /apm-demo-server/%7Bfoo%7D//bar HTTP/1.1

Nginx请求到upstream
19:16:37.953191 IP6 ::1.62055 > ::1.8111: Flags [P.]
POST /apm-demo-server/%7Bfoo%7D//bar HTTP/1.1
X-Forwarded-Url: http://ya-dev.test.seewo.com/apm-demo-server/%7Bfoo%7D//bar
可以看到转发到后端服务器那里的时候已经是未解码过的%7Bfoo%7D
```

