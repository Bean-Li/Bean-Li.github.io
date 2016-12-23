---
layout: post
title: leveldb中的SSTable (2)
date: 2016-12-22 22:29
categories: linux
tag: leveldb
excerpt: 本文介绍leveldb中的重要数据结构SSTable
---

# 前言

上面介绍了SSTable的layout，以及footer和Data Block的结构。本文介绍index block。

footer中记录了index block和metaindex block的位置信息，这说明两者是非常重要的。 我们先介绍index block的作用。

典型的Data Block大小为4KB，而sstable文件的典型大小为2MB，也就说，一个sstable中，存在很多data block块，如果用户要查找某个key，该key到底位于which data block? 

如果没有index block，只有data block的起始位置和长度 （这个信息一定要有，否则无法确定查找，因为data block是变长的，并不是固定大小的），当用户查找某个key的时候，尽管是数据是有序的，可是还是不得不线性地找遍所有data block。任何一次查找要不得不读取2MB左右的内容，势必造成性能的恶化。

（其实也可以采用瞎猜算法，比如有256个data block的位置信息，先去找第128个 data block，判断第一个key和要找的key的关系，来二分查找。但是这种思路已经非常逼近index block的实现了。）

上帝说要有光，所有就有了光。Leveldb说要有索引，所有就有了index block。

index block的组织形式 data block的组织形式是一模一样的，不同点的地方在于存储的内容。data block存放的客户输入的key-value，而 index block存放的是 索引，我们细细展开。

如果上一个data block的最后一个key是 “helloleveldb”， 而当前data block的第一个key是“helloworld ”，那么index block就会在新block插入第一个key “helloworld”的时候，计算出两个data block之间的分割key（比如leveldb中算出来的hellom）。比分割key（hellom）小的key，在上一个data block，比分割key(hellom)大的key在下一个data block，因此

```
key = 分割key
value = 上一个data block的（offset，size）
```
通过这种手段，可以快速地定位出我们要找的key位于哪个data block(当然也可能压根不存在)。

# index block的实现
前言介绍了index block的原理部分，下面介绍index block的细节。

首先index block中的数据元素，是根据上一个data block的最后一个key和下一个data block的第一个key，计算出来一个分割key，结合上一个data block的位置信息（offset ，size），作为一组key-value存入index block。那么时机就很明确了，就是当data block插入第一个key-value的时候，就需要计算分割key，以及存入index block。

```
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->num_entries > 0) {
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }
  
  /********************************************************************************
  pending_index_entry 标志位用来判定是不是data block的第一个key，
  因此在上一个data block Flush的时候，会将该标志位置位，
  当下一个data block第一个key-value到来后，成功往index block插入分割key之后，就会清零 
  **********************************************************************************/
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    /*****************************************************************************
      计算出来的分割key即r->last_key作为key，而上一个data block的位置信息作为value。
      pending_handle里面存放的是上一个data block的位置信息，BlockHandle类型*
      
      注意index_block的组织形式和上一篇讲的Data block是一模一样的，
      区别在于存放的key-value pair不同。
     *****************************************************************************/
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }

  /*暂时忽略filter block部分*/
  if (r->filter_block != NULL) {
    r->filter_block->AddKey(key);
  }

  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  r->data_block.Add(key, value);

  /*估算当前data block的长度，如果超过了阈值，就要Flush*/
  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}

void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  assert(!r->pending_index_entry);
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    /************************************
     设置pending_index_entry为true
     当下一个key-value到来的时候，就需要计算分割key，
     融合pending_handle中存放的上一个data block位置信息，
     作为key-value pair，插入到index block
     *************************************/
    r->pending_index_entry = true;
    /*文件Flush，写入硬件。*/
    r->status = r->file->Flush();
  }
  if (r->filter_block != NULL) {
    r->filter_block->StartBlock(r->offset);
  }
}

void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;
  Slice raw = block->Finish();

  Slice block_contents;
  CompressionType type = r->options.compression;
  // TODO(postrelease): Support more compression options: zlib?
  switch (type) {
    case kNoCompression:
      block_contents = raw;
      break;

    /*可以将内容压缩，但是一般不开启，走上面那个分支*/
    case kSnappyCompression: {
      std::string* compressed = &r->compressed_output;
      if (port::Snappy_Compress(raw.data(), raw.size(), compressed) &&
          compressed->size() < raw.size() - (raw.size() / 8u)) {
        block_contents = *compressed;
      } else {
        // Snappy not supported, or compressed less than 12.5%, so just
        // store uncompressed form
        block_contents = raw;
        type = kNoCompression;
      }
      break;
    }
  }
  WriteRawBlock(block_contents, type, handle);
  r->compressed_output.clear();
  block->Reset();
}

void TableBuilder::WriteRawBlock(const Slice& block_contents,
                                 CompressionType type,
                                 BlockHandle* handle) {
  Rep* r = rep_;
  
  /*此处handle即为前面传入的 r->pending_handle,记录下上一个data block的offset和size*/
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());
  /*追加写入data block的内容*/
  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {
    char trailer[kBlockTrailerSize];
    trailer[0] = type;
    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // Extend crc to cover block type
    EncodeFixed32(trailer+1, crc32c::Mask(crc));
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}

```

往index block添加 key-value pair的时机，我们基本已经走了一遍，但是我们一直说分割key，那么分割key到底是怎么算出来的。

# 分割key的计算Comparator

Comparator负责计算两个data block之间的分割key，他需要的输入只有2个，

* 上一个data block的最后一个key
* 下一个data block的第一个key

根据两者，就能算出一个key，作为两个data block之间的分割线。

```
  /********************************************************************************
  pending_index_entry 标志位用来判定是不是data block的第一个key，
  因此在上一个data block Flush的时候，会将该标志位置位，
  当下一个data block第一个key-value到来后，成功往index block插入分割key之后，就会清零 
  **********************************************************************************/
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    /*****************************************************************************
      计算出来的分割key即r->last_key作为key，而上一个data block的位置信息作为value。
      pending_handle里面存放的是上一个data block的位置信息，BlockHandle类型*
      
      注意index_block的组织形式和上一篇讲的Data block是一模一样的，
      区别在于存放的key-value pair不同。
     *****************************************************************************/
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }
```
TableBuild中Add函数的这段逻辑就是用来计算分割key，并插入key-value pair到index block。关键函数在

```
r->options.comparator->FindShortestSeparator(&r->last_key, key);
```
FindShortestSeparator的声明如下：

```
FindShortestSeparator(std::string *start, const Slice& limit)
```
该函数的的作用是，如果start<limit,就把start修改为*start和limit的共同前缀后面多一个字符加1。很难理解用例子来说明下：

```
 *start:    helloleveldb        上一个data block的最后一个key
 limit:     helloworld          下一个data block的第一个key
 由于 *start < limit, 所以调用 FindShortSuccessor(start, limit)之后，start变成：
 hellom (保留前缀，第一个不相同的字符+1)
```
这里面有一个特例，即上一个data block的最后一个key是下一个data block第一个可以
的子串，怎么处理?

```
 *start:    hello               上一个data block的最后一个key
 limit:     helloworld          下一个data block的第一个key
 由于 *start < limit, 所以调用 FindShortSuccessor(start, limit)之后，start变成：
 hello (保留前缀，第一个不相同的字符+1)
```
答案是就用上一个 data block的最后一个key作为分割key。详情可以阅读下面代码：

```
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const {
    // Find length of common prefix
    size_t min_length = std::min(start->size(), limit.size());
    size_t diff_index = 0;
    while ((diff_index < min_length) &&
           ((*start)[diff_index] == limit[diff_index])) {
      diff_index++;
    }

    /*上一个data block的key是下一个data block的子串，则不处理*/
    if (diff_index >= min_length) {
      // Do not shorten if one string is a prefix of the other
    } else {
      uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
      if (diff_byte < static_cast<uint8_t>(0xff) &&
          diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) {
        (*start)[diff_index]++;
        start->resize(diff_index + 1);
        assert(Compare(*start, limit) < 0);
      }
    }
  }
```

# index block的落盘

index block的组织形式和 data block一样，区别在于key value的内容。这些index信息最终要写入sstable文件，落在磁盘上。何时做，怎么做？

在TableBuilder::Finish函数，实现了将index block写入sstablefile，并最终落盘。

```
Status TableBuilder::Finish() {
  Rep* r = rep_;
  Flush();
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  if (ok() && r->filter_block != NULL) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != NULL) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    /*写入Index Block的内容，
     *最重要的是，算出index block在file中的offset和size，存放到index_block_handle中，
     *这个信息要记录在footer中
     */
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

注意，写入的Index Block内容之后，记录了Index Block在sstable中的offset和size，存放在index_block_handle中，然后将index_block_handle记录在footer中，由此，就可以完成顺藤摸瓜的大业：

从footer，可以找到index block，而index存放的是{“分割key”：data block的位置信息}，所以根据index block可以找到任何一个data block的起始位置和结束位置。

