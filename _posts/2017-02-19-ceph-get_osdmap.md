---
layout: post
title: Get osdmap at specific epoch 
date: 2017-02-19 11:43:40
categories: ceph
tag: ceph
excerpt: 获取指定epoch的OSDMap
---

# 前言

开始分析Ceph PG的状态迁移，其中OSD的变化，牵动OSDMap的变化， 进而引起PG状态的迁移。

因此第一步，我们要通过现场环境来分析状态切换，需要取到各个epoch的OSDMap


# 获取当前OSDMap的epoch

ceph osd stat命令可以获得当前OSD的epoch：

```
root@BEAN-1:~# ceph osd stat
     osdmap e540: 3 osds: 3 up, 3 in
            flags noout
```

# 获取当前的OSDMap

ceph osd dump是获取当前版本的OSDMap的命令。

# 获取历史版本的OSDMap

分成两步走：

* 第一步获取二进制的osdmap，命令如下：

```
ceph osd getmap xxx -o osdmap_xxx
```
(注： xxx为osdmap的版本，比如539这种数字，当xxx为0 的时候，获取的是当前版本的osdmap)

* 通过osdmaptool 将二进制的osdmap打印成可读的版本

```
osdmaptool --print osdmap_xxx
```

其输出如下：

```
root@BEAN-1:~# ceph osd getmap 539 -o osdmap_539
got osdmap epoch 539

root@BEAN-1:~# osdmaptool --print osdmap_539 
osdmaptool: osdmap file 'osdmap_539'
epoch 539
fsid 23a30fe1-f6af-4641-983a-964b8c0d417e
created 2017-01-04 10:54:06.244070
modified 2017-02-16 06:32:39.529807
flags noout

pool 0 'rbd' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 1 flags hashpspool stripe_width 0
pool 1 '.ezs3' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 9 flags hashpspool stripe_width 0
pool 2 'data' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 12 flags hashpspool stripe_width 0
pool 3 'metadata' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 11 flags hashpspool stripe_width 0
pool 4 '.rgw.buckets' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 13 flags hashpspool stripe_width 0
pool 5 '.ezs3.central.log' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 14 flags hashpspool stripe_width 0
pool 6 '.ezs3.statistic' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 15 flags hashpspool stripe_width 0
pool 7 '.rgw.root' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 16 flags hashpspool stripe_width 0
pool 8 '.rgw.control' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 19 flags hashpspool stripe_width 0
pool 9 '.rgw' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 21 flags hashpspool stripe_width 0
pool 10 '.rgw.gc' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 22 flags hashpspool stripe_width 0
pool 11 '.users.uid' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 23 flags hashpspool stripe_width 0
pool 12 '.users.email' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 25 flags hashpspool stripe_width 0
pool 13 '.users' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 27 flags hashpspool stripe_width 0
pool 14 '.users.swift' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 29 flags hashpspool stripe_width 0

max_osd 3
osd.0 up in weight 1 recovery_weight 1 up_from 539 up_thru 533 down_at 536 last_clean_interval [478,536) 10.11.12.1:6801/4493 10.11.12.1:6806/3004493 10.11.12.1:6807/3004493 10.11.12.1:6808/3004493 exists,up 08c4922b-2a2e-429e-b9d3-b37b0a94e89a
osd.1 up in weight 1 recovery_weight 1 up_from 523 up_thru 536 down_at 520 last_clean_interval [489,522) 10.11.12.2:6801/128106 10.11.12.2:6806/5128106 10.11.12.2:6807/5128106 10.11.12.2:6808/5128106 exists,up 376ce6a9-18ac-4a92-a09c-2c06a7a593cd
osd.2 up in weight 1 recovery_weight 1 up_from 481 up_thru 536 down_at 480 last_clean_interval [414,475) 10.11.12.3:6801/4360 10.11.12.3:6802/4360 10.11.12.3:6803/4360 10.11.12.3:6804/4360 exists,up caa8d048-2ef1-401a-8506-29949e79fe3c
```

注意其中每个OSD的  up_from up_thru last_clean_interval等信息，这些信息是我们分析状态迁移所必须的信息。