---
layout: post
title: MegaCli command on LSI Raid --BBU,write policy and disk cache policy
date: 2016-03-17 14:38:40
categories: Storage
tag: megacli
excerpt: MegaCli是一个非常有用的工具
---

引言
-----
MegaCli是一个非常有用的工具，无论是创建raid、获取raid信息还是修改raid设置。本文focus在BBU和write policy以及disk cache policy信息的获取和修改。



是否存在BBU
-----------
如何查看是否存在BBU：

/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -aAll

当没有BBU时，输出如下：

```
root@yb-test:/opt/MegaRAID/MegaCli# ./MegaCli64 -AdpBbuCmd -aAll
                                     
Adapter 0: Get BBU Status Failed.

FW error description: 
 The required hardware component is not present.  

Exit Code: 0x22

```

BBU的相关信息
-------------
如果系统存在BBU的话，输出的信息如下：

```

root@node-2:/opt/MegaRAID/MegaCli# ./MegaCli64 -AdpBbuCmd -Aall
                                     
BBU status for Adapter: 0

BatteryType: iBBU-09
Voltage: 3996 mV
Current: 0 mA
Temperature: 53 C
**Battery State: Optimal**
Design Mode  : 48+ Hrs retention with a non-transparent learn cycle and moderate service life.

BBU Firmware Status:

  Charging Status              : None
  Voltage                                 : OK
  Temperature                             : OK
  Learn Cycle Requested	                  : Yes
  Learn Cycle Active                      : No
  Learn Cycle Status                      : OK
  Learn Cycle Timeout                     : No
  I2c Errors Detected                     : No
  Battery Pack Missing                    : No
  Battery Replacement required            : No
  Remaining Capacity Low                  : No
  Periodic Learn Required                 : No
  Transparent Learn                       : No
  No space to cache offload               : No
  Pack is about to fail & should be replaced : No
  Cache Offload premium feature required  : No
  Module microcode update required        : No

BBU GasGauge Status: 0x0180 
  Relative State of Charge: 89 %
  Charger System State: 1
  Charger System Ctrl: 0
  Charging current: 0 mA
  Absolute state of charge: 77 %
  Max Error: 0 %
  Battery backup charge time : 48 hours +

BBU Capacity Info for Adapter: 0

  Relative State of Charge: 89 %
  Absolute State of charge: 77 %
  Remaining Capacity: 1169 mAh
  Full Charge Capacity: 1324 mAh
  Run time to empty: Battery is not being charged.  
  Average time to empty: 2 Hour, 20 Min. 
  Estimated Time to full recharge: Battery is not being charged.  
  Cycle Count: 3

BBU Design Info for Adapter: 0

  Date of Manufacture: 09/03, 2013
  Design Capacity: 1500 mAh
  Design Voltage: 4100 mV
  Specification Info: 0
  Serial Number: 7968
  Pack Stat Configuration: 0x0000
  Manufacture Name: LS36691
  Firmware Version   : 
  Device Name: iBBU-09
  Device Chemistry: LION
  Battery FRU: N/A
  Transparent Learn = 0
  App Data = 0

BBU Properties for Adapter: 0

  Auto Learn Period: 28 Days
  Next Learn time: Thu Mar 31 15:16:22 2016
  Learn Delay Interval:0 Hours
  Auto-Learn Mode: Enabled
  BBU Mode = 5

Exit Code: 0x00

```

其中加粗的信息Optimal表示BBU状态正常。

BBU是由锂离子电池和电子控制电路组成，电池的寿命取决于其老化程度，无论是否充放电已经充放电的多少，锂离子电池的容量都会减少。为了记录电池的放电曲线，延长电池的寿命，默认会启用自动校准模式（AutoLearn Mode），在Learn Cycle 内，Raid卡控制器不会使用BBU，这个过程可能持续12小时，这个过程中会禁用Write back，直到其完成校准。

这个Auto-Learn Mode，一般30天执行一次，我们的BBU 28天之星一次。一般不要关闭，因为这可以有效地延长寿命。
如果不做这个校准，寿命会从2年下降到8个月。

充电过程中，BBU相关的信息如下：

```
root@yb-test:/opt/MegaRAID/MegaCli# ./MegaCli64 -AdpBbucmd -aAll
                                     
BBU status for Adapter: 0

BatteryType: iBBU-09
Voltage: 3955 mV
Current: 527 mA
Temperature: 36 C
**Battery State: Degraded(Charging)**
Design Mode  : 48+ Hrs retention with a non-transparent learn cycle and moderate service life.

BBU Firmware Status:

**  Charging Status              : Charging**
  Voltage                                 : OK
  Temperature                             : OK
  Learn Cycle Requested	                  : No
  Learn Cycle Active                      : No
  Learn Cycle Status                      : OK
  Learn Cycle Timeout                     : No
  I2c Errors Detected                     : No
  Battery Pack Missing                    : No
  Battery Replacement required            : No
  Remaining Capacity Low                  : Yes
  Periodic Learn Required                 : No
  Transparent Learn                       : No
  No space to cache offload               : No
  Pack is about to fail & should be replaced : No
  Cache Offload premium feature required  : No
  Module microcode update required        : No

BBU GasGauge Status: 0x0100 
  Relative State of Charge: 63 %
  Charger System State: 1
  Charger System Ctrl: 0
  Charging current: 527 mA
  Absolute state of charge: 52 %
  Max Error: 0 %
  Battery backup charge time : 42 hours

  ...

Exit Code: 0x00

```

BBU相关的cache policy
----------------------

通过执行 MegaCli64 -LDIno -Lall -aALL可以查看默认的cache policy，

```
/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aALL
Default Cache Policy: WriteBack, ReadAheadNone, Direct, No Write Cache if Bad BBU

```

其中No Write Cache if Bad BBU表示BBU有问题的时候，禁用write cache，这种做法是安全负责任的。

另外一种不负责任的选项是 Write Cache OK if Bad BBU。
分别可以采用以下命令修改：

```
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp CachedBadBBU -Lall -aALL


root@node-2:/opt/MegaRAID/MegaCli# ./MegaCli64 -LDSetProp CachedBadBBU -Lall -aALL
                                     
Set Write Cache OK if bad BBU on Adapter 0, VD 0 (target id: 0) success
Set Write Cache OK if bad BBU on Adapter 0, VD 1 (target id: 1) success
Set Write Cache OK if bad BBU on Adapter 0, VD 2 (target id: 2) success
Set Write Cache OK if bad BBU on Adapter 0, VD 3 (target id: 3) success
Set Write Cache OK if bad BBU on Adapter 0, VD 4 (target id: 4) success

改过之后，可以查看，变成了Write Cache OK if Bad BBU
Virtual Drive: 2 (Target Id: 2)
Name                :
RAID Level          : Primary-0, Secondary-0, RAID Level Qualifier-0
Size                : 7.275 TB
Sector Size         : 512
Is VD emulated      : No
Parity Size         : 0
State               : Optimal
Strip Size          : 128 KB
Number Of Drives    : 4
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, Write Cache OK if Bad BBU
Current Cache Policy: WriteBack, ReadAhead, Direct, Write Cache OK if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
Encryption Type     : None
PI type: No PI

Is VD Cached: No

```
当然了这种策略是不负责任的，我们需要改成No Write Cache if Bad BBU，方法如下：

```

/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -NoCachedBadBBU -Immediate -Lall -aAll

```

输出情况及查看是否生效：

```
root@node-2:/opt/MegaRAID/MegaCli#  ./MegaCli64 -LDSetProp -NoCachedBadBBU  -Lall -aAll
                                     
Set No Write Cache if bad BBU on Adapter 0, VD 0 (target id: 0) success
Set No Write Cache if bad BBU on Adapter 0, VD 1 (target id: 1) success
Set No Write Cache if bad BBU on Adapter 0, VD 2 (target id: 2) success
Set No Write Cache if bad BBU on Adapter 0, VD 3 (target id: 3) success
Set No Write Cache if bad BBU on Adapter 0, VD 4 (target id: 4) success

Exit Code: 0x00

Virtual Drive: 2 (Target Id: 2)
Name                :
RAID Level          : Primary-0, Secondary-0, RAID Level Qualifier-0
Size                : 7.275 TB
Sector Size         : 512
Is VD emulated      : No
Parity Size         : 0
State               : Optimal
Strip Size          : 128 KB
Number Of           : 4
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


disk cache policy
------------------

按照最佳设置，这个disk cache policy应该关闭，

关闭的命令上一篇博文有提到，

```

 关闭disk cache
 /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -DisDskCache -Immediate -Lall -aAll
 
 开启disk cache
 /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -EnDskCache -Immediate -Lall -aAll

```








