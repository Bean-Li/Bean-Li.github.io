---
layout: post
title: leveldb之Compaction (2)--何时需要Compaction
date: 2017-01-12 10:29
categories: leveldb
tag: leveldb
excerpt: 本文介绍leveldb中的Compaction
---

# 前言

上篇Compaction简单的介绍了 Compaction的宏观流程，介绍了将MemTable中的内容 dump成SSTable文件的过程，讲述到了CompactMemTable将新生成的sstable文件 置于which level。

本文重点介绍Compaction的trigger时机，Major Compaction的算法。

# 两种Compaction

从Compaction牵扯的动静出发，分成两种Compaction：

一种是MemTable文件已经足够大了，需要dump成sstable，这种情况叫做Minor Compaction；
另外一种情况是Major Compaction,请看如下代码：

```

void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (bg_compaction_scheduled_) {
 
  } else if (shutting_down_.Acquire_Load()) {

  } else if (!bg_error_.ok()) {

  } else if (imm_ == NULL &&
             manual_compaction_ == NULL &&
             !versions_->NeedsCompaction()) {
    // No work to be done
  } else {
    bg_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}
```

这段代码反应了一些需要发起Compction的条件，一下三个条件满足一个即可发起Compaction：

* imm_ != NULL 表示需要将Memtable dump成SSTable，发起Minor Compaction
* manual\_compaction_  != NULL 表示手动发起Compaction
* versions_->NeedsCompaction函数返回True

注意这个NeedsCompaction()函数，这个函数涌来判断，当前的情形，需不需要发起一次Compaction，那么除了MemTable dump成SStable这一种情况外，什么情景需要发起Compaction呢？

我们来看NeedsCompaction函数：

```
  bool NeedsCompaction() const {
    Version* v = current_;
    return (v->compaction_score_ >= 1) || (v->file_to_compact_ != NULL);
  }
```

## 文件数目过多或者某层级文件总大小过大，引起compacction

如果某一层级文件的个数太多（指的是level0），
或者某一层级的文件总大小太大，超过门限值，则设置v->compaction_score为一个大于1的数。

我们看下在什么情况下会重新计算compaction_score_。在Finalize函数中，会遍历各个level的文件数目和该level所有文件的总大小，给各个level打个分，如果没有一个level的分数是大于等于1，表示任何一个层级都不需要Compaction，但是如果存在某个或者某几个层级的score大于等于1，选择分最高的那个level。

```
void VersionSet::Finalize(Version* v) {
  // Precomputed best level for next compaction
  int best_level = -1;
  double best_score = -1;

  for (int level = 0; level < config::kNumLevels-1; level++) {
    double score;
    if (level == 0) {
      // We treat level-0 specially by bounding the number of files
      // instead of number of bytes for two reasons:
      //
      // (1) With larger write-buffer sizes, it is nice not to do too
      // many level-0 compactions.
      //
      // (2) The files in level-0 are merged on every read and
      // therefore we wish to avoid too many files when the individual
      // file size is small (perhaps because of a small write-buffer
      // setting, or very high compression ratios, or lots of
      // overwrites/deletions).
      score = v->files_[level].size() /
          static_cast<double>(config::kL0_CompactionTrigger);
    } else {
      // Compute the ratio of current size to size limit.
      const uint64_t level_bytes = TotalFileSize(v->files_[level]);
      score =
          static_cast<double>(level_bytes) / MaxBytesForLevel(options_, level);
    }

    if (score > best_score) {
      best_level = level;
      best_score = score;
    }
  }

  v->compaction_level_ = best_level;
  v->compaction_score_ = best_score;
}
```

这里面有一个特殊情况，就是level0是根据文件数目来计算score，而其他层级(level 1~level 6)是根据该层级所有文件的总大小来计算score。

对于level 0

```
score =  （level 0 文件总数目） ／ config::kL0_CompactionTrigger

static const int kL0_CompactionTrigger = 4;
```

对于 level 1～ level 6

```
score = （该level 所有文件的总大小）／ （该level的大小的理论上限：MaxBytesForLevel）
```

先说level 0 为什么搞特殊。 

注释说的很明白，level 0的文件之间，key可能是交叉重叠的，因此不希望level 0的文件数特别多。我们考虑write buffer 比较小的时候，如果使用size来限制，那么level 0的文件数可能太多。

另一个方面，如果write buffer过大，使用固定大小的size 来限制level 0的话，可能算出来的level 0的文件数又太少，触发 level 0 compaction的情况发生的又太频繁。因此level 0 走了一个特殊。


再说level 1～level 6，每一级别都有一个希望的上限：

```
static double MaxBytesForLevel(const Options* options, int level) {

  //level 0 不会调用该函数，因为level 0靠文件个数来计算得分
  double result = 10. * 1048576.0;
  while (level > 1) {
    result *= 10;
    level--;
  }
  return result;
}
```

```
level 1               10M 
level 2              100M
level 3             1000M
level 4            10000M
level 5           100000M
level 6          1000000M
```

我以level 1 为力，期望上限为10M，如果当前version ，level 1 的所有文件加在一起，总大小为 17MB,那么

level 1的得分 score : 17 ／ 10 ＝ 1.7

注意score 得分是double类型，各个层级计算自己的得分，如果得分越高，说明该层级触发compaction的要求就越迫切， v->compaction\_level_ 就会设置成得分最高的那个层级。


# seek 次数太多，引发compaction

出了上面这种情况，引起compaction以外，还有一种情况也可能会引起Compaction，就是某个文件seek的次数太多。

有同学说了，seek是非常正常的操作为什么会触发Compaction呢：

我们考虑下Get操作，当用户尝试获取某个key对应的Value的时候，查找的顺序如下：

```
MemTable --> Immutable MemTable --> Level 0 files --> Level 1 files -->Level 2 files ......-->Level 6 files
```

如下面函数所示：

```
Status DBImpl::Get(const ReadOptions& options,
                   const Slice& key,
                   std::string* value) {
  Status s;
  MutexLock l(&mutex_);
  SequenceNumber snapshot;
  if (options.snapshot != NULL) {
    snapshot = reinterpret_cast<const SnapshotImpl*>(options.snapshot)->number_;
  } else {
    snapshot = versions_->LastSequence();
  }

  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
  mem->Ref();
  if (imm != NULL) imm->Ref();
  current->Ref();

  bool have_stat_update = false;
  Version::GetStats stats;

  // Unlock while reading from files and memtables
  {
    mutex_.Unlock();
    // First look in the memtable, then in the immutable memtable (if any).
    LookupKey lkey(key, snapshot);
    
    /*MemTable毫无疑问是第一优先级，在内存中查找*/
    if (mem->Get(lkey, value, &s)) {
      // Done
    } else if (imm != NULL && imm->Get(lkey, value, &s)) {
      /*Immutable MemTable是第二优先级，也在内存*/
    } else {
    
      /*都没有命中，那么寻找sstable文件*/
      s = current->Get(options, lkey, value, &stats);
      have_stat_update = true;
    }
    mutex_.Lock();
  }

```

除了level 0以外，任何一个level的文件内部是有序的，文件之间也是有序的。但是level（n）和level （n＋1）中的两个文件的key可能存在交叉。正是因为这种交叉，查找某个key值的时候， level（n） 的查找无功而返，而不得不resort to level(n＋1)。

我们考虑寻找某一个key，如果找了曾经查找了level (n) ,但是没找到，然后去level (n+1)查找，结果找到了，那么对level (n)的某个文件而言，该文件就意味着有一次 未命中。 

我们可以很容易想到，如果查找了多次，某个文件不得不查找，却总也找不到，总是去高一级的level，才能找到。这说明该层级的文件和上一级的文件，key的范围重叠的很严重，这是不合理的，会导致效率的下降。因此，需要对该level 发起一次Major compaction，减少 level 和level ＋ 1的重叠情况。

这就是所谓的 Seek Compaction。

当新建一个sstable文件的时候，都会初始化一个 allow_seek的变量，表示最多容忍seek miss多少次：

```
  void Apply(VersionEdit* edit) {
  
    ...
    // Add new files
    for (size_t i = 0; i < edit->new_files_.size(); i++) {
      const int level = edit->new_files_[i].first;
      FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
      f->refs = 1;

      // We arrange to automatically compact this file after
      // a certain number of seeks.  Let's assume:
      //   (1) One seek costs 10ms
      //   (2) Writing or reading 1MB costs 10ms (100MB/s)
      //   (3) A compaction of 1MB does 25MB of IO:
      //         1MB read from this level
      //         10-12MB read from next level (boundaries may be misaligned)
      //         10-12MB written to next level
      // This implies that 25 seeks cost the same as the compaction
      // of 1MB of data.  I.e., one seek costs approximately the
      // same as the compaction of 40KB of data.  We are a little
      // conservative and allow approximately one seek for every 16KB
      // of data before triggering a compaction.
      f->allowed_seeks = (f->file_size / 16384);
      if (f->allowed_seeks < 100) f->allowed_seeks = 100;

      levels_[level].deleted_files.erase(f->number);
      levels_[level].added_files->insert(f);
    }
 }
```
计算的方法即 :

```
f->allow_seeks = 文件长度 ／ 16KB 
```

代码中的注释解释了这么计算的原因，seek 1次，等于compaction 16KB的内容花费的时间，如果seek了allow_seeks 这么多次，effort就等同于做了一遍文件的Compaction，就把这个值作为门限值。发生这么多次seek，就拒绝容忍这种seek带来的损失，发起Compaction。


# 综合两种可能 

当非manual compaciton的时候，由PickCompaction函数来计算是否需要发起Compaction，如果需要，该函数来确定哪些层级的哪些文件需要参与其中。

需要不要发起，就是看前面讨论的逻辑

* size compaction : 因为文件数过多或者总文件大小过大，需要comapction
* seek compaction : 某个层级的某个文件无效seek过多，需要compaction


```
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != NULL);
  
  
  if (size_compaction) {
  
    /*size compaction是第一种情况，根据size来决定是否发起compaction，从which层级发起compaction*/
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
  
    /*seek_compaction 是第二种情况，无效seek 次数太多，所以依据文件以及其所属层级发起compaction*/
    level = current_->file_to_compact_level_;
    c = new Compaction(options_, level);
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return NULL;
  }

...
  
}
```

至于哪些文件需要参与本轮Compaction，以及如何Compaction，下篇zai jiang
