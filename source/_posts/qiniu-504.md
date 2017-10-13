---
layout: post
title:  "七牛上传504问题debug过程分析"
date:   2017-10-13 23:17:43
categories: "tcpdump 网络"
---
# 七牛上传504问题debug过程分析

现象是这样的，我们今天增加了一台内网的业务服务器，这台机器会生成pdf文件通过`up.qbox.me`域名上传到七牛云，但是这台机器上传每次都报504错误

![](/images/qiniu-1.jpeg)
业务流程如下

![](/images/qiniu-2.png)

先抓包看一下：业务机到`nginx`这段

`sudo tcpdump -vvv -i any host 10.10.14.60  -s 1500 -A -nn -w dump.pcap`

![](/images/qiniu-3.jpeg)


wireshark打开看一下，发现都比较正常，除了有60s在等待nginx返回而已

在nginx机器上抓包看一下 

![](/images/qiniu-4.png)


`sudo tcpdump -vvv -i any host 192.254.94.44  -s 1500 -A -nn -w cap.pcap`

发现三次握手成功了以后，nginx机器想建立`ssl`连接，发了一个`client hello`过去，七牛服务器一直没有回应，导致nginx机器一直重传。

这里可以大概看到是`nginx`到七牛云这段有问题，那么为什么有问题呢？

看了一下`192.254.94.44`这个ip，发现是美国的，很不正常对吗，怀疑是dns的锅
我们有两台dns服务器， `172.17.82.12` 和 `172.18.70.5`

分别`dig`看一下

```
第一台dns服务器

dig up.qbox.me @172.17.82.12

; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> up.qbox.me @172.17.82.12
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45131
;; flags: qr rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;up.qbox.me.      IN  A

;; ANSWER SECTION:
up.qbox.me.   32  IN  CNAME upload-z0.qiniukodo.com.
upload-z0.qiniukodo.com. 314  IN  CNAME proxy.qiniukodo.com.
proxy.qiniukodo.com.  314 IN  CNAME default.kodo-acc.vdn.pilidns.com.
default.kodo-acc.vdn.pilidns.com. 314 IN CNAME  default.acc.vdn.pilidns.com.
default.acc.vdn.pilidns.com. 59 IN  A 192.254.94.44

;; Query time: 87 msec
;; SERVER: 172.17.82.12#53(172.17.82.12)
;; WHEN: 五 10月 13 17:27:15 CST 2017
;; MSG SIZE  rcvd: 181


第二台dns服务器

dig up.qbox.me @172.18.70.5

; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> up.qbox.me @172.18.70.5
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1697
;; flags: qr rd ra; QUERY: 1, ANSWER: 12, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;up.qbox.me.      IN  A

;; ANSWER SECTION:
up.qbox.me.   171 IN  CNAME nb-gate-up.qiniu.com.
nb-gate-up.qiniu.com. 171 IN  A 115.238.101.32
nb-gate-up.qiniu.com. 171 IN  A 115.238.101.33
nb-gate-up.qiniu.com. 171 IN  A 115.238.101.34
nb-gate-up.qiniu.com. 171 IN  A 115.238.101.35
nb-gate-up.qiniu.com. 171 IN  A 115.238.101.36
nb-gate-up.qiniu.com. 171 IN  A 115.231.97.57
nb-gate-up.qiniu.com. 171 IN  A 183.131.7.18
nb-gate-up.qiniu.com. 171 IN  A 115.231.97.59
nb-gate-up.qiniu.com. 171 IN  A 115.231.97.46
nb-gate-up.qiniu.com. 171 IN  A 122.224.95.111
nb-gate-up.qiniu.com. 171 IN  A 122.224.95.105

;; Query time: 0 msec
;; SERVER: 172.18.70.5#53(172.18.70.5)
;; WHEN: 五 10月 13 17:30:37 CST 2017
;; MSG SIZE  rcvd: 249
```

看到 `172.17.82.12` 这台dns服务器解析`up.qbox.me`域名到了一个美国地址。可能IT那边有什么特殊规则，跟IT联系，说明哪台服务器有问题，哪个域名解析有问题，马上就解决了。


