---
layout: post
title: 获取apache请求的header信息
date: 2016-03-31 23:20:40
categories: linux
tag: linux
excerpt: 有些时候需要获取web 请求的header信息，来定位问题
---

前言
----
今天某个客户访问我们集群的amazon s3功能，总是返回403错误。我们的售后工程师定位不出来原因，他在用cloudberry访问对应账户的bucket，一切正常，让我帮忙定位一下。

```
127.0.0.1:80 172.20.2.130 - - [31/Mar/2016:11:20:00 +0800] "GET / HTTP/1.1" 403 310 "-" "JetS3t/0.9.0 (Linux/2.6.32-279.el6.x86_64; amd64; zh; JVM 1.7.0_71)"
```

我登录环境,用s3cmd 执行了s3cmd ls，发现一切正常，返回是200，注意，请求是一样的，都是GET ／，但是，s3cmd ls返回200，但是从客户的平台上发过来的请求就是403错误。

分析
----

403到底是何意呢？查了查ceph的文档：

* 403  AccessDenied
* 403	  UserSuspended
* 403  RequestTimeTooSkewed

很明显第一个是AccessDenied，简单地说就是access key和secret key可能不对。但是考虑到这也不是第一次和客户集成，这种可能性并不大。

最明显的当然是client端和ceph 存储节点时间有较大的偏差。之前用cloudberry连接s3时经常因为时间偏差大，导致出错。

问了我们的售后，是不是时间因素，结果售后没有回应，他并没帮我问客户。没法子，只能继续搞。

很明显，如果不单单能看到请求，同时看到请求的header信息，比较s3cmd ls和客户请求的差异，这个问题也就迎刃而解了。问题就演化成了如何过去请求的更多信息。


方法：
----
我们的存储是基于Ubuntu的，用的apache2，可以采用如下方法获取到请求的header信息：

1 修改  ／etc/apache2/apache2.conf

```
LoadModule log_forensic_module  /usr/lib/apache2/modules/mod_log_forensic.so
<IfModule log_forensic_module>
</IfModule>
ForensicLog /var/log/apache2/forensic_log
```

2 重启apache

然后我执行了一遍s3cmd ls，让售后触发了一遍客户的请求，在/var/log/apache2/forensic.log中，看到如下对比内容：

```
+27450:56fcbe22:1|GET / HTTP/1.1|Host:172.20.1.4|Accept-Encoding:identity|content-length:0|Authorization:AWS 31WB0EK0APFRRUYQY7SY%3aFwyI/qxZCd5URuLq4LfyKOGElJE=|x-amz-date:Thu, 31 Mar 2016 06%3a05%3a22 GMT
-27450:56fcbe22:1

+6787:56fcbe88:b|GET / HTTP/1.1|Date:Thu, 31 Mar 2016 14%3a08%3a24 GMT|Authorization:AWS 31WB0EK0APFRRUYQY7SY%3a8Nw7uNfvxopXNtJxrIz2VCe2miI=|Host:172.20.1.7%3a80|Connection:Keep-Alive|User-Agent:JetS3t/0.9.0 (Linux/2.6.32-279.el6.x86_64; amd64; zh; JVM 1.7.0_71)
-6787:56fcbe88:b
```

我几乎是同时发送的请求，在存储节点上执行时间是06:05:22秒，但是从客户那边发过来的请求是14:08:24秒，我在存储节点上执行s3cmd，不会和ceph集群的提供的s3 服务有时间的偏差，但是客户的client端很明显和ceph集群有8个小时左右的偏差，从而导致了403错误。


代码
----

我翻了翻ceph的代码，找到了这个：


```

   #define RGW_AUTH_GRACE_MINS 15

	time_t req_sec = s->header_time.sec();

	if ((req_sec < now - RGW_AUTH_GRACE_MINS * 60 ||
	     req_sec > now + RGW_AUTH_GRACE_MINS * 60) && !qsr) {
	  dout(10) << "req_sec=" << req_sec << " now=" << now << "; now - RGW_AUTH_GRACE_MINS=" << now - RGW_AUTH_GRACE_MINS * 60 << "; now + RGW_AUTH_GRACE_MINS=" << now + RGW_AUTH_GRACE_MINS * 60 << dendl;
	  dout(0) << "NOTICE: request time skew too big now=" << utime_t(now, 0) << " req_time=" << s->header_time << dendl;
	  return -ERR_REQUEST_TIME_SKEWED;
	}
	
	
	const static struct rgw_http_errors RGW_HTTP_ERRORS[] = {
	    { EPERM, 403, "AccessDenied" },
        { ERR_USER_SUSPENDED, 403, "UserSuspended" },
        { ERR_REQUEST_TIME_SKEWED, 403, "RequestTimeTooSkewed" },
    }
```

因此时间偏移正负15分钟，就会返回403错误。