---
layout: post
title: s3cmd 快速评估RADOSGW的性能
date: 2016-04-23 16:20:40
categories: ceph
tag: ceph
excerpt: s3cmd来测试RADOSGW的性能，其中参数send_chunk默认值4096不适合用来测试性能
---


## 前言

前面介绍了如何使用s3cmd 测试RADOSGW，基本局限在功能测试。近日，有人报CEPH集群 纠删码Pool的S3功能太弱，3节点集群，9个OSD，性能只能达到100MBps左右，因为上传的file大小都在50～100M之间，所以这个值有点低。

客户并没有好的测试手段，用6个Windows客户端，使用CloudBerry拖拽的方式上传object，那集群S3的性能到底如何呢？摆在面前的问题是如何快速的测试RADOSGW的上传性能 100MBps是否合理呢。

我想到了s3cmd这个工具，只要有足够的并发数，不难测试集群的汇聚带宽。

在3个客户端分别连3个存储节点，在客户端上分别执行如下命令：

```
seq 0 9999       | xargs -I{} -P 40 s3cmd put bean s3://bucket/{}
seq 10000 19999  | xargs -I{} -P 40 s3cmd put bean s3://bucket/{}
seq 20000 29999  | xargs -I{} -P 40 s3cmd put bean s3://bucket/{}
```


发现速度很慢：

![](/assets/s3cmd/send_chunk_4K_performance.png)

查看atop信息，系统并不繁忙

![](/assets/s3cmd/send_chunk_4K_atop.png)

磁盘和CPU都没有到达瓶颈，而client到存储节点的网络是万兆，也不是瓶颈，那问题出在哪里呢？

通过strace跟踪一个s3cmd put，看出了端倪，原来上传的buffer是4K。

## 解决办法

s3cmd 提供了send_chunk和recv_chunk这两个选项，默认都是4096字节。毫无疑问，对于传输几十M甚至上G的文件到s3 bucket，这个buffer太小了。

对于测试上传，我将send_chunk调整成了262144字节（256KB），整体性能立刻显著提升，直到达到系统某项设备达到瓶颈：

![](/assets/s3cmd/send_chunk_256K_performance.png)

![](/assets/s3cmd/send_chunk_256K_atop.png)


性能提升了三倍，从atop的信息来看，磁盘已经到达极限。此外因为纠删码会消耗CPU资源，因此，考虑到存储服务器只有8核，因此，CPU也成为了瓶颈。
注意，设个send_chunk也并非是越大越好，太大的send\_chunk会导致s3cmd传输失败而不得不重传，反而降低效率。


## 其它

另外一个值得关注的问题是每个磁盘150MB以下的速度也值得怀疑，从iostat的avgrq-sz看出，基本都在300（sector）以下，即低于150K。这个问题就不在此处讨论了。