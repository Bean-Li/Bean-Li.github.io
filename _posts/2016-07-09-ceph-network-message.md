---
layout: post
title: ceph 网络层代码分析(2)
date: 2016-07-09 14:43:40
categories: ceph-internal
tag: ceph-internal
excerpt: ceph 网络层代码分析
---

## 前言
网络层的代码确实太长，所以不得不分成多篇文章来讲。

上一篇博客，Accepter，Connection，Pipe，PipeConnection，都亮相了。还有最重要的部分，即到底如何和发送消息，如何处理消息，以及如何回应。这篇文章会介绍client端和server端会话的建立。


## Pipe的connect和accept

对于client端:

```
conn = messenger->get_connection(dest_server);
```
如果需要创建新的Pipe，会调用connect_rank函数，如下所示：


```
Pipe *SimpleMessenger::connect_rank(const entity_addr_t& addr,
				    int type,
				    PipeConnection *con,
				    Message *first)
{
  assert(lock.is_locked());
  assert(addr != my_inst.addr);
  
  ldout(cct,10) << "connect_rank to " << addr << ", creating pipe and registering" << dendl;
  
  // 创建新的Pipe，并且将Pipe的状态设置为STATE_CONNECTING,这很重要
  Pipe *pipe = new Pipe(this, Pipe::STATE_CONNECTING,
			static_cast<PipeConnection*>(con));
  pipe->pipe_lock.Lock();
  pipe->set_peer_type(type);
  pipe->set_peer_addr(addr);
  pipe->policy = get_policy(type);
  
  /*启动Pipe的写线程*/
  pipe->start_writer();
  if (first)
    pipe->_send(first);
  pipe->pipe_lock.Unlock();
  pipe->register_pipe();
  pipes.insert(pipe);

  return pipe;
}

void Pipe::start_writer()
{
  assert(pipe_lock.is_locked());
  assert(!writer_running);
  writer_running = true;
  writer_thread.create("ms_pipe_write", msgr->cct->_conf->ms_rwthread_stack_bytes);
}

```

client端会创建Pipe的写进程，写进程的主函数是Pipe::writer , 而此时Pipe处于Pipe::STATE_CONNECTING状态，

```
void Pipe::writer()
{
  pipe_lock.Lock();
  while (state != STATE_CLOSED) {// && state != STATE_WAIT) {
    ldout(msgr->cct,10) << "writer: state = " << get_state_name()
			<< " policy.server=" << policy.server << dendl;

    // standby?
    if (is_queued() && state == STATE_STANDBY && !policy.server)
      state = STATE_CONNECTING;

    // connect?
    if (state == STATE_CONNECTING) {
      assert(!policy.server);
      connect();
      continue;
    }
    ...
    
}
    
```
当Pipe处于Pipe::STATE_CONNECTING状态，writer函数会调用Pipe::connect函数，该函数负责与服务器建立连接，真正意义上的通信通道，我们不妨看看该函数：

```
int Pipe::connect()
{
  bool got_bad_auth = false;

  ldout(msgr->cct,10) << "connect " << connect_seq << dendl;
  assert(pipe_lock.is_locked());

  __u32 cseq = connect_seq;
  __u32 gseq = msgr->get_global_seq();

  // stop reader thrad
  join_reader();

  pipe_lock.Unlock();
  
  char tag = -1;
  int rc = -1;
  struct msghdr msg;
  struct iovec msgvec[2];
  int msglen;
  char banner[strlen(CEPH_BANNER) + 1];  // extra byte makes coverity happy
  entity_addr_t paddr;
  entity_addr_t peer_addr_for_me, socket_addr;
  AuthAuthorizer *authorizer = NULL;
  bufferlist addrbl, myaddrbl;
  const md_config_t *conf = msgr->cct->_conf;

  // close old socket.  this is safe because we stopped the reader thread above.
  if (sd >= 0)
    ::close(sd);

  // create socket?
  sd = ::socket(peer_addr.get_family(), SOCK_STREAM, 0);
  if (sd < 0) {
    rc = -errno;
    lderr(msgr->cct) << "connect couldn't created socket " << cpp_strerror(rc) << dendl;
    goto fail;
  }

  recv_reset();

  set_socket_options();

  /*发起连接，事实上在Server端Accepter类的对应的工作线程正阻塞在accept系统调用上，等待client端的连接*/
  ldout(msgr->cct,10) << "connecting to " << peer_addr << dendl;
  rc = ::connect(sd, peer_addr.get_sockaddr(), peer_addr.get_sockaddr_len());
  if (rc < 0) {
    rc = -errno;
    ldout(msgr->cct,2) << "connect error " << peer_addr
	     << ", " << cpp_strerror(rc) << dendl;
    goto fail;
  }
```

client端通过Pipe::connect函数，会真正地调用connect系统调用，尝试连接服务器端的监听地址；在通信的另一端，服务器端的Accepter线程正阻塞在accept系统调用上，等待client调用connect系统调用来连，一旦服务器端的accept函数返回，Accepter中的线程就会调用add_accept_pipe函数来创建一个新的Pipe，全权负责和client的通信。创建出来Pipe之后，Accepter类的主线程不敢恋战，而是继续下一个循环，即调用accept系统调用，等待新的连接。如下所示：

```
void *Accepter::entry()
{

....
    sockaddr_storage ss;
    socklen_t slen = sizeof(ss);
    int sd = ::accept(listen_sd, (sockaddr*)&ss, &slen);
    if (sd >= 0) {
      int r = set_close_on_exec(sd);
      if (r) {
	     ldout(msgr->cct,0) << "accepter set_close_on_exec() failed "
	      << cpp_strerror(r) << dendl;
      }
      errors = 0;
      ldout(msgr->cct,10) << "accepted incoming on sd " << sd << dendl;
      
      msgr->add_accept_pipe(sd);
    } else {
      ldout(msgr->cct,0) << "accepter no incoming connection?  sd = " << sd
	      << " errno " << errno << " " << cpp_strerror(errno) << dendl;
      if (++errors > 4)
	     break;
    }
    
    ....
    
}
    
```

而SimpleMessenger的add_accept_pipe函数也是创建一个Pipe对象，同时启动reader_thread：

```
Pipe *SimpleMessenger::add_accept_pipe(int sd)
{
  lock.Lock();
  
  /*创建Pipe，接管与Client端的通信事宜，以便让Accepter线程即使返回，继续调用accept，等待新连接*/
  Pipe *p = new Pipe(this, Pipe::STATE_ACCEPTING, NULL);
  p->sd = sd;
  p->pipe_lock.Lock();
  
  /*启动Pipe的读线程，由于新的Pipe初始状态是ACCEPTING，因此会读线程会调用Pipe::accept函数，和client端建立会话*/
  p->start_reader();
  p->pipe_lock.Unlock();
  pipes.insert(p);
  accepting_pipes.insert(p);
  lock.Unlock();
  return p;
}
```
注意，新创建的Pipe处于Pipe::STATE_ACCEPTING，Pipe的读线程的主函数是Pipe::reader,在该函数中，如果Pipe状态是STATE_ACCEPTING，会调用accept函数和client进行通信，创建会话。如下所示

```
void Pipe::reader()
{
  pipe_lock.Lock();

  if (state == STATE_ACCEPTING) {
    accept();
    assert(pipe_lock.is_locked());
  }
```

当Pipe处于STATE_ACCEPTING状态时，读线程会执行accept函数。在通信的初始化阶段，Pipe::connect和Pipe::accept是一对，他俩互相协商，互相通信，建立连接关系。

接下来client端和server，交换了哪些信息呢？我们阅读Pipe::connect和Pipe::accept不难得到下图（图来自麦子迈的[解析Ceph: 网络层的处理](解析Ceph: 网络层的处理)）：

![](/assets/ceph_internals/pipe_connection_setup.png)


client端的代码如下：

```
 // verify banner
  // FIXME: this should be non-blocking, or in some other way verify the banner as we get it.
  
   /*接收server端Pipe::accept函数中发送过来的CEPH_BANNER*/
  rc = tcp_read((char*)&banner, strlen(CEPH_BANNER));
  if (rc < 0) {
    ldout(msgr->cct,2) << "connect couldn't read banner, " << cpp_strerror(rc) << dendl;
    goto fail;
  }
  if (memcmp(banner, CEPH_BANNER, strlen(CEPH_BANNER))) {
    ldout(msgr->cct,0) << "connect protocol error (bad banner) on peer " << peer_addr << dendl;
    goto fail;
  }
  
  /*发送自己的地址信息给Server端*/
  memset(&msg, 0, sizeof(msg));
  msgvec[0].iov_base = banner;
  msgvec[0].iov_len = strlen(CEPH_BANNER);
  msg.msg_iov = msgvec;
  msg.msg_iovlen = 1;
  msglen = msgvec[0].iov_len;
  rc = do_sendmsg(&msg, msglen);
  if (rc < 0) {
    ldout(msgr->cct,2) << "connect couldn't write my banner, " << cpp_strerror(rc) << dendl;
    goto fail;
  }
  
  ...
```

server端的代码我就不贴了.

上图给出了一个会话建立的过程，BANNER有点是建立连接的接头暗号一般，如同天王盖地虎，宝塔镇河妖的一位，BANNER是常量。


```
/*
 * tcp connection banner.  include a protocol version. and adjust
 * whenever the wire protocol changes.  try to keep this string length
 * constant.
 */
#define CEPH_BANNER "ceph v027"
```

通过BANNER暗号之后，互相向对方通告自己的地址。这一部分逻辑并无特别，就不展开了。

最重要的逻辑是connection message的沟通和交互。很明显Pipe::connect和Pipe::accept函数下半段有一大段很难懂的代码。这段代码的用途在于，服务器端会校验这些连接信息并确保面向这个地址的连接只有一条。

client端首先会发送一个ceph_msg_connect的结构体，

```
/*
 * connection negotiation
 */
struct ceph_msg_connect {
	__le64 features;     /* supported feature bits */
	__le32 host_type;    /* CEPH_ENTITY_TYPE_* */
	__le32 global_seq;   /* count connections initiated by this host */
	__le32 connect_seq;  /* count connections initiated in this session */
	__le32 protocol_version;
	__le32 authorizer_protocol;
	__le32 authorizer_len;
	__u8  flags;         /* CEPH_MSG_CONNECT_* */
} __attribute__ ((packed));


/*client端发送connection negotiation的代码部分*/

 while (1) {
    delete authorizer;
    authorizer = msgr->get_authorizer(peer_type, false);
    bufferlist authorizer_reply;

    ceph_msg_connect connect;
    connect.features = policy.features_supported;
    connect.host_type = msgr->get_myinst().name.type();
    connect.global_seq = gseq;
    connect.connect_seq = cseq;
    connect.protocol_version = msgr->get_proto_version(peer_type, true);
    connect.authorizer_protocol = authorizer ? authorizer->protocol : 0;
    connect.authorizer_len = authorizer ? authorizer->bl.length() : 0;
    if (authorizer) 
      ldout(msgr->cct,10) << "connect.authorizer_len=" << connect.authorizer_len
	       << " protocol=" << connect.authorizer_protocol << dendl;
    connect.flags = 0;
    if (policy.lossy)
      connect.flags |= CEPH_MSG_CONNECT_LOSSY;  // this is fyi, actually, server decides!
    memset(&msg, 0, sizeof(msg));
    msgvec[0].iov_base = (char*)&connect;
    msgvec[0].iov_len = sizeof(connect);
    msg.msg_iov = msgvec;
    msg.msg_iovlen = 1;
    msglen = msgvec[0].iov_len;
    if (authorizer) {
      msgvec[1].iov_base = authorizer->bl.c_str();
      msgvec[1].iov_len = authorizer->bl.length();
      msg.msg_iovlen++;
      msglen += msgvec[1].iov_len;
    }

    ldout(msgr->cct,10) << "connect sending gseq=" << gseq << " cseq=" << cseq
	     << " proto=" << connect.protocol_version << dendl;
    rc = do_sendmsg(&msg, msglen);
    if (rc < 0) {
      ldout(msgr->cct,2) << "connect couldn't write gseq, cseq, " << cpp_strerror(rc) << dendl;
      goto fail;
    }

```

client端将 本端的global\_seq和 connect_seq发送到了服务器端。服务器端会首先查找是否已经存在负责和client端通信的的Pipe

```
 existing = msgr->_lookup_pipe(peer_addr);
```

如果不存在已有的Pipe，问题就简单了，根据client端发过来的connect.connect_seq是否为0，分成两种情况：

```
else if (connect.connect_seq > 0) {
      // we reset, and they are opening a new session
      /*服务器端reset了，通过CEPH_MSGR_TAG_RESETSESSION tag告诉client端，reset session*/
      ldout(msgr->cct,0) << "accept we reset (peer sent cseq " << connect.connect_seq << "), sending RESETSESSION" << dendl;
      msgr->lock.Unlock();
      reply.tag = CEPH_MSGR_TAG_RESETSESSION;
      goto reply;
    } else {
      
      /*创建新的session*/
      ldout(msgr->cct,10) << "accept new session" << dendl;
      existing = NULL;
      goto open;
    }
```

对于新连接而言，一般connect.connect\_seq 总是等于0。这时候，只需要将状态转换成STATE\_OPEN，同时发送回应给client 端就行了，如下所示：

```
open:
  // open
  assert(pipe_lock.is_locked());
  connect_seq = connect.connect_seq + 1;
  peer_global_seq = connect.global_seq;
  assert(state == STATE_ACCEPTING);
  
  /*将Pipe的状态设置为 STATE_OPEN*/
  state = STATE_OPEN;
  ldout(msgr->cct,10) << "accept success, connect_seq = " << connect_seq << ", sending READY" << dendl;

  // send READY reply
  reply.tag = (reply_tag ? reply_tag : CEPH_MSGR_TAG_READY);
  reply.features = policy.features_supported;
  reply.global_seq = msgr->get_global_seq();
  reply.connect_seq = connect_seq;
  reply.flags = 0;
  reply.authorizer_len = authorizer_reply.length();
  if (policy.lossy)
    reply.flags = reply.flags | CEPH_MSG_CONNECT_LOSSY;

  connection_state->set_features((uint64_t)reply.features & (uint64_t)connect.features);
  ldout(msgr->cct,10) << "accept features " << connection_state->get_features() << dendl;

  session_security.reset(
      get_auth_session_handler(msgr->cct,
			       connect.authorizer_protocol,
			       session_key,
			       connection_state->get_features()));

  // notify
  msgr->dispatch_queue.queue_accept(connection_state.get());
  msgr->ms_deliver_handle_fast_accept(connection_state.get());

  // ok!
  if (msgr->dispatch_queue.stop)
    goto shutting_down;
  removed = msgr->accepting_pipes.erase(this);
  assert(removed == 1);
  register_pipe();
  msgr->lock.Unlock();
  pipe_lock.Unlock();

  /*回复client 端*/
  r = tcp_write((char*)&reply, sizeof(reply));
  if (r < 0) {
    goto fail_registered;
  }

  if (reply.authorizer_len) {
    r = tcp_write(authorizer_reply.c_str(), authorizer_reply.length());
    if (r < 0) {
      goto fail_registered;
    }
  }


  if (reply_tag == CEPH_MSGR_TAG_SEQ) {
    if (tcp_write((char*)&existing_seq, sizeof(existing_seq)) < 0) {
      ldout(msgr->cct,2) << "accept write error on in_seq" << dendl;
      goto fail_registered;
    }
    if (tcp_read((char*)&newly_acked_seq, sizeof(newly_acked_seq)) < 0) {
      ldout(msgr->cct,2) << "accept read error on newly_acked_seq" << dendl;
      goto fail_registered;
    }
  }

  pipe_lock.Lock();
  discard_requeued_up_to(newly_acked_seq);
  if (state != STATE_CLOSED) {
    ldout(msgr->cct,10) << "accept starting writer, state " << get_state_name() << dendl;
    
    /*启动Pipe的写线程*/
    start_writer();
  }
  ldout(msgr->cct,20) << "accept done" << dendl;

  maybe_start_delay_thread();

  return 0;   // success.

```

但是如果client端发过来的connect_seq不是0，而服务器端又找不到负责和client地址通信的Pipe，那就说明服务器端reset了，需要把这个情况通过CEPH\_MSGR\_TAG\_RESETSESSION tag告知client端，client端收到这个tag之后，会执行was\_session\_reset函数，然后将cseq设置成0，然后重新发送 connect message。

```
/*client端收到CEPH_MSGR_TAG_RESETSESSION，执行的动作*/
    if (reply.tag == CEPH_MSGR_TAG_RESETSESSION) {
      ldout(msgr->cct,0) << "connect got RESETSESSION" << dendl;
      was_session_reset();
      cseq = 0;
      pipe_lock.Unlock();
      continue;
    }
    
```
下面我们看下 was\_session\_reset 函数：

```
void Pipe::was_session_reset()
{
  assert(pipe_lock.is_locked());

  ldout(msgr->cct,10) << "was_session_reset" << dendl;
  in_q->discard_queue(conn_id);
  if (delay_thread)
    delay_thread->discard();
  discard_out_queue();

  msgr->dispatch_queue.queue_remote_reset(connection_state.get());

  if (randomize_out_seq()) {
    lsubdout(msgr->cct,ms,15) << "was_session_reset(): Could not get random bytes to set seq number for session reset; set seq number to " << out_seq << dendl;
  }

  in_seq = 0;
  connect_seq = 0;
}
```

前面讨论的是服务器端找不到client addr通信的Pipe，如果在服务器端找到了面向client端的Pipe，那么需要根据收到的client端发过来的global\_seq和connect\_seq和服务器端的global_seq和connect\_seq的值的关系，来判断当前的情况，采取不同的行动。

比如 client端发过来的connect\_seq ＝ 0 ，而服务器端的connect\_seq不是0，那么表明，client端的会话重置了，而服务器的正确的行为是清掉老的的会话，然后用新的Pipe来替换。

这中间其他的可能性就不再赘述了。

至此，client端和服务器端的会话就建立起来，后续的通信，就是有事说事。后面的部分和Dispatcher以及DispatchQueue关系比较密切，放倒下一篇介绍。



