---
layout: post
title: leveldb之Compaction (3)
date: 2017-01-12 10:29
categories: leveldb
tag: leveldb
excerpt: 本文介绍leveldb中的Compaction
---

# 前言

前面已经有了两篇关于Compaction的文章。第一篇文章介绍的是从MemTable到SSTable，第二篇文章介绍的是因何而Compaction，给了两种场景，一种是某一层级的文件数过多或者文件总大小超过预定门限，另一种是level n 和level n+1重叠严重，无效seek次数太多。

这篇文章来介绍，参战部队。

随着时间的流逝，LevelDB各个层级都有多个文件。剔除level 0不论，对于任何一个层级来说，层级的内的任意一个文件本身是有序的，而位于同一层级的内部的多个文件，他们也是有序的，而且key是不交叉的。

但是很不幸的是，level n 和level n＋1的文件，key的范围可能交叉，这种交叉，就可能带来 seek miss，即数据有可能位于level n的某个文件中（根据该文件的最小key和最大key和用户要查找的key来推算），但是实际情况是并不在level n的该文件中，不得不去level n＋1的文件查找。这种seek miss不解决，就会造成查询效率的下降。

如何解决？

出现这种无效seek的原因是level n和level n ＋ 1，在某些key的范围内，是有交叉的，解决的方法即： 整理level n和level n＋1 该key范围内的所有文件和key，将其整理后，归入level n＋1，而删除level n的该部分文件，从而消除该key范围内，level n和level n+1的重叠。

那为什么LevelDB非要搞出来level 0 ～level 6 这么多的层级呢？就level 0和level 1不好吗？

我们设想下，如果只有level 0和level 1，随着levelDB 数据的增长，level 1上的数据会异常的多，数据异常的紧致，文件数特别多，一旦level 0需要和level 1的某些文件合并，根据level 0的某些文件的确定的范围 ［smallest_key, largest_key］,需要参与本次compact的level 1的文件数势必非常多。我们想象一下，如果level 1中有几千几万个文件参与Compaction，其耗时必然非常大，这明显不是我们期待的，我们希望每次Compaction参战的文件非常少，瞬时爆发的读写和merge在一个非常小的范围之内，而不是单次Compaction就有几千几万的文件参与排序。

但是我们有多个level的话，情况就不同，我无法用简单的语言说清楚为什么多级level就可以解决这个难题，但是大家不妨细细想想，细细体会。我介绍完本节，大家想想可不可以设计出一种反 leveldb的输入，导致每次leveldb的compaction都劳民伤财，伤筋动骨，参战部队异常的多。


对于非manual 而言，确定参战部队的番号，是由PickCompaction函数来实现的。如果需要compaction的level是n，那么参加compaction的文件要么全部位于level n，要么既有level n的某些文件，也有level n＋1的某些文件。

![](/assets/LevelDB/compact_related_file.png)

(为什么上图中，level n＋1的线段比较短，而level n的线段反而比较长呢？ 因为线段的长度表示负责的key的范围，level n＋1的文件数可能更多，因为理论上每一层的文件总长度是下一层文件总长度的10倍，因此，每个文件负责的key的范围会更狭窄，因此看起来线段更短。而level n以更少的文件，负责一定的key的范围，因此相对于level n＋1的单个文件，它负责的key的范围更加宽广。通过上图基本也可以领会leveldb 的分出7层原因，希望从上倒下稀疏有致)

确定参与Compaction的参战文件，分成两部分，

* 第一部分确定level n的参战文件
* 第二部分确定level n＋1的参战文件，当然也可能没有 level n＋1的文件需要参战。

# 确定level(n)的参战文件

```
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != NULL);
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level+1 < config::kNumLevels);
    c = new Compaction(options_, level);

    // Pick the first file that comes after compact_pointer_[level]
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
      }
    }
    if (c->inputs_[0].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(options_, level);
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return NULL;
  }

  c->input_version_ = current_;
  c->input_version_->Ref();

  // Files in level 0 may overlap each other, so pick up all overlapping ones
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }

  SetupOtherInputs(c);

  return c;
}
```

上面这个函数，出了最后的SetupOtherInputs函数，负责计算Level n＋1的参战文件，其他的语句都是用来计算level n的参战文件。

它分成了两个部分，如果是size 触发的compaction是一个计算方法，如果是seek触发的compaction，是另外一个计算方法。

比较简单的是seek触发的compaction，哪个文件无效seek的次数到了门限值，那个文件就是level n的参与compaction的文件。
而size 触发的compaction稍微复杂一点，它需要考虑上一次compaction做到了哪个key，什么地方，然后大于该key的第一个文件即为level n的参战文件。

对于n >0的情况，初选情况下（即调用SetOtherInput之前）level n的参战文件只会有1个，如果n＝0，因为level 0的文件之间，key可能交叉重叠，因此，根据选定的level 0的该文件，得到该文件负责的最小key和最大key，找到所有和这个key 区间有交叠的level 0文件，都加入到参战文件。

基本原理就是这个原理。

简单概括，即当n>0的时候，初选 level n的参战文件只会有1个，而n ＝ 0的时候，初选的情看下，level n的文件也可能有多个。


# 确定level(n＋1)的参战文件

确定了level n的参战文件，根据 level n参战文件，就可以确定出level n参战文件的最小key和最大key，有了最小key和最大key，就可以进一步确定level n＋1需要的参战部队。


```
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;
  
  /获取level n所有参战文件的最小key和最大key/
  GetRange(c->inputs_[0], &smallest, &largest);

  /*根据最小key和最大 key，计算level n＋1的文件中于该范围有重叠的文件，放入c->inputs_[1]中*/
  current_->GetOverlappingInputs(level+1, &smallest, &largest, &c->inputs_[1]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
  
  /*综合level n 和level n+1的所有文件，计算出key的范围，即新的最小key和新的最大key*/
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);

  // See if we can grow the number of inputs in "level" without
  // changing the number of "level+1" files we pick up.
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level+1, &new_start, &new_limit,
                                     &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
            level,
            int(c->inputs_[0].size()),
            int(c->inputs_[1].size()),
            long(inputs0_size), long(inputs1_size),
            int(expanded0.size()),
            int(expanded1.size()),
            long(expanded0_size), long(inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
    }
  }

  // Compute the set of grandparent files that overlap this compaction
  // (parent == level+1; grandparent == level+2)
  if (level + 2 < config::kNumLevels) {
    current_->GetOverlappingInputs(level + 2, &all_start, &all_limit,
                                   &c->grandparents_);
  }

  if (false) {
    Log(options_->info_log, "Compacting %d '%s' .. '%s'",
        level,
        smallest.DebugString().c_str(),
        largest.DebugString().c_str());
  }

  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
}
```

选择level n＋1的步骤如下：

* 根据level n的参战文件，计算得到最小key和最大key
* 根据最小key和最大key圈定的key的范围，从level n＋1中选择和该范围有交叠的所有文件，计入c->inputs_[1]，作为level n＋1的参战文件
* 根据第一步和第二步的到的所有的level n的文件和level n＋1的文件，重新计算新的最小key和新的最大key

如上图蓝色框线圈定的level n 和level n＋1的文件。

事实上，上述步骤已经足矣完成任务了，但是紧接着，LevelDB有来了一大段 ：

```
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level+1, &new_start, &new_limit,
                                     &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
            level,
            int(c->inputs_[0].size()),
            int(c->inputs_[1].size()),
            long(inputs0_size), long(inputs1_size),
            int(expanded0.size()),
            int(expanded1.size()),
            long(expanded0_size), long(inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
    }
  }

```

这一部分逻辑是干啥的呢？

看下面的图：

![](/assets/LevelDB/calc_level_n_1.png)

注意，根据上图中的 level n中的参战文件A 计算出了 level n＋1 中的B C D需要参战，这是没有任何问题的，但是 由于B C D的加入，key的范围扩大了，又一个问题是 level n层的E需不需要参战？从上图看，参战是比较好的，因为A＋E确定的范围，完全笼罩在计算出来的B C D的范围之内，不会因为E的加入，而扩大level n＋1的文件。



如下图所示的情况，很明显E是不能加入战局的原因是 leven n＋1层的B C D无法笼罩 A＋E确定的范围，如果不管不顾，（A＋E）和（B＋C＋D） 一起Compaction造成的恶果就是，很可能和level n＋1 的某个已有文件(F)发生重叠，这就破坏了 level 1～level 6 同一层的文件内不许重叠的要求，因此，下图的情况，是不允许 level n的E参战的。

![](/assets/LevelDB/level_n_1_not.png)

说笼罩，其实也不确切了，因为level n新加入了文件，很大key 可能造成key的范围扩大，只要扩大后的key的范围，不会involve 新的level n＋1的文件就行。


如果满足这个条件是不是level n的文件 E是不是一定可以参战呢？ 也不一定，看参战文件是否已经超出了上限。

```
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
```

注意，单次compaction的文件，我们不希望太多，造成瞬时的读写压力。因此，到底扩不扩容，看扩容之后参战文件的总大小。

如果加入了level n的扩容文件，leven n和level n＋1的文件总长度超过了上限，那么就放弃扩容 level n中的文件。

上限是多少？


```
static int64_t ExpandedCompactionByteSizeLimit(const Options* options) {
  return 25 * TargetFileSize(options);
}
```

25倍的TargetFileSize，以典型的2M大小为例，如果level n和level n＋1的总大小超过了50MB，就不再主动扩容，将level n的某些符合条件的文件involve 进来了。


最后的最后是记录下Compaction_pointer_ ，对于size compaction, 下一次要靠该值来选择 level n的参战文件

```
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
```

# 结束语

上一篇介绍了什么时候需要Compaction，本文重点介绍如何选择参与Compaction的文件，理解了这些，下一篇如何进行Compaction我都不想介绍了，基本上一个归并排序就可以概括了。我反倒是想介绍下Compaction的触发时机。