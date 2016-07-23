---
layout: post
title: 理解flashcache（1）
date: 2016-07-17 14:43:40
categories: linux
tag: flashcache linux
excerpt: 本文介绍flashcache
---

## 前言
flashcache 是facebook 发布的开源的混合存储方案。它是基于内核的DeviceMapper机制，允许讲SSD设备影射成机械存储设备的缓存，堆叠成一个虚拟设备供用户读写，从OS的角度看，就是一块普通的磁盘。

为什么要讲SSD和普通的机械硬盘绑在一起呢？传统的机械硬盘，对于顺序读写的能力还是不错的，但是，如果存在大量的随机小io，大量的时间花在寻道上，性能就会下降很多。SSD不存在这个寻道的问题，对于随机小io，性能一样很好。但是我们知道SSD的价格目前还是非常贵的，所有的存储都是用SSD，价格上很难承担。

如果存在一种技术，将SSD和HDD绑在一起，对于连续的大IO直接写入HDD，对于离散的小io写入SSD即为成功，则相当于同时享受了SSD对离散小io的高性能和HDD的大容量。这种技术就是facebook的Mohan Srinivasan 大神发布的flashcache。


## flashcache layout

flashcache 本质是一种cache，它的原理和CPU L1 Cache非常相似。因为L1 Cache容量有限，不可能容纳内存中的所有内容，同样道理SSD因为容量的原因，也不可能容纳后备的HDD的所有内容。那么哪些内容会在SSD中，OS系统如何从SSD中找到对应的块或扇区呢，OS又如何判断目前查找的扇区是否落在SSD之中呢？

flashcache将SSD的几乎所有内容（superblock和元数据除外），组织成如下结构：

![](/assets/LINUX/flashcache_grid.png)

首先，上图中的每一个方格都代表一个block，默认值是4K。 每一行都代表一个cache set，默认情况下，512个block组成一个cache size（即上图中的m＝512），因此，cache size的容量为512*4K ＝ 2M。 

假如说，我拿一个100G的SSD来做flashcache的，很容易算出一共可以构造出多少个这样的cache set，即

```
    100G/2M=51200
```
因此上图中的n为51200 。

那么问题来了，如果手头上有一个flashcache，如何确定flashcache的layout呢？

```
root@node-186:~# lsblk
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                      8:0    0   1.8T  0 disk 
├─sda1                   8:1    0  30.5M  0 part 
├─sda2                   8:2    0 488.3M  0 part 
├─sda3                   8:3    0  93.1G  0 part /
├─sda4                   8:4    0   128G  0 part [SWAP]
└─sda5                   8:5    0   1.6T  0 part 
sdb                      8:16   0   7.3T  0 disk 
└─sdb1                   8:17   0   7.3T  0 part 
  └─OSD-SATA-01 (dm-0) 252:0    0   7.3T  0 dm   /data/osd.0
sdc                      8:32   0   7.3T  0 disk 
└─sdc1                   8:33   0   7.3T  0 part 
  └─OSD-SATA-02 (dm-1) 252:1    0   7.3T  0 dm   /data/osd.1
sdd                      8:48   0 372.6G  0 disk 
├─sdd1                   8:49   0    32G  0 part 
└─sdd2                   8:50   0 340.6G  0 part 
  └─OSD-SATA-01 (dm-0) 252:0    0   7.3T  0 dm   /data/osd.0
sde                      8:64   0 372.6G  0 disk 
├─sde1                   8:65   0    32G  0 part 
└─sde2                   8:66   0 340.6G  0 part 
  └─OSD-SATA-02 (dm-1) 252:1    0   7.3T  0 dm   /data/osd.1
rbd0                   251:0    0    10T  0 disk 

```

从上图可以看出，7.3T的sdc1 和 340.6G的sd32， 组成了一个flashcache，通过dmsetup，创建成了一个device mapper，即dm-1。通过dmsetup table命令可以得到flashcache的layout

```
root@node-186:~# dmsetup table /dev/mapper/OSD-SATA-01 
0 15623780319 flashcache conf:
	ssd dev (/dev/disk/by-partlabel/OSD-SATA-01-ssd), disk dev (/dev/disk/by-partlabel/OSD-SATA-01-data) cache mode(WRITE_BACK)
	capacity(347422M), associativity(512), data block size(4K) metadata block size(4096b)
	disk assoc(256K)
	skip sequential thresh(32K)
	total blocks(88940032), cached blocks(38453498), cache percent(43)
	dirty blocks(25037914), dirty percent(28)
	nr_queued(0)
Size Hist: 512:760473 1024:2 3584:8 4096:2591203166 
```

data block size : 即上图中每一个方格的大小，即每个方格为4K
associativity   : 每一行方格的个数，本例为默认值512
capacity        : 即所有data 部分的容量。 本例为347422M

通过如下公式，可以算出

```
 n = capacity / (data block size) / associativity
```
因此不难算出，对于我们的flashcache实例共有173711个cache set。


上面的讨论简化了flashcache的模型，事实上，flashcache不仅仅有data部分，还有superblock和元数据。 和文件系统的superblock一样，flashcache的superblock解释了flashcache是如何组织的。

```
struct flash_superblock {
	sector_t size;		/* Cache size */
	u_int32_t block_size;	/* Cache block size */
	u_int32_t assoc;	/* Cache associativity */
	u_int32_t cache_sb_state;	/* Clean shutdown ? */
	char cache_devname[DEV_PATHLEN]; /* Contains dm_vdev name as of v2 modifications */
	sector_t cache_devsize;
	char disk_devname[DEV_PATHLEN]; /* underlying block device name (use UUID paths!) */
	sector_t disk_devsize;
	u_int32_t cache_version;
	u_int32_t md_block_size;
	u_int32_t disk_assoc;
	u_int32_t write_only_cache;
};

```

虽然flash_superblock消耗的空间很少，但是，flashcache仍然预留了1个data block的size，即默认情况下4K。4K的空间为将来的扩展预留了足够的空间。详情可见 flashcache_writeback_create函数中的如下语句：

```
#define DEFAULT_BLOCK_SIZE		8
#define DEFAULT_MD_BLOCK_SIZE	8
#define MD_BLOCK_BYTES(DMC)		((DMC)->md_block_size * 512)

	header = (struct flash_superblock *)vmalloc(MD_BLOCK_BYTES(dmc));
```

除此以外，对于每一个方格，必须要有元数据描述该方格：

很明显，SSD的size是远远小于HDD的size的，因此，对于每一个方格，必须要知道该方格对应的是HDD的那个block。因此，每一个data block，对应的元数据中必须要有dbn，即disk block number。

除此以外，无论方格可能有数据，可能无数据，SSD方格中的数据可能要比后备的HDD对应的block的数据新（writeback模式，先写入flashcache，lazy地刷入HDD），因此，必须要记录方格中block data的状态信息,因此，SSD上每一个block data都要有状态信息。状态有 （INVALID，VALID，DIRTY）。

```
struct flash_cacheblock {
	sector_t 	dbn;	/* Sector number of the cached block */
	u_int32_t	cache_state; /* INVALID | VALID | DIRTY */
} __attribute__ ((aligned(16)));
```

注意，对于flashcache上的每一个block data的元数据，占据的空间是16B，因此我们其实并不难计算总共需要多大的空间来存放所有block size的元数据信息。该计算是在flashcache_writeback_create 函数中进行的（都是以writeback模式为例进行的）

输入：
 
* dm->size  SSD设备扇区个数
* dm->block_size 每个数据块占用扇区的个数
* MD_SLOTS_PER_BLOCK  每个block可以存放元数据的个数

```
dm->size / dm->block_size  = 数据块的个数 
```
可以初步算出一共有多少个数据块（即有多少个方格）。因为每一个数据块都需要一个1个元数据，也就初步估算出了需要元数据的个数。

```
MD_SLOTS_PER_BLOCK  ＝ block_size / sizeof(flash_cacheblock)  = 每个block能存放多少个元数据
```   
因此，不难算出所有的元数据需要多少个数据块来存放：

```
   /*需要多少个block来存放元数据和superblock在内*/
	dmc->md_blocks = INDEX_TO_MD_BLOCK(dmc, dmc->size / dmc->block_size) + 1 + 1;
	
	/*总扇区数，剪掉用于存放元数据和superblock的扇区数，即用语存放数据的区域的扇区个数*/
	dmc->size -= dmc->md_blocks * MD_SECTORS_PER_BLOCK(dmc);
	
	
   /*将扇区个数转化成block 个数，即计算出一共有data block*/
	dmc->size /= dmc->block_size;
	
	/*注意 data block的个数，必须是assoc的倍数，否则最后一个cache set无法凑满足够的assoc*/
	dmc->size = (dmc->size / dmc->assoc) * dmc->assoc	
```

因此，flashcache的整体layout如下图所示：

![](/assets/LINUX/flashcache_layout.png)

值得一提的是，除了落在SSD上的元数据信息外，在内存中也要想好相当可观的内存。flashcache早期的版本每一个数据块要消耗24Byte的内存，因此，300G的SSD，如果4K一个block块，对于WriteBack模式，需要消耗1.8G左右的内存。

后来69fae2ff72c0606e8c00249621365cfbb877531f commit做了改进，每一个数据块在内存中的数据结构消耗18 Byte，节省了不少内存。还是以300G的SSD为例，如果4K一个block块，对于writeback模式，消耗1.35G左右的内存。

```
commit 69fae2ff72c0606e8c00249621365cfbb877531f
Author: Mohan Srinivasan <mohan@fb.com>
Date:   Tue Aug 6 08:28:33 2013 -0700

    Make each in-memory cacheblock state packed - 18 bytes each.

    Summary: Make 'struct cacheblock' a packed struct. So it fits in
    18 bytes (down from 24 bytes).

    Ported forward from facebook's internal repo.

    Test Plan:

    Reviewers:

    CC:

    Task ID: #

    Blame Rev:

diff --git a/src/flashcache.h b/src/flashcache.h
index 0f37e99..281eab8 100644
--- a/src/flashcache.h
+++ b/src/flashcache.h
@@ -425,7 +425,7 @@ struct cacheblock {
 #ifdef FLASHCACHE_DO_CHECKSUMS
        u_int64_t       checksum;
 #endif
-};
+} __attribute__((packed));

 struct flash_superblock {
        sector_t size;          /* Cache size */
(END)
```

那么内存中的数据结构到底存放了哪些内容呢？ 我们看下定义：

```
/* Cache block metadata structure */
struct cacheblock {
	u_int16_t	cache_state;
	int16_t 	nr_queued;	/* jobs in pending queue */
	u_int16_t	lru_prev, lru_next;
	u_int8_t        use_cnt;
	u_int8_t        lru_state;
	sector_t 	dbn;	/* Sector number of the cached block */
	u_int16_t	hash_prev, hash_next;
#ifdef FLASHCACHE_DO_CHECKSUMS
	u_int64_t 	checksum;
#endif
} __attribute__((packed));
```

## flashcache mode

对flashcache layout有了初步了解之后，接下来学习flashcache的工作模式。
flashcacheyou三种模式：

```
/* Cache Modes */
enum {
	FLASHCACHE_WRITE_BACK=1,
	FLASHCACHE_WRITE_THROUGH=2,
	FLASHCACHE_WRITE_AROUND=3,
};
```

* WRITEBACK : 对于写入，首先会写入到Cache中，同时将对于block的元数据dirty bit，但是并不会立即写入后备的device
* WRITETHROUGH : 对于写入，写入到Cache中，同时也会将数据写入backing device，知道写完backing device，才算写完
* WRITEAROUND : 写入的时候，绕过Cache，直接写入backing device，即SSD只当读缓存。

这三者的区别如下图

![](/assets/LINUX/flashcache_mode.png)


在最常用的writeback模式中，还有一个特殊的选项，即writecache：只有write才会使用SSD做的cache，而读直接读取慢速的硬盘。
这种场景适用于读缺失杂乱无章，读命中的概率非常低的情况。

当我们调用flashcache_create 创建flashcache的时候，加上－w option即可变成writecache

```
-w : write cache mode. Only writes are cached, not reads
```

在下面提到的flashcache_read中有如下语句：

```
spin_lock_irqsave(&dmc->ioctl_lock, flags);
	if (res == -1 || dmc->write_only_cache || flashcache_uncacheable(dmc, bio)) {
		spin_unlock_irqrestore(&dmc->ioctl_lock, flags);
		/* No room , non-cacheable or sequential i/o means not wanted in cache */
		if ((res > 0) && 
		    (dmc->cache[index].cache_state == INVALID))
			/* 
			 * If happened to pick up an INVALID block, put it back on the 
			 * per cache-set invalid list
			 */
			flashcache_invalid_insert(dmc, index);
		flashcache_setlocks_multidrop(dmc, bio);
		DPRINTK("Cache read: Block %llu(%lu):%s",
			bio->bi_sector, bio->bi_size, "CACHE MISS & NO ROOM");
		if (res == -1)
			flashcache_clean_set(dmc, hash_block(dmc, bio->bi_sector), 0);
		/* Start uncached IO */
		flashcache_start_uncached_io(dmc, bio);
		return;
	} else 
		spin_unlock_irqrestore(&dmc->ioctl_lock, flags);
```

注意dmc->write_only_cache, 当我们设置了WRITECACHE模式，我们就会调用flashcache_start_uncached_io，绕过SSD，直接从HDD中读取内容。


## SSD中数据块于HDD中数据块的对应关系

本节主要以WriteBack模式来讲解，即读和写都会用到flashcache的情景。

因为flashcache的缓存作用，当页高速缓存（Page Cache）没有命中，就会发起对磁盘的读写，都会调用到submit_io函数。既然SSD在页高速缓存之外，又提供了一个缓存，因此无论是读还是写，第一步都是要在SSD的数据块中寻找对应的块。

对于读，如果在SSD中找到对应的数据块，并且内容是VALID，那么就不需要去HDD寻道读出对应的内容了，直接从SSD中读出即可。

```
static void
flashcache_read(struct cache_c *dmc, struct bio *bio)
{
	int index;
	int res;
	struct cacheblock *cacheblk;
	int queued;
	unsigned long flags;
	
	DPRINTK("Got a %s for %llu (%u bytes)",
	        (bio_rw(bio) == READ ? "READ":"READA"), 
		bio->bi_sector, bio->bi_size);

	flashcache_setlocks_multiget(dmc, bio);
	res = flashcache_lookup(dmc, bio, &index);
	/* Cache Read Hit case */
	if (res > 0) {
		cacheblk = &dmc->cache[index];
		if ((cacheblk->cache_state & VALID) && 
		    (cacheblk->dbn == bio->bi_sector)) {
			flashcache_read_hit(dmc, bio, index);
			return;
		}
	}
```

对于WRITEBACK模式下的写入而言，写入到SSD的对应数据块即可，不需要写入到HDD。

```
static void
flashcache_write(struct cache_c *dmc, struct bio *bio)
{
	int index;
	int res;
	struct cacheblock *cacheblk;
	int queued;

	flashcache_setlocks_multiget(dmc, bio);
	res = flashcache_lookup(dmc, bio, &index);
	if (res != -1) {
		/* Cache Hit */
		cacheblk = &dmc->cache[index];		
		if ((cacheblk->cache_state & VALID) && 
		    (cacheblk->dbn == bio->bi_sector)) {
			/* Cache Hit */
			flashcache_write_hit(dmc, bio, index);
		} else {
			/* Cache Miss, found block to recycle */
			flashcache_write_miss(dmc, bio, index);
		}
		return;
	}
```

无论是哪一种情况，找到SSD中合适的数据块是第一要务。让我们看着flashcache layout小节中的第一张图，其实很容易想到。

10G的SSD和100G的HDD，通过dmsetup组成一个dm。 每个数据块大小是4K，每一个cache set中有512个数据块，即每行有512个方格，2M的空间。10G的SSD可以有5120行，即一共有5120个set。

如果用户要读取磁盘上偏移54321MB处的2K内容，只需要对54321M这个位置做hash。因为每一个cache set为2M，因此54321M对应的 set number是27160，因为总共有5120个set，因此，21760对应的set是：

```
21760 % 5120 ＝ 1560 
```

即我们应该去方格表的1560行去寻找正确的block。但是每行有512个data block，which one才是正确的哪个呢？

比对元数据中的dbn。因为54321M的扇区号为：

```
54321 * 1024 * 1024 / 512 = 111249408
```

因此，挨个比较512个data block对应的dbn，如果值等于111249408，则表示 SSD中存在对应的data block。


