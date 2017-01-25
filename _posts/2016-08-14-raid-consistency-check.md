---
layout: post
title: LSI MegaRaid Consistency Check
date: 2016-08-14 14:43:40
categories: Storage
tag: megacli
excerpt: Megacli 查看和控制一致性检查相关的内容
---

# 前言

今天查看集群，发现写入速度明显下降了。

```
Virtual Drive: 4 (Target Id: 4)
Name                :
RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
Size                : 32.743 TB
Sector Size         : 512
Is VD emulated      : No
Parity Size         : 3.637 TB
State               : Optimal
Strip Size          : 128 KB
Number Of Drives    : 10
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAdaptive, Cached, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAdaptive, Cached, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disabled
Ongoing Progresses:
  Check Consistency        : Completed 4%, Taken 616 min.
Encryption Type     : None
Bad Blocks Exist: No
PI type: No PI

Is VD Cached: No



Exit Code: 0x00

```

发现集群中的RAID在做一致性检查。

LSI MegaRaid的一致性检查分成两种

* Patrol Read
* Consistency Check

本文介绍Consistency Check。其相关的几个参数如下：

* Delay Interval：  默认是168小时，即一周。
* mode  ： 值有两种 并发模式 ModeConc 和顺序模式ModeSeq
* Consistency Check Rate ： 默认30%


# 查看当前配置

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpCCSched -Info -Aall
                                     
Adapter #0

Operation Mode: Concurrent
Execution Delay: 168
Next start time: 08/13/2016, 10:00:

Current State: Active
Number of iterations: 0
Number of VD completed: 0
Excluded VDs          : None
Exit Code: 0x00
root@IPTV:~# date
Sun Aug 14 19:31:13 CST 2016
```

# 设置下次周期性一致性检查的时间

因为周期性检查会损害性能，所以正常使用过程中如果进行这个检查会造成性能显著地下降。因此我们一般有需要调整周期行检查的时间和周期。

## 设置RAID Controller的时间

```

root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpSetTime `date +%Y%m%d` `date +%H:%M:%S` -Aall -NoLOG
                                     
New date/time is set on Adapter 0

Exit Code: 0x00

root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpGetTime -Aall
                                     

Adapter 0:
    Date: 08/14/2016
    Time: 19:35:46

Exit Code: 0x00
```

## 设置下次检查的时间



```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64  -AdpCcSched -SetStartTime 20160830 02 -Aall
                                     
Adapter 0: Scheduled CC start time is set.

Exit Code: 0x00
```

设置完毕后，通过如下命令查看是否生效：

```

root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64  -AdpCcSched -Info -Aall
                                     
Adapter #0

Operation Mode: Concurrent
Execution Delay: 168
Next start time: 08/30/2016, 02:00:00
Current State: Active
Number of iterations: 0
Number of VD completed: 0
Excluded VDs          : None
Exit Code: 0x00
root@IPTV:~# 
```

# 设置检查周期

默认检查周期是168小时，即7天的时间。可以将检查周期变成30天，如下所示：

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -SetDelay 720 -Aall
                                     
Adapter 0: Scheduled CC execution delay is set to 720 hours.

Exit Code: 0x00
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64  -AdpCcSched -Info -Aall
                                     
Adapter #0

Operation Mode: Concurrent
Execution Delay: 720
Next start time: 08/30/2016, 02:00:00
Current State: Active
Number of iterations: 0
Number of VD completed: 0
Excluded VDs          : None
Exit Code: 0x00
root@IPTV:~# 

```

#  检查进度

如果已经在进行一致性检查了，可能希望了解当前的进度，包括什么时候开始的检查。

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -ldcc -progdsply -lall -aall

 Progress of Virtual Drives...

  Virtual Drive #              Percent Complete                       Time Elps
          3         ##                     04 %                        11:46:53
          4         ##                     04 %                        13:13:39

    Press <ESC> key to quit...

root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -ldcc -showprog -lall -aall
                                     
Check Consistency on VD #0 is not in progress.
Check Consistency on VD #3 (target id #3) Completed 4% in 808 Minutes.
Check Consistency on VD #4 (target id #4) Completed 4% in 840 Minutes.

Exit Code: 0x00

```

# 禁止一致性检查

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -Dsbl -Aall
                                     
Adapter 0: Scheduled CC mode is set to Disabled.

Exit Code: 0x00
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64  -AdpCcSched -Info -Aall
                                     
Adapter #0

Operation Mode: Disabled
Execution Delay: 720
Next start time: 07/28/2135, 02:00:00
Current State: Active
Number of iterations: 0
Number of VD completed: 0
Excluded VDs          : None
Exit Code: 0x00
root@IPTV:~# 

```

# 设置模式

设置为ModeConc

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -ModeConc -Aall
                                     
Adapter 0: Scheduled CC mode is set to Concurrent.

Exit Code: 0x00
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpCCSched -Info -Aall
                                     
Adapter #0

Operation Mode: Concurrent
Execution Delay: 168
Next start time: 07/28/2135, 02:00:00
Current State: Active
Number of iterations: 0
Number of VD completed: 0
Excluded VDs          : None
Exit Code: 0x00
root@IPTV:~# 
```

设置为顺序模式：

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -ModeSeq -Aall
                                     
Adapter 0: Scheduled CC mode is set to Sequential.

Exit Code: 0x00
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64  -AdpCcSched -Info -Aall
                                     
Adapter #0

Operation Mode: Sequential
Execution Delay: 720
Next start time: 07/28/2135, 02:00:00
Current State: Active
Number of iterations: 0
Number of VD completed: 0
Excluded VDs          : None
Exit Code: 0x00

```
注意，如果Disable了一致性检查，然后设置mode，需要设置下一次检查的时间，否则时间为2135年07月28日。


# 设置Consistency Check Rate

```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 AdpGetProp CCRate -Aall
                                     
Adapter 0: Check Consistency Rate = 30%

Exit Code: 0x00
```


```
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 -AdpSetprop CCRate 10 -A0
                                     
Adapter 0: Set Check consistency rate to 10% success.

Exit Code: 0x00
root@IPTV:~# /opt/MegaRAID/MegaCli/MegaCli64 AdpGetProp CCRate -Aall
                                     
Adapter 0: Check Consistency Rate = 10%

Exit Code: 0x00
```
