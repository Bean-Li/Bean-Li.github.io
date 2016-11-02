---
layout: post
title: atop 获取进程退出码或导致退出的信号值
date: 2016-11-02 14:43:40
categories: linux
tag: linux
excerpt:
---

# 前言

Daemon进程凌晨无故退出了，log中没有任何有效信息判断退出的原因。 QA找我确定下退出的原因，是收到信号被杀死，还是自己异常退出了。

幸好有atop,会纪录进程的退出码或者收到的信号值。

# 方法

请看下图：

![](/assets/LINUX/atop_exit.png)

上图中第一行 ＃exit 20305 表示在过去10分钟内，有20305个进程退出了。

其中<ceph-osd>这一行表示，在两个采样时间点中间，ceph-osd退出了，<> 保护的进程表示退出的进程。如何判断它是正常退出，还是收到信号，如果是前者，其返回值是多少，如果是后者，又收到了什么信号呢？

atop中的ST和EXC这两个字段，可以告诉我们答案

```
ST
The status of a process.
The first position indicates if the process has been started during the last interval (the value N means 'new process').
The second position indicates if the process has been finished during the last interval.
The value E means 'exit' on the process' own initiative; the exit code is displayed in the column 'EXC'.
The value S means that the process has been terminated unvoluntarily by a signal; the signal number is displayed in the in the column 'EXC'.
The value C means that the process has been terminated unvoluntarily by a signal, producing a core dump in its current directory; the signal number is displayed in the column 'EXC'.

```

ST第一个字段如果是N表示，新启动了一个进程，关键是ST的第二个字母：

* E 正常退出， EXC的值表示的是退出码
* S 收到了信号
* C 收到了会产生coredump的信号


S和C，表示收到了信号，不得不退出，这时候， EXC字段纪录就是导致进程退出的信号值。

```
EXC
The exit code of a terminated process (second position of column 'ST' is E) or the fatal signal number (second position of column 'ST' is S or C).
```

对于本例， ST＝ NS，表示收到了信号，才导致退出， EXC＝10 表示收到了10号信号。


