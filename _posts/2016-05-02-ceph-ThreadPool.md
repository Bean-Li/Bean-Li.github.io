---
layout: post
title: ceph internal 之 ThreadPool和WorkQueue
date: 2016-05-02 22:34:40
categories: ceph-internal
tag: ceph-internal
excerpt: 本文介绍ceph 的线程池ThreadPool和工作队列WorkQueue
---

## 前言

线程池和工作队列其实是密不可分的，从ceph的代码中也可以看出来。让任务推入工作队列，而线程池中的线程负责从工作队列中取出任务进行处理。这种处理任务的模式任何一个有C编程经验的人都不会陌生。当年我在ZTE和Trend工作的时候，都写过或者维护过类似的代码。

工作队列和线程池的关系，类似于狡兔和走狗的关系，正是因为有任务，所以才需要雇佣线程来完成任务，没有了狡兔，走狗也就失去了存在的意义。而线程必须要可以从工作队列中认领任务并完成，这就类似于猎狗要有追捕狡兔的功能。正因为两个数据结构拥有如此紧密的关系，因此，Ceph中他们的相关函数都位于WorkQueue.cc和WorkQueue.h中。


## 线程池线程个数调整

线程池的关键在于线程的主函数做的事情。首先是工作线程.

线程池中会有很多的WorkThread，它的基类就是Thread。线程的主函数为pool->worker，即ThreadPool::worker函数。

```
  struct WorkThread : public Thread {
    ThreadPool *pool;
    // cppcheck-suppress noExplicitConstructor
    WorkThread(ThreadPool *p) : pool(p) {}
    void *entry() {
      pool->worker(this);
      return 0;
    }
  };
```

```
void ThreadPool::worker(WorkThread *wt)

```

线程池和heartbeat是交织在一起的，后面会有专门的文章介绍，在此先略过。
线程池是支持动态调整线程个数的。所谓调整，有两种可能性，一种是线程个数增加，一种线程个数减少。我们知道，当添加OSD的时候，数据会重分布，恢复的速度可以调节，其中一个重要的参数为osd-max-recovery-threads，该值修改可以实时生效。

```
ceph tell osd.* injectargs '--osd-max-recovery-threads 8'
```

该值之所以可以实时生效，说到底，不过是因为OSD类中的recovery_tp就是一种普通的ThreadPool而已

```

private:

  ThreadPool osd_tp;
  ShardedThreadPool osd_op_tp;
  ThreadPool recovery_tp;
  ThreadPool disk_tp;
  ThreadPool command_tp;

```

### 线程个数减少

线程个数减少相对比较简单，线程自杀即可。

```
void ThreadPool::worker(WorkThread *wt)
{  
  _lock.Lock();
  ldout(cct,10) << "worker start" << dendl;
  
  std::stringstream ss;
  ss << name << " thread " << (void*)pthread_self();
  heartbeat_handle_d *hb = cct->get_heartbeat_map()->add_worker(ss.str());

  while (!_stop) {

    // manage dynamic thread pool
    join_old_threads();
    if (_threads.size() > _num_threads) {
      ldout(cct,1) << " worker shutting down; too many threads (" << _threads.size() << " > " << _num_threads << ")" << dendl;
      _threads.erase(wt);
      _old_threads.push_back(wt);
      break;
    }
```

线程本身是一个loop，不停地处理WorkQueue中的任务，在一个loop的开头，线程个数是否超出了配置的个数，如果超出了，就需要自杀，所谓自杀即将自身推送到\_old\_threads中，然后跳出loop，直接返回了。线程池中的其他兄弟在busy-loop开头的join_old_threads函数会判断是否存在自杀的兄弟，如果存在的话，执行join，为兄弟收尸。

```
void ThreadPool::join_old_threads()
{
  assert(_lock.is_locked());
  while (!_old_threads.empty()) {
    ldout(cct, 10) << "join_old_threads joining and deleting " << _old_threads.front() << dendl;
    _old_threads.front()->join();
    delete _old_threads.front();
    _old_threads.pop_front();
  }
}
```

### 线程个数增加

线程池的线程个数如果不够用，也可以动态的增加，通过配置的变化来做到：

```
void ThreadPool::handle_conf_change(const struct md_config_t *conf,
				    const std::set <std::string> &changed)
{
  if (changed.count(_thread_num_option)) {
    char *buf;
    int r = conf->get_val(_thread_num_option.c_str(), &buf, -1);
    assert(r >= 0);
    int v = atoi(buf);
    free(buf);
    if (v > 0) {
      _lock.Lock();
      _num_threads = v;
      start_threads();
      _cond.SignalAll();
      _lock.Unlock();
    }
  }
}

void ThreadPool::start_threads()
{
  assert(_lock.is_locked());
  while (_threads.size() < _num_threads) {
    WorkThread *wt = new WorkThread(this);
    ldout(cct, 10) << "start_threads creating and starting " << wt << dendl;
    _threads.insert(wt);

    int r = wt->set_ioprio(ioprio_class, ioprio_priority);
    if (r < 0)
      lderr(cct) << " set_ioprio got " << cpp_strerror(r) << dendl;

    wt->create(thread_name.c_str());
  }
}
```

start\_threads函数不仅仅可以用在初始化时启动所有工作线程，而且可以用于动态增加，它会根据配置要求的线程数_num_threads和当前线程池中线程的个数，来创建WorkThread，当然了，他会调整线程的io优先级。


### 工作线程的暂停执行和恢复执行

线程池的工作线程，绝大部分时间内，自然是busy－loop中处理工作队列上的任务，但是有一种场景是，需要让工作暂时停下来，停止工作，不要处理WorkQueue中的任务。

线程池提供了一个标志为_pause,只要_pause不等于0，那么线程池中线程就在loop中就不会处理工作队列中的任务，而是空转。为了能够及时的醒来，也不是sleep，而是通过条件等待，等待执行的时间。

```
void ThreadPool::pause()
{
  ldout(cct,10) << "pause" << dendl;
  _lock.Lock();
  _pause++;
  while (processing)
    _wait_cond.Wait(_lock);
  _lock.Unlock();
  ldout(cct,15) << "paused" << dendl;
}

void ThreadPool::pause_new()
{
  ldout(cct,10) << "pause_new" << dendl;
  _lock.Lock();
  _pause++;
  _lock.Unlock();
}



void ThreadPool::worker(WorkThread *wt)
{
  while (!_stop) {

    // manage dynamic thread pool
    join_old_threads();
    if (_threads.size() > _num_threads) {
      ldout(cct,1) << " worker shutting down; too many threads (" << _threads.size() << " > " << _num_threads << ")" << dendl;
      _threads.erase(wt);
      _old_threads.push_back(wt);
      break;
    }

    if (!_pause && !work_queues.empty()) {
      ...
    }

    ldout(cct,20) << "worker waiting" << dendl;
    
    /*重新设置timeout时间，放置误判timeout*/
    cct->get_heartbeat_map()->reset_timeout(
      hb,
      cct->_conf->threadpool_default_timeout,
      0);
      
    /*此处相当于sleep，但是条件变量的存在，一旦情况有变（比如调用了unpause函数）能及时醒来*/
    _cond.WaitInterval(cct, _lock,
      utime_t(
	            cct->_conf->threadpool_empty_queue_max_wait, 0));
  }
  
  ...
}
```

那么处理pause_new和pause 函数做的事情差不多，两者有什么区别呢？关键在于

```
  while (processing)
    _wait_cond.Wait(_lock);
```

当下达pause指令的时候，很可能线程池中的某几个线程正在处理工作队列中的任务，这种情况下并不是立刻就能停下的，只有处理完手头的任务，在下一轮loop中检查_pause标志位才能真正地停下。那么pause指令就面临选择，要不要等工作线程WorkThread处理完手头的任务。pause函数是等，pauser_new函数并不等，pause_new函数只负责设置标志位，当其返回的时候，某几个线程可能仍然在处理工作队列中的任务。



### 工作线程的工作内容

讲了这么多，基本都是旁枝，而不是工作线程的主干，主干部分是处理工作队列中的任务：

```
if (!_pause && !work_queues.empty()) {
      WorkQueue_* wq;
      int tries = work_queues.size();
      bool did = false;
      while (tries--) {
	       last_work_queue++;
	       last_work_queue %= work_queues.size();
	       wq = work_queues[last_work_queue];
	
			   void *item = wq->_void_dequeue();
			   if (item) {
			      processing++;
			      ldout(cct,12) << "worker wq " << wq->name << " start processing " << item
					    << " (" << processing << " active)" << dendl;
			      TPHandle tp_handle(cct, hb, wq->timeout_interval, wq->suicide_interval);
			      /*重设heartbeat的超时时间*/
			      tp_handle.reset_tp_timeout();
			      _lock.Unlock();
			      wq->_void_process(item, tp_handle);
			      _lock.Lock();
			      wq->_void_process_finish(item);
			      processing--;
			      ldout(cct,15) << "worker wq " << wq->name << " done processing " << item
					    << " (" << processing << " active)" << dendl;
			      if (_pause || _draining)
			         _wait_cond.Signal();
			      did = true;
			      break;
			   }
      }
     if (did)
	    continue;
}
```

其中\_void\_process和\_void\_process\_finish是WorkQueue基类中定义的函数，真正定义工作队列的时候，可以继承该基类，定义自己的_process函数和_process_finish	函数，来雇用线程完成特定的任务。

```
template<class T>
  class WorkQueue : public WorkQueue_ {
    ThreadPool *pool;
    
    /// Add a work item to the queue.
    virtual bool _enqueue(T *) = 0;
    /// Dequeue a previously submitted work item.
    virtual void _dequeue(T *) = 0;
    /// Dequeue a work item and return the original submitted pointer.
    virtual T *_dequeue() = 0;
    virtual void _process_finish(T *) {}

    // implementation of virtual methods from WorkQueue_
    void *_void_dequeue() {
      return (void *)_dequeue();
    }
    void _void_process(void *p, TPHandle &handle) {
      _process(static_cast<T *>(p), handle);
    }
    void _void_process_finish(void *p) {
      _process_finish(static_cast<T *>(p));
    }

  protected:
    /// Process a work item. Called from the worker threads.
    virtual void _process(T *t, TPHandle &) = 0;

  public:
    WorkQueue(string n, time_t ti, time_t sti, ThreadPool* p) : WorkQueue_(n, ti, sti), pool(p) {
      pool->add_work_queue(this);
    }
    ~WorkQueue() {
      pool->remove_work_queue(this);
    }
```

我们不妨以FileStore的op\_tp和op\_wq为例查看下该线程池中工作线程的日常任务。

```
ThreadPool op_tp;
  struct OpWQ : public ThreadPool::WorkQueue<OpSequencer> {
    FileStore *store;
    OpWQ(FileStore *fs, time_t timeout, time_t suicide_timeout, ThreadPool *tp)
      : ThreadPool::WorkQueue<OpSequencer>("FileStore::OpWQ", timeout, suicide_timeout, tp), store(fs) {}

    bool _enqueue(OpSequencer *osr) {
      store->op_queue.push_back(osr);
      return true;
    }
    void _dequeue(OpSequencer *o) {
      assert(0);
    }
    bool _empty() {
      return store->op_queue.empty();
    }
    OpSequencer *_dequeue() {
      if (store->op_queue.empty())
	       return NULL;
      OpSequencer *osr = store->op_queue.front();
      store->op_queue.pop_front();
      return osr;
    }
    void _process(OpSequencer *osr, ThreadPool::TPHandle &handle) override {
      store->_do_op(osr, handle);
    }
    void _process_finish(OpSequencer *osr) {
      store->_finish_op(osr);
    }
    void _clear() {
      assert(store->op_queue.empty());
    }
  } op_wq;

```

## 工作队列和线程池建立合作关系

好像至今也没有介绍如何建立起合作关系，还是以FileStore的op_tp和op_wq为例：

```
  op_tp(g_ceph_context, "FileStore::op_tp", "tp_fstore_op", g_conf->filestore_op_threads, "filestore_op_threads"),
  
  op_wq(this, g_conf->filestore_op_thread_timeout,
	g_conf->filestore_op_thread_suicide_timeout, &op_tp),
```

FileStore实例化的时候，op_tp作为参数传递给了op_wq的构造函数。

我们不妨看看WorkQueue的构造函数和析构函数：

```
 public:
    WorkQueue(string n, time_t ti, time_t sti, ThreadPool* p) : WorkQueue_(n, ti, sti), pool(p) {
      pool->add_work_queue(this);
    }
    ~WorkQueue() {
      pool->remove_work_queue(this);
    }
    
    
```

而ThreadPool中的add_work_queue和remove_work_queue就是用来建立和移除与WorkQueue关联的函数

```
  /// assign a work queue to this thread pool
  void add_work_queue(WorkQueue_* wq) {
    Mutex::Locker l(_lock);
    work_queues.push_back(wq);
  }
  /// remove a work queue from this thread pool
  void remove_work_queue(WorkQueue_* wq) {
    Mutex::Locker l(_lock);
    unsigned i = 0;
    while (work_queues[i] != wq)
      i++;
    for (i++; i < work_queues.size(); i++) 
      work_queues[i-1] = work_queues[i];
    assert(i == work_queues.size());
    work_queues.resize(i-1);
  }
```

因此建立狼狈为奸的关系，需要先创建线程池，然后创建WorkQueue的时候，将线程池作为参数传递给WorkQueue，就能建立关系。

