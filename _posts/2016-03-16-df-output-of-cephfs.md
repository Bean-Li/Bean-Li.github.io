---
layout: post
title: stat -f or df -hi output of cephfs
date: 2016-03-16 22:00:40
categories: ceph
tag: cephfs ceph
excerpt: stat -f or df -hi output of cephfs
---

前言
----
我目前有机会接触到各式各样的客户，了解客户的需求，处理客户报的问题，解决客户的困惑。其实很多问题都是客户发现，push我厘清这些细节。

今天有一个客户的存储工程师执行了df －hi，发现我们的cephfs返回如下信息：

```
Filesystem                                            Inodes    IUsed    IFree    IUse%    Mounted on
172.30.30.211,172.30.30.212,172.30.30.213:/             413K        -        -        -    /var/share/ezfs
```

客户看到413K，感觉到很担忧，认为我们cephfs只能提供413K个inode，即最多可以创建41万多文件。

显然这是不对的，在日常测试中，我们曾经多次测试创建5亿个文件，那么这个413K是什么东东呢，为什么IUsed  IFree  IUse的值都是－呢？

测试
---- 

接到客户的邮件之后，我马上在我的测试环境里看了下，果然Inodes数目非常奇怪，是一个很奇怪的数值389.

```
root@BASIC:~# stat -f /var/share/ezfs/
  File: "/var/share/ezfs/"
    ID: d30a1a4fe2c80caf Namelen: 255     Type: UNKNOWN (0xc36400)
Block size: 4194304    Fundamental block size: 4194304
Blocks: Total: 23117      Free: 23058      Available: 23050
Inodes: Total: 389        Free: -1


root@BASIC:~# df -ih
Filesystem              Inodes IUsed IFree IUse% Mounted on
/dev/sda3                 2.0M   66K  1.9M    4% /
udev                      493K   503  492K    1% /dev
tmpfs                     495K   469  494K    1% /run
none                      495K    10  495K    1% /run/lock
none                      495K     1  495K    1% /run/shm
/dev/sdb2                 2.9M  7.1K  2.9M    1% /data/osd.0
/dev/sdc2                 2.9M  7.4K  2.9M    1% /data/osd.1
10.20.1.88,10.20.1.99:/    389     -     -     - /var/share/ezfs

```


代码
-----

我看了下，Inodes：Total只有389，而Inode Free的值是－1。 我查了下手册，statfs接口相关的结构体定义如下。


```
       struct statfs {
           __SWORD_TYPE f_type;    /* type of file system (see below) */
           __SWORD_TYPE f_bsize;   /* optimal transfer block size */
           fsblkcnt_t   f_blocks;  /* total data blocks in file system */
           fsblkcnt_t   f_bfree;   /* free blocks in fs */
           fsblkcnt_t   f_bavail;  /* free blocks available to
                                      unprivileged user */
           fsfilcnt_t   f_files;   /* total file nodes in file system */
           fsfilcnt_t   f_ffree;   /* free file nodes in fs */
           fsid_t       f_fsid;    /* file system id */
           __SWORD_TYPE f_namelen; /* maximum length of filenames */
           __SWORD_TYPE f_frsize;  /* fragment size (since Linux 2.6) */
           __SWORD_TYPE f_spare[5];
       };
```
 
好在，源码面前没有秘密，查看kernel的代码，从fs/ceph/super.c中不难找到答案：

```
	err = ceph_monc_do_statfs_pool(&fsc->client->monc, &st, poolid);
	...
	
	buf->f_bsize = 1 << CEPH_BLOCK_SHIFT;
	buf->f_frsize = 1 << CEPH_BLOCK_SHIFT;
	buf->f_blocks = le64_to_cpu(st.kb) >> (CEPH_BLOCK_SHIFT-10);
	buf->f_bfree = le64_to_cpu(st.kb - st.kb_used) >> (CEPH_BLOCK_SHIFT-10);
	buf->f_bavail = le64_to_cpu(st.kb_avail) >> (CEPH_BLOCK_SHIFT-10);

	buf->f_files = le64_to_cpu(st.num_objects);
	buf->f_ffree = -1;
	buf->f_namelen = NAME_MAX;
	
```

从buf->f_files 不难看出，该值是当前集群的object数，更准确的含义，不妨看ceph代码，但是很明显，该值并不是cephfs容许的最大inode数。
而free inode值为－1，表示并没有什么限制。

```
root@BASIC:~# rados df
pool name       category                 KB      objects       clones     degraded      unfound           rd        rd KB           wr        wr KB
.ezs3           -                          0            0            0            0           0            0            0            0            0
.ezs3.central.log -                          6            6            0            0           0            0            0            8            9
.ezs3.statistic -                       2100           38            0            0           0         4772         4289       837795       838831
.rgw            -                          0            0            0            0           0            0            0            0            0
.rgw.buckets    -                          0            0            0            0           0            0            0            0            0
.rgw.control    -                          0            8            0            0           0            0            0            0            0
.rgw.gc         -                          0           32            0            0           0          295          295          228            0
.rgw.root       -                          1            3            0            0           0        13724         8235            3            3
.users          -                          1            1            0            0           0            0            0            1            1
.users.email    -                          1            1            0            0           0            0            0            1            1
.users.swift    -                          1            1            0            0           0            0            0            1            1
.users.uid      -                          1            1            0            0           0           12           10            4            2
data            -                          4            3            0            0           0            2            8            4            4
metadata        -                      60150          295            0            0           0           26           30        40608       102058
rbd             -                          0            0            0            0           0            0            0            0            0
  total used          484808          389
  total avail      188831912
  total space      189382256
```          
      
