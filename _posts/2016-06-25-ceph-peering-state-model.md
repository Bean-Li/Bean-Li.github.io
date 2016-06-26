---
layout: post
title: ceph peering state model
date: 2016-06-26 14:43:40
categories: ceph-internal
tag: ceph-internal
excerpt: peer state machine 
---

## 前言

对ceph的学习，从filestore到了OSD和PG。 在工作中也遇到了三副本集群，由于一个node上的OSD无法启动，按照常理，由于其他副本存在，集群应该是可以提供服务的。但是发现有多个PG处于down＋peering的状态，ceph集群无法对外提供服务。

理解Ceph的状态机是非常关键的。我们从peering state machine 入手。

Loïc Dachary 有篇文章，[Ceph Placement Groups peering](http://dachary.org/?p=2061)，介绍了peering state machine的相关内容。其中很有意思的是，ceph源码提供了一种产生状态迁移图的脚本：

##  产生状态机

在ceph的源码路径下，执行如下命令：

```
cat src/osd/PG.h src/osd/PG.cc | doc/scripts/gen_state_diagram.py > ~/Pictures/peering_graph.generated.dot
```

就可以获得.dot 文件

```
digraph G {
	size="512,512"
	compound=true;
	subgraph cluster0 {
		label = "RecoveryMachine";
		color = "blue";
		Crashed;
		Initial[shape=Mdiamond];
		Reset;
		subgraph cluster1 {
			label = "Started";
			color = "blue";
			Start[shape=Mdiamond];
			subgraph cluster2 {
				label = "Primary";
				color = "blue";
				WaitActingChange;
				subgraph cluster3 {
					label = "Peering";
					color = "blue";
					GetInfo[shape=Mdiamond];
					GetLog;
					GetMissing;
					WaitUpThru;
					Incomplete;
				}
				subgraph cluster4 {
					label = "Active";
					color = "blue";
					Clean;
					Recovered;
					Backfilling;
					WaitRemoteBackfillReserved;
					WaitLocalBackfillReserved;
					NotBackfilling;
					Recovering;
					WaitRemoteRecoveryReserved;
					WaitLocalRecoveryReserved;
					Activating[shape=Mdiamond];
				}
			}
			subgraph cluster5 {
				label = "ReplicaActive";
				color = "blue";
				RepRecovering;
				RepWaitBackfillReserved;
				RepWaitRecoveryReserved;
				RepNotRecovering[shape=Mdiamond];
			}
			Stray;
		}
	}
GetInfo -> WaitActingChange [label="NeedActingChange",ltail=cluster2,];
Clean -> WaitLocalRecoveryReserved [label="DoRecovery",];
Activating -> WaitLocalRecoveryReserved [label="DoRecovery",];
Reset -> Start [label="ActMap",lhead=cluster1,];
Recovered -> Clean [label="GoClean",];
Start -> GetInfo [label="MakePrimary",lhead=cluster2,];
Initial -> Crashed [label="boost::statechart::event_base",];
Reset -> Crashed [label="boost::statechart::event_base",];
Start -> Crashed [label="boost::statechart::event_base",ltail=cluster1,];
GetLog -> GetMissing [label="GotLog",];
Initial -> GetInfo [label="MNotifyRec",lhead=cluster2,];
Incomplete -> GetLog [label="MNotifyRec",];
Initial -> Stray [label="MLogRec",];
Stray -> RepNotRecovering [label="MLogRec",lhead=cluster5,];
Activating -> Recovered [label="AllReplicasRecovered",];
Recovering -> Recovered [label="AllReplicasRecovered",];
WaitRemoteRecoveryReserved -> Recovering [label="AllRemotesReserved",];
Initial -> Reset [label="Initialize",];
RepNotRecovering -> RepWaitRecoveryReserved [label="RequestRecovery",];
NotBackfilling -> WaitLocalBackfillReserved [label="RequestBackfill",];
Activating -> WaitLocalBackfillReserved [label="RequestBackfill",];
Recovering -> WaitRemoteBackfillReserved [label="RequestBackfill",];
Initial -> Reset [label="Load",];
GetMissing -> WaitUpThru [label="NeedUpThru",];
RepWaitRecoveryReserved -> RepRecovering [label="RemoteRecoveryReserved",];
WaitLocalRecoveryReserved -> WaitRemoteRecoveryReserved [label="LocalRecoveryReserved",];
RepNotRecovering -> RepWaitBackfillReserved [label="RequestBackfillPrio",];
WaitRemoteBackfillReserved -> Backfilling [label="AllBackfillsReserved",];
Backfilling -> Recovered [label="Backfilled",];
Initial -> Stray [label="MInfoRec",];
Stray -> RepNotRecovering [label="MInfoRec",lhead=cluster5,];
RepRecovering -> RepNotRecovering [label="RecoveryDone",];
RepNotRecovering -> RepNotRecovering [label="RecoveryDone",];
RepRecovering -> RepNotRecovering [label="RemoteReservationRejected",];
Backfilling -> NotBackfilling [label="RemoteReservationRejected",];
WaitRemoteBackfillReserved -> NotBackfilling [label="RemoteReservationRejected",];
RepWaitBackfillReserved -> RepNotRecovering [label="RemoteReservationRejected",];
GetLog -> Incomplete [label="IsIncomplete",];
WaitLocalBackfillReserved -> WaitRemoteBackfillReserved [label="LocalBackfillReserved",];
GetInfo -> Activating [label="Activate",ltail=cluster3,lhead=cluster4,];
GetInfo -> GetLog [label="GotInfo",];
Start -> Reset [label="AdvMap",ltail=cluster1,];
GetInfo -> Reset [label="AdvMap",ltail=cluster3,];
GetLog -> Reset [label="AdvMap",];
WaitActingChange -> Reset [label="AdvMap",];
Incomplete -> Reset [label="AdvMap",];
RepWaitBackfillReserved -> RepRecovering [label="RemoteBackfillReserved",];
Start -> Stray [label="MakeStray",];
}

```

注意，默认的尺寸是"7,7"即：

```
digraph G {
	size="7,7"
	compound=true;
	subgraph cluster0 {
```

产生的图片非常模糊，所以被我调整成了512，512。

我们将dot文件转换成png图片

```
dot -Tpng peering_graph.generated.dot -o peering.png
```

图片内容如下：

![](/assets/ceph_internals/peering.png)

接下来一周的工作，就是深入地理解这个状态机的各个状态之间的迁移。

