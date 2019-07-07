---
layout: post
title: 解决UEFI安装无法启动的问题
date: 2019-07-07 12:40:40
categories: linux	
tag: linux
excerpt: 某些硬件UEFI安装之后无法启动，探索原理和解决办法
---

# 前言

我们产品是支持UEFI安装的，在很多款机器上都正常的安装。今日在浪潮服务器和技嘉服务器上都遇到一次，可以安装，但是无法正常启动。

所以我们必须要解决此问题，来支持更多的硬件。

# 基础知识

 EFI的全称是，Extensible Firmware Interface ， UEFI是Unified EFI。

我们以安装好的Linux系统为例，如何查看我们机器使用的是UEFI还是Legacy呢？

答案是查看如下文件是否存在：

```
/sys/firmware/efi
```

下面的输出分别是以Legacy和UEFI模式的情况下，各自的输出情况。

```bash
# 没有efi 目录，Legacy模式
root@node243:/sys/firmware# ll
total 0
drwxr-xr-x  4 root root 0 Jul  2 11:16 ./
dr-xr-xr-x 12 root root 0 Jul  2 11:15 ../
drwxr-xr-x  5 root root 0 Jul  2 16:43 acpi/
drwxr-xr-x 12 root root 0 Jul  2 16:43 memmap/

# 有efi 目录， UEFI模式
root@node245:/sys/firmware# ll
total 0
drwxr-xr-x  5 root root 0 Jul  2 16:33 ./
dr-xr-xr-x 12 root root 0 Jul  2 11:29 ../
drwxr-xr-x  5 root root 0 Jul  2 16:43 acpi/
drwxr-xr-x  5 root root 0 Jul  2 16:33 efi/
drwxr-xr-x 16 root root 0 Jul  2 16:43 memmap/
```

在我们的系统盘上，有单独的一个分区，是boot 分区：

```bash
root@node245:/sys/firmware# parted /dev/sda print 
Model: AVAGO MR9361-8i (scsi)
Disk /dev/sda: 16.0TB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start  End     Size    File system     Name  Flags
 1      262kB  538MB   538MB   fat32           EFI   boot, esp
 2      538MB  103GB   102GB   ext4
 3      103GB  131GB   27.6GB  linux-swap(v1)
 4      131GB  16.0TB  15.9TB
```

在fstab中也会有相应的条目：

```bash
root@node245:/sys/firmware# cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda2 during installation
UUID=efc5dbbf-8eb5-45ba-94b0-ce5253ed5d8d /               ext4    errors=remount-ro 0       1
# /boot/efi was on /dev/sda1 during installation
UUID=92A1-DB43  /boot/efi       vfat    defaults        0       1
# swap was on /dev/sda3 during installation
UUID=59a84533-d365-4e06-8a55-a2fa63b7e82e none            swap    sw              0       0
```

如何查看我们的引导项呢：

![](/assets/LINUX/efibootmgr_normal.png)

这个efibootmgr我们系统默认自带，如果安装模式是EFI的话。其中该命令会自动的显示当前所有启动项，包括启动顺序。

BootCurrent表示我们当前的系统是哪个启动项。因此我们当前的系统是：

```
Boot0000* grub	HD(1,200,100600,3a92c3e7-adc5-4fcc-8197-ac21b84b601d)File(\EFI\grub\grubx64.efi)
```

对于我们的系统来讲，EFI分区占据sda的第一个分区：

```bash
root@node245:/sys/firmware# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0  14.6T  0 disk 
├─sda1   8:1    0 512.8M  0 part /boot/efi
├─sda2   8:2    0  95.4G  0 part /
├─sda3   8:3    0  25.7G  0 part [SWAP]
└─sda4   8:4    0  14.4T  0 part 
```

其中 HD（1,200,100600, 3a92c3e7-adc5-4fcc-8197-ac21b84b601d）

* 1 : partition number

* 200 : partition offset 

* 100600 : partition size

* 3a92c3e7-adc5-4fcc-8197-ac21b84b601d : partition GUID:

  ```bash
  root@node245:/sys/firmware# ll /dev/disk/by-partlabel/
  total 0
  drwxr-xr-x 2 root root 180 Jul  2 17:35 ./
  drwxr-xr-x 9 root root 180 Jul  2 11:29 ../
  lrwxrwxrwx 1 root root  10 Jul  2 11:29 EFI -> ../../sda1
  ....
  root@node245:/sys/firmware# ll /dev/disk/by-partuuid
  ...
  lrwxrwxrwx 1 root root  10 Jul  2 11:29 3a92c3e7-adc5-4fcc-8197-ac21b84b601d -> ../../sda1
  ....
  ```

紧接着的是如下内容

```
File(\EFI\grub\grubx64.efi)
```

这指定了我们的OS bootloader 这个文件的类型如下：

```bash
root@node245:/boot/efi/EFI/grub# file grubx64.efi 
grubx64.efi: PE32+ executable (EFI application) x86-64 (stripped to external PDB), for MS Windows
```

一般来讲，正常安装好的系统，某个启动条目后面都有File指定某个文件，我们本次探索技嘉设备，安装好了之后并不能正确的启动，我们发现，启动条目后面并没有指定对应的File：

![](/assets/LINUX/gigabyte_uefi_items.png)

我们看到，上面输出中，并没有指定File，我们怀疑这也是为什么技嘉设备不能正常启动的原因，尽管我们在/boot/efi/EFI/grub/存在 grubx64.efi文件。

# Fallback Path （回退路径）

debian帮助文档中提到，一些机器的EFI可能存在bug：

* ### Weak EFI implementation only recognizes the fallback bootloader

某些比较脆弱的EFI实现，并不能认识我们的bootloader，导致启动的时候，他会查找默认的回退路径。

UEFI 规范定义了一种“回退”路径 (Fallback path)，用于启动此类启动管理器项，其工作原理类似于 BIOS 驱动器启动：它会在标准位置查找某些启动装载程序代码。但是其中的细节和 BIOS 不同。

当尝试以这种方式启动时，固件真正执行的操作相当简单。固件会遍历磁盘上的每个 EFI 系统分区（按照磁盘上的分区顺序）。在 ESP 内，固件将查找位于特定位置的具有特定名称的文件。在 x86-64 PC 上，固件会查找文件 \EFI\BOOT\BOOTx64.EFI。固件实际查找的是 \EFI\BOOT\BOOT{计算机类型简称}.EFI，其中，“x64”是 x86-64 PC 的“计算机类型简称”。文件名还有可能是 BOOTIA32.EFI (x86-32)、BOOTIA64.EFI (Itanium)、BOOTARM.EFI（AArch32，即32位ARM）和 BOOTAA64.EFI（AArch64，即64位ARM）。然后，固件将执行找到的第一个有效文件（当然，文件需要符合UEFI规范中定义的可执行格式）。

回到我们的情况，技嘉的机器可能只认 /boot/efi/EFI/BOOT/BOOTx64.EFI 这个回退路径，因此昨晚我和胜国，将我们自己的文件 /boot/efi/EFI/grub/grubx64.efi 拷贝到了 回退路径，这样，之前不能启动的路径就可以自如的启动了。

所以对于我们的问题，比较正确的处理方法是，grub-install执行完毕后， efibootmgr -v |grep grubx64.efi ，如果可以找到我们的File，那么什么也不做，如果找不到，则执行紧急补救不错，即cp我们的文件到回退路径。

# efibootmgr

这个工具非常有用，可以创建新的启动项：

## 创建新的启动项

- –create (-c) 表示要创建条目
- –part (-p) 用于提供ESP所在的分区号
- –disk (-d) 用于提供ESP所在的磁盘名称
- –label (-L) 用于提供条目名称
- –loader (-l) 用于提供要加载的EFI image

![](/assets/LINUX/efibootmgr_create.png)

## 删除某启动项

可以使用-B参数删除某个启动项：

```bash
       -b | --bootnum XXXX
              Modify BootXXXX (hex)
       -B | --delete-bootnum
              Delete bootnum
```

![](/assets/LINUX/efibootmgr_delete.png)

## 改变启动顺序

```
-o | --bootorder XXXX,YYYY,ZZZZ
Explicitly set BootOrder (hex).  Any value from 0 to FFFF is accepted so long as it corresponds to an existing Boot#### variable, and zero padding is not required.

```

出现在前面的启动项，启动优先级要高。

## 启用禁用启动项：

```
efibootmgr -a -b X  ==> 启用标号为X的启动项 
efibootmgr -A -b X  ==> 禁用标号为X的启动项
```



### 参考文献

* <https://blog.uncooperative.org/blog/2014/02/06/the-efi-system-partition/#fnref:3>
* <https://blog.woodelf.org/2014/05/28/uefi-boot-how-it-works.html>
* <http://www.rodsbooks.com/efi-bootloaders/installation.html#accessing>
* <https://lockless.github.io/2018/05/13/UEFI-tool/>