---
layout: post
title: ceph写流程（2）
date: 2017-02-07 14:43:40
categories: ceph-internal
tag: ceph
excerpt: 通过debug log来整理学习ceph的写入流程
---

# 前言

通过设置debug\_osd/debug\_filestore/debug\_journal为20，写入一个3M的文件，然后通过log 梳理写入流程。

ceph的版本是giant。


# 写入测试
写入一个3MB的文件 un


```
dd if=/dev/zero of=un  bs=1M count=3
```
```
root@BEAN-2:/var/share/ezfs/shareroot# cephfs ./un map
WARNING: This tool is deprecated.  Use the layout.* xattrs to query and modify layouts.
    FILE OFFSET                    OBJECT        OFFSET        LENGTH  OSD
              0      10000000619.00000000             0       4194304  2
root@BEAN-2:/var/share/ezfs/shareroot# ceph osd map data 10000000619.00000000 
osdmap e323 pool 'data' (2) object '10000000619.00000000' -> pg 2.38fbabae (2.3ae) -> up ([2,0], p2) acting ([2,0], p2)
root@BEAN-2:/var/share/ezfs/shareroot# 
```

Primary OSD 为osd.2 
Replica OSD 为osd.0


我们今天分析的重点是FileStore部分，消息流通部分上面一篇博客已经讲过了。入口点为 queue_transactions。

0. OSD::osd_op_tp -> 提交写请求到journal队列
1. FileJournal::write_thread -> 发起aio请求
2. FileJournal::write_finish_thread ->监听aio的完成情况，完成某aio写入后，完成事先约定的回调。
3. 上一步中的queue_completions_thru承前启后，调用C_JournaledAhead回调，做了两件事：第一 将op放入FileStore队列，等待第二阶段，写入data partitions；第二 将实现约定好的回调 ondisk放入ondisk\_finisher的队列。
4. 根据上一步，花开两朵，各表一枝，首先回调部分会调用实现约好的C\_OSD\_OnOpCommit中的op\_commit函数，通知到PG，Primary OSD写入Journal的部分已经完成。
5. 另一支是： FileStore中的op\_tp线程池，会继续处理工作队列中的任务，通过\_do\_op函数处理事务，写入OSD的data partitions。事实上是写入Linux的Page Cache，并不会调用sync，不保证落盘。这一步是比较干货的一部，真正写入本地文件系统和omap;
6. 完成写入后，通过工作队列的\_process\_finish调用FileStore的\_finish\_op函数，将onreadable的回调插入op\_finisher。而onreadable回调即之前约定的C\_OSD\_OnOpApplied,会通知PG，该OSD第二阶段的任务也完成了，已经写入了OSD data partitions，数据可以读了。当然了遗留了隐患，即没有sync。
7. FileStore中的sync\_thread负责周期性地执行syncfs（对于EXT4和XFS 而言，btrfs不一样）。同时记录下一个Journal中的op_seq,写入OSD路径下的 current/commit\_op\_seq ，表示截止到这个op\_seq，Journal中数据和元数据，一定是执行过了syncfs，安全地落了盘（是否真正安全，也要考虑硬件的配置，安全指的是从本地文件系统层面安全）。因为Journal 是一块分区，一般4G～50G这种规模的大小。因为Journal的某些部分安全落了盘，所以Journal可以像ring buffer一样，安全地回滚，覆盖写入。
8. 如果出现了异常关机，那么从current/commit_op_seq对应的序列号之后的Journal上的事务，都需要replay，重新执行5～7


上面提到的部分，主要指的是Primary的OSD，对于Replica的OSD也是大差不差，主要的区别在第4步的回调和第6步的回调。

之所以有这种区别在于，Replica完成对应的MileStone之后，需要向Primary汇报，因此是需要透过网络，向Primary OSD发送消息，而Primary是不需要发送消息的。

| milestone | Primary回调上下文|Primary 回调函数 | Replica 回调上下文 | Replica 回调函数 
| ------| ------ | ------ | ------| ---------|
|完成Journal的写入| C\_OSD\_OnOpCommit | op_commit| C\_OSD\_RepModifyCommit | sub\_op\_modify\_commit
|完成OSD data partition的写入|C\_OSD\_OnOpApplied |op_applied| C\_OSD\_RepModifyApply | sub\_op\_modify\_applied



这里面有一个细节，即Replica OSD到底会向Primary OSD发送几个消息？

从上一篇博文上看，应当是两个消息，消息类型都是MOSDSubOpReply，区别在标志位。第一阶段写入Journal完成的，如果发送消息，则标志位为CEPH_OSD_FLAG_ONDISK，第二阶段写入Data Partition完成，如果发送消息，则标志位为CEPH_OSD_FLAG_ACK。 应该是2个消息。其实不然。

当Replica OSD完成Journal的写入，会通过C_OSD_RepModifyCommit上下文，调用sub_op_modify_commit函数，当Replica OSD完成OSD data partitions的时候，会通过C_OSD_RepModifyApply 上下文，调用sub_op_modify_applied。我们细细阅读代码，就能看出端倪。

```c
void ReplicatedBackend::sub_op_modify_applied(RepModifyRef rm)
{
  rm->op->mark_event("sub_op_applied");
  rm->applied = true; 

  dout(10) << "sub_op_modify_applied on " << rm << " op "
           << *rm->op->get_req() << dendl;
  MOSDSubOp *m = static_cast<MOSDSubOp*>(rm->op->get_req());
  assert(m->get_header().type == MSG_OSD_SUBOP);
  
  if (!rm->committed) {
    // send ack to acker only if we haven't sent a commit already
    MOSDSubOpReply *ack = new MOSDSubOpReply(
      m, parent->whoami_shard(),
      0, get_osdmap()->get_epoch(), CEPH_OSD_FLAG_ACK);
    ack->set_priority(CEPH_MSG_PRIO_HIGH); // this better match commit priority!
    get_parent()->send_message_osd_cluster(
      rm->ackerosd, ack, get_osdmap()->get_epoch());
  }
                                                                                                                                                                            
  parent->op_applied(m->version);
}
void ReplicatedBackend::sub_op_modify_commit(RepModifyRef rm)
{
  rm->op->mark_commit_sent();
  /*一上来先将commited标志设为true，后续执行sub_op_modify_applied的时候，会首先判断该标志位
   *如果发现该标志位为true，则sub_op_modify_applied就不会发送ACK给Primary OSD*/
  rm->committed = true; 

  // send commit.
  dout(10) << "sub_op_modify_commit on op " << *rm->op->get_req()                                                                                                           
           << ", sending commit to osd." << rm->ackerosd
           << dendl;
  
  assert(get_osdmap()->is_up(rm->ackerosd));
  get_parent()->update_last_complete_ondisk(rm->last_complete);
  
  /*无论是commit阶段还是apply阶段完成， Replica OSD都是发送同一种类型的消息，区别在于flag
   * commit完成，标志位为CEPH_OSD_FLAG_ONDISK，apply完成，标志位为CEPH_OSD_FLAG_ACK*/
  MOSDSubOpReply *commit = new MOSDSubOpReply(
    static_cast<MOSDSubOp*>(rm->op->get_req()),
    get_parent()->whoami_shard(),
    0, get_osdmap()->get_epoch(), CEPH_OSD_FLAG_ONDISK); 
  commit->set_last_complete_ondisk(rm->last_complete);
  commit->set_priority(CEPH_MSG_PRIO_HIGH); // this better match ack priority!
  get_parent()->send_message_osd_cluster(
    rm->ackerosd, commit, get_osdmap()->get_epoch());
  
  log_subop_stats(get_parent()->get_logger(), rm->op, l_osd_sop_w);
}
```

从上面的注释也不难看出，当第二阶段完成，只有当我们没有发送过commit完成的ACK消息的时候，才会发送带apply已经完成的消息到Primary OSD。反言之，如果已经发送过第一阶段的commit 已经完成的消息，就不会再发送apply阶段已经完成的ACK消息到Primary OSD。



因此，Replica OSD基本上只会发送一条ACK消息到Primary OSD。细细揣摩，这也是合理的，无论是第一阶段commit完成，还是第二阶段apply完成，都至少说明数据写入了持久化的Journal上，哪怕后面还没来得及写入data partition，OSD重启之后也可以通过Journal Replay将数据写入data partitions。

我们看一下Primary OSD收到Replica OSD发过来的ACK之后的处理逻辑：

```c
     if (r->ack_type & CEPH_OSD_FLAG_ONDISK) {
      assert(ip_op.waiting_for_commit.count(from));
      ip_op.waiting_for_commit.erase(from);                                                                                                                                 
      if (ip_op.op)
        ip_op.op->mark_event("sub_op_commit_rec");
    } else {
      assert(ip_op.waiting_for_applied.count(from));
      if (ip_op.op)
        ip_op.op->mark_event("sub_op_applied_rec");
    } 
    /*无论是哪种消息，都要从waiting_for_applied中删除该Replica OSD
     *原因上面已经提到*/
    ip_op.waiting_for_applied.erase(from);

    parent->update_peer_last_complete_ondisk(
      from,
      r->get_last_complete_ondisk());

    if (ip_op.waiting_for_applied.empty() &&
        ip_op.on_applied) {
      /*调用InProgressOp中on_applied上下文中的回调
       *如果Primary OSD早早完成了第二阶段apply，则需要在此处回调
       *如果Primary OSD 还未完成第二阶段apply，则无需回调
       *另一个可能调用InProgressOp中on_applied上下文中的回调的时机是下面提到的
       *Primary完成第二阶段之后，op_applied函数中，会判断是需要回调*/
      ip_op.on_applied->complete(0);
      ip_op.on_applied = 0;
    }   
    if (ip_op.waiting_for_commit.empty() &&
        ip_op.on_commit) {
      ip_op.on_commit->complete(0);
      ip_op.on_commit= 0;
    }   
    if (ip_op.done()) {
      assert(!ip_op.on_commit && !ip_op.on_applied);
      in_progress_ops.erase(iter);
    } 
```

我初读这个代码的时候，始终写不明白，为什么无论收到commit ACK还是apply ACK都要执行

```
ip_op.waiting_for_applied.erase(from);
```

这条语句，总觉得这句话应该放在else里面，只有收到Replica OSD发来的apply ACK才执行这句话。其则不然，因为Replica OSD一般一会发送一条消息给Primary OSD，无论是commit还是apply，都从waiting_for_applied中抹掉该Replica OSD。



这里面暗含的逻辑是，对于写入，所有的OSD都必须完成commit阶段，才叫完成了commit阶段，才会通知client写入已经完成，但是第二阶段apply，不需要每个OSD都完成，只需要Primary OSD完成该阶段，就可以通知client数据处于可读的状态。事实上，读取，仅仅是读取Primary OSD上的数据，不会去Replica OSD请求数据。



因此Primary OSD完成第二阶段apply，Primary OSD从waiting_for_applied中清除，才有机会调用InProgressOp类中的on_applied->complete函数。

```c++
void ReplicatedBackend::op_applied(
  InProgressOp *op)
{
  dout(10) << __func__ << ": " << op->tid << dendl;
  if (op->op)
    op->op->mark_event("op_applied");

  op->waiting_for_applied.erase(get_parent()->whoami_shard());
  parent->op_applied(op->v);

  if (op->waiting_for_applied.empty()) {
    op->on_applied->complete(0);
    op->on_applied = 0;
  }
  if (op->done()) {
    assert(!op->on_commit && !op->on_applied);
    in_progress_ops.erase(op->tid);
  }
}
```

而InProgressOp中的on_applied定义自issue_repop函数中的如下语句：

```c
  Context *on_all_commit = new C_OSD_RepopCommit(this, repop);
  Context *on_all_applied = new C_OSD_RepopApplied(this, repop);
```

该上下文的定义如下：

```c++
class C_OSD_RepopCommit : public Context {
  ReplicatedPGRef pg;
  boost::intrusive_ptr<ReplicatedPG::RepGather> repop;
public:
  C_OSD_RepopCommit(ReplicatedPG *pg, ReplicatedPG::RepGather *repop)
      : pg(pg), repop(repop) {}
  void finish(int) {
    pg->repop_all_committed(repop.get());
  }
};

void ReplicatedPG::repop_all_committed(RepGather *repop)
{
  dout(10) << __func__ << ": repop tid " << repop->rep_tid << " all committed "
           << dendl;
  repop->all_committed = true; /*所有的OSD都已经完成了第一阶段commit，而且是确实完成了第一阶段*/

  if (!repop->rep_aborted) {
    if (repop->v != eversion_t()) {
      last_update_ondisk = repop->v;
      last_complete_ondisk = repop->pg_local_last_complete;
    }
    eval_repop(repop);
  }
}
class C_OSD_RepopApplied : public Context {
  ReplicatedPGRef pg;
  boost::intrusive_ptr<ReplicatedPG::RepGather> repop;
public:
  C_OSD_RepopApplied(ReplicatedPG *pg, ReplicatedPG::RepGather *repop)
  : pg(pg), repop(repop) {}
  void finish(int) {
    pg->repop_all_applied(repop.get());
  }
};


void ReplicatedPG::repop_all_applied(RepGather *repop)
{
  dout(10) << __func__ << ": repop tid " << repop->rep_tid << " all applied "
           << dendl;
  /*所有的OSD都已经applied，
   *根据上面的讨论，事实上并不一定
   *但是这是可以保证所有的OSD完成commit + PrimaryOSD完成apply*/
  repop->all_applied = true; 
  if (!repop->rep_aborted) {
    eval_repop(repop);                                                                                                                                                      
    if (repop->on_applied) {
     repop->on_applied->complete(0);
     repop->on_applied = NULL;
    }
  }
}
```

我们可以看到，repop_all_committed函数也好，repop_all_applied也罢，最终都要执行同一个函数：eval_repop。这个函数是负责和client交互的，所有的OSD都commit了，就发送写入完成ACK给client端，所有的OSD都applied，就发送数据可读 ACK给client端。






# Primary OSD log on write



```
2017-02-06 18:01:05.258532 7fce051fd700 15 osd.2 323 enqueue_op 0x7fce2e065a00 prio 127 cost 3145728 latency 0.009482 osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728 [1@-1]] 2.38fbabae snapc 1=[] ondisk+write e323) v4
2017-02-06 18:01:05.259124 7fce17fe6700 10 osd.2 323 dequeue_op 0x7fce2e065a00 prio 127 cost 3145728 latency 0.010074 osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728 [1@-1]] 2.38fbabae snapc 1=[] ondisk+write e323) v4 pg pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean]
2017-02-06 18:01:05.259173 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] op_has_sufficient_caps pool=2 (data ) owner=0 need_read_cap=0 need_write_cap=1 need_class_read_cap=0 need_class_write_cap=0 -> yes
2017-02-06 18:01:05.259197 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] handle_message: 0x7fce2e065a00
2017-02-06 18:01:05.259218 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] do_op osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728 [1@-1]] 2.38fbabae snapc 1=[] ondisk+write e323) v4 may_write -> write-ordered flags ondisk+write
2017-02-06 18:01:05.259256 7fce17fe6700 15 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/head//2 '_'
2017-02-06 18:01:05.259378 7fce17fe6700 10 filestore(/data/osd.2) error opening file /data/osd.2/current/2.3ae_head/10000000619.00000000__head_38FBABAE__2 with flags=2: (2) No such file or directory
2017-02-06 18:01:05.259397 7fce17fe6700 10 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/head//2 '_' = -2
2017-02-06 18:01:05.259401 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] get_object_context: no obc for soid 38fbabae/10000000619.00000000/head//2 but can_create
2017-02-06 18:01:05.259414 7fce17fe6700 15 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/head//2 'snapset'
2017-02-06 18:01:05.259433 7fce17fe6700 10 filestore(/data/osd.2) error opening file /data/osd.2/current/2.3ae_head/10000000619.00000000__head_38FBABAE__2 with flags=2: (2) No such file or directory
2017-02-06 18:01:05.259438 7fce17fe6700 10 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/head//2 'snapset' = -2
2017-02-06 18:01:05.259441 7fce17fe6700 15 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/snapdir//2 'snapset'
2017-02-06 18:01:05.259464 7fce17fe6700 10 filestore(/data/osd.2) error opening file /data/osd.2/current/2.3ae_head/10000000619.00000000__snapdir_38FBABAE__2 with flags=2: (2) No such file or directory
2017-02-06 18:01:05.259475 7fce17fe6700 10 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/snapdir//2 'snapset' = -2
2017-02-06 18:01:05.259487 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] create_object_context 0x7fce2e2a9900 38fbabae/10000000619.00000000/head//2 
2017-02-06 18:01:05.259513 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] populate_obc_watchers 38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.259523 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] ReplicatedPG::check_blacklisted_obc_watchers for obc 38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.259531 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] get_object_context: 0x7fce2e2a9900 38fbabae/10000000619.00000000/head//2 rwstate(none n=0 w=0) oi: 38fbabae/10000000619.00000000/head//2(0'0 unknown.0.0:0 wrlock_by=unknown.0.0:0 s 0 uv0) ssc: 0x7fce03eb3400 snapset: 0=[]:[]
2017-02-06 18:01:05.259543 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] find_object_context 38fbabae/10000000619.00000000/head//2 @head oi=38fbabae/10000000619.00000000/head//2(0'0 unknown.0.0:0 wrlock_by=unknown.0.0:0 s 0 uv0)
2017-02-06 18:01:05.259560 7fce17fe6700 15 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/snapdir//2 '_'
2017-02-06 18:01:05.259582 7fce17fe6700 10 filestore(/data/osd.2) error opening file /data/osd.2/current/2.3ae_head/10000000619.00000000__snapdir_38FBABAE__2 with flags=2: (2) No such file or directory
2017-02-06 18:01:05.259591 7fce17fe6700 10 filestore(/data/osd.2) getattr 2.3ae_head/38fbabae/10000000619.00000000/snapdir//2 '_' = -2
2017-02-06 18:01:05.259593 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] get_object_context: no obc for soid 38fbabae/10000000619.00000000/snapdir//2 and !can_create
2017-02-06 18:01:05.259605 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] execute_ctx 0x7fce02a6ce00
2017-02-06 18:01:05.259615 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] do_op 38fbabae/10000000619.00000000/head//2 [write 0~3145728 [1@-1]] ov 0'0 av 323'4 snapc 1=[] snapset 0=[]:[]
2017-02-06 18:01:05.259626 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] do_osd_op 38fbabae/10000000619.00000000/head//2 [write 0~3145728 [1@-1]]
2017-02-06 18:01:05.259634 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] do_osd_op  write 0~3145728 [1@-1]
2017-02-06 18:01:05.259652 7fce17fe6700 15 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] do_osd_op_effects on session 0x7fcdfd843800
2017-02-06 18:01:05.259669 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] make_writeable 38fbabae/10000000619.00000000/head//2 snapset=0x7fce03eb3438  snapc=1=[]
2017-02-06 18:01:05.259679 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean]  setting DIRTY flag
2017-02-06 18:01:05.259687 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] make_writeable 38fbabae/10000000619.00000000/head//2 done, snapset=1=[]:[]+head
2017-02-06 18:01:05.259695 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] finish_ctx 38fbabae/10000000619.00000000/head//2 0x7fce02a6ce00 op modify  
2017-02-06 18:01:05.259705 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean]  set mtime to 2017-02-06 18:01:00.229845
2017-02-06 18:01:05.259732 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean]  final snapset 1=[]:[]+head in 38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.259754 7fce17fe6700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean]  zeroing write result code 0
2017-02-06 18:01:05.259769 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] new_repop rep_tid 224265 on osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4
2017-02-06 18:01:05.259783 7fce17fe6700  7 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] issue_repop rep_tid 224265 o 38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.259815 7fce17fe6700 20 osd.2 323 share_map_peer 0x7fce2e03e600 already has epoch 323
2017-02-06 18:01:05.259832 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] append_log log((0'0,172'3], crt=172'1) [323'4 (0'0) modify   38fbabae/10000000619.00000000/head//2 by client.48424379.1:4 2017-02-06 18:01:00.229845]
2017-02-06 18:01:05.259860 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] add_log_entry 323'4 (0'0) modify   38fbabae/10000000619.00000000/head//2 by client.48424379.1:4 2017-02-06 18:01:00.229845
2017-02-06 18:01:05.259879 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] append_log: trimming to 0'0 entries 
2017-02-06 18:01:05.259901 7fce17fe6700 10 write_log with: dirty_to: 0'0, dirty_from: 4294967295'18446744073709551615, dirty_divergent_priors: 0, writeout_from: 323'4, trimmed: 


从此处开始进入FileStore部分，入口函数为queue_transactions
2017-02-06 18:01:05.259932 7fce17fe6700  5 filestore(/data/osd.2) queue_transactions existing osr(2.3ae 0x7fce2c46cbc0)/0x7fce2c46cbc0
2017-02-06 18:01:05.259944 7fce17fe6700 10 journal op_submit_start 3835063

XFS和EXT4都是writeahead，btrfs 自己是parallel
2017-02-06 18:01:05.259947 7fce17fe6700  5 filestore(/data/osd.2) queue_transactions (writeahead) 3835063 0x7fce09d28980

调用_op_journal_transactions函数将当前的写入请求推入到FileJournal类中的writeq，并通过条件变量通知到write_thread_entry

2017-02-06 18:01:05.259948 7fce17fe6700 10 journal op_journal_transactions 3835063 0x7fce09d28980
2017-02-06 18:01:05.259952 7fce17fe6700  5 journal submit_entry seq 3835063 len 3147522 (0x7fce2e360b20)
2017-02-06 18:01:05.259960 7fce17fe6700 10 journal op_submit_finish 3835063
2017-02-06 18:01:05.259963 7fce17fe6700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] eval_repop repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=0 applied?=0 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4) wants=d


2017-02-06 18:01:05.259978 7fce17fe6700 10 osd.2 323 dequeue_op 0x7fce2e065a00 finish
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
FileJournal类中write_thread成员对应的线程被条件变量唤醒，调用linux aio函数，直接写入journal

2017-02-06 18:01:05.261443 7fce3bdf3700 20 journal write_thread_entry woke up
2017-02-06 18:01:05.261472 7fce3bdf3700 10 journal room 4294963199 max_size 4294967296 pos 790171648 header.start 790171648 top 4096
2017-02-06 18:01:05.261478 7fce3bdf3700 10 journal check_for_full at 790171648 : 3153920 < 4294963199
2017-02-06 18:01:05.261496 7fce3bdf3700 15 journal prepare_single_write 1 will write 790171648 : seq 3835063 len 3147522 -> 3153920 (head 40 pre_pad 2739 ebl 3147522 post_pad 3579 tail 40) (ebl alignment 2779)
2017-02-06 18:01:05.261734 7fce3bdf3700 20 journal prepare_multi_write queue_pos now 793325568
2017-02-06 18:01:05.261765 7fce3bdf3700 15 journal do_aio_write writing 790171648~3153920 + header
2017-02-06 18:01:05.261770 7fce3bdf3700 20 journal write_aio_bl 0~4096 seq 0
2017-02-06 18:01:05.261779 7fce3bdf3700 20 journal write_aio_bl .. 0~4096 in 1
2017-02-06 18:01:05.261974 7fce3bdf3700 20 journal write_aio_bl 790171648~3153920 seq 3835063
2017-02-06 18:01:05.261994 7fce3bdf3700 20 journal write_aio_bl .. 790171648~3153920 in 3
2017-02-06 18:01:05.281449 7fce3bdf3700 20 journal write_thread_entry going to sleep


－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
上面一段的作用是发起aio，下面到了write_finish_thread_entry，这个线程的作用是听，听aio操作是否完成，如果完成调用相关的Finisher回调

2017-02-06 18:01:05.281451 7fce3b5f2700 20 journal write_finish_thread_entry waiting for aio(s)
2017-02-06 18:01:05.281553 7fce3b5f2700 10 journal write_finish_thread_entry aio 0~4096 done
2017-02-06 18:01:05.281560 7fce3b5f2700 20 journal check_aio_completion
2017-02-06 18:01:05.281561 7fce3b5f2700 20 journal check_aio_completion completed seq 0 0~4096
2017-02-06 18:01:05.281566 7fce3b5f2700 20 journal write_finish_thread_entry waiting for aio(s)
2017-02-06 18:01:05.281757 7fce3b5f2700 10 journal write_finish_thread_entry aio 790171648~3153920 done
2017-02-06 18:01:05.281770 7fce3b5f2700 20 journal check_aio_completion

下面两句表示seq 3835063对应写，写入Journal部分已经完成
2017-02-06 18:01:05.281771 7fce3b5f2700 20 journal check_aio_completion completed seq 3835063 790171648~3153920
2017-02-06 18:01:05.281776 7fce3b5f2700 20 journal check_aio_completion queueing finishers through seq 3835063


queue_completions_thru是个非常重要的函数,最重要的是调用回调函数：

2017-02-06 18:01:05.281778 7fce3b5f2700 10 journal queue_completions_thru seq 3835063 queueing seq 3835063 0x7fce2e360b20 lat 0.021823
2017-02-06 18:01:05.281792 7fce3b5f2700 20 journal write_finish_thread_entry sleeping
2017-02-06 18:01:05.281803 7fce3a9f1700  5 filestore(/data/osd.2) _journaled_ahead 0x7fce449386f0 seq 3835063 osr(2.3ae 0x7fce2c46cbc0) 0x7fce09d28980
2017-02-06 18:01:05.281811 7fce3a9f1700  5 filestore(/data/osd.2) queue_op 0x7fce449386f0 seq 3835063 osr(2.3ae 0x7fce2c46cbc0) 3147516 bytes   (queue has 1 ops and 3147516 bytes)
2017-02-06 18:01:05.281840 7fce3a9f1700 10 filestore(/data/osd.2)  queueing ondisk 0x7fce03ffc1e0
2017-02-06 18:01:05.281857 7fce39bff700 10 journal op_apply_start 3835063 open_ops 0 -> 1
2017-02-06 18:01:05.281865 7fce39bff700  5 filestore(/data/osd.2) _do_op 0x7fce449386f0 seq 3835063 osr(2.3ae 0x7fce2c46cbc0)/0x7fce2c46cbc0 start
2017-02-06 18:01:05.281870 7fce39bff700 10 filestore(/data/osd.2) _do_transaction on 0x7fce09d28980
2017-02-06 18:01:05.281883 7fce39bff700 15 filestore(/data/osd.2) _omap_setkeys 2.3ae_head/3ae//head//2
2017-02-06 18:01:05.281876 7fce34bff700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] op_commit: 224265
2017-02-06 18:01:05.282142 7fce39bff700 10 filestore oid: 3ae//head//2 not skipping op, *spos 3835063.0.0
2017-02-06 18:01:05.282154 7fce39bff700 10 filestore  > header.spos 0.0.0
2017-02-06 18:01:05.282221 7fce39bff700 15 filestore(/data/osd.2) _omap_setkeys 2.3ae_head/3ae//head//2
2017-02-06 18:01:05.282267 7fce39bff700 10 filestore oid: 3ae//head//2 not skipping op, *spos 3835063.0.1
2017-02-06 18:01:05.282271 7fce39bff700 10 filestore  > header.spos 0.0.0
2017-02-06 18:01:05.282310 7fce39bff700 15 filestore(/data/osd.2) write 2.3ae_head/38fbabae/10000000619.00000000/head//2 0~3145728
2017-02-06 18:01:05.285252 7fce39bff700 10 filestore(/data/osd.2) write 2.3ae_head/38fbabae/10000000619.00000000/head//2 0~3145728 = 3145728
2017-02-06 18:01:05.285329 7fce39bff700 15 filestore(/data/osd.2) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.285368 7fce39bff700 10 filestore(/data/osd.2) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2 = 0
2017-02-06 18:01:05.285386 7fce39bff700 20 filestore(/data/osd.2) fgetattrs 5522 getting '_'
2017-02-06 18:01:05.285392 7fce39bff700 15 filestore(/data/osd.2) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.285404 7fce39bff700 10 filestore(/data/osd.2) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2 = 0
2017-02-06 18:01:05.285410 7fce39bff700 10 journal op_apply_finish 3835063 open_ops 1 -> 0
2017-02-06 18:01:05.285412 7fce39bff700 10 filestore(/data/osd.2) _do_op 0x7fce449386f0 seq 3835063 r = 0, finisher 0x7fce09c88f40 0x7fce2e19da00
2017-02-06 18:01:05.285417 7fce39bff700 10 filestore(/data/osd.2) _finish_op 0x7fce449386f0 seq 3835063 osr(2.3ae 0x7fce2c46cbc0)/0x7fce2c46cbc0
2017-02-06 18:01:05.285452 7fce35bf7700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] op_applied: 224265
2017-02-06 18:01:05.285498 7fce35bf7700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] op_applied version 323'4
2017-02-06 18:01:05.301501 7fce08bf3700 10 osd.2 323 handle_replica_op osd_sub_op_reply(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] ondisk, result = 0) v2 epoch 323
2017-02-06 18:01:05.301556 7fce08bf3700 20 osd.2 323 should_share_map osd.0 10.11.12.1:6803/483148 323
2017-02-06 18:01:05.301575 7fce08bf3700 15 osd.2 323 enqueue_op 0x7fce2f4bcc00 prio 196 cost 0 latency 0.000121 osd_sub_op_reply(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] ondisk, result = 0) v2
2017-02-06 18:01:05.301620 7fce157e1700 10 osd.2 323 dequeue_op 0x7fce2f4bcc00 prio 196 cost 0 latency 0.000166 osd_sub_op_reply(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] ondisk, result = 0) v2 pg pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean]
2017-02-06 18:01:05.301663 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] handle_message: 0x7fce2f4bcc00

--------------------------------------------------------------------------------------------
注意下面语句中的sub_op_modify_reply函数，该函数处理Replica OSD发过来的ACK，ack_type 4 表示消息的FLAG为CEPH_OSD_FLAG_ONDISK，因此Primary OSD收到的是第一阶段commit完成的ACK，继续往下找，是不会找到第二条sub_op_modify_reply相关的日志，原因无他，就是因为 Replica OSD不会发送两条消息。

2017-02-06 18:01:05.301683 7fce157e1700  7 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] sub_op_modify_reply: tid 224265 op  ack_type 4 from 0
2017-02-06 18:01:05.301704 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] repop_all_applied: repop tid 224265 all applied 
2017-02-06 18:01:05.301718 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] eval_repop repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=0 applied?=1 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4) wants=d
2017-02-06 18:01:05.301756 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 luod=172'3 crt=172'1 lcod 0'0 mlcod 0'0 active+clean] repop_all_committed: repop tid 224265 all committed 
2017-02-06 18:01:05.301773 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 0'0 active+clean] eval_repop repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=1 applied?=1 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4) wants=d
2017-02-06 18:01:05.301795 7fce157e1700 15 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 0'0 active+clean] log_op_stats osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4 inb 3145868 outb 0 rlat 0.052703 lat 0.052743
2017-02-06 18:01:05.301820 7fce157e1700 15 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 0'0 active+clean] publish_stats_to_osd 323:307
2017-02-06 18:01:05.301835 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 0'0 active+clean]  sending commit on repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=1 applied?=1 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4) 0x7fce03f92f00
2017-02-06 18:01:05.301874 7fce157e1700 10 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 172'3 active+clean]  removing repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=1 applied?=1 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4)
2017-02-06 18:01:05.301895 7fce157e1700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 172'3 active+clean]    q front is repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=1 applied?=1 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4)
2017-02-06 18:01:05.301914 7fce157e1700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 172'3 active+clean] remove_repop repgather(0x7fce0c04cf40 323'4 rep_tid=224265 committed?=1 applied?=1 lock=0 op=osd_op(client.48424379.1:4 10000000619.00000000 [write 0~3145728] 2.38fbabae snapc 1=[] ondisk+write e323) v4)
2017-02-06 18:01:05.301933 7fce157e1700 20 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 172'3 active+clean]  obc obc(38fbabae/10000000619.00000000/head//2 rwstate(write n=1 w=0))
2017-02-06 18:01:05.301948 7fce157e1700 15 osd.2 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=0 lpr=322 crt=172'1 lcod 172'3 mlcod 172'3 active+clean]  requeue_ops 
2017-02-06 18:01:05.302037 7fce157e1700 10 osd.2 323 dequeue_op 0x7fce2f4bcc00 finish
2017-02-06 18:01:05.944603 7fce11fda700 20 osd.2 323 update_osd_stat osd_stat(2720 MB used, 33419 MB avail, 36156 MB total, peers [0,1]/[] op hist [])
2017-02-06 18:01:05.944647 7fce11fda700  5 osd.2 323 heartbeat: osd_stat(2720 MB used, 33419 MB avail, 36156 MB total, peers [0,1]/[] op hist [])
root@BEAN-3:/var/log/ceph# 
```

# Replica OSD log on write

```
2017-02-06 18:01:05.268922 7fa05b1e2700 15 osd.0 323 enqueue_op 0x7fa048c0c200 prio 127 cost 3146350 latency 0.008316 osd_sub_op(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] v 323'4 snapset=0=[]:[] snapc=0=[]) v11
2017-02-06 18:01:05.268963 7fa0587fd700 10 osd.0 323 dequeue_op 0x7fa048c0c200 prio 127 cost 3146350 latency 0.008356 osd_sub_op(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] v 323'4 snapset=0=[]:[] snapc=0=[]) v11 pg pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active]
2017-02-06 18:01:05.269004 7fa0587fd700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] handle_message: 0x7fa048c0c200
2017-02-06 18:01:05.269016 7fa0587fd700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=2 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] sub_op_modify trans 38fbabae/10000000619.00000000/head//2 v 323'4 (transaction) 156
2017-02-06 18:01:05.269036 7fa0587fd700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 172'3 (0'0,172'3] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] append_log log((0'0,172'3], crt=172'3) [323'4 (0'0) modify   38fbabae/10000000619.00000000/head//2 by client.48424379.1:4 2017-02-06 18:01:00.229845]
2017-02-06 18:01:05.269064 7fa0587fd700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] add_log_entry 323'4 (0'0) modify   38fbabae/10000000619.00000000/head//2 by client.48424379.1:4 2017-02-06 18:01:00.229845
2017-02-06 18:01:05.269085 7fa0587fd700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] append_log: trimming to 0'0 entries 
2017-02-06 18:01:05.269110 7fa0587fd700 10 write_log with: dirty_to: 0'0, dirty_from: 4294967295'18446744073709551615, dirty_divergent_priors: 0, writeout_from: 323'4, trimmed: 
2017-02-06 18:01:05.269135 7fa0587fd700  5 filestore(/data/osd.0) queue_transactions existing osr(2.3ae 0x7fa0705b2aa0)/0x7fa0705b2aa0
2017-02-06 18:01:05.269141 7fa0587fd700 10 journal op_submit_start 2497991
2017-02-06 18:01:05.269143 7fa0587fd700  5 filestore(/data/osd.0) queue_transactions (writeahead) 2497991 0x7fa04992c578
2017-02-06 18:01:05.269144 7fa0587fd700 10 journal op_journal_transactions 2497991 0x7fa04992c578
2017-02-06 18:01:05.269150 7fa0587fd700  5 journal submit_entry seq 2497991 len 3147522 (0x7fa050ecb5e0)
2017-02-06 18:01:05.269158 7fa0587fd700 10 journal op_submit_finish 2497991
2017-02-06 18:01:05.269161 7fa0587fd700 15 osd.0 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] do_sub_op osd_sub_op(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] v 323'4 snapset=0=[]:[] snapc=0=[]) v11
2017-02-06 18:01:05.269173 7fa0587fd700 10 osd.0 323 dequeue_op 0x7fa048c0c200 finish
2017-02-06 18:01:05.269183 7fa07d7fd700 20 journal write_thread_entry woke up
2017-02-06 18:01:05.269186 7fa07d7fd700 10 journal room 4294918143 max_size 4294967296 pos 1082961920 header.start 1082916864 top 4096
2017-02-06 18:01:05.269195 7fa07d7fd700 10 journal check_for_full at 1082961920 : 3153920 < 4294918143
2017-02-06 18:01:05.269196 7fa07d7fd700 15 journal prepare_single_write 1 will write 1082961920 : seq 2497991 len 3147522 -> 3153920 (head 40 pre_pad 2739 ebl 3147522 post_pad 3579 tail 40) (ebl alignment 2779)
2017-02-06 18:01:05.269486 7fa07d7fd700 20 journal prepare_multi_write queue_pos now 1086115840
2017-02-06 18:01:05.269489 7fa07d7fd700 15 journal do_aio_write writing 1082961920~3153920
2017-02-06 18:01:05.270795 7fa07d7fd700 20 journal write_aio_bl 1082961920~3153920 seq 2497991
2017-02-06 18:01:05.270801 7fa07d7fd700 20 journal write_aio_bl .. 1082961920~3153920 in 1
2017-02-06 18:01:05.301087 7fa07cffc700 20 journal write_finish_thread_entry waiting for aio(s)
2017-02-06 18:01:05.301098 7fa07cffc700 10 journal write_finish_thread_entry aio 1082961920~3153920 done
2017-02-06 18:01:05.301100 7fa07cffc700 20 journal check_aio_completion
2017-02-06 18:01:05.301101 7fa07cffc700 20 journal check_aio_completion completed seq 2497991 1082961920~3153920
2017-02-06 18:01:05.301108 7fa07cffc700 20 journal check_aio_completion queueing finishers through seq 2497991
2017-02-06 18:01:05.301110 7fa07cffc700 10 journal queue_completions_thru seq 2497991 queueing seq 2497991 0x7fa050ecb5e0 lat 0.031957
2017-02-06 18:01:05.301119 7fa07cffc700 20 journal write_finish_thread_entry sleeping
2017-02-06 18:01:05.301182 7fa07c7fb700  5 filestore(/data/osd.0) _journaled_ahead 0x7fa0739d1e20 seq 2497991 osr(2.3ae 0x7fa0705b2aa0) 0x7fa04992c578
2017-02-06 18:01:05.301189 7fa07c7fb700  5 filestore(/data/osd.0) queue_op 0x7fa0739d1e20 seq 2497991 osr(2.3ae 0x7fa0705b2aa0) 3147516 bytes   (queue has 1 ops and 3147516 bytes)
2017-02-06 18:01:05.301197 7fa07c7fb700 10 filestore(/data/osd.0)  queueing ondisk 0x7fa04dd0aa80
2017-02-06 18:01:05.301212 7fa07b7f9700 10 journal op_apply_start 2497991 open_ops 0 -> 1
2017-02-06 18:01:05.301220 7fa07b7f9700  5 filestore(/data/osd.0) _do_op 0x7fa0739d1e20 seq 2497991 osr(2.3ae 0x7fa0705b2aa0)/0x7fa0705b2aa0 start
2017-02-06 18:01:05.301225 7fa07b7f9700 10 filestore(/data/osd.0) _do_transaction on 0x7fa04992c578

---------------------------------------------------------------------------------------------
注意下面这行，Replica OSD还是调用回调函数sub_op_modify_commit，向Primary OSD发送消息，通知Primary OSD第一阶段的任务commit已经完成。

2017-02-06 18:01:05.301208 7fa0777f1700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 0'0 active] sub_op_modify_commit on op osd_sub_op(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] v 323'4 snapset=0=[]:[] snapc=0=[]) v11, sending commit to osd.2
2017-02-06 18:01:05.301235 7fa07b7f9700 15 filestore(/data/osd.0) _omap_setkeys 2.3ae_head/3ae//head//2
2017-02-06 18:01:05.301239 7fa0777f1700 20 osd.0 323 share_map_peer 0x7fa05a06a480 already has epoch 323
2017-02-06 18:01:05.301381 7fa07d7fd700 20 journal write_thread_entry going to sleep
2017-02-06 18:01:05.301474 7fa07b7f9700 10 filestore oid: 3ae//head//2 not skipping op, *spos 2497991.0.0
2017-02-06 18:01:05.301485 7fa07b7f9700 10 filestore  > header.spos 0.0.0
2017-02-06 18:01:05.301543 7fa07b7f9700 15 filestore(/data/osd.0) _omap_setkeys 2.3ae_head/3ae//head//2
2017-02-06 18:01:05.301569 7fa07b7f9700 10 filestore oid: 3ae//head//2 not skipping op, *spos 2497991.0.1
2017-02-06 18:01:05.301571 7fa07b7f9700 10 filestore  > header.spos 0.0.0
2017-02-06 18:01:05.301601 7fa07b7f9700 15 filestore(/data/osd.0) write 2.3ae_head/38fbabae/10000000619.00000000/head//2 0~3145728
2017-02-06 18:01:05.304352 7fa07b7f9700 10 filestore(/data/osd.0) write 2.3ae_head/38fbabae/10000000619.00000000/head//2 0~3145728 = 3145728
2017-02-06 18:01:05.304400 7fa07b7f9700 15 filestore(/data/osd.0) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.304441 7fa07b7f9700 10 filestore(/data/osd.0) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2 = 0
2017-02-06 18:01:05.304464 7fa07b7f9700 20 filestore(/data/osd.0) fgetattrs 4676 getting '_'
2017-02-06 18:01:05.304470 7fa07b7f9700 15 filestore(/data/osd.0) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2
2017-02-06 18:01:05.304482 7fa07b7f9700 10 filestore(/data/osd.0) setattrs 2.3ae_head/38fbabae/10000000619.00000000/head//2 = 0
2017-02-06 18:01:05.304487 7fa07b7f9700 10 journal op_apply_finish 2497991 open_ops 1 -> 0
2017-02-06 18:01:05.304490 7fa07b7f9700 10 filestore(/data/osd.0) _do_op 0x7fa0739d1e20 seq 2497991 r = 0, finisher 0x7fa04dccd100 0
2017-02-06 18:01:05.304493 7fa07b7f9700 10 filestore(/data/osd.0) _finish_op 0x7fa0739d1e20 seq 2497991 osr(2.3ae 0x7fa0705b2aa0)/0x7fa0705b2aa0
2017-02-06 18:01:05.304596 7fa05b2e3700 10 osd.0 323 handle_replica_op osd_sub_op(mds.0.5:692 3.74 6e5f474/200.00000001/head//3 [] v 323'1474 snapset=0=[]:[] snapc=0=[]) v11 epoch 323
2017-02-06 18:01:05.304613 7fa05b2e3700 20 osd.0 323 should_share_map osd.1 10.11.12.2:6802/4401 323
2017-02-06 18:01:05.304623 7fa05b2e3700 15 osd.0 323 enqueue_op 0x7fa04b020900 prio 196 cost 2657 latency 0.000079 osd_sub_op(mds.0.5:692 3.74 6e5f474/200.00000001/head//3 [] v 323'1474 snapset=0=[]:[] snapc=0=[]) v11

----------------------------------------------------------------------------------------------
Replica OSD 执行完毕了第二阶段apply，开始调用sub_op_modify_applied回调函数，但是该函数并不会向Primary OSD 发送消息。
2017-02-06 18:01:05.304655 7fa077ff2700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 172'3 active] sub_op_modify_applied on 0x7fa04992c480 op osd_sub_op(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] v 323'4 snapset=0=[]:[] snapc=0=[]) v11
```