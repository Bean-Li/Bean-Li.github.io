---
layout: post
title: ceph-mon的Leader Elect机制
date: 2017-09-02 17:20:40
categories: ceph-internal
tag: ceph-internal
excerpt: 本文介绍ceph-mon中的Elect机制
---

# 前言

在ceph monitor 运行过程中，需要选举leader，所有的ceph monitor中只会有一个节点会被选举成leader，其他的节点为peon。

后续所有的update相关的操作，通过leader 发出Paxos propose完成，如果是peon节点收到更新请求，那么会转发到leader，让leader代为执行。

注意ceph中的monitor leader的选举算法，并不是paxos算法，ceph讨了一个巧，它利用了节点在monmap中的rank值，人为地制造了地位不平等，rand最小的节点获胜，从而简单快速地达到选举的目的。

# 发起

其实整个选举过程，我们可以以start_election为起点研究，那么何时会调用start_election函数呢？

* 节点调用bootstrap函数引导启动，接着会probing，查询其他monitor信息(有可能需要同步数据)，完成后发起选举
* 节点收到选举消息MMonElection，如果节点自己已经处于quorum或自己的编号更小，也会重新发起选举
* 节点收到quorum enter/exit命令

第三种情况是一个测试用的工具，我们可以忽略不提。很多种情况都会导致某个节点调用bootstrap，比如某个ceph-mon重新启动，比如上篇lease中提到的两种情形。

如果发生了mon 选举，我们从ceph.log中可以看到如下的内容：

```
2017-08-27 17:33:43.144946 mon.1 172.1.1.197:6789/0 152076 : cluster [INF] mon.hymwq calling new monitor election
2017-08-27 17:33:43.151244 mon.0 172.1.1.196:6789/0 282354 : cluster [INF] mon.nhgfb calling new monitor election
```

注意，当集群节点个数比较多的时候，在同一时间点我们可能看到一条到多条上述信息，这是为何？这要深入了解mon的选举过程

# 选举过程

整个选举过程，在Elector类中实现。此类之中实现了一个election_epoch:

```shell
root@scaler02:~# ceph quorum_status |json_pp
{
   "quorum_leader_name" : "skmif",
   "monmap" : {
      "mons" : [
         {
            "name" : "skmif",
            "addr" : "10.10.1.1:6789/0",
            "rank" : 0
         },
         {
            "name" : "vqdtz",
            "addr" : "10.10.1.2:6789/0",
            "rank" : 1
         },
         {
            "name" : "lzhsg",
            "addr" : "10.10.1.3:6789/0",
            "rank" : 2
         }
      ],
      "created" : "2017-08-29 09:27:11.587301",
      "epoch" : 3,
      "modified" : "2017-08-29 09:28:05.098478",
      "fsid" : "6e74645a-9894-4d7b-9e94-9f4b9596d59f"
   },
   "quorum_names" : [
      "skmif",
      "vqdtz"
   ],
   "quorum" : [
      0,
      1
   ],
   "election_epoch" : 24
}
```

当这个election_epoch为偶数的时候，表示处于稳定状态，为奇数的时候，表示还在选举过程中，mon leader的宝座还在竞争，鹿死谁手尚未可知。

下面梳理下流程。

```c++
void Monitor::start_election()                                                   
{
  /*这条日志我们一般看不到*/
  dout(10) << "start_election" << dendl;
  wait_for_paxos_write();
  _reset();
  state = STATE_ELECTING;

  logger->inc(l_mon_num_elections);
  logger->inc(l_mon_election_call);

  cancel_probe_timeout();

  clog->info() << "mon." << name << " calling new monitor election\n";
  elector.call_election();
}
```

当我们从ceph.log中看到如下的打印的时候，表示选举已经开始，某一时间段内第一条打印，是发起选举的mon。它率先觉察到异常，调用了bootstrap，最终走到了start_election。

```
2017-08-27 17:33:43.144946 mon.1 172.1.1.197:6789/0 152076 : cluster [INF] mon.hymwq calling new monitor election
```

如果我们打印更多debug信息的时候我们可能看到如下的流程：

```shell
因为10秒内没有收到延长租约的消息，最终触发了election，有PEON发起，调用了bootstrap
2017-09-02 15:17:13.687831 7fe3d3c5a700  1 mon.vqdtz@1(peon).paxos(paxos updating c 1051189..1051804) lease_timeout -- calling new election
2017-09-02 15:17:13.687849 7fe3d3c5a700 10 mon.vqdtz@1(peon) e3 bootstrap
2017-09-02 15:17:13.687856 7fe3d3c5a700 10 mon.vqdtz@1(peon) e3 sync_reset_requester
2017-09-02 15:17:13.687859 7fe3d3c5a700 10 mon.vqdtz@1(peon) e3 unregister_cluster_logger
2017-09-02 15:17:13.687865 7fe3d3c5a700 10 mon.vqdtz@1(peon) e3 cancel_probe_timeout (none scheduled)
2017-09-02 15:17:13.687869 7fe3d3c5a700 10 mon.vqdtz@1(probing) e3 _reset
2017-09-02 15:17:13.687871 7fe3d3c5a700 10 mon.vqdtz@1(probing) e3 cancel_probe_timeout (none scheduled)
2017-09-02 15:17:13.687873 7fe3d3c5a700 10 mon.vqdtz@1(probing) e3 timecheck_finish
2017-09-02 15:17:13.687882 7fe3d3c5a700 10 mon.vqdtz@1(probing) e3 scrub_reset
....
此处的cancel_probe_timeout即函数中cancel_probe_timeout()语句
2017-09-02 15:17:13.688846 7fe3d3459700 10 mon.vqdtz@1(electing) e3 cancel_probe_timeout (none scheduled)
2017-09-02 15:17:13.688875 7fe3d3459700  5 mon.vqdtz@1(electing).elector(42) start -- can i be leader?
2017-09-02 15:17:13.688915 7fe3d3459700  1 mon.vqdtz@1(electing).elector(42) init, last seen epoch 42
2017-09-02 15:17:13.688919 7fe3d3459700 10 mon.vqdtz@1(electing).elector(42) bump_epoch 42 to 43
2017-09-02 15:17:13.690523 7fe3d3459700 10 mon.vqdtz@1(electing) e3 join_election
2017-09-02 15:17:13.690540 7fe3d3459700 10 mon.vqdtz@1(electing) e3 _reset
2017-09-02 15:17:13.690543 7fe3d3459700 10 mon.vqdtz@1(electing) e3 cancel_probe_timeout (none scheduled)
2017-09-02 15:17:13.690545 7fe3d3459700 10 mon.vqdtz@1(electing) e3 timecheck_finish
2017-09-02 15:17:13.690549 7fe3d3459700 10 mon.vqdtz@1(electing) e3 scrub_reset


```

从 elector.call_election()开始，就开始调用elector类定义的方法，开始选举。

```c++
  void call_election() {                                                     
    start();
  }

void Elector::start()
{
  if (!participating) {
    dout(0) << "not starting new election -- not participating" << dendl;
    return;
  }
  dout(5) << "start -- can i be leader?" << dendl;

  acked_me.clear();
  classic_mons.clear();
  init();
  
  /*从稳定态进入选举态，需要将版本号从偶数往上抬，抬成奇数*/
  if (epoch % 2 == 0) 
    bump_epoch(epoch+1);  // odd == election cycle
  start_stamp = ceph_clock_now(g_ceph_context);
  electing_me = true;
  acked_me[mon->rank] = CEPH_FEATURES_ALL;
  leader_acked = -1;

  /*向每一个成员广播消息，提议开始重新选举*/
  for (unsigned i=0; i<mon->monmap->size(); ++i) {
    if ((int)i == mon->rank) continue;
    Message *m = new MMonElection(MMonElection::OP_PROPOSE, epoch, mon->monmap);
    mon->messenger->send_message(m, mon->monmap->get_inst(i));
  }              
  reset_timer();
}
```

当其他成员收到OP_PROPOSE的消息时，就知道了，需要开始新一轮的选举了。其他的成员收到消息之后，反应可以分成三种：

* 赞成
* 给其他所有成员发消息，选我
* 不理

因为到底谁当选leader，取决于rank的大小，rank小者胜，因此收到消息之后，会比对rank，来决定做什么事情。

## 如果选举发起方的rank比自身的rank大

天子宁有种乎，兵强马壮者为之耳！如果从未收到过更强者(rank更小者)发来的选举请求，调用start_election，给所有成员发消息，让他们选自己为mon leader。这里面有一种场景，即连续两个弱者要求当leader，这时候，第一次的时候，如果已经调用了start_election，要求大家选自己，就没有必要重新再发一次选自己当leader的请求了。

```c++
  if (mon->rank < from) {
    // i would win over them.
    if (leader_acked >= 0) { // we already acked someone
      /*自己曾经认过怂，消息来源的mon还不如自己，直接不理，
       *来源的mon太弱，是不可能被自己承认的*/
      assert(leader_acked < from);  // and they still win, of course
      dout(5) << "no, we already acked " << leader_acked << dendl;
    } else {
      /*注意，electing_me记录了自己是否发出过选我为leader的请求
       *如果先后收到两个弱小者发来的选举请求，处理第一个的时候，本节点已经发出了选自己当leader的请求，
       *当第二个弱者消息到来的时候，没必要再发送选自己当leader的请求*/
      if (!electing_me) {
        mon->start_election();
      }
    }
  } else {
    // they would win over me
    if (leader_acked < 0 ||      // haven't acked anyone yet, or
        leader_acked > from ||   // they would win over who you did ack, or
        leader_acked == from) {  // this is the guy we're already deferring to
      defer(from);
    } else {
      // ignore them!
      dout(5) << "no, we already acked " << leader_acked << dendl;
    }
  }
```

这里面还有另外一种情况，即更强者曾经来过，自己曾经认过怂，承认过别人更强，那么这种情况下，采用的是不理的策略，自己都认了怂，这个消息的来源mon还不如自己，肯定是不能承认其leader的地位。



## 如果选举发起方的rank比自身的rank小

如果消息的来源mon，rank比自己小，要强于自己，发出选举倡议，让大家选自己是不可能了，只剩两种可能，要么承认它，要么不理它。

````c++
    if (leader_acked < 0 ||      /*从未承认过别人，从未认过怂*/ 
        leader_acked > from ||   /*虽然承认过别人，认过怂，无奈这次来的更强大，所以还是得认怂，承认它*/
        leader_acked == from) {  
      defer(from);  /*defer函数的作用是认可对方可以当leader*/
    }else {
      /*曾经认可过更强者，不可能向不够强的mon发送认可，不理*/
      dout(5) << "no, we already acked " << leader_acked << dendl;
    }

````

从上面可以看出，一个节点根据时序的不同，可能调用defer多次，承认多个mon当leader的请求。

如果曾经认可过更强的mon，当处于中间水平的mon到来的时候，自己是不可能再向该mon发送认可的回应。

通过上面的讨论可以看出，只有rank最小者，即最强者才有可能搜集到最多的承认。

```c++
void Elector::defer(int who)
{
  dout(5) << "defer to " << who << dendl;

  if (electing_me) {
    /*注意，认怂就要清零，哪怕曾竖起过大旗，要求别人选自己*/
    acked_me.clear();
    classic_mons.clear();
    electing_me = false;
  }

  // ack them
  leader_acked = who;
  ack_stamp = ceph_clock_now(g_ceph_context);
  /*发送OP_ACK承认对方可以当leader*/
  MMonElection *m = new MMonElection(MMonElection::OP_ACK, epoch, mon->monmap);
  m->sharing_bl = mon->get_supported_commands_bl();
  mon->messenger->send_message(m, mon->monmap->get_inst(who));
  
  // set a timer
  reset_timer(1.0);  // give the leader some extra time to declare victory
}
```

## 处理OP_ACK 消息 

收到这个OP_ACK消息，表示对方认可自己当leader的请求，我们来看下如何处理：

```c++
void Elector::handle_ack(MMonElection *m)
{                                                                                                                                                      
  dout(5) << "handle_ack from " << m->get_source() << dendl;
  int from = m->get_source().num();

  assert(m->epoch % 2 == 1); // election
  if (m->epoch > epoch) {
    dout(5) << "woah, that's a newer epoch, i must have rebooted.  bumping and re-starting!" << dendl;
    bump_epoch(m->epoch);
    start();
    m->put();
    return;
  }
  assert(m->epoch == epoch);
  uint64_t required_features = mon->get_required_features();
  if ((required_features ^ m->get_connection()->get_features()) &
      required_features) {
    dout(5) << " ignoring ack from mon" << from
            << " without required features" << dendl;
    return;
  }
  
  if (electing_me) {
    // thanks
    /*搜集到一枚承认，记录在acked_me*/
    acked_me[from] = m->get_connection()->get_features();
    if (!m->sharing_bl.length())
      classic_mons.insert(from);
    dout(5) << " so far i have " << acked_me << dendl;
    
    /*如果所有成员都承认了自己的leader地位，那么宣布获胜，调用victory*/
    if (acked_me.size() == mon->monmap->size()) {
      // if yes, shortcut to election finish
      victory();
    }
  } else {
    // ignore, i'm deferring already.
    assert(leader_acked >= 0);
  }
  
  m->put();
}  
```

注意，如果所有的人都承认自己leader地位，那么可以宣布获胜。但是有些情况下，无法等到所有的回应。比如某个ceph-mon进程已经不在了，是不可能得到其承认的。为了防止出现这种情况下，在通知其他节点选自己的start函数设置了定时器“

```C++
void Elector::start() 
{
    ...
    reset_timer();
}
void Elector::reset_timer(double plus)                                                    
{
  // set the timer
  cancel_timer();
  expire_event = new C_ElectionExpire(this);
  mon->timer.add_event_after(g_conf->mon_lease + plus,
                             expire_event);
}
void Elector::expire()
{
  dout(5) << "election timer expired" << dendl;
  
  /*注意，超过半数，就能宣布获胜
   *注意，如果认过怂，electing_me就变成false了，你就不可能宣布胜利了*/
  if (electing_me &&
      acked_me.size() > (unsigned)(mon->monmap->size() / 2)) {
    // i win
    victory();
  } else {
    // whoever i deferred to didn't declare victory quickly enough.
    if (mon->has_ever_joined)
      start();
    else                                            
      mon->bootstrap();
  }
}
```

如果自己获胜的话，会给其他节点发送OP_VICTORY消息，告诉别的节点，自己已经当选leader了。

```c++
void Elector::victory()
{

  /*选举过程完成，需要抬election_epoch，变成偶数*/
  assert(epoch % 2 == 1);  // election
  bump_epoch(epoch+1);     // is over!  
  // tell everyone!
  for (set<int>::iterator p = quorum.begin();
       p != quorum.end();
       ++p) {
    if (*p == mon->rank) continue;
    MMonElection *m = new MMonElection(MMonElection::OP_VICTORY, epoch, mon->monmap);
    m->quorum = quorum;
    m->quorum_features = features;
    m->sharing_bl = *cmds_bl;
    mon->messenger->send_message(m, mon->monmap->get_inst(*p));
  }    
  // tell monitor                                                  
  mon->win_election(epoch, quorum, features, cmds, cmdsize, &copy_classic_mons);
}
```

而竞选的失败者，在handle_victory函数中处理OP_VICTORY消息：

```c++
void Elector::handle_victory(MMonElection *m)
{
  ...
   bump_epoch(m->epoch) ;
   // they win
   mon->lose_election(epoch, m->quorum, from, m->quorum_features);
   ...
}
```

注意，胜利者通过win_election重新初始化，成为Leader，而失败者，通过lost_election重新初始化，变成PEON
