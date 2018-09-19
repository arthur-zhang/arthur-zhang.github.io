---
layout: post
title:  "url中出现大括号的有趣问题"
date:   2018-09-19 16:11:18
categories: Nginx
---

# url中出现大括号的有趣问题

### 背景

线上又出现了一起问题
请求是： /easicar/v1/subCourses/{subCourseId}/comments/create
因为node服务的有bug{subCourseId}，没有用值覆盖它，在线上不正常返回400，在测试环境正常（虽然没赋值，后端也没用到subCourseId这个值）
原因还是因为上篇文章讲过的原因：http://arthur-zhang.github.io/2018/09/11/nginx-merge-slash/

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

可以看到`Nginx`这个坑货，在`server`后面有内容的时候，会把`url`解码、合并斜杆后的`url`传给`upstream`服务器，但是`tomcat`会拒绝掉部分未经编码的他觉得不合法的字符。
无论发起方编码的多么好，都会有问题

解决办法有两种：

1. 修改`Nginx`配置
2. 修改`tomcat`配置`Connector`中`relaxedQueryChars`属性，使之支持`{`这些特殊字符


那么哪些是http协议认为的合法的字符呢？
写了一段代码，经过一层`Nginx`，转发到`tomcat`，遍历了0-127的所有字符，以下字符是一定不被`tomcat`允许的

```
" 0x22
< 0x3c
> 0x3e
^ 0x5e
` 0x60
{ 0x7b
| 0x7c
} 0x7d
```

看`tomcat`源码也可以知道

```
if (IS_CONTROL[i] || i > 127 ||
        i == ' ' || i == '\"' || i == '#' || i == '<' || i == '>' || i == '\\' ||
        i == '^' || i == '`'  || i == '{' || i == '|' || i == '}') {
    if (!REQUEST_TARGET_ALLOW[i]) {
        IS_NOT_REQUEST_TARGET[i] = true;
    }
}
```

细心的人可以发现，没有完全对应上，可以留做下一篇文章