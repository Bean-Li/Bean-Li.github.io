---
layout: post
title: check e2fsck progress realtime
date: 2016-05-16 21:39:40
categories: linux
tag: linux
excerpt: 实时检查e2fsck 的进度
---

## 前言

对于e2fsck 而言，又两个特点，如果文件系统是健康的，可以很快完成，3秒之内解决战斗，但是确实存在error的情况下，可能耗时非常久，这种情况下，进度汇报是非常重要的，如果一个文件系统需要修复几个小时，又没有进度汇报，人会抓狂的。

本文提供一个实时汇报进度的方法

## 实时检查 e2fsck 进度的方法

在e2fsck 进行的时候，在另外一个终端向e2fsck 进程发送SIGUSR1信号。

```
    watch -n 5 kill -10 `pidof e2fsck`
```

在e2fsck调用的终端上，就会每5秒钟显示一下实时的进度。

```
e2fsck 1.42 (29-Nov-2011)
/dev/dm-19 contains a file system with errors, check forced.
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure                                           
Pass 3: Checking directory connectivity                                        
Pass 4: Checking reference counts                                              
Pass 5: Checking group summary information                                     
/dev/dm-19: |======================================================  \ 95.8%   
```


## 结束语

并非只有e2fsck 这个工具会响应SIGUSR1信号，dd工具也有同样的特点。dd 拷贝大文件的时候，只有在拷贝结束的时候，才会汇报时间以及速度等信息，但是像我种等待焦虑综合症的选手，一定是抓狂的，同样道理，通过实时向dd进程发送 SIGUSR1信号，dd进程就会时时汇报速率信息，不妨试试。

对于开发各种工具的人来说，如果进度可能耗时很久，通过注册SIGUSR1信号的处理函数，实时汇报进度是个不错的方法。

