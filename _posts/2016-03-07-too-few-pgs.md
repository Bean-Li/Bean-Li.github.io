---
layout: post
title: pool has too few pgs appears in ceph health
date: 2016-03-07 13:12:40
categories: ceph
tag: ceph
excerpt: 执行ceph health，输出中有 pool xxx has too few pgs，what does it mean？
---


现象
------

在客户的集群环境中出现了如下输出：

```
   ceph health detail
   HEALTH_WARN pool volume-flashcache-xxx has too few pgs; noout flag(s) set
   pool volume-flashcache-xxx objects per pg (3290) is more than 10.2174 times cluster average (322)
   noout flag(s) set
```

这个has too few pgs到底是何意？ 是否有危害？

原因
------
从objects per pgs is more than 10.2174 times cluster average 不难看出，对于pool volume-flashcache-xxx而言，该pool上的每个PG有3290个object，超过了集群的平均值的10倍。

不难推出，这个警告的含义是，绝大多数的数据，即object都分布在pool volume-flashcache-xxx，而其他pool只有少量的object。

从ceph df的输出不难验证这一点：

```
       GLOBAL:
           SIZE     AVAIL     RAW USED     %RAW USED 
           674T      619T       25688G          3.72 
       POOLS:
           NAME                           ID     USED       %USED     MAX AVAIL     OBJECTS 
           rbd                            0           8         0          302T           1 
           .ezs3                          1          78         0          302T           1 
           data                           2         175         0          302T        1287 
           metadata                       3      41687k         0          302T         825 
           .rgw.buckets                   4           0         0          302T           0 
           .ezs3.central.log              5      24509k         0          302T       10008 
           .ezs3.statistic                6      33561k         0          302T         532 
           .rgw.root                      7         854         0          302T           3 
           .rgw.control                   8           0         0          302T           8 
           .rgw                           9        1266         0          302T           6 
           .rgw.gc                        10          0         0          302T          32 
           .users.uid                     11       4167         0          302T           8 
           .users.email                   12        133         0          302T           7 
           .users                         13        133         0          302T           7 
           .users.swift                   14        257         0          302T          13 
           volume-flashcache-langfang     15     12883G      1.87          269T     3368949
           volume-sata-langfang           16     10980M         0        36796G        2782 
           volume-ssd-langfang            17     11014M         0          345G        2780 
           .usage                         18          0         0          302T           3 
           .rgw.buckets.index             19          0         0          302T           4 
 
```

可以看出，几乎所有的数据都位于volume-flashcache-xxx 中，因此会存在这样一条警告。事实上，存在一个控制参数来控制该条警告

```
   ceph --admin-daemon /var/run/ceph/ceph-mon.vdxys.asok config show |grep skew
   "mon_pg_warn_max_object_skew": "10",
```

只要某个pool中objects per pg的平均值大于集群的objects per pg，就会有此警告。

代码
-----

相关的代码位于src/mon/PGMonitor.cc

```
      int average_objects_per_pg = pg_map.pg_sum.stats.sum.num_objects / pg_map.pg_stat.size();
      if (average_objects_per_pg > 0 &&
          pg_map.pg_sum.stats.sum.num_objects >= g_conf->mon_pg_warn_min_objects &&
          p->second.stats.sum.num_objects >= g_conf->mon_pg_warn_min_pool_objects) {
	int objects_per_pg = p->second.stats.sum.num_objects / pi->get_pg_num();
	float ratio = (float)objects_per_pg / (float)average_objects_per_pg;
	if (g_conf->mon_pg_warn_max_object_skew > 0 &&
	    ratio > g_conf->mon_pg_warn_max_object_skew) {
	  ostringstream ss;
	  ss << "pool " << name << " has many more objects per pg than average (too few pgs?)";
	  summary.push_back(make_pair(HEALTH_WARN, ss.str()));
	  if (detail) {
	    ostringstream ss;
	    ss << "pool " << name << " objects per pg ("
	       << objects_per_pg << ") is more than " << ratio << " times cluster average ("
	       << average_objects_per_pg << ")";
	    detail->push_back(make_pair(HEALTH_WARN, ss.str()));
	  }
	}
      }
```

取消警告
--------
看完代码，可以看出该警告并不是什么严重的问题，仅仅是用户对pool的使用并不均匀引起的。如果用户将其所有的数据放在一个pool中，就会引起此警告。我们不妨取消该警告。

```
	mon pg warn max object skew = 0
```