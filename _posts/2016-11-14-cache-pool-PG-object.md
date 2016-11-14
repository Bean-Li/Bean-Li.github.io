---
layout: post
title: Cache Tier中的两个水位控制
date: 2016-11-12 14:43:40
categories: ceph
tag: ceph
excerpt: 介绍Cache Tier中的两个水位控制项
---

#前言

Ceph提供了Cache Tier的功能，这个功能和纠删码配合使用，可以提高空间利用率。Cache Tier将两个Pool融合在一起，其中容量小，性能高的设备组成cachepool，而容量大，性能低的设备组成data pool。cache pool作为 data pool 缓存而存在，通过少量的高性能的设备，提升整体性能。 Cache Tier对于存在明显 “热”数据集合的场景比较适合，但是对于不存在明显热数据集的场景，性能并不高。



# 原理

Cache pool，作为缓存，容量是有限的。我们指定了cache\_target\_max_\bytes就指定了cache pool的最大空间。在此基础上，存在两个参数控制cache pool的水位：

* cache\_target\_full\_ratio
* cache\_target\_dirty\_ratio

注意，这两个水位，是对PG生效的。比如Cache Pool占用40T的空间，而pool中有1024个PG的话， 平均下来，每个PG分摊到40G的空间，如果底层对象大小 object size 为4MB，那么平均每个PG最多存放10240个对象。

如果cache\_target\_full\_ratio  = 0.8 ，那么cache pool如果某个PG已经存在了10240*0.8 ＝ 8192个对象，表示已经到了full的水位。
如果cache\_target\_dirty\_ratio = 0.4 ,那么cachepool中，如果某个PG脏的对象个数达到了4096个，表示已经到达了dirty水位，再有dirty的内容进来，就应该flush已有内容了。

对于cache pool而言，如果dirty的内容超过了dirty_ratio或者对象总数超过了full ratio，读写性能会有显著的下降。其实关键点在于cache pool的缓存替换算法做的比较初级，因此，Cache Tier的效果，并没有想象的那么好。


# 如果获得统计信息 

为了定位问题，我们可能需要知道，对于Cache Pool中的某个PG，到底有多少个对象在cache pool中，到底有多少个是dirty的对象。

如下命令可以提供帮助。

```
ceph pg x.xx query
```

如果想统计cache pool中所有PG 对象的个数和dirty 对象的个数，可以用如下指令：

以pool id 为16 ，pgnum ＝ 1024 为例：

```
for i in {0..1023} ; do pgid=$(printf %x $i) ;echo 16.$pgid |tee -a pg_dirty_statis.log2;  ceph pg 16.$pgid  query |grep -m2   -Ei "num_objects\"|num_objects_dirty"  |tee -a  pg_dirty_statis.log2  ;  done 

```
命令的输出如下：

```

16.0
              "num_objects": 5193,
              "num_objects_dirty": 5183,
16.1
              "num_objects": 5164,
              "num_objects_dirty": 4281,
16.2
              "num_objects": 5310,
              "num_objects_dirty": 5304,
16.3
              "num_objects": 5033,
              "num_objects_dirty": 5025,
16.4
              "num_objects": 4961,
              "num_objects_dirty": 4953,
16.5
              "num_objects": 4993,
              "num_objects_dirty": 4985,
16.6
              "num_objects": 5040,
              "num_objects_dirty": 4279,
16.7
              "num_objects": 9458,
              "num_objects_dirty": 7264,
16.8
              "num_objects": 9698,
              "num_objects_dirty": 4271,
16.9
              "num_objects": 8999,
              "num_objects_dirty": 4327,



```

从 pg query中还可以得到flush 和evict的状态信息，如果超过水位，可以看到PG的对应状态是active，否则是idle。