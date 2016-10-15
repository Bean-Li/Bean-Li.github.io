---
layout: post
title: How to get scsi_id
date: 2016-08-21 14:43:40
categories: Storage
tag: 
excerpt: 获取scsi_id的几种方法
---

# 前言

有时候需要获取磁盘的SCSI ID，Linux下有多重方法可以做到

# lsscsi

lsscsi 0.27以及更新的版本提供了 -i选项

    --scsi_id|-i      show udev derived /dev/disk/by-id/scsi* entry

```
root@node1:~# lsscsi --scsi_id
[0:0:8:0]    enclosu AIC CORP SAS 6G Expander  0b01  -          -
[0:2:0:0]    disk    LSI      MR9271-8i        3.24  /dev/sda   3600605b009e978401e960bd0128244e7
[0:2:1:0]    disk    LSI      MR9271-8i        3.24  /dev/sdb   3600605b009e978401e20e9d80e73eb04
[0:2:2:0]    disk    LSI      MR9271-8i        3.24  /dev/sdc   3600605b009e978401e20e9d80e742269
[0:2:3:0]    disk    LSI      MR9271-8i        3.24  /dev/sdd   3600605b009e978401e960bd012824a5c
[3:0:0:0]    disk    ATA      INTEL SSDSC2BB24 0370  /dev/sde   SATA_INTEL_SSDSC2BB2BTWL30940045240MGN
```

# udev提供的scsi_id

/lib/udev/scsi_id这个tool是udev这个deb提供的：

```
root@node1:~# dpkg -S /lib/udev/scsi_id
udev: /lib/udev/scsi_id
```

```
root@node1:~# /lib/udev/scsi_id -g /dev/sda
3600605b009e978401e960bd0128244e7
root@node1:~# /lib/udev/scsi_id -g /dev/sdb
3600605b009e978401e20e9d80e73eb04
root@node1:~# /lib/udev/scsi_id -g /dev/sdc
3600605b009e978401e20e9d80e742269
root@node1:~# /lib/udev/scsi_id -g /dev/sdd
3600605b009e978401e960bd012824a5c
```

# /dev/disk/by-id

```
root@node1:~# ll /dev/disk/by-id/
total 0
drwxr-xr-x 2 root root 740 Oct 14 14:09 ./
drwxr-xr-x 9 root root 180 Oct 14 14:09 ../
lrwxrwxrwx 1 root root   9 Oct 14 14:10 ata-INTEL_SSDSC2BB240G4_BTWL30940045240MGN -> ../../sde
lrwxrwxrwx 1 root root  10 Oct 14 14:09 ata-INTEL_SSDSC2BB240G4_BTWL30940045240MGN-part1 -> ../../sde1
lrwxrwxrwx 1 root root  10 Oct 14 14:09 ata-INTEL_SSDSC2BB240G4_BTWL30940045240MGN-part2 -> ../../sde2
lrwxrwxrwx 1 root root  10 Oct 14 14:09 ata-INTEL_SSDSC2BB240G4_BTWL30940045240MGN-part3 -> ../../sde3
lrwxrwxrwx 1 root root  10 Oct 14 14:09 ata-INTEL_SSDSC2BB240G4_BTWL30940045240MGN-part4 -> ../../sde4
lrwxrwxrwx 1 root root   9 Oct 15 13:09 scsi-3600605b009e978401e20e9d80e73eb04 -> ../../sdb
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-3600605b009e978401e20e9d80e73eb04-part1 -> ../../sdb1
lrwxrwxrwx 1 root root   9 Oct 15 13:09 scsi-3600605b009e978401e20e9d80e742269 -> ../../sdc
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-3600605b009e978401e20e9d80e742269-part1 -> ../../sdc1
lrwxrwxrwx 1 root root   9 Oct 15 13:09 scsi-3600605b009e978401e960bd0128244e7 -> ../../sda
lrwxrwxrwx 1 root root  10 Oct 14 14:12 scsi-3600605b009e978401e960bd0128244e7-part1 -> ../../sda1
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-3600605b009e978401e960bd0128244e7-part2 -> ../../sda2
lrwxrwxrwx 1 root root   9 Oct 15 13:09 scsi-3600605b009e978401e960bd012824a5c -> ../../sdd
lrwxrwxrwx 1 root root  10 Oct 14 14:12 scsi-3600605b009e978401e960bd012824a5c-part1 -> ../../sdd1
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-3600605b009e978401e960bd012824a5c-part2 -> ../../sdd2
lrwxrwxrwx 1 root root   9 Oct 14 14:10 scsi-SATA_INTEL_SSDSC2BB2BTWL30940045240MGN -> ../../sde
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-SATA_INTEL_SSDSC2BB2BTWL30940045240MGN-part1 -> ../../sde1
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-SATA_INTEL_SSDSC2BB2BTWL30940045240MGN-part2 -> ../../sde2
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-SATA_INTEL_SSDSC2BB2BTWL30940045240MGN-part3 -> ../../sde3
lrwxrwxrwx 1 root root  10 Oct 14 14:09 scsi-SATA_INTEL_SSDSC2BB2BTWL30940045240MGN-part4 -> ../../sde4
```