---
layout: post
title: leveldb之log文件
date: 2016-11-30 20:54
categories: leveldb
tag: leveldb
excerpt: 本文介绍leveldb中的log文件
---

# 前言

leveldb在内存存储为Memtable，但是很明显内存不是持久化设备，当异常掉电的时候，会data loss，leveldb是用纪录log的方式来防范异常掉电的。即所有的内容，写入内存之前，先写入log文件，这种情况下，即使发生异常，Memtable中的内容没有来得及Dump到磁盘的SSTable文件，也没有关系，完全可以根据log文件将内容恢复Memtable中的内容，避免数据丢失。


从上面的讨论可以看出，写入动作，可以分解为2步

* 追加写入log文件
* 写入内存

因为追加写入log文件属于连续写入，而写入内存的速度又比较快，因此，写入会比较高效。


# log文件的布局

log相关的代码在

* db/log_format.h
* db/log_reader.h
* db/log_reader.cc
* db/log_writer.h
* db/log_writer.cc

我们首先了解下log文件的布局，即数据在log文件中是怎么组织的。

leveldb存放的是key-value对，因为键值和value值的长度是可变的，因此，每一笔记录都必须有个length字段来表明当前记录的长度。
当然了，leveldb为了校验数据的一致性，同时会计算checksum，作为记录的一个字段.

还有另外一个字段，即type。注意，Log文件是分block存放的，每个block的大小为32KB，这就会存在一个问题，如某个key－value对过大，无法存放在单个block中，可能存放在多个不同的block中,因此引入了另一个字段RecordType


```
#ifndef STORAGE_LEVELDB_DB_LOG_FORMAT_H_
#define STORAGE_LEVELDB_DB_LOG_FORMAT_H_

namespace leveldb {
namespace log {

enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,

  kFullType = 1,

  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
static const int kMaxRecordType = kLastType;

static const int kBlockSize = 32768;  /*每个block为32KB*/

// Header is checksum (4 bytes), length (2 bytes), type (1 byte).
static const int kHeaderSize = 4 + 2 + 1;  /*header由三部分组成，checksum length和type*/

}  // namespace log
}  // namespace leveldb

#endif  // STORAGE_LEVELDB_DB_LOG_FORMAT_H_
```

![](/assets/LevelDB/leveldb-log1.png)

```
    checksum: uint32           // type及data[]对应的crc32值
    length:   uint16           // 数据长度
    type:     uint8            // FULL/FIRST/MIDDLE/LAST中的一种
    data:     uint8[length]    // 实际存储的数据
```

类型存在4种：

* kFullType   ： 记录完全在一个block中
* kFirstType  ： 当前block容纳不下所有的内容，记录的第一片在本block中
* kMiddleType ： 记录的内容的起始位置不在本block，结束未知也不在本block
* kLastType   ： 记录的内容起始位置不在本block，但 结束位置在本block

![](/assets/LevelDB/leveldb-log2.png)

有了上述layout的信息，普通水平的程序员也可以完成如何add一笔记录到Log文件的功能：


# 实现

```
namespace leveldb {
namespace log {

static void InitTypeCrc(uint32_t* type_crc) {
  for (int i = 0; i <= kMaxRecordType; i++) {
    char t = static_cast<char>(i);
    type_crc[i] = crc32c::Value(&t, 1);
  }
}

Writer::Writer(WritableFile* dest)
    : dest_(dest),
      block_offset_(0) {
  InitTypeCrc(type_crc_);
}

Writer::Writer(WritableFile* dest, uint64_t dest_length)
    : dest_(dest), block_offset_(dest_length % kBlockSize) {
  InitTypeCrc(type_crc_);
}

Writer::~Writer() {
}

Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  Status s;
  bool begin = true;
  do {
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0);
    if (leftover < kHeaderSize) {
       如果当前block剩下的空间已经不足7字节，即header3个字段的长度
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        assert(kHeaderSize == 7);
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;

    RecordType type;
    const bool end = (left == fragment_length);
    
    if (begin && end) {
      /*当前记录的开始和结束都在当前block，则为kFullType*/
      type = kFullType;
    } else if (begin) {
      /*当前记录的开始字段在当前block，则为kFirstType*/
      type = kFirstType;
    } else if (end) {
      /*当前记录的结束位置在当前block，则为kLastType*/
      type = kLastType;
    } else {
      /*当前记录的开始和结束都不在当前block，则为kMiddleType*/
      type = kMiddleType;
    }

    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);
  return s;
}

Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr, size_t n) {
  assert(n <= 0xffff);  // Must fit in two bytes
  assert(block_offset_ + kHeaderSize + n <= kBlockSize);

  // Format the header
  char buf[kHeaderSize];
  
  /*buf[4]和buf[5] 记录的是记录的size */
  buf[4] = static_cast<char>(n & 0xff);
  buf[5] = static_cast<char>(n >> 8);  
  buf[6] = static_cast<char>(t);  //第7个字节记录的是四种type之一

  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, n);
  crc = crc32c::Mask(crc);                 // Adjust for storage
  EncodeFixed32(buf, crc);

  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, n));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + n;
  return s;
}

}  // namespace log
}  // namespace leveldb

```

注意，buf[4]和buf[5]中记录的是当前记录的size，可以容乃

