---
layout: post
title: ceph internal 之 Throttle
date: 2016-04-26 21:34:40
categories: ceph internal
tag: ceph internal
excerpt: 本文介绍ceph Throttle机制
---

前言
----

其实熟悉kernel的都应该听说过throttle这个单词，这个单词含义是节流阀，节气阀，作用顾名思义就是流量控制的。对于一个大型的系统，上下游特别地多，如果没有流量控制，很可能因为处理速度不匹配，导致大量事务积压在某处，导致系统处于不可控的状态。

这个功能，其实是一个限速的功能。我们关注的某个指标到了一定的值，我们就拒不接受，让会导致指标的上涨的进程或线程陷入等待，直到指标回落到正常的方位。

其实想想，限速有很多种，最简单的就类似停车场，一共100个车位，当当前值已经到了100的时候，后面的请求一律无限期阻塞，知道某个车位空出来为止，这种情况是刚性的限流。

还有一种是柔性的限速，按照当前的水位来调节，有低水位阈值和高水位阈值。如果指标的当前值处于低水位阈值之下，那么无需调节，立刻处理，如果当前的水位处于低水位阈值和高水位阈值之间，轻微调节下，让线程或者进程等待少量的时间后再处理请求，如果当前的水位高于了高水位阈值，那么就让线程或进程多等待一些时间。 比较优秀的算法，自然是水位越高，需要等待的时间越多。ceph也实现了这种Throttle，即 BackoffThrottle。


## 刚性throttle 


看下成员变量，不难猜测其含义：

```
class Throttle {
  CephContext *cct;
  const std::string name;
  PerfCounters *logger;
  ceph::atomic_t count, max;
  Mutex lock;
  list<Cond*> cond;
  const bool use_perf;
```

其中max即前文提到的车位个数，即最大容许值，如果当前请求（流量，指标 Whatever）会导致count值超过max，一律停止响应，陷入等待，即刚性的Throttle。

count毫无疑问就是当前的指标值。该值不能超过max，否则请求就会阻塞。

看下其判断是否需要等待的函数：

```
    bool _should_wait(int64_t c) const {
	    int64_t m = max.read();
	    int64_t cur = count.read();
	    return
	      m &&
	      ((c <= m && cur + c > m) || // normally stay under max
	       (c >= m && cur > m));     // except for large c
  }
```

简单说，当前值 ＋ 请求的值如果超过上限，就返回true，应该等待。

这是典型的多值信号量的行为，可是ceph采用了条件变量配置互斥量来实现。

```
bool Throttle::_wait(int64_t c)
{
	utime_t start;
	bool waited = false;
	
	/*如果需要等待，或者已经有人再等待，毫无疑问，我们需要等待*/
	if (_should_wait(c) || !cond.empty()) { // always wait behind other waiters.
		Cond *cv = new Cond;
		cond.push_back(cv);
		do {
			if (!waited) {
				ldout(cct, 2) << "_wait waiting..." << dendl;
				if (logger)
					start = ceph_clock_now(cct);
			}
			waited = true;
			cv->Wait(lock);
		} while (_should_wait(c) || cv != cond.front());

		if (waited) {
			ldout(cct, 3) << "_wait finished waiting" << dendl;
			if (logger) {
				utime_t dur = ceph_clock_now(cct) - start;
				logger->tinc(l_throttle_wait, dur);
			}
		}

		delete cv;
		cond.pop_front();

		// wake up the next guy
		if (!cond.empty())
			cond.front()->SignalOne();
	}
	return waited;
}
```

如果需要等待的话，将创建一个条件变量，推送到条件变量列表中，然后通过cv->Wait函数等待在互斥量上。除非有人释放资源（腾出车位），这时候，释放资源的人会掉用cond.front()->SignalOne();唤醒排在列表第一个等待变量上的线程。这样就实现了先来先服务。当然可能存在虚假唤醒，醒来的时候，还要在此判断是否需要继续等待，即do while循环的条件部分。

如果等待结束，被唤醒了，那么会更新等待的时间，同时删除条件变量，

```
int64_t Throttle::put(int64_t c)
{
  if (0 == max.read()) {
    return 0;
  }

  assert(c >= 0);
  ldout(cct, 10) << "put " << c << " (" << count.read() << " -> " << (count.read()-c) << ")" << dendl;
  Mutex::Locker l(lock);
  if (c) {
    if (!cond.empty())
      cond.front()->SignalOne();
    assert(((int64_t)count.read()) >= c); //if count goes negative, we failed somewhere!
    count.sub(c);
    if (logger) {
      logger->inc(l_throttle_put);
      logger->inc(l_throttle_put_sum, c);
      logger->set(l_throttle_val, count.read());
    }
  }
  return count.read();
}
```

如果某线程释放了资源，腾出了车位，需要通知陷入阻塞的线程（如果存在的话），还要更新count的值

```
    if (!cond.empty())
      cond.front()->SignalOne();
      
    count.sub(c)      
```


这里面又个很有意思的唤醒链条，如果有很多线程被阻塞，都通过条件变量等待条件满足，Throttle::put会唤醒condition_variable列表的第一个线程，

 if (!cond.empty())
      cond.front()->SignalOne();
      
但是很有意思的是列表上的第一个线程在Throttle::_wait会尝试唤醒下一个，这是为何？

		// wake up the next guy
		if (!cond.empty())
			cond.front()->SignalOne();

原因是，释放资源可能不止释放一个资源，比如条件变量列表上的线程分别等待2 3 4 5 6 个资源，而put一下子释放了10个资源，那么，条件等待列表中的第一个线程自然被掉用put的线程唤醒，而只消耗了2个资源，还有8个资源可用，因此第一个线程负责叫醒第二个线程，第二个线程叫醒第三个线程，但是当第三个线程叫醒第四个线程时，第四个线程一看，还剩余10-2 -3 -4 = 1个资源，但是自己需要5个资源，就再次陷入阻塞，自然也不会叫醒第五个线程了。


## 柔性Throttle


有一个故事讲和珅去赈灾，在施粥棚的的锅里撒了一些沙土，别人不解，和珅说，只有粥里掺了沙土，真正的灾民才不会在乎，而不是真正的灾民，穷苦人家里有口吃的，就不会去吃这些粥了。

这个故事和柔型Throttle有些类似，就是通过某些手段，减缓到来的流量。还是以施舍粥为例，一开始灾民少，粥多，自然大家随便吃；再后来人也来越多，可以在粥里多參水，让粥稀一些；在后来人还是很多，碗可以不盛满，每个灾民7分满，再后来，估计就要撒沙土。

柔型Throttle也是如此，不过是在低水位，可以不限速，到了低水位和高水位之间，可以睡眠一小段时间，高过高水位，就睡眠时间变长。和珅参的是沙土，我们參的是睡眠时间。

到底要參多少水呢？

低水位门限值值归一化（low_threshold/max)定位l，将高水位门限值值归一化为h(high_threshold/max)，当前的值归一化为r(current/max),
同时定义两个指标，即high_delay和max_delay，所以high delay是指，当 

```
 * [0, low_threshold)                 delay = 0 
 * [low_threshold, high_threshold)    delay = (r - l) * (e / (h - l))
 * [high_threshold,1)                 delay = e + (r - h)((m - e)/(1 - h))
```

注意，公式第三行并不是笔误，而是ceph代码的注释有错误。

这个公式乍看之下很唬人，其实比较简单，就是定义两个参数，high_multiple和max_multiple,以及expect_throught，我个人觉得，expected_throughput反倒不容易理解，我们这么理解：

high_delay = high_multiple / expected_throughput
max_delay =  max_multiple ／ expecte_throughput

从此刻起，我们忘记high_multiple,max_multiple ，也忘记expected_throughput,单单记住high_delay和max_delay。
当水位的值时high时，延迟的值为high_delay,当水位是l时，延迟的值时0，因此，在［l,h)这段区域内延迟的值线性增长到high_delay.

当水位的值时1时，即max，延迟的值时max_delay,因此，从［h,1)这段区间内，也是线性增长，从high_delay一直增长到max_delay,只不过斜率不同罢了。

然后我们再次回到expect_throught， 这个变量的作用和它的名字一样，即期待的吞吐量，如果取倒数，我们会得到期待的处理速度：

expect_delay = 1 / expected_throughput 

我们以high_multiple = 2  max_multiple = 10为例来讲解， 如果当前的水位是high，正好等于high，那么此时的delay是

high_delay = 2* expected_delay
max_delay = 10*expect_delay 

即，如果消费者线程处理消息的速度是2*expected_delay, 那么水位会维持在high值，始终不变。 2＊expected_delay的时间正好处理完毕一个，而正好有一个新的消息进入需要处理。

如果消费者处理线程的速度进一步下降，当水位到达1时，Throttle会将消息delay 10*expected_delay这么久，只要处理一个消息的时间低于10＊expected_delay,那么处理消息的速度还是会超过消息到来的速度，从而让积压的消息越来越少，水位也开始下降。

我们以low水位为0.4，高水位为0.6，high_multiple =2  max_multiple = 10 为例，sleep的时间如下图所示。

![](/assets/ceph_internals/backoff_throttle.png)

当然了，delay的单位为 expected_delay,即 expected_throughput的倒数。


这部分逻辑体现在如下部分代码：

```
bool BackoffThrottle::set_params(
  double _low_threshhold,
  double _high_threshhold,
  double _expected_throughput,
  double _high_multiple,
  double _max_multiple,
  uint64_t _throttle_max,
  ostream *errstream)
{
  bool valid = true;
  if (_low_threshhold > _high_threshhold) {
    valid = false;
    if (errstream) {
      *errstream << "low_threshhold (" << _low_threshhold
		 << ") > high_threshhold (" << _high_threshhold
		 << ")" << std::endl;
    }
  }

  if (_high_multiple > _max_multiple) {
    valid = false;
    if (errstream) {
      *errstream << "_high_multiple (" << _high_multiple
		 << ") > _max_multiple (" << _max_multiple
		 << ")" << std::endl;
    }
  }

  if (_low_threshhold > 1 || _low_threshhold < 0) {
    valid = false;
    if (errstream) {
      *errstream << "invalid low_threshhold (" << _low_threshhold << ")"
		 << std::endl;
    }
  }

  if (_high_threshhold > 1 || _high_threshhold < 0) {
    valid = false;
    if (errstream) {
      *errstream << "invalid high_threshhold (" << _high_threshhold << ")"
		 << std::endl;
    }
  }

  if (_max_multiple < 0) {
    valid = false;
    if (errstream) {
      *errstream << "invalid _max_multiple ("
		 << _max_multiple << ")"
		 << std::endl;
    }
  }

  if (_high_multiple < 0) {
    valid = false;
    if (errstream) {
      *errstream << "invalid _high_multiple ("
		 << _high_multiple << ")"
		 << std::endl;
    }
  }

  if (_expected_throughput < 0) {
    valid = false;
    if (errstream) {
      *errstream << "invalid _expected_throughput("
		 << _expected_throughput << ")"
		 << std::endl;
    }
  }

  if (!valid)
    return false;

  locker l(lock);
  low_threshhold = _low_threshhold;
  high_threshhold = _high_threshhold;
  high_delay_per_count = _high_multiple / _expected_throughput;
  max_delay_per_count = _max_multiple / _expected_throughput;
  max = _throttle_max;

  if (high_threshhold - low_threshhold > 0) {
    s0 = high_delay_per_count / (high_threshhold - low_threshhold);
  } else {
    low_threshhold = high_threshhold;
    s0 = 0;
  }

  if (1 - high_threshhold > 0) {
    s1 = (max_delay_per_count - high_delay_per_count)
      / (1 - high_threshhold);
  } else {
    high_threshhold = 1;
    s1 = 0;
  }

  _kick_waiters();
  return true;
}

std::chrono::duration<double> BackoffThrottle::_get_delay(uint64_t c) const
{
  if (max == 0)
    return std::chrono::duration<double>(0);

  double r = ((double)current) / ((double)max);
  if (r < low_threshhold) {
    return std::chrono::duration<double>(0);
  } else if (r < high_threshhold) {
    return c * std::chrono::duration<double>(
      (r - low_threshhold) * s0);
  } else {
    return c * std::chrono::duration<double>(
      high_delay_per_count + ((r - high_threshhold) * s1));
  }
}

```

说它时柔性限速，是因为它并不死等，而是根据是否是条件等待队列的第一个元素来决定是傻等还是最多等待有限时间。

### 如果线程是第一个进入队列等待的元素：

```
 while (((start + delay) > std::chrono::system_clock::now()) ||
	 !((max == 0) || (current == 0) || ((current + c) <= max))) {
    assert(ticket == waiters.begin());
    (*ticket)->wait_until(l, start + delay);
    delay = _get_delay(c);
  }
  
  waiters.pop_front();
  _kick_waiters();

  current += c;
  return std::chrono::system_clock::now() - start;
```
该线程最多等delay这么久（通过条件变量的wait_util），哪怕没有等待资源释放，因为如下条件满足也就不等了。

```
((start + delay) > std::chrono::system_clock::now()) 
```

另一种可能性是在等待过程中，由于其它线程释放了资源，导致wait_util及时醒来，发现current ＋ c < max 因此也不需要等待了。


### 如果线程不是第一个进入队列等待

很可惜，策略是傻等

```
  while (waiters.begin() != ticket) {
    (*ticket)->wait(l);
  }
```

但是也有转机，转机有两个，

转机1 ： 是有线程释放资源，

```
uint64_t BackoffThrottle::put(uint64_t c)
{
  locker l(lock);
  assert(current >= c);
  current -= c;
  _kick_waiters();
  return current;
}

```
转机2: 队列曾经的头元素等了足够长的时间，或者因为有资源了，已经通过waiters.pop_front()离开队列，并且，通过_kick_waiters将消息传递出去，通知其它人情况有变. 队列的第二个元素就有了转机，它就不死等了，可以转入有限期的等待了。

```
 void _kick_waiters() {
    if (!waiters.empty())
      waiters.front()->notify_all();
  }
```

## SimpleThrottle

顾名思义，这是个简化版，简化在它每次只申请一个资源，释放自然也是只释放一个元素，不像通用的版本，可以申请或者释放n个资源。至于等待策略，自然也是无限期傻等。

```
void SimpleThrottle::start_op()
{
  Mutex::Locker l(m_lock);
  while (m_max == m_current)
    m_cond.Wait(m_lock);
  ++m_current;
}

void SimpleThrottle::end_op(int r)
{
  Mutex::Locker l(m_lock);
  --m_current;
  if (r < 0 && !m_ret && !(r == -ENOENT && m_ignore_enoent))
    m_ret = r;
  m_cond.Signal();
}
```


