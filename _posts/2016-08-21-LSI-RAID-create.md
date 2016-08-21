---
layout: post
title: LSI MegaRaid 创建RAID和删除RAID
date: 2016-08-21 14:43:40
categories: linux
tag: megacli
excerpt: Megacli 创建和删除RAID
---

# 前言

LSI的WebBIOS不太好用，尽管有Alt快捷键，但是我更喜欢Command Line。

如何使用MegaCli创建RAID


# 创建RAID

创建RAID的命令是CfgLdAdd,如下所示：

```
root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 CfgLDAdd -r5 [41:5,41:6,41:7,41:8,41:9,41:10,41:11,41:12,41:13,41:14] -A0 
                                     
Adapter 0: Created VD 2

Adapter 0: Configured the Adapter!!

Exit Code: 0x00
root@scaler:~# 

root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -L2 -A0
                                     

Adapter 0 -- Virtual Drive Information:
Virtual Drive: 2 (Target Id: 2)
Name                :
RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
Size                : 49.117 TB
Sector Size         : 512
Is VD emulated      : Yes
Parity Size         : 5.457 TB
State               : Optimal
Strip Size          : 256 KB
Number Of Drives    : 10
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
Ongoing Progresses:
  Background Initialization: Completed 0%, Taken 36 min.
Encryption Type     : None
Bad Blocks Exist: No
PI type: No PI

Is VD Cached: No



Exit Code: 0x00
root@scaler:~# 
```

上例是创建了一个RAID 5 ，其中中括号中的内容是Enclosure Device ID：Slot Number，可以通过 

```
/opt/MegaRAID/MegaCli/MegaCli64 PDList -Aall
```

来查看。

注意，RAID的很多属性都可以事后调节，比如 WriteBack 还是WriteThrough， 关于ReadAhead是选择 RA 还是NORA还是ADRA，Disk Cache Policy 是Enable 还是Disable，比较例外的是条带大小，如果不指定的话，默认是256KB，如果对条带有特殊要求，需要在创建RAID的时候指定，如下所示：

方式是 －strpsz128 （以128KB为例）

```
root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 CfgLDAdd -r5 [41:5,41:6,41:7,41:8,41:9,41:10,41:11,41:12,41:13,41:14] -strpsz128 -A0 
                                     
Adapter 0: Created VD 2

Adapter 0: Configured the Adapter!!

Exit Code: 0x00
root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -L2 -A0
                                     

Adapter 0 -- Virtual Drive Information:
Virtual Drive: 2 (Target Id: 2)
Name                :
RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
Size                : 49.117 TB
Sector Size         : 512
Is VD emulated      : Yes
Parity Size         : 5.457 TB
State               : Optimal
Strip Size          : 128 KB
Number Of Drives    : 10
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
Encryption Type     : None
Bad Blocks Exist: No
PI type: No PI

Is VD Cached: No



Exit Code: 0x00
root@scaler:~# 
```

# 删除RAID

删除RAID的指令是 CfgLdDel

```
root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 -CfgLdDel -L2 -A0
                                     
Adapter 0: Deleted Virtual Drive-2(target id-2)
```