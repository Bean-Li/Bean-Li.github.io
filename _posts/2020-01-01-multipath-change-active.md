---
layout: post
title: Multipath 切换主路径
date: 2020-01-01 15:49:40
categories: Linux
tag: multipath
excerpt:  multipath 切换主路径的方法。
---
# 前言

本文记录multipath相关指令, Multipath的path
path_grouping_policy – Paths are grouped into path groups. The policy determines how path groups are formed. There are five different policies.
* failover: One path per priority group
* multibus: All paths in one priority group. This is the default.
* group_by_serial: One priority group per storage controller (serial number)
* group_by_prio: One priority group per priority value
* group_by_node_name: One priority group per target node name

有时候使用failover，需要切换主路径，本文记录切换主路径的方法

# 查看当前配置

```bash
root@host243:~# multipath -ll
2c2c5d201aba719ee dm-0 Bigtera ,VirtualStor_Scal
size=5.0T features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
| `- 49:0:0:0 sdg 8:96 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 48:0:0:0 sdf 8:80 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 46:0:0:0 sdd 8:48 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 47:0:0:0 sde 8:64 active ready running
```

# 切换主路径

从上面的输出可以看出，sdd/sde/sdf/sdg 都是同一个设备的4条不同的路径。如果设备是分布式存储提供的iSCSI设备，每条路径可能对应一个Gateway。有时候可能需要切换主路径，比如某个存储节点要关机维护，这种情况，要怎么做呢？

首先执行如下语句，进入交互页面：

```bash
multipathd -k 
```

通过如下指令可以实时查看某个multipath的多个group信息：

```bash
multipathd> show topology
2c2c5d201aba719ee dm-0 Bigtera ,VirtualStor_Scal
size=5.0T features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
| `- 49:0:0:0 sdg 8:96 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 48:0:0:0 sdf 8:80 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 46:0:0:0 sdd 8:48 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 47:0:0:0 sde 8:64 active ready running
multipathd>
```

对于我这个环境比较简单，如果存在多个，可以通过如下指令指定：

```bash
multipathd> show map 2c2c5d201aba719ee topology
2c2c5d201aba719ee dm-0 Bigtera ,VirtualStor_Scal
size=5.0T features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
| `- 49:0:0:0 sdg 8:96 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 48:0:0:0 sdf 8:80 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 46:0:0:0 sdd 8:48 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 47:0:0:0 sde 8:64 active ready running
multipathd>
```

从上面的输出可以看出，我们设备下有多条路径，第一条sdg 为group 1，第二个sdf为group 2， 如果想切换主路径到sdd，那么需要执行：

![](/assets/LINUX/multipath-change-active.png)

执行完毕后，可以执行quit 退出multipathd的交互，然后执行multipath -ll来确认主路径是否发生了变化。

有些时候，没有I/O流量的情况下，multipath -ll的输出并不会及时发生变化，不过没关系，引入一些IO，就可以看到active 路径发生了变化。

```bash
oot@host243:~# multipath -ll
2c2c5d201aba719ee dm-0 Bigtera ,VirtualStor_Scal
size=5.0T features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 49:0:0:0 sdg 8:96 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 48:0:0:0 sdf 8:80 active ready running
|-+- policy='round-robin 0' prio=1 status=active
| `- 46:0:0:0 sdd 8:48 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 47:0:0:0 sde 8:64 active ready running
```



# 验证

对于分布式存储的iSCSI，每一条路径对应某一个存储Gateway，如何判断sdX和存储Gateway的映射关系呢？

```bash
root@host243:~# iscsiadm  -m session -P 3
iSCSI Transport Class version 2.0-870
version 2.0-873
Target: iqn.2020-01.lenovo:test
	Current Portal: 10.16.172.105:3260,1
	Persistent Portal: 10.16.172.105:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.org.debian:01:f077d70d498
		Iface IPaddress: 10.16.172.243
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 16
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 120
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
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
		Host Number: 46	State: running
		scsi46 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdd		State: running
	Current Portal: 10.16.172.106:3260,1
	Persistent Portal: 10.16.172.106:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.org.debian:01:f077d70d498
		Iface IPaddress: 10.16.172.243
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 17
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 120
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
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
		Host Number: 47	State: running
		scsi47 Channel 00 Id 0 Lun: 0
			Attached scsi disk sde		State: running
	Current Portal: 10.16.172.107:3260,1
	Persistent Portal: 10.16.172.107:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.org.debian:01:f077d70d498
		Iface IPaddress: 10.16.172.243
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 18
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 120
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
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
		Host Number: 48	State: running
		scsi48 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdf		State: running
	Current Portal: 10.16.172.108:3260,1
	Persistent Portal: 10.16.172.108:3260,1
		**********
		Interface:
		**********
		Iface Name: default
		Iface Transport: tcp
		Iface Initiatorname: iqn.1993-08.org.debian:01:f077d70d498
		Iface IPaddress: 10.16.172.243
		Iface HWaddress: <empty>
		Iface Netdev: <empty>
		SID: 19
		iSCSI Connection State: LOGGED IN
		iSCSI Session State: LOGGED_IN
		Internal iscsid Session State: NO CHANGE
		*********
		Timeouts:
		*********
		Recovery Timeout: 120
		Target Reset Timeout: 30
		LUN Reset Timeout: 30
		Abort Timeout: 15
		*****
		CHAP:
		*****
		username: <empty>
		password: ********
		username_in: <empty>
		password_in: ********
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
		Host Number: 49	State: running
		scsi49 Channel 00 Id 0 Lun: 0
			Attached scsi disk sdg		State: running
root@host243:~#
```

通过上面的输出，可以看到 /dev/sdd对应的是10.16.172.105:3260 ，因此找到了对应关系。

修改完毕了主路径，如何从网路上判定已经发生了切换呢？

```
iftop -i bond0
```

通过iftop，查看网络流向，确定是否流向了正确的Gateway。

![image-20200101152049825](/assets/LINUX/iftop-bond0.png)