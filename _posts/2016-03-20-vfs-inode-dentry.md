---
layout: post
title: VFS中的file，dentry和inode
date: 2016-03-17 14:38:40
categories: kernel
tag: kernel
excerpt: VFS中有三个比较核心的数据结构
---

前言
----
毕业以来，我花了很多时间阅读内核的代码，深入Linux内核架构，深入理解Linux内核，Robert Love的Linux内核设计与实现，Linux的虚拟文件系统对应章节，也读了很多遍，每一次读，都有新的心得和体会。我觉得单纯流水账的读书，并不会有很好的效果，早起对着代码读，往往陷入细节，而不能体会为何如此设计。花了很多时间思考，也阅读了很多前辈的书籍博客，分享下我对VFS层几个关键数据结构的理解。


struct file
-----------
严格来讲，这个结构体不仅仅和文件系统相关，它也和进程相关联。对于虚拟文件系统而言，它的重要性一般，而相反，这个结构体对进程管理更重要，它记录了进程打开文件的上下文。该结构体，浮于表面，更贴近用户，更贴近进程。

但是，绝不应该低估这个数据结构的意义，有个成语叫做顺藤摸瓜，如果说文件系统是瓜，那么这个结构体就是藤。

我们都知道应用层调用open系统调用，返回的是一个int型的数字，称为文件描述符。

```
    int fd = open(FILEPATH,O_RDONLY);

```
用户不需要知道内核VFS，更不需要知道具体的文件系统实现，就可以自如地读写文件，获取文件信息，修改文件属性等。对于用户来讲，文件描述符这个数字才是用户层的藤。

应该说open系统调用之返回一个int型的数字，而并没有将更多的细节暴露给客户是非常高明的，因为隐藏了实现。只要接口的语义不变，内核层无论怎么修改，都和应用层编程无关。

如果说文件描述符这个数字，是用户层的藤，那么struct file就是内核层的藤。先来讲用户层的藤，怎么关联到内核层的藤。

首先来看进程描述符中文件系统相关的结构体

```
struct task_struct {
    struct files_struct *files;  
}

struct files_struct {
	atomic_t count;
	struct fdtable __rcu *fdt;
	～～struct fdtable fdtab;  ～～            

	spinlock_t file_lock ____cacheline_aligned_in_smp;
	int next_fd;
	～～struct embedded_fd_set close_on_exec_init;～～
	～～struct embedded_fd_set open_fds_init;～～
	～～struct file __rcu * fd_array[NR_OPEN_DEFAULT];～～
};

struct fdtable
{
	unsigned int max_fds;
	struct file __rcu **fd;      /* current fd array */
	fd_set *close_on_exec;       /*位图，用来记录那些fd需要close_on_exec*/
	fd_set *open_fds;            /*位图，用来记录那些fd已经在用，那些还处于free状态*/
	struct rcu_head rcu;
	struct fdtable *next;
};
struct embedded_fd_set {
	unsigned long fds_bits[1];
};


```

这个files_struct结构体会给初学者带来困扰，因为又有fdtable类型的指针fdt，又有fdtable类型的变量fdtab，files_struct结构体中有两个位图，偏偏fdtable类型的结构体中也有两个名字几乎雷同的位图。

为什么？

其实，如果将files_struct结构体中波浪线对应的成员删除，会更好理解。带波浪线的成员之所以存在，是出于性能优化的目的。Linux内核实现中有一个假设，即大多数的进程打开的文件个数不会太多，对于64位系统，内核假定绝大多数进程打开的文件数不会超过64个，因此fork创建进程的时候，就已经预先分配了可能需要的fdtable，以及两个长度为64的位图。

```
   atomic_set(&newf->count, 1);

	spin_lock_init(&newf->file_lock);
	newf->next_fd = 0;
	new_fdt = &newf->fdtab;
	new_fdt->max_fds = NR_OPEN_DEFAULT;
	new_fdt->close_on_exec = (fd_set *)&newf->close_on_exec_init;
	new_fdt->open_fds = (fd_set *)&newf->open_fds_init;
	new_fdt->fd = &newf->fd_array[0];
	new_fdt->next = NULL;
```

如下图所示：

![](/assets/VFS/files_struct.png)

注意，默认情况下使用的是预分配fdtable和两个位图，如果进程打开的文件超过了64，那么就不得不expand_fdtable分配一个更大的能够容纳更多struct file指针的fdtable,然后将老的fdtab中file指针和两个位图，拷贝到新的fdtable。

![](／assets/VFS/alloc_fdtable.png)


注意，fdtable里面的fd不过是一个指针，指向一个数组，数组中的每一个元素都是一个struct file类型的指针，到目前为止，终于走到了struct file这个最表层的结构体。

前面说过，struct file并非文件系统最核心的结构体，文件系统并非围绕该结构体全面展开，更确切地讲，它只是用户使用该文件的上下文信息。我们不妨看下该结构体的成员变量：

```
unsigned int f_flags ;

f_mode_t f_mode;

loff_t  f_pos ;

```

比如f_pos纪录了文件偏移量的位置，因为无论是read还是write系统调用，并没有一个参数指定偏移量，因此内核的struct file纪录了这个信息。
再比如文件打开时的标志位，是O_RDONLY还是O_RDWR，是否带了O_DIRECT标志位等信息，这些信息强烈地透漏出，struct file不过是进程打开文件的上下文信息。他们并不是文件的本质信息，用户可以读打开自然也可以写打开文件（只要有权限），这次操作的偏移量是4k，另一个进程操作该文件的偏移量可能是200K，甚至同一个进程可以打开同一个文件两次，有两个struct file结构体，分别纪录不同的上下文信息。


dentry
---------------
上一节提到了，struct file并不是文件系统的核心数据结构，那么dentry和inode，这两个结构体谁是文件系统的核心数据结构呢，它们存在的目的又分别是什么呢？

首先dentry是目录项缓存，是一个存放在内存里的缩略版的磁盘文件系统目录树结构,他是directory entry的缩写。我们知道文件系统内的文件可能非常庞大，目录树结构可能很深，该树状结构中，可能存在几千万，几亿的文件。

首先假设不存在dentry这个数据结构，我们看下我们可能会面临什么困境：

比如我要打开/usr/bin/vim 	文件，
1 首先需要去／所在的inode找到／的数据块，从／的数据块中读取到usr这个条目的inode，
2 跳转到user 对应的inode，根据/usr inode 指向的数据块，读取到/usr 目录的内容，从中读取到bin这个条目的inode
3 跳转到/usr/bin/对应的inode，根据/usr/bin/指向的数据块，从中读取到/usr/bin/目录的内容，从里面找到vim的inode


我们都知道，Linux提供了page cache页高速缓存，很多文件的内容已经缓存在内存里，如果没有dentry，文件名无法快速地关联到inode，即使文件的内容已经缓存在页高速缓存，但是每一次不得不重复地从磁盘上找出来文件名到VFS inode的关联。

因此理想情况下，我们需要将文件系统所有文件名到VFS inode的关联都纪录下来，但是这么做并不现实，首先并不是所有磁盘文件的inode都会纪录在内存中，其次磁盘文件数字可能非常庞大，我们无法简单地建立这种关联，耗尽所有的内存也做不到将文件树结构照搬进内存。

dentry就是为了解决这个难题的，我们先看下dentry结构体的定义：

```
struct dentry {
	/* RCU lookup touched fields */
	unsigned int d_flags;		/* protected by d_lock */
	seqcount_t d_seq;		/* per dentry seqlock */
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	/* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	unsigned long d_time;		/* used by d_revalidate */
	void *d_fsdata;			/* fs-specific data */

	struct list_head d_lru;		/* LRU list */
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
	/*
	 * d_alias and d_rcu can share memory
	 */
	union {
		struct hlist_node d_alias;	/* inode alias list */
	 	struct rcu_head d_rcu;
	} d_u;
};
```
struct inode ＊d\_inode 这个成员变量出现在dentry中，我显然毫不意外，因为dentry之所以需要存在，就是要建立文件名到inode的mapping关系，但是找了一圈没找到文件名，其实藏在 struct qstr d_name中。

```
#ifdef __LITTLE_ENDIAN
#define HASH_LEN_DECLARE u32 hash; u32 len;
#else
#define HASH_LEN_DECLARE u32 len; u32 hash;
#endif

struct qstr {
	union {
		struct {
			HASH_LEN_DECLARE;
		};
		u64 hash_len;
	};
	const unsigned char *name;
};
```

qstr是quick string的缩写，注意，qstr中的name只存放路径的最后一个分量，即basename，/usr/bin/vim 只会存放vim这个名字。当然了，如果路径名比较短，就存放在d_iname中,注意此结构体中有hash，还有一个len变量隐藏在struct HASH_LEN_DECLARE中。


因为每个dentry的父目录是唯一的，所以dentry 有成员变量d_parent，也就是说根据dentry很容易找到其父目录。但是dentry也会有子目录对应的dentry，所以提供了d_subdirs，所有子目录对应的dentry都会挂在该链表上。

dentry靠什么挂在其父dentry的以d_subdirs为头部的链表上？靠的是d_child成员比变量。



根据上面的结构体，已经可以查找某个目录是否在内存的dentry中，但是用链表查找太慢了，所以内核提供了hash表,d_hash会放置到hash表某个头节点所在的链表。

但是计算hash值并不是简单的根据目录的basename来hash，否则会有大量的冲突，比如home下每一个用户的家目录中都会有Pictures，Video，Project等目录，名字重复的概率太高，因此，计算hash的时候，将父dentry的地址也放入了hash计算当中，影响最后的结果，这大大降低了发生碰撞的机会。

也就是说一个dentry的hash值，取决于两个值：父dentry的地址和该dentry路径的basename。

```
107 static inline struct hlist_bl_head *d_hash(const struct dentry *parent,
108                                         unsigned int hash)
109 {
110         hash += (unsigned long) parent / L1_CACHE_BYTES;
111         return dentry_hashtable + hash_32(hash, d_hash_shift);
112 }

```

注意，如果一个目录book，但是每一次都要计算该basename的hash值，就会每次查找不得不计算一次book的hash，那效率就低了，因此，
qstr结构体中有一个字段hash，一次算好，再也不算了。此处是稍微牺牲空间效率来提升时间效率，用空间来换时间，可以看出Linux将性能压榨到了极致，能提升性能的地方，绝不放过。


当然了，一开始可能某个目录对应的dentry根本就不在内存中，所以会有d_lookup函数，以父dentry和qstr类型的name为依据，来查找内存中是否已经有了对应的dentry，当然，如果没有，就需要分配一个dentry，这是d_alloc函数负责分配dentry结构体，初始化相应的变量，建立与父dentry的关系。

这一部分代码在fs/dcache.c，就按下不提了。

inode
------
inode 应该算是整个VFS的核心，无数功能围绕着该结构体展开。dentry很明显属于文件系统层面，和进程并无瓜葛，但是它不能成为VFS的核心，原因是它和文件并不是一对一的关系，通过文件链接，同一个文件可以有多个dentry。

inode，因为和文件是一一对应的关系，因此，它实际上成为了VFS的核心，无数的读写功能都最终围绕它组织。

我们知道，现代的操作系统，都会设计文件缓存。因为文件位于慢速的块设备上，如果操作系统不设计缓存，每一次对文件的读写都要走到块设备，速度是不能容忍的。对于Linux而言，实现了页高速缓存。我们有这个感觉如果一次读某个文件慢的话，紧接着读这个文件第二次，速度会有明显的提升。原因是Linux已经将文件的部分内容或者全部内容缓存到了页高速缓存。

一个很神奇的事情是，A用户的a进程操作文件，将文件带入缓存，过一会B用户的b进程操作通一个文件时，同样可以享受文件内容在页高速缓存带来的福利。为什么，内核时如何做到的？

如果我要读文件F的第N个页面，内核是如何判断该页面是否在页高速缓存，如果在，如何找到该页的内容？这就牵扯到了页高速缓存的组织形式。页高速缓存在内核中以基数树的形式组织。

我们知道，很多文件是非常大的，比如EXT4就支持16T的文件，如果组织不善，查找太慢，会严重影响性能。基数树，你看下它的样子，就会明白为什么这货适合存放文件的页面。

![](／assets／VFS／radix_tree.png)

