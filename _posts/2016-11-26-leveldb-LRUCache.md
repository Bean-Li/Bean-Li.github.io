---
layout: post
title: leveldb中的LRUCache设计
date: 2016-11-26 22:29
categories: leveldb
tag: leveldb
excerpt: 本文介绍leveldb中的重要数据结构LRUCache
---

# 前言
leveldb是Google大神Jeff Dean的作品，只有2万行出头的代码量，非常精彩，非常值得解剖学习。本文介绍leveldb的LRUCache设计。
我们知道，相对于块设备，内存永远都是奢侈品。局部性原理应该算是计算机科学中数一数二的重要原理，无数精巧的设计，不过是让有限的内存提供更高的用户数据访问命中，茫茫海量的数据，我需要你的时候，你恰巧在内存，这才是工程师不不懈的追求。

内存是稀缺资源，缓存替换算法是计算机科学的重要Topic。LRU （Last Recent Used）是一个重要的缓存替换算法。Leveldb采用的就是这种算法。但是仅仅知道LRU这三个字母是没啥用的，我们一起学习下leveldb的缓存替换算法，体会下大神的精巧设计。


# 内部实现

没有文档的情况下，沉入代码细节，容易只见树木不见森林，需要花费好久才能体会出作者的设计思路。代码告诉你How，而设计文档才告诉你Why。

LevelDB的LRUCache设计有4个数据结构，是依次递进的关系，分别是：

* LRUHandle
* HandleTable
* LRUCache
* ShardedLRUCache

事实上到了第三个数据结构LRUCache，LRU的缓存管理数据结构已经实现了，之所以引入第四个数据结构，就是因为减少竞争。因为多线程访问需要加锁，为了减少竞争，提升效率，ShardedLRUCache内部有16个LRUCache，查找key的时候，先计算属于哪一个LRUCache，然后在相应的LRUCache中上锁查找。

```
class ShardedLRUCache : public Cache {  
 private:  
  LRUCache shard_[kNumShards];  
  ...
}
```
这不是什么高深的思路，这种减少竞争的策略非常常见。因此，读懂缓存管理策略的关键在前三个数据结构。

LevelDB的Cache管理，维护有2个双向链表和一个哈希表。哈希表是非常容易理解的。如何确定一个key值到底存不存在，如果存在如何快速获取key值对应的value值。我们都学过数据结构，这活，哈希表是比较适合的。

注意，我们都知道，hash表存在一个重要的问题，就是碰撞，有可能多个不同的键值hash之后值相同，解决碰撞的一个重要思路是链表，将hash之后计算的key相同的元素链入同一个表头对应的链表。

可是我们并不满意这种速度，LevelDB做了进一步的优化，即及时扩大hash桶的个数，尽可能地不会发生碰撞。因此LevelDB自己实现了一个hash表，即HandleTable数据结构。

说句题外话，我不太喜欢数据结构的命名方式，比如HandleTable，命名就是个HashTable，如果出现Hash会好理解很多。这个名字还自罢了，LRUHandle这个名字更是让人摸不到头脑，明明就是一个数据节点，如果名字中出现Node，整个代码都会好理解很多。好了吐槽结束，看下HandleTable的数据结构：


```
class HandleTable {
 public:
  
 ...
 
 private:

  uint32_t length_;
  uint32_t elems_;
  LRUHandle** list_;
```

第一个元素length_ 纪录的就是当前hash桶的个数，第二个元素 elems_维护在整个hash表中一共存放了多少个元素。第三个就更好理解了，二维指针，每一个指针指向一个桶的表头位置。

为了提升查找效率，提早增加桶的个数来尽可能地保持一个桶后面只有一个元素，在插入的算法中有如下内容：

```
  LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h->key(), h->hash);
    LRUHandle* old = *ptr;  //老的元素返回，LRUCache会将相同key的老元素释放，详情看LRUCache的Insert函数。
    h->next_hash = (old == NULL ? NULL : old->next_hash);
    *ptr = h;
    if (old == NULL) {
      ++elems_;
      if (elems_ > length_) {
        // Since each cache entry is fairly large, we aim for a small
        // average linked list length (<= 1).
        Resize();
      }
    }
    return old;
  }
```

注意，当整个hash表中元素的个数超过 hash表桶的的个数的时候，调用Resize函数，该函数会将桶的个数增加一倍，同时将现有的元素搬迁到合适的桶的后面。正是这种提早扩大桶的个数，良好的hash函数会保证每个桶对应的链表中尽可能的只有1个元素，从这个角度讲，LevelDB使用这种优化后的哈希表，查找的效率为O（1）。

```
  void Resize() {
    uint32_t new_length = 4;
    while (new_length < elems_) {
      new_length *= 2;
    }
    LRUHandle** new_list = new LRUHandle*[new_length];
    memset(new_list, 0, sizeof(new_list[0]) * new_length);
    uint32_t count = 0;
    for (uint32_t i = 0; i < length_; i++) {
      LRUHandle* h = list_[i];
      while (h != NULL) {
        LRUHandle* next = h->next_hash;
        uint32_t hash = h->hash;
        LRUHandle** ptr = &new_list[hash & (new_length - 1)]; //各个已有的元素重新计算，应该落在哪个桶的链表中。
        h->next_hash = *ptr;
        *ptr = h;
        h = next;
        count++;
      }
    }
    assert(elems_ == count);
    delete[] list_;
    list_ = new_list;
    length_ = new_length;
  }
```


设计缓存，快速找到位置是一个重要指标，但是毫无疑问，不仅仅只有这一个设计指标。因为使用LRU，当空间不够的时候，需要踢出某些元素的时候，必需能够快速地找到，哪些元素Last Recent Used，作为替换的牺牲品。这个指标哈希表可就爱莫能助了。

当然，我们可以讲最近访问时间作为元素的一个字段保存起来，但是我们不得不扫描整个hash表，将访问时间排序才能知道哪个元素更应该被剔除。毫无疑问效率太低。

LRUCache不管需要哈希表来快速查找，还需要链表能够快速插入和删除。LRUCache 维护有两条双向链表：

* lru_
* in_use_


```
  LRUHandle lru_;      // lru_ 是冷链表，属于冷宫，

  LRUHandle in_use_;   // in_use_ 属于热链表，热数据在此链表

  HandleTable table_; // 哈希表部分已经讲过
```


Ref 函数表示要使用该cache，因此如果对应元素位于冷链表，需要将它从冷链表溢出，链入到热链表：

```
void LRUCache::Ref(LRUHandle* e) {
  if (e->refs == 1 && e->in_cache) {  // If on lru_ list, move to in_use_ list.
    LRU_Remove(e);
    LRU_Append(&in_use_, e);
  }
  e->refs++;
}

void LRUCache::LRU_Remove(LRUHandle* e) {
  e->next->prev = e->prev;
  e->prev->next = e->next;
}

void LRUCache::LRU_Append(LRUHandle* list, LRUHandle* e) {
  // Make "e" newest entry by inserting just before *list
  e->next = list;
  e->prev = list->prev;
  e->prev->next = e;
  e->next->prev = e;
}
```

Unref正好想法，表示客户不再访问该元素，需要将引用计数－－，如果彻底没人用了，引用计数为0了，就可以删除这个元素了，如果引用计数为1，则可以将元素打入冷宫，放入到冷链表：

```

void LRUCache::Unref(LRUHandle* e) {
  assert(e->refs > 0);
  e->refs--;
  if (e->refs == 0) { // Deallocate. 彻底没人访问了，而且也在冷链表中，可以删除了
    assert(!e->in_cache);
    (*e->deleter)(e->key(), e->value); //元素的deleter函数，此时回调。
    free(e);
  } else if (e->in_cache && e->refs == 1) {  // 移入冷链表 lru_
    LRU_Remove(e);
    LRU_Append(&lru_, e);
  }
}
```

注意，缓存必须要要有容量的概念，超过了容量，缓存必须要踢出某些元素，对于我们这种场景而言，就是从冷链表中踢人。


对于LevelDB而言，插入的时候，会判断是否超过了容量，如果超过了事先规划的容量，就会从冷链表中踢人：

```
size_t capacity_; // LRUCache的容量
size_t usage_;    // 当前使用的容量


Cache::Handle* LRUCache::Insert(
    const Slice& key, uint32_t hash, void* value, size_t charge,
    void (*deleter)(const Slice& key, void* value)) {
  MutexLock l(&mutex_);

  LRUHandle* e = reinterpret_cast<LRUHandle*>(
      malloc(sizeof(LRUHandle)-1 + key.size()));
  e->value = value;
  e->deleter = deleter;
  e->charge = charge;
  e->key_length = key.size();
  e->hash = hash;
  e->in_cache = false;
  e->refs = 1;  // for the returned handle.
  memcpy(e->key_data, key.data(), key.size());

  if (capacity_ > 0) {
    e->refs++;  // for the cache's reference.
    e->in_cache = true;
    LRU_Append(&in_use_, e);  //链入热链表
    usage_ += charge;         //使用的容量增加
    FinishErase(table_.Insert(e));  // 如果是更新的话，需要回收老的元素
  } // else don't cache.  (Tests use capacity_==0 to turn off caching.)

  while (usage_ > capacity_ && lru_.next != &lru_) { 
   //如果容量超过了设计的容量，并且冷链表中有内容，则从冷链表中删除所有元素
    LRUHandle* old = lru_.next;
    assert(old->refs == 1);
    bool erased = FinishErase(table_.Remove(old->key(), old->hash));
    if (!erased) {  // to avoid unused variable when compiled NDEBUG
      assert(erased);
    }
  }

  return reinterpret_cast<Cache::Handle*>(e);
}

bool LRUCache::FinishErase(LRUHandle* e) {
  if (e != NULL) {
    assert(e->in_cache);
    LRU_Remove(e);
    e->in_cache = false;
    usage_ -= e->charge;
    Unref(e);
  }
  return e != NULL;
}

```

我们要重点注意下插入逻辑中的

```
  if (capacity_ > 0) {
    e->refs++;  // for the cache's reference.
    e->in_cache = true;
    LRU_Append(&in_use_, e);  //链入热链表
    usage_ += charge;         //使用的容量增加
    FinishErase(table_.Insert(e));  // 如果是更新的话，需要回收老的元素
  } // else don't cache.  (Tests use capacity_==0 to turn off caching.)

```

为什么会调用：

``` 
FinishErase(table_.Insert(e));
```
插入hash表的时候，如果找到了同一个key值的元素已经存在，HandleTable的Insert函数会将老的元素返回：

```

  LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h->key(), h->hash);
    LRUHandle* old = *ptr;  //老的元素返回，LRUCache会将相同key的老元素释放，详情看LRUCache的Insert函数。
    h->next_hash = (old == NULL ? NULL : old->next_hash);
    *ptr = h;
    if (old == NULL) {
      ++elems_;
      if (elems_ > length_) {
        // Since each cache entry is fairly large, we aim for a small
        // average linked list length (<= 1).
        Resize();
      }
    }
    return old;
  }

```

因此LRU的Insert函数内部隐含了更新的操作，会将新的Node加入到Cache中，而老的元素会调用FinishErase函数来决定是移入冷宫还是彻底删除。



最后的最后，看下元素长什么样，就是很坑爹的LRUHandle数据结构，名字太坑爹了，如果叫LRUNode，会好理解很多

```
struct LRUHandle {
  void* value;
  void (*deleter)(const Slice&, void* value);
  LRUHandle* next_hash;
  LRUHandle* next;
  LRUHandle* prev;
  size_t charge;      // TODO(opt): Only allow uint32_t?
  size_t key_length;
  bool in_cache;      // Whether entry is in the cache.
  uint32_t refs;      // References, including cache reference, if present.
  uint32_t hash;      // Hash of key(); used for fast sharding and comparisons
  char key_data[1];   // Beginning of key

  Slice key() const {
    // For cheaper lookups, we allow a temporary Handle object
    // to store a pointer to a key in "value".
    if (next == this) {
      return *(reinterpret_cast<Slice*>(value));
    } else {
      return Slice(key_data, key_length);
    }
  }
};

```
这里面的next_hash 会链入到哈希表对应的桶对应的链表中，而next和prev，不是在热链表就是在冷链表。


```
// LRUHandle数据结构

  LRUHandle* next_hash;
  LRUHandle* next;
  LRUHandle* prev;
  
//LRUCahe对应的数据结构
  LRUHandle lru_;      // lru_ 是冷链表，属于冷宫，

  LRUHandle in_use_;   // in_use_ 属于热链表，热数据在此链表

  HandleTable table_; // 哈希表部分已经讲过
```

服务关系一目了然。

打完收工。

