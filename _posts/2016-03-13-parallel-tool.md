---
layout: post
title: user parallel to exec command on multiple machine
date: 2016-03-13 01:06:40
categories: linux
tag: linux
excerpt: 测试分布式存储时，经常需要将任务分解到多台机器之行。
---

前言
-----
我从事的是分布式存储的研发，在日常工作中，经常需要测试存储的性能。

我个人认为，单机版的Linux，生产效率最高的工具是xargs。

比如我想测试400路并发写10G的大文件，一共写入2000个文件

```
seq 1 2000 |xargs -P 400 -I {} dd if=/dev/zero of=file_{} bs=1M count=10240 oflag=direct
 
```

xargs 参数中 -P 选项相当逆天，保持400路并发。

但是如果需要多台测试机，比如多台client机通过NFS测试写入，怎么办？其实我们需要的是将xargs 逆天的能力扩展到多台机器。

parallel作为主角，是时候登场了

安装步骤
-------
1  安装parallel deb包

```
	sudo apt-get install parallel
```

2 修改parallel的配置文件

简单地说，就是删除--tollef

```
cat /etc/parallel/config
--tollef
```

3 测试可用性

可以用简单的sleep命令来测试parallel的可用性：

```
seq 1 100 ｜parallel -j 2 -S 10.16.17.17 -S 10.16.17.169 sleep {}
```

参数－j表示每次分发几个job，比如本例子中，每次发两个任务，即每台机器上会有两个sleep任务在跑。

```
10.16.17.17 zsh上的输出
➜  ~ ps ax|grep sleep
 8389 ?        Ss     0:00 sleep 44
 8629 ?        Ss     0:00 sleep 45

10.16.17.169  bash上的输出
manu@NJ-BUILDMAN:~$ ps ax|grep sleep
 2388 ?        Ss     0:00 bash -c eval `echo $SHELL | grep -E "/(t)?csh" > /dev/null  && echo setenv PARALLEL_SEQ 42\;  setenv PARALLEL_PID 3256  || echo PARALLEL_SEQ=42\;export PARALLEL_SEQ\;  PARALLEL_PID=3256\;export PARALLEL_PID` ; sleep 42
 2392 ?        S      0:00 sleep 42
 2535 ?        Ss     0:00 bash -c eval `echo $SHELL | grep -E "/(t)?csh" > /dev/null  && echo setenv PARALLEL_SEQ 43\;  setenv PARALLEL_PID 3256  || echo PARALLEL_SEQ=43\;export PARALLEL_SEQ\;  PARALLEL_PID=3256\;export PARALLEL_PID` ; sleep 43
 2539 ?        S      0:00 sleep 43
 
```

注意节点之间需要设置ssh无需输入密码。
