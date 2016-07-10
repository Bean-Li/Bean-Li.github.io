---
layout: post
title: ceph 网络层代码分析(3)
date: 2016-07-10 14:43:40
categories: ceph-internal
tag: ceph-internal
excerpt: ceph 网络层代码分析之下
---

## 前言

上面两篇博客，介绍了网络模块基本的数据结构，以及client端和server端会话的建立。本节的重点是消息的发送和接收。在这一篇中，Dispatcher和DispatchQueue会粉墨登场，在网络通信中发挥重要作用。


## 消息的发送
前面讲过，Connection是一个上层的抽象概念，消息的发送，一般如下：

```
conn = messenger->get_connection(dest_server);

Message *m;
m = new MPing();
conn.sendmsg(m)
```

对于SimpleMessenger而言，conn不过是Pipe中的PipeConnectionRef connection_state成员，该类的sendmsg就是调用了SimpleMessenger的sendmsg方法，如下：

```
int PipeConnection::send_message(Message *m)
{
  assert(msgr);
  return static_cast<SimpleMessenger*>(msgr)->send_message(m, this);
}
```

OK ,我们的起点是SimpleMessenger的sendmsg方法。
（下图来自YY哥的[Ceph源码解析：网络模块](https://hustcat.github.io/ceph-internal-network/)）


![](/assets/ceph_internals/simple_sendmsg.jpg)

对于Pipe的_send函数而言，就是简单地将消息放到out_q这个优先队列中。

```
    void _send(Message *m) {
      assert(pipe_lock.is_locked());
      out_q[m->get_priority()].push_back(m);
      cond.Signal();
    }
```

Pipe类中有如下成员：

```
  map<int, list<Message*> > out_q;  // priority queue for outbound msgs
```

该成员就是个优先队列。send_message并没有将消息发送给目的端，仅仅是丢掉了对应优先级的队列中，那么何时才真正地发送出去呢？Pipe的写线程负责此事。

Pipe::_send函数中中的cond.Signal会唤醒Pipe的写线程，如果写线程在沉睡的话：

```
void Pipe::writer()
{
  pipe_lock.Lock();
  while (state != STATE_CLOSED) {// && state != STATE_WAIT) {
  ...
      // wait
    ldout(msgr->cct,20) << "writer sleeping" << dendl;
    cond.Wait(pipe_lock);
  
  }
  ...
}
```
消息进了队列，写线程也可以被及时唤醒。如此，写线程就可以处理消息了。处理消息的逻辑即上图中的

```
Pipe::_get_net_outgoing   从优先队列out_q中取出下一跳要发送的消息
Pipe::write_message       发送到对端
```

发送部分的脉络比较清晰，下面看一下消息的接收。


## 消息的接收

消息的接收，是有Pipe的读线程负责的，

```
 while (state != STATE_CLOSED &&
           state != STATE_CONNECTING) {
        // 读取消息类型，某些消息会马上激活 writer 线程先处理
    if (tcp_read((char*)&tag, 1) < 0) {
        continue;
    }
    if (tag == CEPH_MSGR_TAG_KEEPALIVE) {
        continue;
    }
    if (tag == CEPH_MSGR_TAG_KEEPALIVE2) {
        continue;
    }
    if (tag == CEPH_MSGR_TAG_KEEPALIVE2_ACK) {
        continue;
    }
    if (tag == CEPH_MSGR_TAG_ACK) {
        continue;
    }
    
    else if (tag == CEPH_MSGR_TAG_MSG) {
      ldout(msgr->cct,20) << "reader got MSG" << dendl;
      Message *m = 0;
      
      /*将消息读取到new 创建出来的Message，同时m指针指向该结构体*/
      int r = read_message(&m, auth_handler.get());

      pipe_lock.Lock();
      
      if (!m) {
	     if (r < 0)
	       fault(true);
	     continue;
      }

      if (state == STATE_CLOSED || 
          state == STATE_CONNECTING) {
	      in_q->dispatch_throttle_release(m->get_dispatch_throttle_size());
	      m->put();
	      continue;
      }

      // check received seq#.  if it is old, drop the message.  
      // note that incoming messages may skip ahead.  this is convenient for the client
      // side queueing because messages can't be renumbered, but the (kernel) client will
      // occasionally pull a message out of the sent queue to send elsewhere.  in that case
      // it doesn't matter if we "got" it or not.
      
      /*检查收到的seq，如果小于等于当前的in_seq,表示收到了老的消息，丢弃即可*/
      if (m->get_seq() <= in_seq) {
	     ldout(msgr->cct,0) << "reader got old message "
		            << m->get_seq() << " <= " << in_seq << " " << m << " " << *m
		            << ", discarding" << dendl;
	     in_q->dispatch_throttle_release(m->get_dispatch_throttle_size());
	     m->put();
	     if (connection_state->has_feature(CEPH_FEATURE_RECONNECT_SEQ) &&
	          msgr->cct->_conf->ms_die_on_old_message)
	        assert(0 == "old msgs despite reconnect_seq feature");
	      continue;
      }
      /*如果比in_seq+1还要大，表示曾经有消息没有收到，*/
      if (m->get_seq() > in_seq + 1) {
	      ldout(msgr->cct,0) << "reader missed message?  skipped from seq "
			         << in_seq << " to " << m->get_seq() << dendl;
	      if (msgr->cct->_conf->ms_die_on_skipped_message)
	         assert(0 == "skipped incoming seq");
      }

      m->set_connection(connection_state.get());

      // note last received message.
      in_seq = m->get_seq();

      cond.Signal();  // wake up writer, to ack this
      
      ldout(msgr->cct,10) << "reader got message "
	       << m->get_seq() << " " << m << " " << *m
	       << dendl;
      in_q->fast_preprocess(m);

      //此处有三种情况，
      //1 正常处理，将消息放入in_q这个DispatchQueue，DispatchQueue关联的线程会从该优先队列中取出消息处理
      //2 直接在reader线程中处理，如上面的CEPH_MSGR_TAG_ACK消息
      //3 延迟发送

      if (delay_thread) {
        utime_t release;
        if (rand() % 10000 < msgr->cct->_conf->ms_inject_delay_probability * 10000.0) {
          release = m->get_recv_stamp();
          release += msgr->cct->_conf->ms_inject_delay_max * (double)(rand() % 10000) / 10000.0;
          lsubdout(msgr->cct, ms, 1) << "queue_received will delay until " << release << " on " << m << " " << *m << dendl;
        }
        delay_thread->queue(release, m);
      } else {
        if (in_q->can_fast_dispatch(m)) {
          reader_dispatching = true;
          pipe_lock.Unlock();
          in_q->fast_dispatch(m);
          pipe_lock.Lock()；
          reader_dispatching = false;
          if (state == STATE_CLOSED ||
	         notify_on_dispatch_done) { // there might be somebody waiting
	         notify_on_dispatch_done = false;
	         cond.Signal();
	       }
        } else {
          in_q->enqueue(m, m->get_priority(), conn_id);
        }
      }
    }
    
```

消息接收及处理部分，牵扯到Pipe下属的一个重要数据结构，DispatchQueue in_q

```
    DispatchQueue *in_q;
```
在DispatchQueue这个类中，有两个很重要的成员：mqueue和dispatch_thread

```
class DispatchQueue {
...

  PrioritizedQueue<QueueItem, uint64_t> mqueue;

  class DispatchThread : public Thread {
    DispatchQueue *dq;
  public:
    explicit DispatchThread(DispatchQueue *dq) : dq(dq) {}
    void *entry() {
      dq->entry();
      return 0;
    }
  } dispatch_thread;
 ...
}
```

mqueue是一个优先队列PrioritizedQueue，队列中的元素有优先级，处理的时候，优先处理优先级高的消息，因此in_q->enqueue调用的时候，需指定消息的优先级，比如上面队列的代码：

```
 in_q->enqueue(m, m->get_priority(), conn_id);
```
DispatchQueue的enqueue函数实现如下：

```
void DispatchQueue::enqueue(Message *m, int priority, uint64_t id)
{

  Mutex::Locker l(lock);
  ldout(cct,20) << "queue " << m << " prio " << priority << dendl;
  add_arrival(m);
  if (priority >= CEPH_MSG_PRIO_LOW) {
    mqueue.enqueue_strict(
        id, priority, QueueItem(m));
  } else {
    mqueue.enqueue(
        id, priority, m->get_cost(), QueueItem(m));
  }
  cond.Signal();
}
```
首先是消息入队（优先队列PrioritizedQueue的实现是个很有意思的话题，我们此处不展开讲）。然后示通过cond.Signal，给DispatchQueue配合的线程ispatchQueue::dispatch_thread::entry() 发个消息，告诉它来任务了，如果你在睡觉，就别睡了。


```
void DispatchQueue::entry()
{
  lock.Lock();
  while (true) {
    while (!mqueue.empty()) {
      QueueItem qitem = mqueue.dequeue();
      if (!qitem.is_code())
			remove_arrival(qitem.get_message());
      lock.Unlock();

      if (qitem.is_code()) {

      } else {
			Message *m = qitem.get_message();
			if (stop) {
			  ldout(cct,10) << " stop flag set, discarding " << m << " " << *m << dendl;
			  m->put();
			} else {
			  /*接下来的三行是处理消息，大部分时间走本分支*/
			  uint64_t msize = pre_dispatch(m);
			  msgr->ms_deliver_dispatch(m);
			  post_dispatch(m, msize);
			}
      }

      lock.Lock();
    }
    if (stop)
      break;

    // wait for something to be put on queue
    /*如果队列为空，就会再次沉睡，DispatchQueue::enqueue的时候，cond.Signal会将其唤醒*/
    cond.Wait(lock);
  }
  lock.Unlock();
}
```

处理消息的主干部分在：

```
			  uint64_t msize = pre_dispatch(m);
			  msgr->ms_deliver_dispatch(m);
			  post_dispatch(m, msize);
```

其中ms_deliver_dispatch是主干中的主干，

```
  void ms_deliver_dispatch(Message *m) {
    m->set_dispatch_stamp(ceph_clock_now(cct));
    for (list<Dispatcher*>::iterator p = dispatchers.begin();
	 p != dispatchers.end();
	 ++p) {
      	if ((*p)->ms_dispatch(m))
				return;
    }
    lsubdout(cct, ms, 0) << "ms_deliver_dispatch: unhandled message " << m << " " << *m << " from "
			 << m->get_source_inst() << dendl;
    assert(!cct->_conf->ms_die_on_unhandled_msg);
    m->put();
  }
```

到此处，我们终于引出了网络模块中最后一个重量级的数据结构，Dispatcher。

无论消息出入多少个队列，在多少个数据结构和函数中流转，最终终究要有处理消息的模块。负责处理消息的模块就是Dispatcher。
在SimpleMessenger中有两个元素为Dispatcher类型的链表，分别为dispatcher和fast_dispatcher。

```
class Messenger {
private:
  list<Dispatcher*> dispatchers;
  list <Dispatcher*> fast_dispatchers;

```

从上面可以看出，ms_deliver_dispatch方法是一个通用的处理方法，具体到每个SimpleMessenger，由注册的Dispatcher成员来处理消息。

在网络模块第一篇博文中提到过，在通信之中，如下部分是不可缺少的

```
	dispatcher = new SimpleDispatcher(messenger);
	messenger->add_dispatcher_head(dispatcher);
```
原因就在于，dispatcher是真正处理消息的模块，当真正牵扯双方通信的时候，需要约定好双方之间会发送哪些类型的消息，每一方收到某类型消息该如何处理该类型的消息，如果没有这个约定，那没法通信。A向B发送了消息X，B没有处理X消息的模块，存在该处理消息的模块，但是不知道如何处理X消息，那么没办法通信了。

我们以OSD负责心跳的SimpleMessenger为例：

```
  hbclient_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_front_server_messenger->add_dispatcher_head(&heartbeat_dispatcher);
  hb_back_server_messenger->add_dispatcher_head(&heartbeat_dispatcher);
```
由于负责心跳的SimpleMessenger责任很明确，就是心跳，我们看下该Dispatcher的定义情况：

```
 struct HeartbeatDispatcher : public Dispatcher {
    OSD *osd;
    explicit HeartbeatDispatcher(OSD *o) : Dispatcher(o->cct), osd(o) {}

    bool ms_can_fast_dispatch_any() const { return true; }
    bool ms_can_fast_dispatch(Message *m) const {
      switch (m->get_type()) {
			case CEPH_MSG_PING:
			case MSG_OSD_PING:
		          return true;
			default:
		          return false;
		}
    }
    void ms_fast_dispatch(Message *m) {
      osd->heartbeat_dispatch(m);
    }
    bool ms_dispatch(Message *m) {
      return osd->heartbeat_dispatch(m);
    }
    bool ms_handle_reset(Connection *con) {
      return osd->heartbeat_reset(con);
    }
    void ms_handle_remote_reset(Connection *con) {}
    bool ms_verify_authorizer(Connection *con, int peer_type,
			      int protocol, bufferlist& authorizer_data, bufferlist& authorizer_reply,
			      bool& isvalid, CryptoKey& session_key) {
      isvalid = true;
      return true;
    }
  } heartbeat_dispatcher;
  
```

从上面的ms_dispatch函数可以看出，真正处理心跳信息的函数是osd->heartbeat\_dispatch函数：

```
bool OSD::heartbeat_dispatch(Message *m)
{
  dout(30) << "heartbeat_dispatch " << m << dendl;
  switch (m->get_type()) {

  case CEPH_MSG_PING:
    dout(10) << "ping from " << m->get_source_inst() << dendl;
    m->put();
    break;

  case MSG_OSD_PING:
    handle_osd_ping(static_cast<MOSDPing*>(m));
    break;

  default:
    dout(0) << "dropping unexpected message " << *m << " from " << m->get_source_inst() << dendl;
    m->put();
  }

  return true;
}
```

处理流程很清爽，就是根据不同的消息类型，调用不同的函数来处理。因为心跳相关的SimpleMessenger只会收到上面提到的消息类型，因此注册的dispatcher只要能处理这些消息就足够了，不需要处理其它消息。

从此处可以看出，不同的SimpleMessenger之间通信的时候，双方会存在哪些消息类型是约定好的，因此需要注册的dispatcher也是不同的。

注意，处理完消息之后，需要回应，哪怕是最简单的心跳也要回复。
我们看一下负责处理MSG\_OSD\_PING消息的handle\_osd\_ping，该函数中有如下语句，他会构造一个MSG\_OSD\_PING消息类型，回复给心跳的发送者。

```
      Message *r = new MOSDPing(monc->get_fsid(),
				curmap->get_epoch(),
				MOSDPing::PING_REPLY,
				m->stamp);
      m->get_connection()->send_message(r);
```

注意了，此处回复的方法是

```
m->get_connection()->send_message(r);
```

又到了send_message，这是第一小节消息发送部分的内容。

```
        --> Messenger::send_message()
            --> SimpleMessenger::submit_message()
                 --> Pipe::_send()
                     --> Pipe::out_q[].push_back(m) -> cond.Signal 激活 writer 线程
                         --> ::sendmsg() // 发送到 socket
```

为了方便理解流程，使用一张图来概括消息接收的流程，该图来自YY哥的[Ceph源码解析：网络模块](https://hustcat.github.io/ceph-internal-network/)

![](/assets/ceph_internals/simple_process_message.jpg)

到此为止，通信模块全部介绍完毕了

## 尾声
Pipe代码非常有意思，有很多相互对应的逻辑，比如

```
write_message和read_message 是一对好基友
Pipe::connect 和 Pipe::accept是一对好基友
Pipe::reader 和Pipe::writer 是一对好基友
```
互相对照着看代码，很有意思。

另外一个值得深入探讨的是PioritizeQueue这个数据结构，也很有意思，值得一篇文章深入探讨下。




## 参考文献
* [Ceph 深入解析(1) — Ceph 的消息处理架构](http://mathslinux.org/?p=664)
* [解析Ceph: 网络层的处理](http://www.wzxue.com/ceph-network/)
* [Ceph 网络通信源代码分析](http://www.voidcn.com/blog/changtao381/article/p-5736297.html)
* [Ceph源码解析：网络模块](https://hustcat.github.io/ceph-internal-network/)