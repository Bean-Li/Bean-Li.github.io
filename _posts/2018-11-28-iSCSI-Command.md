---
layout: post
title: iSCSI command
date: 2018-11-28 23:12:40
categories: storage
tag: iSCSI
excerpt: iSCSI常用命令
---

# 前言

iSCSI客户端常用命令总是忘记，在此处记录下。

# 常用命令

##  查看当前session

挂载之前，一般如下图所示：

```
root@node-242:~# iscsiadm -m session 
iscsiadm: No active sessions.
```

挂载之后：

```
root@node-242:~# iscsiadm -m session 
tcp: [2] 10.16.172.247:3260,1 iqn.2018-11.com:BEAN

root@node-242:~# iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 2.0-871
Target: iqn.2018-11.com:BEAN
	Current Portal: 10.16.172.247:3260,1
	Persistent Portal: 10.16.172.247:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.org.debian:01:c9c12dd76e
		Iface IPaddress: 10.16.172.242
		Iface HWaddress: (null)
		Iface Netdev: (null)
		SID: 2
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 1048576
		FirstBurstLength: 262144
		MaxBurstLength: 1048576
		ImmediateData: Yes
		InitialR2T: No
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 25	State: running
		scsi25 Channel 00 Id 0 Lun: 0
			Attached scsi disk sde		State: running

```

##  根据IP 发现target

```
iscsiadm -m discovery -t st -p 10.16.172.246
```

输出如下：

```
root@node-242:~# iscsiadm -m discovery -t st -p 10.16.172.246
10.16.172.246:3260,1 iqn.2018-11.com:BEAN
10.16.172.247:3260,1 iqn.2018-11.com:BEAN
10.16.172.248:3260,1 iqn.2018-11.com:BEAN
```



##  登录到指定target

```
iscsiadm -m node -T [target_name] -p [ip:3260] -l
 如下所示：
iscsiadm -m node -T iqn.2018-11.com:BEAN -p 10.16.172.246:3260 -l
```

输出如下：

```
root@node-242:~# iscsiadm -m node -T iqn.2018-11.com:BEAN -p 10.16.172.246:3260 -l
Logging in to [iface: default, target: iqn.2018-11.com:BEAN, portal: 10.16.172.246,3260]
Login to [iface: default, target: iqn.2018-11.com:BEAN, portal: 10.16.172.246,3260]: successful
```

登录之后，可以用iscsiadm -m session查看。结果一般如下所示：

````
root@node-242:~# iscsiadm -m session 
tcp: [3] 10.16.172.246:3260,1 iqn.2018-11.com:BEAN
````

## 登出指定target

```
iscsiadm -m node -T [target_name] -p [ip:3260] -u
```

具体指令如下所示：

```
root@node-242:~# iscsiadm -m node -T iqn.2018-11.com:BEAN -p 10.16.172.246:3260 -u
Logging out of session [sid: 3, target: iqn.2018-11.com:BEAN, portal: 10.16.172.246,3260]
Logout of [sid: 3, target: iqn.2018-11.com:BEAN, portal: 10.16.172.246,3260]: successful
```

登出之后，可以用iscsiadm -m session 检查效果。

```
root@node-242:~# iscsiadm -m session 
iscsiadm: No active sessions.
```

# 信息

一般来讲，登录target之后会新增一个盘符，登录之前，lsblk输出如下：

```
root@node2:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0    30G  0 disk 
├─sda1   8:1    0     7M  0 part 
├─sda2   8:2    0  22.2G  0 part /
├─sda3   8:3    0   7.5G  0 part [SWAP]
└─sda4   8:4    0   261M  0 part 
sdb      8:16   0   100G  0 disk 
├─sdb1   8:17   0     8G  0 part 
└─sdb2   8:18   0    92G  0 part /data/osd.2
sdc      8:32   0     2T  0 disk 
sr0     11:0    1  1024M  0 rom 
```

执行登录target之后：

```
root@node2:~# iscsiadm -m node -T iqn.2018-11.com:BEAN -p 10.16.172.246:3260 -l
Logging in to [iface: default, target: iqn.2018-11.com:BEAN, portal: 10.16.172.246,3260] (multiple)
Login to [iface: default, target: iqn.2018-11.com:BEAN, portal: 10.16.172.246,3260] successful.
root@node2:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0    30G  0 disk 
├─sda1   8:1    0     7M  0 part 
├─sda2   8:2    0  22.2G  0 part /
├─sda3   8:3    0   7.5G  0 part [SWAP]
└─sda4   8:4    0   261M  0 part 
sdb      8:16   0   100G  0 disk 
├─sdb1   8:17   0     8G  0 part 
└─sdb2   8:18   0    92G  0 part /data/osd.2
sdc      8:32   0     2T  0 disk 
sr0     11:0    1  1024M  0 rom  
```

我们可以看到新增了一个sdc。

如果确定sdc和iSCSI target的关系呢:

```
iscsiadm -m session -P 3
```

比如之前的输出, sde这块磁盘即iSCSI，来自 10.16.172.247:3260的Target: iqn.2018-11.com:BEAN

```
root@node-242:~# iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 2.0-871
Target: iqn.2018-11.com:BEAN
	Current Portal: 10.16.172.247:3260,1
	Persistent Portal: 10.16.172.247:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.org.debian:01:c9c12dd76e
		Iface IPaddress: 10.16.172.242
		Iface HWaddress: (null)
		Iface Netdev: (null)
		SID: 2
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		************************
		Negotiated iSCSI params:
		************************
		HeaderDigest: None
		DataDigest: None
		MaxRecvDataSegmentLength: 262144
		MaxXmitDataSegmentLength: 1048576
		FirstBurstLength: 262144
		MaxBurstLength: 1048576
		ImmediateData: Yes
		InitialR2T: No
		MaxOutstandingR2T: 1
		************************
		Attached SCSI devices:
		************************
		Host Number: 25	State: running
		scsi25 Channel 00 Id 0 Lun: 0
			Attached scsi disk sde		State: running
```



