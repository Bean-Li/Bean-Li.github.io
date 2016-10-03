---
layout: post
title: 给网络注入点延迟
date: 2016-10-03 11:52:30
categories: linux	
tag: linux
excerpt: 人为引入网络延迟的方法
---


＃ 前言

有些时候，需要人为引入网络延迟，模拟网络条件比较差的情况下，集群的行为。这时候，tc这个工具就横空出世了。
tc是 Traffic Control的简写，用来控制网络流量的。它的功能还是很强大的，我们本文仅简单介绍如何惹人为地注入网络延迟。


# 注入延迟

注入延迟之前，看下网路延迟：

```
root@BEAN-1:~# ping 10.10.125.1
PING 10.10.125.1 (10.10.125.1) 56(84) bytes of data.
64 bytes from 10.10.125.1: icmp_req=1 ttl=64 time=0.178 ms
64 bytes from 10.10.125.1: icmp_req=2 ttl=64 time=0.090 ms
64 bytes from 10.10.125.1: icmp_req=3 ttl=64 time=0.270 ms
64 bytes from 10.10.125.1: icmp_req=4 ttl=64 time=0.156 ms
64 bytes from 10.10.125.1: icmp_req=5 ttl=64 time=0.145 ms
^C
--- 10.10.125.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 3998ms
rtt min/avg/max/mdev = 0.090/0.167/0.270/0.061 ms

```

给eth0 注入8ms的延迟：

```
tc qdisc add dev eth0 root netem delay 8ms
```

然后通过ping查看是否生效：

```
root@BEAN-1:~# ping 10.10.125.1
PING 10.10.125.1 (10.10.125.1) 56(84) bytes of data.
64 bytes from 10.10.125.1: icmp_req=1 ttl=64 time=8.17 ms
64 bytes from 10.10.125.1: icmp_req=2 ttl=64 time=8.17 ms
64 bytes from 10.10.125.1: icmp_req=3 ttl=64 time=8.18 ms
64 bytes from 10.10.125.1: icmp_req=4 ttl=64 time=8.28 ms
64 bytes from 10.10.125.1: icmp_req=5 ttl=64 time=8.13 ms
```

可以看到网络延迟增加了8ms左右。

# 查看当前的延迟信息

```
tc qdisc show dev eth1 
qdisc netem 8121: root refcnt 2 limit 1000 delay 8.0ms
```

# 改变网络延迟大小

注意，如果已经引入了延迟，但是要修改延迟大小，使用add就不行了，要用change

```
tc qdisc add  dev eth1 root netem delay 10ms
RTNETLINK answers: File exists

tc qdisc change  dev eth1 root netem delay 10ms

tc qdisc show dev eth1 
qdisc netem 8121: root refcnt 2 limit 1000 delay 10.0ms

```

或者我们不想区别第一次和修改，可以一律使用replace

```
删除引入的网络延迟
tc qdisc del dev eth1 root netem delay 1ms

显示当前的状态，无延迟
tc qdisc show dev eth1 
qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1

＃replace设置延迟为9ms
tc qdisc replace  dev eth1 root netem delay 9ms

# 显示当前状态，的确是delay 9ms
tc qdisc show dev eth1 
qdisc netem 8123: root refcnt 2 limit 1000 delay 9.0ms

# 用replace来修改网络延迟为4ms
tc qdisc replace  dev eth1 root netem delay 4ms

# 显示当前网络延迟，的确是4ms
tc qdisc show dev eth1 
qdisc netem 8123: root refcnt 2 limit 1000 delay 4.0ms

```


# 删除引入的延迟

```
tc qdisc del dev eth1 root netem delay 1ms

```

# 其他

我们在测试脚本中引入延迟，但是须知，脚本可能会被信号杀死，我们希望，脚本退出的时候，网络延迟被重置为0，这时候，对于shell脚本而言，需要使用trap，做清理工作。

```
function clean
{
   #此函数为清理函数，一般是将人为引入的网络延迟删除
}

trap clean EXIT

```