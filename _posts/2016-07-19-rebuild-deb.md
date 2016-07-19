---
layout: post
title: 重新打包deb
date: 2016-07-19 14:43:40
categories: linux
tag: 重新打包deb
excerpt: 
---

## 前言

有些时候需要重新打包deb文件。比如近期升级了内核，而linux-firmware中的固件版本太老，找到一个firmware_bnx2x deb 包，但是无奈有些文件和linux-firmware的文件冲突，因此，需要剔除掉重复的文件。这就牵扯的如何重打包。


## 方法

1 创建工作台目录，会讲deb的内容解压到extract目录

```
mkdir -p extract/DEBIAN
```


2 提取deb中的某些文件到extract目录

```
root@node-186:~# dpkg-deb -x firmware-bnx2x_20160110-1_all.deb  extract/
root@node-186:~# ll extract/
total 28
drwxr-xr-x  5 root root  4096 Jan 11  2016 ./
drwx------ 11 root root 12288 Jul 19 18:03 ../
drwxr-xr-x  2 root root  4096 Jul 19 18:03 DEBIAN/
drwxr-xr-x  3 root root  4096 Jan 11  2016 lib/
drwxr-xr-x  3 root root  4096 Jan 11  2016 usr/
```

3 提取control文件到extract/DEBIAN

```
dpkg-deb -e firmware-bnx2x_20160110-1_all.deb extract/DEBIAN
```

4 此事，所有的文件都已经就位，做你想做的任何修改

```
root@node-186:~/extract/lib/firmware/bnx2x# ll
total 3224
drwxr-xr-x 2 root root   4096 Jan 11  2016 ./
drwxr-xr-x 3 root root   4096 Jan 11  2016 ../
-rw-r--r-- 1 root root 161368 Jan 11  2016 bnx2x-e1-7.0.29.0.fw
-rw-r--r-- 1 root root 164392 Jan 11  2016 bnx2x-e1-7.10.51.0.fw
-rw-r--r-- 1 root root 170192 Jan 11  2016 bnx2x-e1-7.12.30.0.fw
-rw-r--r-- 1 root root 170096 Jan 11  2016 bnx2x-e1-7.13.1.0.fw
-rw-r--r-- 1 root root 163592 Jan 11  2016 bnx2x-e1-7.8.19.0.fw
-rw-r--r-- 1 root root 168680 Jan 11  2016 bnx2x-e1h-7.0.29.0.fw
-rw-r--r-- 1 root root 173016 Jan 11  2016 bnx2x-e1h-7.10.51.0.fw
-rw-r--r-- 1 root root 178984 Jan 11  2016 bnx2x-e1h-7.12.30.0.fw
-rw-r--r-- 1 root root 178992 Jan 11  2016 bnx2x-e1h-7.13.1.0.fw
-rw-r--r-- 1 root root 171920 Jan 11  2016 bnx2x-e1h-7.8.19.0.fw
-rw-r--r-- 1 root root 289848 Jan 11  2016 bnx2x-e2-7.0.29.0.fw
-rw-r--r-- 1 root root 321456 Jan 11  2016 bnx2x-e2-7.10.51.0.fw
-rw-r--r-- 1 root root 321320 Jan 11  2016 bnx2x-e2-7.12.30.0.fw
-rw-r--r-- 1 root root 320936 Jan 11  2016 bnx2x-e2-7.13.1.0.fw
-rw-r--r-- 1 root root 310440 Jan 11  2016 bnx2x-e2-7.8.19.0.fw

#下面文件，在Linux-firmware deb中也存在，所以冲突了，删除之，并重新打包
root@node-186:~/extract/lib/firmware/bnx2x# rm bnx2x-e1h-7.0.29.0.fw 
root@node-186:~/extract/lib/firmware/bnx2x# rm bnx2x-e1-7.0.29.0.fw
root@node-186:~/extract/lib/firmware/bnx2x# rm bnx2x-e2-7.0.29.0.fw
```

5 创建build子目录，并重新打包deb到build子目录下

```
mkdir build
dpkg-deb -b extract/ build
```

### 结尾语

注意，本文的终点是重新打包deb，并非处理deb包的冲突，其实本文重打包firmware-bnx2x并非优雅解决之道。

