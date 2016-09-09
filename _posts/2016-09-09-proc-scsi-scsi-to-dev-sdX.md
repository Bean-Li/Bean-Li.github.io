---
layout: post
title: mapping between /proc/scsi/scsi and /dev/sdX
date: 2016-09-09 21:53:40
categories: Storage
tag: storage
excerpt: 从/proc/scsi/scsi到/dev/sdX的映射
---

# 前言

最近帮忙定位了一个问题，即客户环境中有4个800G的SSD，其中2个通过RAID卡，两个不经过RAID卡。这是背景，这4个SSD分别给OSD做Journal和Flashcache，压力应该是一样的，但是监控显示，压力差别很大，有两个SSD的 disk utils明显比另外两个高很多。

![](/assets/LINUX/ssd_iostat.jpeg)


为什么会如此。

首先sdb和sdg是其中2个SSD，disk utils明显高于另外两个SSD，那么问题就来了，这两个SSD是否都在RAID卡上，抑或是都不在RAID上，还是一个在RAID，另外一个不在RAID上？

# 排查

首先通过 /proc/scsi/scsi/信息可以得到如下信息：

```
root@Storage-b6:/sys/class/scsi_host/host1/device/port-1:0/end_device-1:0/target1:0:0/1:0:0:0# cat /proc/scsi/scsi
Attached devices:
Host: scsi0 Channel: 00 Id: 08 Lun: 00
  Vendor: LSI      Model: SAS2X36          Rev: 0e12
  Type:   Enclosure                        ANSI  SCSI revision: 05
Host: scsi0 Channel: 00 Id: 33 Lun: 00
  Vendor: LSI      Model: SAS2X28          Rev: 0e12
  Type:   Enclosure                        ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 00 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 01 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 02 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 03 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 04 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 05 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi0 Channel: 02 Id: 06 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
Host: scsi1 Channel: 00 Id: 00 Lun: 00
  Vendor: ATA      Model: INTEL SSDSC2BB80 Rev: 0370
  Type:   Direct-Access                    ANSI  SCSI revision: 06
Host: scsi1 Channel: 00 Id: 01 Lun: 00
  Vendor: ATA      Model: INTEL SSDSC2BB80 Rev: 0370
  Type:   Direct-Access                    ANSI  SCSI revision: 06
```

其中MR9271-8i是LSI的RAID卡，我们可以看到7个scsi设备是在RAID卡上的，我们通过MegaCli 命令也验证了这个结论

那问题就来了，我们disk util比较高的两个device，到底在不在RAID上呢？

问题其实就简化成了，一直到Host Channel Id LUN，如何确定对应的是/dev/sdX呢？


方法如下：

```
root@Storage-b6:/sys/class#  ls -ld /sys/block/sd*/device  
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sda/device -> ../../../0:2:0:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdb/device -> ../../../0:2:1:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdc/device -> ../../../0:2:2:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdd/device -> ../../../0:2:3:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sde/device -> ../../../0:2:4:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdf/device -> ../../../0:2:5:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdg/device -> ../../../0:2:6:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdh/device -> ../../../1:0:0:0
lrwxrwxrwx 1 root root 0 Sep  8 12:29 /sys/block/sdi/device -> ../../../1:0:1:0
```

很明显，可以看出，

```
sdb 0:2:1:0
sdg 0:2:6:0
```

对应的内容为：

```
Host: scsi0 Channel: 02 Id: 01 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
  
Host: scsi0 Channel: 02 Id: 06 Lun: 00
  Vendor: LSI      Model: MR9271-8i        Rev: 3.24
  Type:   Direct-Access                    ANSI  SCSI revision: 05
```

很明显，disk utils比较高的两块盘sdb和sdg都在RAID卡上。

我们通过MegaCli查到，SSD的条带比较大，是256KB，方便进一步的定位。

```
Virtual Drive: 6 (Target Id: 6)
Name                :
RAID Level          : Primary-0, Secondary-0, RAID Level Qualifier-0
Size                : 744.687 GB
Sector Size         : 512
Is VD emulated      : Yes
Parity Size         : 0
State               : Optimal
Strip Size          : 256 KB
Number Of Drives    : 1
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Enabled
Encryption Type     : None
PI type: No PI

Is VD Cached: No
```

其实本文引出来一个话题，对于用作flashcache的SSD，到底RAID stripe设置成多大比较合适。
网上有很多的benchmark，我们也有一些经验数据，不在此处赘述。



# 尾声

还有一个比较有用的工具，systool，能分析sysfs下的很多信息。虽然不能提供mapping关系，但是能提供设备的很多信息：

```
root@Storage-b6:/sys/class# systool -c scsi_disk -v  
Class = "scsi_disk"

  Class Device = "0:0:0"
  Class Device path = "/sys/devices/pci0000:00/0000:00:02.2/0000:03:00.0/host1/port-1:0/end_device-1:0/target1:0:0/1:0:0:0/scsi_disk/1:0:0:0"
    FUA                 = "1"
    allow_restart       = "1"
    app_tag_own         = "0"
    cache_type          = "write back"
    manage_start_stop   = "0"
    max_medium_access_timeouts= "2"
    max_write_same_blocks= "0"
    protection_mode     = "none"
    protection_type     = "0"
    provisioning_mode   = "unmap"
    thin_provisioning   = "1"
    uevent              = 

    Device = "1:0:0:0"
    Device path = "/sys/devices/pci0000:00/0000:00:02.2/0000:03:00.0/host1/port-1:0/end_device-1:0/target1:0:0/1:0:0:0"
      delete              = <store method only>
      device_blocked      = "0"
      dh_state            = "detached"
      evt_media_change    = "0"
      iocounterbits       = "32"
      iodone_cnt          = "0x31c20b0"
      ioerr_cnt           = "0x18de9"
      iorequest_cnt       = "0x31c20b0"
      modalias            = "scsi:t-0x00"
      model               = "INTEL SSDSC2BB80"
      queue_depth         = "32"
      queue_ramp_up_period= "120000"
      queue_type          = "none"
      rescan              = <store method only>
      rev                 = "0370"
      sas_address         = "0x4433221103000000"
      sas_device_handle   = "0x000a"
      scsi_level          = "7"
      state               = "running"
      timeout             = "30"
      type                = "0"
      uevent              = "DEVTYPE=scsi_device
DRIVER=sd
MODALIAS=scsi:t-0x00"
      vendor              = "ATA     "


...

  Class Device = "2:0:0"
  Class Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:0/0:2:0:0/scsi_disk/0:2:0:0"
    FUA                 = "0"
    allow_restart       = "0"
    app_tag_own         = "0"
    cache_type          = "write back"
    manage_start_stop   = "0"
    max_medium_access_timeouts= "2"
    max_write_same_blocks= "0"
    protection_mode     = "none"
    protection_type     = "0"
    provisioning_mode   = "full"
    thin_provisioning   = "0"
    uevent              = 

    Device = "0:2:0:0"
    Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:0/0:2:0:0"
      delete              = <store method only>
      device_blocked      = "0"
      dh_state            = "detached"
      evt_media_change    = "0"
      iocounterbits       = "32"
      iodone_cnt          = "0x280bfe"
      ioerr_cnt           = "0x2"
      iorequest_cnt       = "0x280bfe"
      modalias            = "scsi:t-0x00"
      model               = "MR9271-8i       "
      queue_depth         = "256"
      queue_ramp_up_period= "120000"
      queue_type          = "none"
      rescan              = <store method only>
      rev                 = "3.24"
      scsi_level          = "6"
      state               = "running"
      timeout             = "90"
      type                = "0"
      uevent              = "DEVTYPE=scsi_device
DRIVER=sd
MODALIAS=scsi:t-0x00"
      vendor              = "LSI     "


  Class Device = "2:1:0"
  Class Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:1/0:2:1:0/scsi_disk/0:2:1:0"
    FUA                 = "0"
    allow_restart       = "0"
    app_tag_own         = "0"
    cache_type          = "write back"
    manage_start_stop   = "0"
    max_medium_access_timeouts= "2"
    max_write_same_blocks= "0"
    protection_mode     = "none"
    protection_type     = "0"
    provisioning_mode   = "full"
    thin_provisioning   = "0"
    uevent              = 

    Device = "0:2:1:0"
    Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:1/0:2:1:0"
      delete              = <store method only>
      device_blocked      = "0"
      dh_state            = "detached"
      evt_media_change    = "0"
      iocounterbits       = "32"
      iodone_cnt          = "0x3111eb2"
      ioerr_cnt           = "0xa"
      iorequest_cnt       = "0x3111ee5"
      modalias            = "scsi:t-0x00"
      model               = "MR9271-8i       "
      queue_depth         = "256"
      queue_ramp_up_period= "120000"
      queue_type          = "none"
      rescan              = <store method only>
      rev                 = "3.24"
      scsi_level          = "6"
      state               = "running"
      timeout             = "90"
      type                = "0"
      uevent              = "DEVTYPE=scsi_device
DRIVER=sd
MODALIAS=scsi:t-0x00"
      vendor              = "LSI     "


  Class Device = "2:2:0"
  Class Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:2/0:2:2:0/scsi_disk/0:2:2:0"
    FUA                 = "0"
    allow_restart       = "0"
    app_tag_own         = "0"
    cache_type          = "write back"
    manage_start_stop   = "0"
    max_medium_access_timeouts= "2"
    max_write_same_blocks= "0"
    protection_mode     = "none"
    protection_type     = "0"
    provisioning_mode   = "full"
    thin_provisioning   = "0"
    uevent              = 

    Device = "0:2:2:0"
    Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:2/0:2:2:0"
      delete              = <store method only>
      device_blocked      = "0"
      dh_state            = "detached"
      evt_media_change    = "0"
      iocounterbits       = "32"
      iodone_cnt          = "0x8b6588"
      ioerr_cnt           = "0x2"
      iorequest_cnt       = "0x8b6588"
      modalias            = "scsi:t-0x00"
      model               = "MR9271-8i       "
      queue_depth         = "256"
      queue_ramp_up_period= "120000"
      queue_type          = "none"
      rescan              = <store method only>
      rev                 = "3.24"
      scsi_level          = "6"
      state               = "running"
      timeout             = "90"
      type                = "0"
      uevent              = "DEVTYPE=scsi_device
DRIVER=sd
MODALIAS=scsi:t-0x00"
      vendor              = "LSI     "


  Class Device = "2:3:0"
  Class Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:3/0:2:3:0/scsi_disk/0:2:3:0"
    FUA                 = "0"
    allow_restart       = "0"
    app_tag_own         = "0"
    cache_type          = "write back"
    manage_start_stop   = "0"
    max_medium_access_timeouts= "2"
    max_write_same_blocks= "0"
    protection_mode     = "none"
    protection_type     = "0"
    provisioning_mode   = "full"
    thin_provisioning   = "0"
    uevent              = 

    Device = "0:2:3:0"
    Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:3/0:2:3:0"
      delete              = <store method only>
      device_blocked      = "0"
      dh_state            = "detached"
      evt_media_change    = "0"
      iocounterbits       = "32"
      iodone_cnt          = "0x9b4088"
      ioerr_cnt           = "0x2"
      iorequest_cnt       = "0x9b4088"
      modalias            = "scsi:t-0x00"
      model               = "MR9271-8i       "
      queue_depth         = "256"
      queue_ramp_up_period= "120000"
      queue_type          = "none"
      rescan              = <store method only>
      rev                 = "3.24"
      scsi_level          = "6"
      state               = "running"
      timeout             = "90"
      type                = "0"
      uevent              = "DEVTYPE=scsi_device
DRIVER=sd
MODALIAS=scsi:t-0x00"
      vendor              = "LSI     "


...


  Class Device = "2:6:0"
  Class Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:6/0:2:6:0/scsi_disk/0:2:6:0"
    FUA                 = "0"
    allow_restart       = "0"
    app_tag_own         = "0"
    cache_type          = "write back"
    manage_start_stop   = "0"
    max_medium_access_timeouts= "2"
    max_write_same_blocks= "0"
    protection_mode     = "none"
    protection_type     = "0"
    provisioning_mode   = "full"
    thin_provisioning   = "0"
    uevent              = 

    Device = "0:2:6:0"
    Device path = "/sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0/host0/target0:2:6/0:2:6:0"
      delete              = <store method only>
      device_blocked      = "0"
      dh_state            = "detached"
      evt_media_change    = "0"
      iocounterbits       = "32"
      iodone_cnt          = "0x344b278"
      ioerr_cnt           = "0xa"
      iorequest_cnt       = "0x344b378"
      modalias            = "scsi:t-0x00"
      model               = "MR9271-8i       "
      queue_depth         = "256"
      queue_ramp_up_period= "120000"
      queue_type          = "none"
      rescan              = <store method only>
      rev                 = "3.24"
      scsi_level          = "6"
      state               = "running"
      timeout             = "90"
      type                = "0"
      uevent              = "DEVTYPE=scsi_device
DRIVER=sd
MODALIAS=scsi:t-0x00"
      vendor              = "LSI     "


root@Storage-b6:/sys/class# 

```