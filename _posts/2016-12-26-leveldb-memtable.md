---
layout: post
title: leveldb中的memtable
date: 2016-12-26 10:29
categories: leveldb
tag: leveldb
excerpt: 本文介绍leveldb中的重要数据结构memtable
---

# 前言

前面将的是SSTable，这种文件是LevelDB中重要的文件，它是leveldb磁盘文件的一种，它和log文件是类似的，都是持久化的。

用户使用leveldb 插入或者删除key-value pair，并不是直接向SSTable插入。从前面的讨论我们可以看出，SSTable的输入是有序的。很显然用户输入key-value pair的行为是不可预知的，不可能是有序的。

使用TableBuilder创建SSTable的时机是：

* minor compaction ： immutable-memtable 中的key/value dump到磁盘，生成sstable
* major compaction ：sstable compact（level-n sstable(s)与level-n+1 sstables多路归并）生成level-n+1的sstable

那用户层的输入的key-value pair，由leveldb的哪个模块负责?

答案是memtable。

# MemTable 和  Immutable Memtable

MemTable 和 Immutable MemTable两者都位于内存，属于内存的数据结构，这是和前面介绍的SSTable文件最大的不同。但是它俩之间是一样的数据结构，两者的区别在于MemTable可读可写，而Immutable MemTable是只读的，不允许写入。MemTable承接来自客户的key-value请求，当写入的数据占用内存到指定门限，就会自动转成Immutable Memtable，等待Dump到磁盘中。当然leveldb会产生新的MemTable供写操作写入新的key-value pair。

这种思想并不罕见，陈硕的muduo中设计log也是采用了类似的思想，按下不表。

因此，理解的关键是MemTable，理解了MemTable，自然也就理解了Immutable MemTable。

# MemTable的数据结构

MemTable的核心组件有三个：

* SkipTable
* Arena
* KeyComparator 

```
class MemTable {
 public:
  // MemTables are reference counted.  The initial reference count
  // is zero and the caller must call Ref() at least once.
  explicit MemTable(const InternalKeyComparator& comparator);

  // Increase reference count.
  void Ref() { ++refs_; }

  // Drop reference count.  Delete if no more references exist.
  void Unref() {
    --refs_;
    assert(refs_ >= 0);
    if (refs_ <= 0) {
      delete this;
    }
  }

  // Returns an estimate of the number of bytes of data in use by this
  // data structure. It is safe to call when MemTable is being modified.
  size_t ApproximateMemoryUsage();

  // Return an iterator that yields the contents of the memtable.
  //
  // The caller must ensure that the underlying MemTable remains live
  // while the returned iterator is live.  The keys returned by this
  // iterator are internal keys encoded by AppendInternalKey in the
  // db/format.{h,cc} module.
  Iterator* NewIterator();

  // Add an entry into memtable that maps key to value at the
  // specified sequence number and with the specified type.
  // Typically value will be empty if type==kTypeDeletion.
  void Add(SequenceNumber seq, ValueType type,
           const Slice& key,
           const Slice& value);

  // If memtable contains a value for key, store it in *value and return true.
  // If memtable contains a deletion for key, store a NotFound() error
  // in *status and return true.
  // Else, return false.
  bool Get(const LookupKey& key, std::string* value, Status* s);

 private:
  ~MemTable();  // Private since only Unref() should be used to delete it

  struct KeyComparator {
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) { }
    int operator()(const char* a, const char* b) const;
  };
  friend class MemTableIterator;
  friend class MemTableBackwardIterator;

  typedef SkipList<const char*, KeyComparator> Table;

  KeyComparator comparator_;
  int refs_;
  Arena arena_;
  Table table_;

  // No copying allowed
  MemTable(const MemTable&);
  void operator=(const MemTable&);
}
```
最核心的无疑是跳表SkipList，内存分配器Arena和键值比较器KeyComparator都是为跳表打工的。SkipList本质是有序的链表。

我们首先考虑传统的有序链表有什么缺点。当链表中的元素只有10或者几十个的时候，插入一个新的元素并不费劲，只需要依次遍历找到这样一个位置：

* 链表中当前元素的key值小于要插入的key值
* 链表中当前元素的下一个元素的key值大于要插入的key值。

这种实现有一个致命的缺点，就是当链表中元素的个数有几千几万甚至几十万上百万的时候，查找效率太低。因此就有了改进：

![](/assets/LevelDB/skiplist_1.png)


如上图所示，{10,30,57,67}这几个元素将整个链表集合中的元素分成了三段，当尝试查找元素59的时候，路径是：

```
10--》30 --》 57 ---》58 ---》59
```
这种方式加快了查找的效率。当然了如果元素的个数进一步膨胀，可以采用多级指针，层层缩小排查范围，方法如下下图：
![](/assets/LevelDB/skiplist_2.png)

* Level 1: 14 too small, 79 too big; go down 14
* Level 2: 14 too small, 50 too small, 79 too big; go down 50
* Level 3: 50 too small, 66 too small, 79 too big; go down 66
* Level 4: 66 too small, 72 spot on

OK，跳表的原理，就不赘述，我们继续看leveldb的MemTable。

# MemTable的Add和Get

有了Memtable，关键的操作就是插入key-value pair和输入key，查询对应为Value，即Add和Get操作。

我们从描述中也看出来了，键值key非常重要，他决定了它在SkipList中的位置，即在MemTable的位置。


![](/assets/LevelDB/leveldb-keys.png)

（本图来自：  [Leveldb源码笔记之读操作](http://blog.1feng.me/2016/09/10/leveldb-read/)）

## Add 接口

我们首先看下Add操作：

```
void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  // Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;
  const size_t encoded_len =
      VarintLength(internal_key_size) + internal_key_size +
      VarintLength(val_size) + val_size;
  char* buf = arena_.Allocate(encoded_len);
  char* p = EncodeVarint32(buf, internal_key_size);
  memcpy(p, key.data(), key_size);
  p += key_size;
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  p = EncodeVarint32(p, val_size);
  memcpy(p, value.data(), val_size);
  assert((p + val_size) - buf == encoded_len);
  table_.Insert(buf);
}

```

这个函数将用户给的key和value组装成一个buf，调用table_.Insert插入到跳表合适的位置。

如何组装的：

![](/assets/LevelDB/leveldb_memtable_entry.png)

（本图来自：  [MemTable与SkipList-leveldb源码剖析(3)](http://www.pandademo.com/2016/03/memtable-and-skiplist-leveldb-source-dissect-3/)）

由上述代码可知SequenceNumber只使用了56bit，小端低位8bit为ValueType；亦可以看到Internal_Key和Val_Slice的大小均使用了varint变长压缩，即使用单字节最高位来区分是否还有后续字节，用前7位存储实际比特数据，对于小整数有比较好的压缩效果，这种对小整数的字节对齐压缩方案在leveldb实现中有较多体现。


## Get接口
我们来看内部重要的LookupKey的定义,这个LookupKey就是我们说的memtable key：

![](/assets/LevelDB/leveldb_key.png)



（本图来自：  [MemTable与SkipList-leveldb源码剖析(3)](http://www.pandademo.com/2016/03/memtable-and-skiplist-leveldb-source-dissect-3/)）

通过上图，不难理解下面的意思：

```
// A helper class useful for DBImpl::Get()
class LookupKey {
 public:
  // Initialize *this for looking up user_key at a snapshot with
  // the specified sequence number.
  LookupKey(const Slice& user_key, SequenceNumber sequence);

  ~LookupKey();

  // Return a key suitable for lookup in a MemTable.
  Slice memtable_key() const { return Slice(start_, end_ - start_); }

  // Return an internal key (suitable for passing to an internal iterator)
  Slice internal_key() const { return Slice(kstart_, end_ - kstart_); }

  // Return the user key
  Slice user_key() const { return Slice(kstart_, end_ - kstart_ - 8); }

 private:
  // We construct a char array of the form:
  //    klength  varint32               <-- start_
  //    userkey  char[klength]          <-- kstart_
  //    tag      uint64
  //                                    <-- end_
  // The array is a suitable MemTable key.
  // The suffix starting with "userkey" can be used as an InternalKey.
  const char* start_;
  const char* kstart_;
  const char* end_;
  char space_[200];      // Avoid allocation for short keys

  // No copying allowed
  LookupKey(const LookupKey&);
  void operator=(const LookupKey&);
};
````

接下来可以看下Get的实现：

```
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  iter.Seek(memkey.data()); /*跳表查找*/
  if (iter.Valid()) {
    // entry format is:
    //    klength  varint32
    //    userkey  char[klength]
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // Check that it belongs to same user key.  We do not check the
    // sequence number since the Seek() call above should have skipped
    // all entries with overly large sequence numbers.
    const char* entry = iter.key();
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry+5, &key_length);
    
    /*比较 user key，如果相同，则判断tag，来确定是否是deleted key */
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8),
            key.user_key()) == 0) {
      // Correct user key
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```

没有找到返回 false，找到了返回true，如果找到了，但是该key已经打上了删除tag，那么设置状态NotFound。注意MemTable删除的时候，并不是真正的删除，而是找到对应的key-value pair在 internal key的ValueType设置成kTypeDeletion。

特别强调下相同user_key在跳表里的排序，以下标作为SequenceNumber，举例如下：
a2, a1, b3, b2, c6, c1，实际是这样排列，能够确保查找时首先获取的是最新版本的数据。


