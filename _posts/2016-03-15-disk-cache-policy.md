---
layout: post
title: disk cache policy in RAID
date: 2016-03-15 11:51:40
categories: linux
tag: linux
excerpt: disk cache policy会严重影响到数据的安全，尤其是可能有异常掉电的情况下
---

引言
-----
之所以写这篇文章，是因为，这篇文章背后，字字都是血泪。曾经因为EXT4-fs error，从夜里10点抢救数据到凌晨7点，
睡一个小时候之后，吃饭，见客户，告诉客户数据恢复回来了。


原理
----
首先，RAID的write policy分成两种：

**write back

**write through

看名字，我们也不难猜出write back会有更好的性能。因为在write back模式下，数据写入Controller Cache，就认为Write IO结束了，而不需要等待写入Hard driver。这种模式很明显存在安全隐患，异常掉电情况下，很容易引起数据丢失。

另一种模式是write through，这种模式比较谨慎，它压根不使用Raid Cache来加速写操作，因此这种模式的性能要比write back 低不少。

但是write back的不安全也是有办法解决的：

第一个方案是UPS--Uninterruptible Power Supply，这个按下不提。

第二个方案是BBU--Backup Battery Unit，有了BBU，就可以安心地采用write-back模式了。

对BBU不太了解的，可以阅读[Barriers, Caches, Filesystems](https://monolight.cc/2011/06/barriers-caches-filesystems/)，介绍的非常好。


讲完了这个write policy，还有一个东东叫disk cache policy，这个东西用来决定磁盘一级的cache是否enable。
一定要注意这个东西，无数血泪都是这个东西引起的。

如果选择write through模式，即不使用RAID cache，这种情况下disk cache对性能的提升是很大的。但是尽管如此，也不要enable disk cache，因为，会有数据丢失的风险。如果可能异常掉电，那么一定不要enable disk cache。

相关命令
--------
查看write policy 和 disk cache policy的命令如下：

```
  /opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aAll
```

输出如下：

```
        Virtual Drive: 1 (Target Id: 1)
        Name                :
        RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
        Size                : 25.466 TB
        Sector Size         : 512
        Is VD emulated      : No
        Parity Size         : 3.637 TB
        State               : Optimal
        Strip Size          : 128 KB
        Number Of Drives    : 8
        Span Depth          : 1
        Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
        Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
        Default Access Policy: Read/Write
        Current Access Policy: Read/Write
        Disk Cache Policy   : Disk's Default
        Encryption Type     : None
        PI type: No PI

        Is VD Cached: No
```

可以看到Disk Cache Policy 是 Disk's Default 。这个default值可以分成以下情况：

1 For virtual disks containing SATA disks ， Enabled

2 For virtual disks containing SAS disks ，  Disabled

可以通过如下命令将Disk Cache Policy的值改成 Disable

```
 /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -DisDskCache -Immediate -Lall -aAll
```
输出如下：

```
                                             
        Set Disk Cache Policy to Disabled on Adapter 0, VD 0 (target id: 0) success
        Set Disk Cache Policy to Disabled on Adapter 0, VD 1 (target id: 1) success
        Set Disk Cache Policy to Disabled on Adapter 0, VD 2 (target id: 2) success
        Set Disk Cache Policy to Disabled on Adapter 0, VD 3 (target id: 3) success
        Set Disk Cache Policy to Disabled on Adapter 0, VD 4 (target id: 4) success
        Set Disk Cache Policy to Disabled on Adapter 0, VD 5 (target id: 5) success
        Set Disk Cache Policy to Disabled on Adapter 0, VD 6 (target id: 6) success

        Exit Code: 0x00
```

此时再次查看输出：

```
        Virtual Drive: 1 (Target Id: 1)
        Name                :
        RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
        Size                : 25.466 TB
        Sector Size         : 512
        Is VD emulated      : No
        Parity Size         : 3.637 TB
        State               : Optimal
        Strip Size          : 128 KB
        Number Of Drives    : 8
        Span Depth          : 1
        Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
        Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
        Default Access Policy: Read/Write
        Current Access Policy: Read/Write
        Disk Cache Policy   : Disabled
        Encryption Type     : None
        PI type: No PI

        Is VD Cached: No
```

推荐设置
-------
1 商用环境，RAID一定要有BBU

2 write policy 采用 write back

3 disk cache policy 一定要为disable

这个推荐设置，和Intel给出的 [Configuring  RAID For Optimal Performance](http://download.intel.com/support/motherboards/server/sb/configuring_raid_for_optimal_perfromance_11.pdf) 是一致的，除此以外，[RAID Controller and Hard Disk Cache Settings](https://www.thomas-krenn.com/en/wiki/RAID_Controller_and_Hard_Disk_Cache_Settings)也给出了类似的结论。这些都是不错的参考文献。




