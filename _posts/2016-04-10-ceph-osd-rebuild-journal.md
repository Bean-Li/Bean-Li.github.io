---
layout: post
title: ceph OSD replace or rebuild journal
date: 2016-04-10 15:21:40
categories: ceph
tag: ceph
excerpt: 本文介绍如何替换ceph的journal
---

前言
-----
在ceph集群的使用过程中，有时候需要替换journal。为什么会有这种需求？不妨考虑一下场景：

1.  用户最初使用SATA盘的一个分区作为ceph OSD的journal，使用过程中发现速度不够快，想增加一块SSD，使用SSD的一个分区作为Journal
2.  用户的最初的journal 就是SSD的一个分区，但是SSD 有寿命，SSD坏掉了。

无论哪种情况，都要抛弃最初的Journal，重建一个信息的journal分区给CEPH OSD使用

方法
----
第一步，重建OSD的journal，总是要将对应的OSD停掉，为了防止OSD down的时间过久，引发recovery，应该设置noout标志位

```
    ceph osd set noout
```

第二部是停掉对应的osd.x

```
/etc/init.d/ceph stop osd.x
```

停掉OSD.x之后，考虑到journal上还有一些没来得及flush到data partition的内容，因此，需要执行flush journal的操作

```
ceph-osd -i x -c /etc/ceph/ceph.conf --cluster ceph --flush-journal
```

第四步是重建journal

重建之前自然是要先将SSD 分区，一般需要将journal所在的分区打上label。我们以osd-x-new-journal作为label的名字。

```
ceph-osd -i x --osd-journal /dev/disk/by-partlabel/osd-x-new-journal --mkjournal
```

第五步就是更新ceph.conf,在对应的osd的section，加入如下内容

```
[osd.x]
osd journal = /dev/disk/by-partlabel/osd-x-new-journal 
```

第六步重启OSD，同时去掉noout标志

```
/etc/init.d/ceph restart osd.x
ceph osd unset noout
```

至此，替换jouarnl或者重建journal的工作就完成了。



