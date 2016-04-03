---
layout: post
title: ceph pool相关的命令
date: 2016-04-03 22:20:40
categories: ceph
tag: ceph
excerpt: 本文介绍pool相关的一些指令
---

前言
-----
pool 是ceph一个重要的概念，pool是管理硬件存储设备的，同时可以实现存储的差异化。比如说，有些数据要求可靠性高，那么可以创建一个3副本的pool，有些数据访问对性能要求非常高，可以将其创建在性能比较好的硬件设备上。

数据在集群存储设备的分布，遵循一个pool -> pg ->OSD这样一个定位流程。

本文介绍pool相关的一些指令


命令
----

**列出所有的pool**

```
root@node2:~# ceph osd lspools
0 rbd,1 .ezs3,2 data,3 metadata,4 .rgw.buckets,5 .ezs3.central.log,6 .ezs3.statistic,7 .rgw.root,8 .rgw.control,9 .rgw,10 .rgw.gc,11 .users.uid,12 .users.email,13 .users,14 .users.swift,15 .usage,16 .rgw.buckets.index,

root@node2:~# rados lspools
rbd
.ezs3
data
metadata
.rgw.buckets
.ezs3.central.log
.ezs3.statistic
.rgw.root
.rgw.control
.rgw
.rgw.gc
.users.uid
.users.email
.users
.users.swift
.usage
.rgw.buckets.index

```

**查看pool的副本个数**

```
ceph osd pool get data  size 
size: 2
```

对于纠删码，稍微有所不同，因为纠删码是k ＋ m这种类型，因此查看纠删码的k＋m的方法如下：

```
ceph osd pool get ec-pool-1 size
size: 20
ceph osd pool get ec-pool-1 min_size
min_size: 17
```

从上面输出可以看出是17+3的纠删码

**查看pool上pg\_num和pgp\_num**

```
ceph osd pool get data  pg_num
pg_num: 1024

ceph osd pool get data  pgp_num
pgp_num: 1024

```

**修改pg\_num和pgp\_num**

```
ceph osd pool set pool-1 pg_num 2048
ceph osd pool set pool-1 pgp_num 2048
```


**查看pool的ruleset id**

```
root@node2:~# ceph osd pool get data crush_ruleset
crush_ruleset: 0
```

**查看pool的crush ruleset信息**

```
root@node2:~# ceph osd crush rule dump
[
    { "rule_id": 0,
      "rule_name": "data",
      "ruleset": 0,
      "type": 1,
      "min_size": 1,
      "max_size": 10,
      "steps": [
            { "op": "take",
              "item": -1,
              "item_name": "default"},
            { "op": "chooseleaf_firstn",
              "num": 0,
              "type": "host"},
            { "op": "emit"}]},
    { "rule_id": 1,
      "rule_name": "metadata",
      "ruleset": 1,
      "type": 1,
      "min_size": 1,
      "max_size": 10,
      "steps": [
            { "op": "take",
              "item": -1,
              "item_name": "default"},
            { "op": "chooseleaf_firstn",
              "num": 0,
              "type": "host"},
            { "op": "emit"}]},
    { "rule_id": 2,
      "rule_name": "rbd",
      "ruleset": 2,
      "type": 1,
      "min_size": 1,
      "max_size": 10,
      "steps": [
            { "op": "take",
              "item": -1,
              "item_name": "default"},
            { "op": "chooseleaf_firstn",
              "num": 0,
              "type": "host"},
            { "op": "emit"}]}]

```

**查看pool上的实时I/O信息**

```
root@node-01:~# ceph osd pool stats
pool rbd id 0
  nothing is going on
pool data id 2
  nothing is going on
pool metadata id 3
  client io 2778 B/s wr, 1 op/s
...
pool ec-pool id 15
  nothing is going on
pool cache-pool id 16
  client io 444 MB/s wr, 446 op/s

```

**获取pool的空间使用情况**

```
root@node2:~# ceph df
GLOBAL:
    SIZE     AVAIL     RAW USED     %RAW USED 
    270G      261G        9388M          3.38 
POOLS:
    NAME                   ID     USED       %USED     MAX AVAIL     OBJECTS 
    rbd                    0           0         0          130G           0 
    .ezs3                  1           0         0          130G           0 
    data                   2        971M      0.35          130G      223181 
    metadata               3      44605k      0.02          130G      510136 
    .rgw.buckets           4        295k         0          130G           1 
    .ezs3.central.log      5        770k         0          130G         158 
    .ezs3.statistic        6       2964k         0          130G          52 
    .rgw.root              7         840         0          130G           3 
    .rgw.control           8           0         0          130G           8 
    .rgw                   9         349         0          130G           2 
    .rgw.gc                10          0         0          130G          32 
    .users.uid             11        835         0          130G           3 
    .users.email           12         20         0          130G           2 
    .users                 13         20         0          130G           2 
    .users.swift           14         31         0          130G           3 
    .usage                 15          0         0          130G           1 
    .rgw.buckets.index     16          0         0          130G           1 

root@node2:~# rados df
pool name       category                 KB      objects       clones     degraded      unfound           rd        rd KB           wr        wr KB
.ezs3           -                          0            0            0            0           0            0            0            0            0
.ezs3.central.log -                        771          158            0            0           0            0            0          211         1105
.ezs3.statistic -                       2965           52            0            0           0      3015628      2721256      1719478      1400229
.rgw            -                          1            2            0            0           0            0            0            5            2
.rgw.buckets    -                        296            1            0            0           0            9            6           32         1121
.rgw.buckets.index -                          0            1            0            0           0           39           38           21            0
.rgw.control    -                          0            8            0            0           0            0            0            0            0
.rgw.gc         -                          0           32            0            0           0        22678        22678        15233            0
.rgw.root       -                          1            3            0            0           0       339394       203635            5            5
.usage          -                          0            1            0            0           0           21           21           42            0
.users          -                          1            2            0            0           0            0            0            2            2
.users.email    -                          1            2            0            0           0            0            0            2            2
.users.swift    -                          1            3            0            0           0            0            0            3            3
.users.uid      -                          1            3            0            0           0           51           44           21            5
data            -                     995013       223181            0            0           0         8952          127       461125       995026
metadata        -                      44455       510136            0            0           0      1479663     12983152      3209403      4802054
rbd             -                          0            0            0            0           0            0            0            0            0
  total used         9610440       733585
  total avail      274364640
  total space      284073384

```

其他
----
纠删码pool会有更多很有意思的控制参数，
可以通过 ceph osd pool get [pool_name] 来试探出支持的参数。

这些参数如下所示，至于具体的含义和作用，内容比较多，就不再此处列出。

```
root@node2:~# ceph osd pool get data
Invalid command:  missing required parameter var(size|min_size|crash_replay_interval|pg_num|pgp_num|crush_ruleset|hit_set_type|hit_set_period|hit_set_count|hit_set_fpp|auid|target_max_objects|target_max_bytes|cache_target_dirty_ratio|cache_target_full_ratio|cache_min_flush_age|cache_min_evict_age|erasure_code_profile|min_read_recency_for_promote)
osd pool get <poolname> size|min_size|crash_replay_interval|pg_num|pgp_num|crush_ruleset|hit_set_type|hit_set_period|hit_set_count|hit_set_fpp|auid|target_max_objects|target_max_bytes|cache_target_dirty_ratio|cache_target_full_ratio|cache_min_flush_age|cache_min_evict_age|erasure_code_profile|min_read_recency_for_promote :  get pool parameter <var>
Error EINVAL: invalid command
```