---
layout: post
title: 谁连了我的NFS目录： showmount
date: 2016-07-19 14:43:40
categories: linux
tag: nfs
excerpt: 哪些client端连了我的NFS 
---

# 前言
有些时候，作为NFS Server，需要了解哪些client端连了我的NFS 目录，这时候，showmount命令就闪亮登场了。

# 使用方法

showmount -a 可以查看，当前到底有哪些client连接了本Server exports的目录：

```
root@Storage-b3:/var/log# showmount -a 
All mount points on Storage-b3:
10.1.226.105:/var/share/ezfs/shareroot/mgt
10.1.227.1:/var/share/ezfs/shareroot/backup
10.1.227.201:/var/share/ezfs/shareroot/mgt
10.1.227.2:/var/share/ezfs/shareroot/backup
10.1.227.3:/var/share/ezfs/shareroot/backup
10.1.227.81:/var/share/ezfs/shareroot/mgt
10.1.227.85:/var/share/ezfs/shareroot/mgt
10.1.232.199:/var/share/ezfs/shareroot/mgt
```

# 尾声
showmount是一个比较综合的命令，还有很多其它的用法。可以通过man 查看。

