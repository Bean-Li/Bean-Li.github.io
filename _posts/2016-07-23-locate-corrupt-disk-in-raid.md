---
layout: post
title: 更换RAID中的故障盘
date: 2016-07-23 14:43:40
categories: Storage
tag: megacli
excerpt: RAID中某个盘发生故障，需要替换，如何确定它的位置，如何替换，如何查看RAID的rebuild进度？
---

# 前言

请看下面的图：

![](/assets/LINUX/iostat_high_srvtm.jpg)

在某客户环境发现，系统盘的util居高不下，很多时候都是100%，但是无论read 还是write的压力并不大，svctm却大的吓人。操作系统是两块盘组成的RAID 1，我们需要看下RAID 1 中是否存在损坏的盘。


# 排查
首先通过megacli的如下命令查看RAID的情况。

```
/opt/MegaRAID/MegaCli/MegaCli64 LDPDInfo -Aall
```
我们的系统盘是2个磁盘组成的RAID1，

```
                                     
Adapter #0

Number of Virtual Disks: 6
Virtual Drive: 0 (Target Id: 0)
Name                :
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 278.875 GB
Sector Size         : 512
Is VD emulated      : No
Mirror Data         : 278.875 GB
State               : Optimal
Strip Size          : 64 KB
Number Of Drives    : 2
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disabled
Encryption Type     : None
PI type: No PI

Is VD Cached: No
Number of Spans: 1
Span: 0 - Number of PDs: 2

PD: 0 Information
Enclosure Device ID: 1
Slot Number: 0
Drive's position: DiskGroup: 0, Span: 0, Arm: 0
Enclosure position: 1
Device Id: 2
WWN: 5000C50074032D54
Sequence Number: 2
Media Error Count: 72605
Other Error Count: 1
Predictive Failure Count: 107
Last Predictive Failure Event Seq Number: 171552
PD Type: SAS

Raw Size: 279.396 GB [0x22ecb25c Sectors]
Non Coerced Size: 278.896 GB [0x22dcb25c Sectors]
Coerced Size: 278.875 GB [0x22dc0000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  512
Firmware state: Online, Spun Up
Commissioned Spare : No
Emergency Spare : No
Device Firmware Level: 000B
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x5000c50074032d55
SAS Address(1): 0x0
Connected Port Number: 1(path0) 
Inquiry Data: SEAGATE ST3300657SS     000B6SJ17KP5            
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Hard Disk Device
Drive:  Not Certified
Drive Temperature :36C (96.80 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Port-1 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : Yes




PD: 1 Information
Enclosure Device ID: 1
Slot Number: 1
Drive's position: DiskGroup: 0, Span: 0, Arm: 1
Enclosure position: 1
Device Id: 3
WWN: 5000C50059EC14A8
Sequence Number: 2
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SAS

Raw Size: 279.396 GB [0x22ecb25c Sectors]
Non Coerced Size: 278.896 GB [0x22dcb25c Sectors]
Coerced Size: 278.875 GB [0x22dc0000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  512
Firmware state: Online, Spun Up
Commissioned Spare : No
Emergency Spare : No
Device Firmware Level: 0008
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x5000c50059ec14a9
SAS Address(1): 0x0
Connected Port Number: 1(path0) 
Inquiry Data: SEAGATE ST3300657SS     00086SJ615M5            
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Hard Disk Device
Drive:  Not Certified
Drive Temperature :35C (95.00 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Port-1 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



```

注意两块盘中，有一块盘有如下信息：

```
PD: 0 Information
Enclosure Device ID: 1
Slot Number: 0
Drive's position: DiskGroup: 0, Span: 0, Arm: 0
Enclosure position: 1
Device Id: 2
WWN: 5000C50074032D54
Sequence Number: 2
Media Error Count: 72605
Other Error Count: 1
Predictive Failure Count: 107
Last Predictive Failure Event Seq Number: 171552
PD Type: SAS


Drive has flagged a S.M.A.R.T alert : Yes
```

从Media Error Count可以看出该盘损坏的已经很厉害了，这也是svctm居高不下的原因。我们需要更换该盘。那我们遇到的首要问题是，如何找到该盘。

megacli提供了让指定位置的盘闪烁的功能。

# 定位盘的位置

我们可以通过让指定位置的盘闪烁的方式来帮助运维人员定位磁盘位置：

```
MegaCli -PdLocate -start -physdrv [E:S]  -aALL
```

其中 E表示 Enclosure Device ID，S表示Slot Number。对于我们的例子，坏盘的位置信息如下：

```
Enclosure Device ID: 1
Slot Number: 0
```

因此我们可以执行如下指令让其闪烁：

```
root@Storage-c2:/opt/MegaRAID/MegaCli# ./MegaCli64 -PdLocate -start  -physdrv[1:0] -a0
                                     
Adapter: 0: Device at EnclId-1 SlotId-0  -- PD Locate Start Command was successfully sent to Firmware 

Exit Code: 0x00
root@Storage-c2:/opt/MegaRAID/MegaCli# 
```

这样就定快速定位到损坏的盘的位置了。

更换完盘之后，关闭闪烁的命令如下

```
MegaCli -PdLocate -stop -physdrv [E:S]  -aALL
```

# 查看进度

很多RAID卡换盘之后，不需要做什么配置，但是RAID1 也好，RAID5也好，新盘进来之后会rebuild，从拔盘到插入新盘，变化遵循如下轨迹：

* Device           
Normal --->Damage  --->Rebuild --->Normal 

* Virtual Drive    
Optimal --->Degraded --->Degraded --->Optimal

* Physical Drive   
Online --->Failed Unconfigured --->Rebuild  --->Online 

换盘之后，VD处于Degraded状态：

```
root@Storage-c2:/opt/MegaRAID/MegaCli# ./MegaCli64  -LDInfo -Lall -Aall
                                     

Adapter 0 -- Virtual Drive Information:
Virtual Drive: 0 (Target Id: 0)
Name                :
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 278.875 GB
Sector Size         : 512
Is VD emulated      : No
Mirror Data         : 278.875 GB
State               : Degraded
Strip Size          : 64 KB
Number Of Drives    : 2
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

换盘以后，PD处于Rebuild状态：

```
Raw Size: 279.396 GB [0x22ecb25c Sectors]
Non Coerced Size: 278.896 GB [0x22dcb25c Sectors]
Coerced Size: 278.875 GB [0x22dc0000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  512
Firmware state: Rebuild
Commissioned Spare : No
Emergency Spare : No
Device Firmware Level: 0008
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x5000c5003b119445
SAS Address(1): 0x0
Connected Port Number: 1(path0) 
Inquiry Data: SEAGATE ST3300657SS     00086SJ2631L            
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Hard Disk Device
Drive:  Not Certified
Drive Temperature :32C (89.60 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
```

可以通过如下指令查看rebuild的进度：

```
/opt/MegaRAID/MegaCli/MegaCli64 -PDRbld -showprog -physDrv [1:0] -a0
```

其输出如下：

```
root@Storage-c2:/opt/MegaRAID/MegaCli# ./MegaCli64 -PDRbld -showprog -physDrv [1:0] -a0
                                     
Rebuild Progress on Device at Enclosure 1, Slot 0 Completed 10% in 0 Minutes.

Exit Code: 0x00
root@Storage-c2:/opt/MegaRAID/MegaCli# 
```
