---
layout: post
title: apache请求的处理时间
date: 2016-06-11 21:43:40
categories: linux
tag: apache
excerpt: 获取apache请求的处理时间
---

## 前言

有时候，需要了解CGI处理程序的效率，知道各个apache请求的处理事件，为将来进一步优化提供判断的依据。


## 方法

/etc/apache2/apache2.conf中，在

```
LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined

```

在这一行中增加 T%.%D，其中T%提供的是CGI请求处理的事件，单位是秒， D%是请求处理的微秒数，两者结合在一起，提供了以微秒为单位，请求处理时间。

```
LogFormat "%v:%p %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" vhost_combined
LogFormat "%h %l %u %t \"%r\" %>s %O %T.%D \"%{Referer}i\" \"%{User-Agent}i\"" combined
LogFormat "%h %l %u %t \"%r\" %>s %O" common
LogFormat "%{Referer}i -> %U" referer
LogFormat "%{User-agent}i" agent
```

从apache的access log中可以看到如下内容：

```
 10.16.17.43 - - [01/Jun/2016:13:41:51 +0800] "GET /cgi-bin/ezs3/json/list_shared_folder?gateway_group=virStorage&_=1464745951979 HTTP/1.1" 200 1322 0.932141 "https://10.16.17.11:8080/" "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:46.0) Gecko/20100101 Firefox/46.0"
```

## 其他
除了T%和D%外，还有其他可以选择的输出内容：

```

%a    Remote IP-address
%A    Local IP-address
%B    Size of response in bytes, excluding HTTP headers.
%b    Size of response in bytes, excluding HTTP headers. In CLF format, i.e. a '-' rather than a 0 when no bytes are sent.
%{Foobar}C    The contents of cookie Foobar in the request sent to the server.
%D    The time taken to serve the request, in microseconds.
%{FOOBAR}e    The contents of the environment variable FOOBAR
%f    Filename
%h    Remote host
%H    The request protocol
%{Foobar}i    The contents of Foobar: header line(s) in the request sent to the server.
%l    Remote logname (from identd, if supplied). This will return a dash unless mod_ident is present and IdentityCheck is set On.
%m    The request method
%{Foobar}n    The contents of note Foobar from another module.
%{Foobar}o    The contents of Foobar: header line(s) in the reply.
%p    The canonical port of the server serving the request
%P    The process ID of the child that serviced the request.
%{format}P    The process ID or thread id of the child that serviced the request. Valid formats are pid, tid, and hextid. hextid requires APR 1.2.0 or higher.
%q    The query string (prepended with a ? if a query string exists, otherwise an empty string)
%r    First line of request
%s    Status. For requests that got internally redirected, this is the status of the *original* request --- %>s for the last.
%t    Time the request was received (standard english format)
%{format}t    The time, in the form given by format, which should be in strftime(3) format. (potentially localized)
%T    The time taken to serve the request, in seconds.
%u    Remote user (from auth; may be bogus if return status (%s) is 401)
%U    The URL path requested, not including any query string.
%v    The canonical ServerName of the server serving the request.
%V    The server name according to the UseCanonicalName setting.
%X    Connection status when response is completed:
X =   connection aborted before the response completed.
+ =   connection may be kept alive after the response is sent.
- =   connection will be closed after the response is sent.
(This directive was %c in late versions of Apache 1.3, but this conflicted with the historical ssl %{var}c syntax.)
%I    Bytes received, including request and headers, cannot be zero. You need to enable mod_logio to use this.
%O    Bytes sent, including headers, cannot be zero. You need to enable mod_logio to use this.
```

可以参考。
