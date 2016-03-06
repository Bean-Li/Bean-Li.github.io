---
layout: post
title: Get PG distribution on Specific Pool
date: 2016-03-06 19:41：30
categories: ceph
---

前言
-----
尽管ceph可以提供出多个pool，很多情况下，客户的所有的业务数据位于一个pool内。这种情况下，主要使用的pool的PG分布是否均匀，就决定了各个OSD的使用是否均匀。

CRUSH算法让PG均匀地分布在各个OSD上，但这仅仅是概率上的平均，实际使用中，经常发生不均匀的情况，甚至出现非常不均匀的情况。

我们以pg_num = 1024的双副本pool为例，对于该pool，一共有2048个PG，这2048个PG分布在OSD上。如果Pool中有10个OSD，那么平均每个OSD应该分到204～205个PG，但是CRUSH算法不是round-robin，无法做到这么平均。

如果OSD.X分到最多的PG，240个，而OSD.Y分到最少的PG 180个，那么会有更多的数据写入OSD.X，这种现象对带来两个弊端：

＊ 业务压力并不平衡，极端情况下，因为OSD.X先到性能瓶颈而影响整个集群的性能

＊ 长此以往，磁盘使用率不均匀，当OSD.X使用率到达95%而不能再写入时，OSD.Y的使用率才到71%

因此，知道Pool上PG在OSD上的分布，在不均匀的情况下采取必要措施是十分必要的。




已有的研究
---------
普通人的菜园子在 PG在OSD里统计工具---pgdump 一文中给出了统计各个pool中PG分布的代码，
CephNotes 也有一篇文章是写这个主题的。

我最初的方法也是这两篇文章提供的思路实现的，直到我看到这个Sage提供的方法。

https://www.spinics.net/lists/ceph-devel/msg26336.html



步骤
------
* 获取OSDMap

```
	ceph osd getmap -o om    
```

* 获取crushmap

```
	osdmaptool om --export-crush cm 
	或者 
	ceph ods getcrushmap -o cm
```

* 获取指定pool上PG的分布

```
	osdmaptool om --import-crush cm --test-map-pgs --pool 2
```

下面以一个实际中的pool为例，查看PG在OSD的分布情况：

```
root@host1:~/bean# osdmaptool om --import-crush cm --test-map-pgs --pool 2
osdmaptool: osdmap file 'om'
osdmaptool: imported 669 byte crush map from cm
pool 2 pg_num 1024
#osd	count	first	primary	c wt	wt
osd.0	224	138	138	0.00563049	1
osd.1	304	139	139	0.00563049	1
osd.2	302	137	137	0.00563049	1
osd.3	223	121	121	0.00563049	1
osd.4	315	131	131	0.00563049	1
osd.5	317	140	140	0.00563049	1
osd.6	363	218	218	0.00947571	1
 in 7
 avg 292 stddev 47.5455 (0.162827x) (expected 15.8359 0.0542325x))
 min osd.3 223
 max osd.6 363
size 0	0
size 1	0
size 2	1024
size 3	0
osdmaptool: writing epoch 144 to om
```

不难看出，PG分布相当的不均匀，osd.6上的PG数最多 363个PG，而osd.3上PG数最少，只有223，平均下来，osd.6的负载要是osd.3负载的1.5倍以上，同时当osd.6使用率超过90时，osd.3的使用率还不到60％。后期必然会受到客户的质疑。

