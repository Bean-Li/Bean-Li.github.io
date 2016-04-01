---
layout: post
title: gdisk 常用操作
date: 2016-04-01 22:20:40
categories: linux
tag: linux
excerpt: gdisk是fdisk的继承人，很重要的分区工具
---
前言
----
传统的老牌的分区工具是fdisk，但是fdisk出道太早，只支持MBR（Master Boot Record），并不支持GPT（GUID Partition Table），无法操作超过2T的磁盘，因此gdisk parted等分区工具横空出世。

gdisk和parted都曾经用过，但是我更喜欢gdisk，因为使用上，gdisk 上承fdisk，没有太多的学习负担。另外我们QA测出过，parted方式分区，在某些使用上会有问题。


使用方法
-------

首先，我有一个sdd，作为我操作的对象。

```
root@node3:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
fd0      2:0    1     4K  0 disk 
sda      8:0    0    40G  0 disk 
├─sda1   8:1    0  30.5M  0 part 
├─sda2   8:2    0 488.3M  0 part 
├─sda3   8:3    0  31.1G  0 part /
├─sda4   8:4    0     8G  0 part [SWAP]
└─sda5   8:5    0 387.8M  0 part 
sdb      8:16   0    50G  0 disk 
├─sdb1   8:17   0     4G  0 part 
└─sdb2   8:18   0    46G  0 part /data/osd.4
sdc      8:32   0    50G  0 disk 
├─sdc1   8:33   0     4G  0 part 
└─sdc2   8:34   0    46G  0 part /data/osd.5
sdd      8:48   0    50G  0 disk 
sr0     11:0    1   1.6G  0 rom  
```

**查看help信息**

进入交互模式之后，输入h可以查看帮助信息

```
root@node3:~# gdisk /dev/sdd
GPT fdisk (gdisk) version 0.8.1

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): h  
b	back up GPT data to a file
c	change a partition's name
d	delete a partition
i	show detailed information on a partition
l	list known partition types
n	add a new partition
o	create a new empty GUID partition table (GPT)
p	print the partition table
q	quit without saving changes
r	recovery and transformation options (experts only)
s	sort partitions
t	change a partition's type code
v	verify disk
w	write table to disk and exit
x	extra functionality (experts only)
?	print this menu

Command (? for help): 

```

**查看分区表信息**

很明显，做出没有分区，所以分区表为空,后面创建分区之后，可以通过p命令查看分区信息

```
Command (? for help): p
Disk /dev/sdd: 104857600 sectors, 50.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): D3328858-7A3F-4A64-BC46-A7040F306F33
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 104857566
Partitions will be aligned on 2048-sector boundaries
Total free space is 104857533 sectors (50.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name

Command (? for help):
```

**创建分区**

打算将50G的空间划分成2个分区，分别是10G和40G：
创建分区是用命令n，你需要选择

1. 分区number
2. 起始扇区
3. 结束扇区

起始扇区可以敲回车选择默认值，而结束扇区用＋10G这种方式来决定分区大小为10G
分区后，可以用p命令查看最新的分区表

```
Command (? for help): n    
Partition number (1-128, default 1): 1
First sector (34-104857566, default = 34) or {+-}size{KMGTP}: 
Information: Moved requested sector from 34 to 2048 in
order to align on 2048-sector boundaries.
Use 'l' on the experts' menu to adjust alignment
Last sector (2048-104857566, default = 104857566) or {+-}size{KMGTP}: +10G
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sdd: 104857600 sectors, 50.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): D3328858-7A3F-4A64-BC46-A7040F306F33
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 104857566
Partitions will be aligned on 2048-sector boundaries
Total free space is 83886013 sectors (40.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20973567   10.0 GiB    8300  Linux filesystem

Command (? for help): n
Partition number (2-128, default 2): 2
First sector (34-104857566, default = 20973568) or {+-}size{KMGTP}: 
Last sector (20973568-104857566, default = 104857566) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sdd: 104857600 sectors, 50.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): D3328858-7A3F-4A64-BC46-A7040F306F33
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 104857566
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20973567   10.0 GiB    8300  Linux filesystem
   2        20973568       104857566   40.0 GiB    8300  Linux filesystem

Command (? for help): 

```

**设置分区标签信息partlabel**

有些时候，磁盘的盘符会漂移，使用sdx这种方式来分辨磁盘是不靠谱，使用lable这种方式是可靠的。那么如何创建分区标签呢？

c是用来设置partlable的

```
Command (? for help): c 
Partition number (1-2): 1
Enter name: bean_part1

Command (? for help): c
Partition number (1-2): 2
Enter name: bean_part2

Command (? for help): p
Disk /dev/sdd: 104857600 sectors, 50.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): D3328858-7A3F-4A64-BC46-A7040F306F33
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 104857566
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        20973567   10.0 GiB    8300  bean_part1
   2        20973568       104857566   40.0 GiB    8300  bean_part2

```

**保存分区信息**

我们将磁盘分成了2个区，给每一个分区贴上了partlabel，但是退出gdisk之前必须保存，否则前功尽弃。

```
Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT).
The operation has completed successfully.
root@node3:~# 

```

效果
-----
愉快地查看分区效果吧：

```
root@node3:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
fd0      2:0    1     4K  0 disk 
sda      8:0    0    40G  0 disk 
├─sda1   8:1    0  30.5M  0 part 
├─sda2   8:2    0 488.3M  0 part 
├─sda3   8:3    0  31.1G  0 part /
├─sda4   8:4    0     8G  0 part [SWAP]
└─sda5   8:5    0 387.8M  0 part 
sdb      8:16   0    50G  0 disk 
├─sdb1   8:17   0     4G  0 part 
└─sdb2   8:18   0    46G  0 part /data/osd.4
sdc      8:32   0    50G  0 disk 
├─sdc1   8:33   0     4G  0 part 
└─sdc2   8:34   0    46G  0 part /data/osd.5
sdd      8:48   0    50G  0 disk 
├─sdd1   8:49   0    10G  0 part 
└─sdd2   8:50   0    40G  0 part 
sr0     11:0    1   1.6G  0 rom  
root@node3:~# ll /dev/disk/by-partlabel/
total 0
drwxr-xr-x 2 root root 180 Apr  1 22:23 ./
drwxr-xr-x 9 root root 180 Apr  1 21:34 ../
lrwxrwxrwx 1 root root  10 Apr  1 22:23 bean_part1 -> ../../sdd1
lrwxrwxrwx 1 root root  10 Apr  1 22:23 bean_part2 -> ../../sdd2
...

```

