---
layout: post
title: Megacli get event log
date: 2017-04-28 11:51:40
categories: Storage
tag: megacli
excerpt: 
---

# 前言

有些时候，追查性能问题的时候，需要查看event log。比如性能突然下降，比如RAID的 cache policy 突然被改成WT。查对应时间点，就需要查看 RAID的event log

# 方法

```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpEventLog -GetEvents -f raid_event_2017_04_28.log -A0
```

# 一致性检查

```
seqNum: 0x00000e06
Time: Sat Aug  1 03:00:00 2015

Code: 0x00000042
Class: 0
Locale: 0x01
Event Description: Consistency Check started on VD 00/0

Event Data:
===========
Target Id: 0
```

# BBU disabled

```
seqNum: 0x00016bef
Time: Fri Apr 28 05:08:02 2017

Code: 0x000000c3
Class: 1
Locale: 0x08
Event Description: BBU disabled; changing WB virtual disks to WT, Forced WB VDs are not affected
Event Data:
===========
None
```

# BBU 开始充电

```
seqNum: 0x00016bf6
Time: Fri Apr 28 05:09:04 2017

Code: 0x00000093
Class: 0
Locale: 0x08
Event Description: Battery started charging
Event Data:
===========
None
```

# BBU enabled

```
seqNum: 0x00016bf7
Time: Fri Apr 28 06:59:34 2017

Code: 0x000000c2
Class: 0
Locale: 0x08
Event Description: BBU enabled; changing WT virtual disks to WB
Event Data:
===========
None

seqNum: 0x00016bf8
Time: Fri Apr 28 06:59:34 2017

Code: 0x00000036
Class: 0
Locale: 0x01
Event Description: Policy change on VD 00/0 to [ID=00,dcp=65,ccp=65,ap=0,dc=2] from [ID=00,dcp=65,ccp=64,ap=0,dc=2]
Event Data:
===========
Target Id: 0
Previous LD Properties
Access Policy: 0
Current Cache Policy: 100
Default Cache Policy: 101
Disk Cache Policy: 2
Name:
NoBgi: 0
New LD Properties
Access Policy: 0
Current Cache Policy: 101
Default Cache Policy: 101
Disk Cache Policy: 2
Name:
NoBgi: 0

```

# RAID时间：

```
root@EBS-005-PUB01-JD-Xian:/var/log# /opt/MegaRAID/MegaCli/MegaCli64 -AdpGetTime -Aall  ; date 
                                     

Adapter 0:
    Date: 04/28/2017
    Time: 09:30:35

Exit Code: 0x00
Fri Apr 28 17:30:34 CST 2017
```

# 修改RAID时间

```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpSetTime `date +%Y%m%d` `date +%H:%M:%S` -Aall -NoLOG
                                     
New date/time is set on Adapter 0

Exit Code: 0x00
```