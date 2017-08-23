---
layout: post
title: ceph-mon的lease机制
date: 2017-08-23 17:57:40
categories: ceph-internal
tag: ceph-internal
excerpt: 本文介绍ceph-mon中的lease机制
---

# 前言

ceph-mon负责很多的功能：
* startup
* data store
* data sync
* data check
* scrub
* leader elect
* timecheck
* lease
* paxos
* paxos service
* consistency

本文介绍lease机制，即租约机制。

ceph-osd之间，会有心跳机制：

```
osd_heartbeat_interval   (默认是6)
osd_heartbeat_grace (默认是20)
```

即OSD Peer之间，其实形成了彼此监控的网络，每 6秒向Peer发送心跳信息，如果超过osd_heartbeat_grace 时间没收到Peer OSD的心跳信息，则send_failure，状告该OSD已经fail。

这种机制的存在确保了当OSD 异常退出或者网络不通的时候，ceph-mon能够发现。

当集群中存在多个ceph-mon的时候，有leader，有peon，ceph-mon进程也可能因为某种原因异常死亡或者网络不通，也必须有机制报障及时发现。这个机制是lease。

monitor内部采用lease协议，保证副本数据在一定时间范围内可读写(写需要是leader节点)，同时也用来发现monitor的异常，然后重新选举。

leader节点会定期发送lease消息，延长各个peon的时间，但是如果某个peon 节点挂掉，leader节点就无法收到lease_ack消息，超时之后，就会重新选举。

同样道理，leader节点也可能会异常宕机，peon节点也要能监督leader节点。如果leader down掉，peon节点就收不到来自leader的lease更新消息，超时之后，也会选举。

这里面有几个参数，比如

* 多久发送一次lease消息：mon_lease_renew_interval  默认3秒
* 每次延长租约多长时间：mon_lease  默认是5秒
* 超时重新选举的timeout时间是多久：mon_lease_ack_timeout   默认是10秒

其中mon_lease_ack_timeout对monitor leader节点和peon节点都是有效。对于monitor leader来说，如果在mon_lease_ack_timeout 的时间内，没有搜集到所有peon的lease ack，就判定超时，调用bootstrap重新选举。在另一个方面，如果peon节点在mon_lease_ack_timeout 时间内，没有收到新的lease 信息，就判定超时，也会发起重新选举。

# A面：leader

我们首先站在leader节点的角度，看下lease相关的操作。lease这个功能的发起点是extend_lease函数：

```c++
void Paxos::extend_lease()
{
  assert(mon->is_leader());
  //assert(is_active());

  /*当前时间+5秒，作为新的lease_expire*/
  lease_expire = ceph_clock_now(g_ceph_context);
  lease_expire += g_conf->mon_lease;
  
  /*已经收到的ack的集合 acked_lease清空，将当前mon leader
   *加入其中*/
  acked_lease.clear();
  acked_lease.insert(mon->rank);

  dout(7) << "extend_lease now+" << g_conf->mon_lease 
          << " (" << lease_expire << ")" << dendl;

  // bcast
  /*向所有的peon发送OP_LEASE消息，消息体中带上lease_expire */
  for (set<int>::const_iterator p = mon->get_quorum().begin();
      p != mon->get_quorum().end(); ++p) {
    if (*p == mon->rank) continue;
    MMonPaxos *lease = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_LEASE,
                                     ceph_clock_now(g_ceph_context));
    lease->last_committed = last_committed;
    lease->lease_timestamp = lease_expire;
    lease->first_committed = first_committed;
    mon->messenger->send_message(lease, mon->monmap->get_inst(*p));
  }

  /*注册ack timeout event，如果在规定时间（默认10秒）内，并未搜集齐ack，
   *那么就执行C_LeaseAckTimeout中定义的超时函数
   *正常情况下，该定时事件总是在收到最后一个ack后，被cancel掉，无法获得执行
   *只有异常发生，才会执行Paxos::lease_ack_timeout*/
  if (!lease_ack_timeout_event) {
    lease_ack_timeout_event = new C_LeaseAckTimeout(this);
    mon->timer.add_event_after(g_conf->mon_lease_ack_timeout, 
                               lease_ack_timeout_event);
  }

  /*因为extend_lease 要一轮一轮的跑下去，因此，注册下一次调用extend_lease的定时事件
   *C_LeaseRenew，触发时间是3秒后，正常情况下总是触发，发起下一轮*/
  lease_renew_event = new C_LeaseRenew(this);
  utime_t at = lease_expire;
  at -= g_conf->mon_lease;
  at += g_conf->mon_lease_renew_interval;
  mon->timer.add_event_at(at, lease_renew_event);
}

```

发送消息之后，mon leader就开始等待peon返回的lease ack消息。收到消息后，monitor leader 

```c++
void Paxos::dispatch(PaxosServiceMessage *m)
{
    switch (m->get_type()) {  
    case MSG_MON_PAXOS:
    {                                                
      MMonPaxos *pm = (MMonPaxos*)m;
      // NOTE: these ops are defined in messages/MMonPaxos.h
      switch (pm->op) {
      ...
        case MMonPaxos::OP_LEASE_ACK:
          handle_lease_ack(pm);
          break;
      }
     ...
    }
    ...
}

void Paxos::handle_lease_ack(MMonPaxos *ack)
{
  int from = ack->get_source().num();

  if (!lease_ack_timeout_event) {
    dout(10) << "handle_lease_ack from " << ack->get_source() 
             << " -- stray (probably since revoked)" << dendl;
  }
  else if (acked_lease.count(from) == 0) {
    acked_lease.insert(from);
    
    if (acked_lease == mon->get_quorum()) {
      // 最后一个peon的消息也收到了，那么没有超时，就取消掉lease_ack_timeout_event
      dout(10) << "handle_lease_ack from " << ack->get_source() 
               << " -- got everyone" << dendl;
      mon->timer.cancel_event(lease_ack_timeout_event);
      lease_ack_timeout_event = 0;
    } else {
      /*并非最后一个peon的消息，除了打印，并不做特殊的处理*/
      dout(10) << "handle_lease_ack from " << ack->get_source() 
               << " -- still need "
               << mon->get_quorum().size() - acked_lease.size()
               << " more" << dendl;
    }
  } else {
    /*已经acked的peon，会记录再acked_lease集合中，如果已经收到对应ack消息了，
     *那么就是重复的消息了，ignore掉*/
    dout(10) << "handle_lease_ack from " << ack->get_source()
             << " dup (lagging!), ignoring" << dendl;
  }
  warn_on_future_time(ack->sent_timestamp, ack->get_source());
  
  ack->put();
}
```

对于monitor leader 来说，每mon_lease_renew_interval 秒（默认3秒）触发依次extend_lease，在该函数中，monitor leader会向所有的peon发送lease 消息，然后设置定时事件C_LeaseAckTimeout，如果在mon_lease_ack_timeout 时间内搜集全所有的lease ack消息，就既往不咎，取消掉C_LeaseAckTimeout定时事件。

如果超过mon_lease_ack_timeout ，也没搜集起所有的lease ack 怎么办？通过lease_ack_timeout函数，调用bootstrap函数，发起选举。

```c++
class C_LeaseAckTimeout : public Context {
    Paxos *paxos;
  public:
    C_LeaseAckTimeout(Paxos *p) : paxos(p) {}
    void finish(int r) { 
      if (r == -ECANCELED)
        return;
      paxos->lease_ack_timeout();
    }                                                                                                                                                  
};
  
void Paxos::lease_ack_timeout()                                                    
{   
  dout(1) << "lease_ack_timeout -- calling new election" << dendl;
  assert(mon->is_leader());
  assert(is_active());
  logger->inc(l_paxos_lease_ack_timeout);
  lease_ack_timeout_event = 0;
  /*bootstrap 发起monitor leader的选举*/
  mon->bootstrap();
} 
```



# B面 peon

对于peon节点而言，收到OP_LEASE消息，是讨论的起点：

```c++
void Paxos::handle_lease(MMonPaxos *lease)                                                   
{
  // sanity
  if (!mon->is_peon() ||
      last_committed != lease->last_committed) {
    dout(10) << "handle_lease i'm not a peon, or they're not the leader,"
             << " or the last_committed doesn't match, dropping" << dendl;
    lease->put();
    return;
  }
  warn_on_future_time(lease->sent_timestamp, lease->get_source());

  /*延长lease 到mon leader指定的时间*/
  if (lease_expire < lease->lease_timestamp) {
    lease_expire = lease->lease_timestamp;
    utime_t now = ceph_clock_now(g_ceph_context);
    /*如果peon和monitor leader的时间差太大，lease_expire小于now，那么警告*/
    if (lease_expire < now) {
      utime_t diff = now - lease_expire;
      derr << "lease_expire from " << lease->get_source_inst() << " is " << diff << " seconds in the past; mons are probably laggy (or possibly clocks are too skewed)" << dendl; 
    }
  }

  state = STATE_ACTIVE;

  /*发送OP_LEASE_ACK消息到mon leader*/
  dout(10) << "handle_lease on " << lease->last_committed
           << " now " << lease_expire << dendl;
  MMonPaxos *ack = new MMonPaxos(mon->get_epoch(), MMonPaxos::OP_LEASE_ACK,
                                 ceph_clock_now(g_ceph_context));
  ack->last_committed = last_committed;
  ack->first_committed = first_committed;
  ack->lease_timestamp = ceph_clock_now(g_ceph_context);
  lease->get_connection()->send_message(ack);

  // (re)set timeout event.
  reset_lease_timeout();

  // kick waiters
  finish_contexts(g_ceph_context, waiting_for_active);
  if (is_readable())
    finish_contexts(g_ceph_context, waiting_for_readable);

  lease->put();
}  

```

前面讲过，mon leader和peon是互相监督，peon对monitor leader的监督，体现在reset_lease_timeout函数。他会以收到OP_LEASE消息的时间为起点，注册一个超时时间为mon_lease_ack_timeout的定时事件。如果该定时器超时了，表示在过去的mon_lease_ack_timeout时间内，没有收到任何的OP_LEASE消息，基本可以确定mon leader出问题了。

```c++
void Paxos::reset_lease_timeout()
{
  dout(20) << "reset_lease_timeout - setting timeout event" << dendl;
  /*先取消掉当前的定时事件
   *事实上，该定时事件几乎总是被cancel掉，因为正常情况下，peon会每隔3秒，源源不断地收到OP_LEASE消息
   */
  if (lease_timeout_event)
    mon->timer.cancel_event(lease_timeout_event);
  lease_timeout_event = new C_LeaseTimeout(this);                                            
  mon->timer.add_event_after(g_conf->mon_lease_ack_timeout, lease_timeout_event);
}

```

通过这个C_LeaseTimeout定时事件，peon也在监督monitor leader，如果monitor leader迟迟不发送OP_LEASE消息，延长租约，那么peon会通过如下方法，发起选举：

```c++
  class C_LeaseTimeout : public Context {
    Paxos *paxos;
  public:
    C_LeaseTimeout(Paxos *p) : paxos(p) {}
    void finish(int r) {
      if (r == -ECANCELED)
        return;
      paxos->lease_timeout();
    }
  };
  
void Paxos::lease_timeout()
{
  dout(1) << "lease_timeout -- calling new election" << dendl;
  /*只有peon节点才会调用该函数*/
  assert(mon->is_peon());
  logger->inc(l_paxos_lease_timeout);
  lease_timeout_event = 0;
  /*调用bootstrap发起选举*/
  mon->bootstrap();
}
```

注意，lease_expire每次续费3秒，但是超时时间是10秒，那么就会有一段时间，租约已经过期，但是还没超时重新选举。这段时间内租约是无效的：

```c++
bool Paxos::is_lease_valid()
{
  return ((mon->get_quorum().size() == 1)
      || (ceph_clock_now(g_ceph_context) < lease_expire));
}   
```

注意这段时间内，是不可读写的：

```c++
bool Paxos::is_readable(version_t v)
{
  bool ret;
  if (v > last_committed)
    ret = false;
  else
    ret =
      (mon->is_peon() || mon->is_leader()) &&
      (is_active() || is_updating() || is_writing()) &&
      last_committed > 0 &&           // must have a value
      (mon->get_quorum().size() == 1 ||  // alone, or
       is_lease_valid()); // have lease                                                                                                                
  dout(5) << __func__ << " = " << (int)ret
          << " - now=" << ceph_clock_now(g_ceph_context)
          << " lease_expire=" << lease_expire
          << " has v" << v << " lc " << last_committed
          << dendl;
  return ret;
}
bool Paxos::is_writeable()
{
  return
    mon->is_leader() &&
    is_active() &&
    is_lease_valid();
}  
```
