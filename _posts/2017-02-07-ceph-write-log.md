---
layout: post
title: ceph write流程
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

下面我们看下相关的debug log


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
2017-02-06 18:01:05.281451 7fce3b5f2700 20 journal write_finish_thread_entry waiting for aio(s)
2017-02-06 18:01:05.281553 7fce3b5f2700 10 journal write_finish_thread_entry aio 0~4096 done
2017-02-06 18:01:05.281560 7fce3b5f2700 20 journal check_aio_completion
2017-02-06 18:01:05.281561 7fce3b5f2700 20 journal check_aio_completion completed seq 0 0~4096
2017-02-06 18:01:05.281566 7fce3b5f2700 20 journal write_finish_thread_entry waiting for aio(s)
2017-02-06 18:01:05.281757 7fce3b5f2700 10 journal write_finish_thread_entry aio 790171648~3153920 done
2017-02-06 18:01:05.281770 7fce3b5f2700 20 journal check_aio_completion

下面两句表示seq 3835063的操作完成
2017-02-06 18:01:05.281771 7fce3b5f2700 20 journal check_aio_completion completed seq 3835063 790171648~3153920
2017-02-06 18:01:05.281776 7fce3b5f2700 20 journal check_aio_completion queueing finishers through seq 3835063


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
2017-02-06 18:01:05.304655 7fa077ff2700 10 osd.0 pg_epoch: 323 pg[2.3ae( v 323'4 (0'0,323'4] local-les=323 n=3 ec=10 les/c 323/323 322/322/308) [2,0] r=1 lpr=322 pi=159-321/7 luod=0'0 crt=172'3 lcod 172'3 active] sub_op_modify_applied on 0x7fa04992c480 op osd_sub_op(client.48424379.1:4 2.3ae 38fbabae/10000000619.00000000/head//2 [] v 323'4 snapset=0=[]:[] snapc=0=[]) v11
```