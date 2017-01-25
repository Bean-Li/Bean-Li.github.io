---
layout: post
title: Disk's S.M.A.R.T info in RAID
date: 2017-01-25 14:43:40
categories: Storage
tag: megacli
excerpt: 获取RAID中某块磁盘的SMART信息
---

# 前言

对于单盘的SMART信息，可以通过smartctl很容易的获取，但是如果磁盘位于RAID中，如何获得其SMART信息？
还是使用smartctl

# smartctl获取RAID组中磁盘的SMART信息

首先通过LDPDinfo -A0 获取某个盘的Device Id。

对于我们而言，存在一个Critical Disks，通过LDPDinfo查看，有一个盘存在 S.M.A.R.T alert，如下图所示：

```
PD: 1 Information
Enclosure Device ID: 59
Slot Number: 2
Drive's position: DiskGroup: 0, Span: 0, Arm: 1
Enclosure position: 1
Device Id: 96
WWN: 5000C500595382D0
Sequence Number: 2
Media Error Count: 2
Other Error Count: 1
Predictive Failure Count: 1
Last Predictive Failure Event Seq Number: 38912
PD Type: SAS
```

我们希望拿到这块盘的SMART信息。那么根据Device Id和盘符/dev/sda可以通过smartctl获取。

```
smartctl -a -d megaraid,[Device Id] /dev/sda
```

其输出如下：

```
root@Storage-07:~# smartctl -a -d megaraid,96 /dev/sda
smartctl 5.41 2011-06-09 r3365 [x86_64-linux-4.1.22-server] (local build)
Copyright (C) 2002-11 by Bruce Allen, http://smartmontools.sourceforge.net
 
Vendor:               SEAGATE
Product:              ST3300657SS    
Revision:             0008
User Capacity:        300,000,000,000 bytes [300 GB]
Logical block size:   512 bytes
Logical Unit id:      0x5000c500595382d3
Serial number:        6SJ5S9N00000N3038WCB
Device type:          disk
Transport protocol:   SAS
Local Time is:        Wed Jan 25 12:45:13 2017 CST
Device supports SMART and is Enabled
Temperature Warning Enabled
SMART Health Status: FAILURE PREDICTION THRESHOLD EXCEEDED [asc=5d, ascq=0]
 
Current Drive Temperature:     32 C
Drive Trip Temperature:        68 C
Elements in grown defect list: 2066
Vendor (Seagate) cache information
  Blocks sent to initiator = 4027599650
  Blocks received from initiator = 2533195074
  Blocks read from cache and sent to initiator = 971048126
  Number of read and write commands whose size <= segment size = 209910133
  Number of read and write commands whose size > segment size = 0
Vendor (Seagate/Hitachi) factory information
  number of hours powered up = 8347.20
  number of minutes until next internal SMART test = 40
 
Error counter log:
           Errors Corrected by           Total   Correction     Gigabytes    Total
               ECC          rereads/    errors   algorithm      processed    uncorrected
           fast | delayed   rewrites  corrected  invocations   [10^9 bytes]  errors
read:   817967014     7436         0  817974450   817974452       4260.902           2
write:         0        0         2         2          3      16814.009           1
verify:     1956        0         0      1956       1956          0.252           0
 
Non-medium error count:        0
 
[GLTSD (Global Logging Target Save Disable) set. Enable Save with '-S on']
No self-tests have been logged
Long (extended) Self Test duration: 3200 seconds [53.3 minutes]
 
```

可以看到

```
SMART Health Status: FAILURE PREDICTION THRESHOLD EXCEEDED [asc=5d, ascq=0]
```
这就是存在Critical Disk的原因。这种情况下一般需要更换磁盘。

[Re: [smartmontools-support] SMARTD Error Messages](https://sourceforge.net/p/smartmontools/mailman/message/18992517/)

