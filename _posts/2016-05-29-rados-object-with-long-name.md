---
layout: post
title: ceph internal 之 long object name
date: 2016-05-29 14:57:40
categories: ceph-internal
tag: ceph-internal
excerpt: 本文学习下ceph底层如何处理long object name
---

## RADOS object with short name

上一篇博文，我们将介绍了对象相关的数据结构ghobject\_t，以及对象在底层文件系统存储的文件名，以及如何从文件名对应到 ghobject\_t对象。

映射关系如下图所示：


![](/assets/ceph_internals/ceph_object_name_in_bottom.png)

这里面又一个漏洞，即object name的长度。

### cephfs

对于cephfs而言，对象的名字是这样的：

```
root@185node:/var/share/ezfs/shareroot/bean_nas# dd if=/dev/zero of=bean bs=1M count=8 
8+0 records in
8+0 records out
8388608 bytes (8.4 MB) copied, 0.00768172 s, 1.1 GB/s
root@185node:/var/share/ezfs/shareroot/bean_nas# cephfs bean map
WARNING: This tool is deprecated.  Use the layout.* xattrs to query and modify layouts.
    FILE OFFSET                    OBJECT        OFFSET        LENGTH  OSD
              0      10000000022.00000000             0       4194304  0
        4194304      10000000022.00000001             0       4194304  1
```

对于cephfs中某个文件的对象是有两个部分组成的：inode 和 文件内object index, 10000000022是文件的inode，而小数点后的数字，表明的对象是文件中的第几个对象。

```
root@185node:/var/share/ezfs/shareroot/bean_nas# ll -li
total 8192
1099511627809 drwxrwxrwx 1 root root       1 May 29 11:39 ./
1099511627776 drwxrwxrwx 1 root root       2 May 29 11:01 ../
1099511627810 -rw-r--r-- 1 root root 8388608 May 29 11:39 bean
root@185node:/var/share/ezfs/shareroot/bean_nas# printf "%x\n" 1099511627810 
10000000022
root@185node:/var/share/ezfs/shareroot/bean_nas#
```

对于这种情况下，objectname是很规整的，长度是有限的，我们去底层查看对象在底层文件系统的文件：

```
注解：bean是pool的名字，下同

root@185node:/var/log# ceph osd map bean 10000000022.00000001 
osdmap e44 pool 'bean' (15) object '10000000022.00000001' -> pg 15.b5ce59c5 (15.1c5) -> up ([1], p1) acting ([1], p1)

root@185node:/data/osd.1/current/15.1c5_head# ll
total 4240
drwxr-xr-x    2 root root    4096 May 29 11:39 ./
drwxr-xr-x 3983 root root  135168 May 29 10:18 ../
-rw-r--r--    1 root root 4194304 May 29 11:39 10000000022.00000001__head_B5CE59C5__f
-rw-r--r--    1 root root       0 May 29 10:18 __head_000001C5__f
```

不出我们预料，对象在底层文件系统的文件名一上来就是对象的名字：10000000022.00000001

### RBD

rbd的情况也类似，我们不妨创建一个rbd：

```
root@185node:/# rbd create -p bean  --image-format=2 --size 100 rbd_test 
root@185node:/# rbd -p bean ls
rbd_test
root@185node:/# rbd -p bean info rbd_test
rbd image 'rbd_test':
	size 102400 kB in 25 objects
	order 22 (4096 kB objects)
	used objects: 0
	block_name_prefix: rbd_data.6n2q5cs0j0o53
	format: 2
	features: layering
root@185node:/# 
root@185node:/# rbd -p bean map rbd_test
/dev/rbd0
```

注意block\_name\_prefix的前缀，是rbd 内对象的前缀，我们从bean 这个pool中可以找到 rbd_data.6n2q5cs0j0o53.0000000000000000 这个objcet name，总体来讲，对象名也很规整，长度有限。

我们深入到底层文件系统，找到给对象的本地文件名，也是以object name 作为文件名的起始部分：rbd\udata.6n2q5cs0j0o53.0000000000000000

```
root@185node:/# ceph osd map bean rbd_data.6n2q5cs0j0o53.0000000000000000 
osdmap e44 pool 'bean' (15) object 'rbd_data.6n2q5cs0j0o53.0000000000000000' -> pg 15.715f761a (15.21a) -> up ([1], p1) acting ([1], p1)

root@185node:/data/osd.1/current/15.21a_head# ll
total 148
drwxr-xr-x    2 root root   4096 May 29 11:57 ./
drwxr-xr-x 3983 root root 135168 May 29 10:18 ../
-rw-r--r--    1 root root      0 May 29 10:18 __head_0000021A__f
-rw-r--r--    1 root root    513 May 29 11:57 rbd\udata.6n2q5cs0j0o53.0000000000000000__head_715F761A__f
```


但是RADOS object是cephfs 和 RBD的基石，RADOS是支持比较长的object name的,如下面的commit所说，RADOS是支持长达2048字节的对象名的。

```
commit 7e0aca18a04a3848af77f5dd2093dc2e009386ec
Author: Sage Weil <sage@redhat.com>
Date:   Wed Jul 16 14:17:27 2014 -0700

    osd: add config for osd_max_object_name_len = 2048 (was hard-coded at 4096)

    Previously we had a hard coded limit of 4096.  Objects > 3k crash the OSD
    when running on ext4, although they probably work on xfs.  But rgw only
    generates objects a bit over 1024 bytes (maybe 1200 tops?), so let set a
    more reasonable limit here.  2048 is a nice round number and should be
    safe.

    Add a test.

    Fixes: #8174
    Signed-off-by: Sage Weil <sage@redhat.com>
```

但是很不幸，本地文件系统并没有这么强悍，支持的文件名长度都有限：

|file system       |max length in bytes  |
|------------------|---------------------|
|EXT4              | 255          
|XFS               | 255
|ZFS               | 255
|btrfs             | 255


这就必然带来问题，因为文件尚且不能存放下object name，更谈不上其他hash之类的字段。Ceph是如何破解这个难题的呢？


## RADOS object with long name 

我们不妨通过rados 命令创建一个具有超长 object name的对象：

首先我们产生一个随机的足够长的名字：

```
root@185node:/# xxd -l $((2048/2)) -p /dev/urandom | tr -d '\n'
acda7ad8b034a90f9b980be5ed47e242209061c2515c2021b83f1f8c49d018d621a14043a68be64ecec025a1434f040c853b7419c0c571c6b20a5e4a25fe7bf2ff181b60508622bf89f7818add55022ba17d6c9f8bd2938d97788964d0da8405a29d5fa77b07b6e4484b5335b20c9e6eb2f89bf7e805256185b580f075008815cad96f79893599b3718d0dbc05796238c2cf22cd4ee0fadc3891951bbffb0602f3b14b3af7b1efe4c96a340de12fa3ba3f4baeb166768326cfe6d79ee210228266f292bdce01eb6d5c6eb4c64ac619d1aa3853d65a614e109638bf7e04389c8b9a06b41492e65a187abc834bfd6fc4988a55c9b2ed5b91a129acf572d6661fa1cac6ce4fb181b005883b38ca600e9004244fb6ff13cde1939c54583a3dc284cd82a6f77ee171a7b7423b040fc6a65070a6ff98a8b45fd3b1de8c325e6ec00c18d077ea6442b9b134fb9d515ea51427ef8dc43bb524c0a2e6958092186e1e3ae6058b114a5d7abfd7056e55596336f9191269731b71c240e1a449b4a83094fe5d5fe2143bcb19a0f913fb4a836f317a32cf74f91b1091b1c16644b39e0ec4dbfc6ec31f9a1da6c2e6c457e976e709b68c921f630fda53185ddc5c9454a63966b5982bc0905a84f134ee7e6187b9e2cd63b4a0fb174bf626c62400517cfb6121df951b3e0e895c1c2c1bd20dc73231f91e2d692c38d2f02f91158c824104c148d08c0ac2e363d7811d964a5fa6415a477e9ac2b304b51e66c52d7ec5d3214bd5f96044a0b96fe6e29a76b2e7818a41ff50db3ebc11eade7089e03237fcb913b17c5ff6de04278ffd7754c62951e493b4044ee916dce246898724a1306c6eae97a689dc9df3f69b42aae6071b00140a8a5d09e67b732c5f093eefc7ca719a7a6d3e5f53f9a36f8a4c9a9e28d19854559f911e1b42ef66ec1a5126ee2adb1d14dc10504a6c00063babea88c1c2b6e97581f771a099388a12d1050a6fe26cba538517195ed399053bd29467422064d8f6dd0661efa9e08f432c0f8ecf42bc589fa357547dc9313da0b172514d4aa102b8a6e01f0205e3c36db2102a7788924d6d314beff379c55d9dc433520355947f4da74038b4f263d74629cac1fa1248b4a89ced59a9005b667f3923b28bb80081429baf8a2748f3f84f31213b660046c22329cf1d3de4f2636be1257c0c8de15cc945f901db2243192802c92162fffef4eee3d4f5aeb9228291d6b89df6ef7c495f9041c65e386a8d77d3ba4b6bc19f0d049d07a49ca95deac3242d0ae8f643df4c65eae119f73516da42e17f8a06b9ea17e1bf248a50b57b870be2cf2269314534a17e77fc0266e05651169a0be11328371dd426d72cb51fa7e1ab5f75f55c0db9453824eeaaa1e156b5c0e0ba27e1f2f99b0733b2b6f004f8dd9f41321b6c24d36ccda327cadc85d97132878c40bb03252cb0
```

其次，我们通过rados 命令，创建一个该名字的object，对象的内容是“hello world”

```
root@185node:/# rados --pool=bean  put acda7ad8b034a90f9b980be5ed47e242209061c2515c2021b83f1f8c49d018d621a14043a68be64ecec025a1434f040c853b7419c0c571c6b20a5e4a25fe7bf2ff181b60508622bf89f7818add55022ba17d6c9f8bd2938d97788964d0da8405a29d5fa77b07b6e4484b5335b20c9e6eb2f89bf7e805256185b580f075008815cad96f79893599b3718d0dbc05796238c2cf22cd4ee0fadc3891951bbffb0602f3b14b3af7b1efe4c96a340de12fa3ba3f4baeb166768326cfe6d79ee210228266f292bdce01eb6d5c6eb4c64ac619d1aa3853d65a614e109638bf7e04389c8b9a06b41492e65a187abc834bfd6fc4988a55c9b2ed5b91a129acf572d6661fa1cac6ce4fb181b005883b38ca600e9004244fb6ff13cde1939c54583a3dc284cd82a6f77ee171a7b7423b040fc6a65070a6ff98a8b45fd3b1de8c325e6ec00c18d077ea6442b9b134fb9d515ea51427ef8dc43bb524c0a2e6958092186e1e3ae6058b114a5d7abfd7056e55596336f9191269731b71c240e1a449b4a83094fe5d5fe2143bcb19a0f913fb4a836f317a32cf74f91b1091b1c16644b39e0ec4dbfc6ec31f9a1da6c2e6c457e976e709b68c921f630fda53185ddc5c9454a63966b5982bc0905a84f134ee7e6187b9e2cd63b4a0fb174bf626c62400517cfb6121df951b3e0e895c1c2c1bd20dc73231f91e2d692c38d2f02f91158c824104c148d08c0ac2e363d7811d964a5fa6415a477e9ac2b304b51e66c52d7ec5d3214bd5f96044a0b96fe6e29a76b2e7818a41ff50db3ebc11eade7089e03237fcb913b17c5ff6de04278ffd7754c62951e493b4044ee916dce246898724a1306c6eae97a689dc9df3f69b42aae6071b00140a8a5d09e67b732c5f093eefc7ca719a7a6d3e5f53f9a36f8a4c9a9e28d19854559f911e1b42ef66ec1a5126ee2adb1d14dc10504a6c00063babea88c1c2b6e97581f771a099388a12d1050a6fe26cba538517195ed399053bd29467422064d8f6dd0661efa9e08f432c0f8ecf42bc589fa357547dc9313da0b172514d4aa102b8a6e01f0205e3c36db2102a7788924d6d314beff379c55d9dc433520355947f4da74038b4f263d74629cac1fa1248b4a89ced59a9005b667f3923b28bb80081429baf8a2748f3f84f31213b660046c22329cf1d3de4f2636be1257c0c8de15cc945f901db2243192802c92162fffef4eee3d4f5aeb9228291d6b89df6ef7c495f9041c65e386a8d77d3ba4b6bc19f0d049d07a49ca95deac3242d0ae8f643df4c65eae119f73516da42e17f8a06b9ea17e1bf248a50b57b870be2cf2269314534a17e77fc0266e05651169a0be11328371dd426d72cb51fa7e1ab5f75f55c0db9453824eeaaa1e156b5c0e0ba27e1f2f99b0733b2b6f004f8dd9f41321b6c24d36ccda327cadc85d97132878c40bb03252cb0  <(echo "hello,world")
```

通过ceph osd map 命令，找到该对象的所在的OSD ：

```
-> pg 15.5939415b (15.15b) -> up ([0], p0) acting ([0], p0)
```

我们去本地文件系统去寻找该对象对应的文件：

```
root@185node:/data/osd.0/current/15.15b_head# ll
total 152
drwxr-xr-x    2 root root   4096 May 29 10:38 ./
drwxr-xr-x 3961 root root 135168 May 29 10:18 ../
-rw-r--r--    1 root root     12 May 29 10:38 acda7ad8b034a90f9b980be5ed47e242209061c2515c2021b83f1f8c49d018d621a14043a68be64ecec025a1434f040c853b7419c0c571c6b20a5e4a25fe7bf2ff181b60508622bf89f7818add55022ba17d6c9f8bd2938d97788964d0da8405a29d5fa77b07b6e4484b5335b20c9e6eb2f_8293d87c929eba91a280_0_long
-rw-r--r--    1 root root      0 May 29 10:18 __head_0000015B__f
```

很明显，本地文件系统是不可能存放下长度达1K这么长的名字的，那ceph是怎么做的呢？对于长的object name，ceph是如何处理的呢？


从存储在本地文件系统的名字来看，文件名分成4个部分

* object name prefix ,长度为FILENAME_PREFIX_LEN 
* object name 的 SHA-1 hash，注意是完整object name的SHA-1 hash
* candidate index , 调用lfn_get_name函数时传递的参数值
* FILENAME_COOKIE 静态字符串，就是‘long’ 这个字符串。

这四个部分通过下划线_分隔开。

这部分逻辑时在build_filename函数实现的：

```

void LFNIndex::build_filename(const char *old_filename, int i, char *filename, int len)
{
  char hash[FILENAME_HASH_LEN + 1];

  assert(len >= FILENAME_SHORT_LEN + 4);

  strncpy(filename, old_filename, FILENAME_PREFIX_LEN);
  filename[FILENAME_PREFIX_LEN] = '\0';
  if ((int)strlen(filename) < FILENAME_PREFIX_LEN)
    return;
  if (old_filename[FILENAME_PREFIX_LEN] == '\0')
    return;

  hash_filename(old_filename, hash, sizeof(hash));
  int ofs = FILENAME_PREFIX_LEN;
  while (1) {
    int suffix_len = sprintf(filename + ofs, "_%s_%d_%s", hash, i, FILENAME_COOKIE.c_str());
    if (ofs + suffix_len <= FILENAME_SHORT_LEN || !ofs)
      break;
    ofs--;
  }
}
```

这部分逻辑比较简单，如果old\_filename 即原始的object name长度有限，比FILENAME\_PREFIX\_LEN 要短的话，那就说明时短的对象名，什么处理也不用做，直接将名字赋值给filename 即可。 但是如果old_filename 很长，就要计算名字的hash，组成长的文件名，即上面提到的4段式。


```

#define CEPH_CRYPTO_SHA1_DIGESTSIZE 20

class LFNIndex : public CollectionIndex {
  /// Hash digest output size.
  static const int FILENAME_LFN_DIGEST_SIZE = CEPH_CRYPTO_SHA1_DIGESTSIZE;
  /// Length of filename hash.
  static const int FILENAME_HASH_LEN = FILENAME_LFN_DIGEST_SIZE;
  /// Max filename size.
  static const int FILENAME_MAX_LEN = 4096;
  /// Length of hashed filename.
  static const int FILENAME_SHORT_LEN = 255;
  /// Length of hashed filename prefix.
  static const int FILENAME_PREFIX_LEN;
  /// Length of hashed filename cookie.
  static const int FILENAME_EXTRA = 4;
  /// Lfn cookie value.
  static const string FILENAME_COOKIE;
  /// Name of LFN attribute for storing full name.
  static const string LFN_ATTR;
  /// Prefix for subdir index attributes.
  static const string PHASH_ATTR_PREFIX;
  /// Prefix for index subdirectories.
  static const string SUBDIR_PREFIX;
```

```

const int LFNIndex::FILENAME_PREFIX_LEN =  FILENAME_SHORT_LEN - FILENAME_HASH_LEN -
                                FILENAME_COOKIE.size() -
                                FILENAME_EXTRA;

const string LFNIndex::FILENAME_COOKIE = "long";

```

有时候需要根据ghoject_t 来生成段的短的文件名：

```
string LFNIndex::lfn_get_short_name(const ghobject_t &oid, int i)
{
  string long_name = lfn_generate_object_name(oid);
  assert(lfn_must_hash(long_name));
  char buf[FILENAME_SHORT_LEN + 4];
  build_filename(long_name.c_str(), i, buf, sizeof(buf));
  return string(buf);
}
```

因为短的文件名是长的object name的摘要，必然会有数据的损失，因此，需要判断短的文件名和长的文件名是否匹配：

```
bool LFNIndex::short_name_matches(const char *short_name, const char *cand_long_name)
{
  const char *end = short_name;
  while (*end) ++end;
  const char *suffix = end;
  if (suffix > short_name)  --suffix;                   // last char
  while (suffix > short_name && *suffix != '_') --suffix; // back to first _
  if (suffix > short_name) --suffix;                   // one behind that
  while (suffix > short_name && *suffix != '_') --suffix; // back to second _

  int index = -1;
  char buf[FILENAME_SHORT_LEN + 4];
  assert((end - suffix) < (int)sizeof(buf));
  int r = sscanf(suffix, "_%d_%s", &index, buf);
  if (r < 2)
    return false;
  if (strcmp(buf, FILENAME_COOKIE.c_str()) != 0)
    return false;
  build_filename(cand_long_name, index, buf, sizeof(buf));
  return strcmp(short_name, buf) == 0;
}
```

注意，刚才我提到了，SHA1本质是摘要，如果文件名从2K截断成200+字节，纵然提供了SHA1摘要，也是有数据损失的，如何根据磁盘上的文件重新获取object的所有信息呢。靠文件名肯定是不行了，有数据丢失，而且不可逆，恢复不回来object的所有信息。

ceph采用的xattr。这几天一直想先写ceph的chain_xattr， 但总觉的简单，而且机缘不到。我们先讲述原理，至于xattr，并不复杂。

```
root@185node:/data/osd.0/current/15.15b_head# getfattr -d acda7ad8b034a90f9b980be5ed47e242209061c2515c2021b83f1f8c49d018d621a14043a68be64ecec025a1434f040c853b7419c0c571c6b20a5e4a25fe7bf2ff181b60508622bf89f7818add55022ba17d6c9f8bd2938d97788964d0da8405a29d5fa77b07b6e4484b5335b20c9e6eb2f_8293d87c929eba91a280_0_long 
# file: acda7ad8b034a90f9b980be5ed47e242209061c2515c2021b83f1f8c49d018d621a14043a68be64ecec025a1434f040c853b7419c0c571c6b20a5e4a25fe7bf2ff181b60508622bf89f7818add55022ba17d6c9f8bd2938d97788964d0da8405a29d5fa77b07b6e4484b5335b20c9e6eb2f_8293d87c929eba91a280_0_long
user.ceph.snapset=0sAgIZAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAA==
user.cephos.lfn3="acda7ad8b034a90f9b980be5ed47e242209061c2515c2021b83f1f8c49d018d621a14043a68be64ecec025a1434f040c853b7419c0c571c6b20a5e4a25fe7bf2ff181b60508622bf89f7818add55022ba17d6c9f8bd2938d97788964d0da8405a29d5fa77b07b6e4484b5335b20c9e6eb2f89bf7e805256185b580f075008815cad96f79893599b3718d0dbc05796238c2cf22cd4ee0fadc3891951bbffb0602f3b14b3af7b1efe4c96a340de12fa3ba3f4baeb166768326cfe6d79ee210228266f292bdce01eb6d5c6eb4c64ac619d1aa3853d65a614e109638bf7e04389c8b9a06b41492e65a187abc834bfd6fc4988a55c9b2ed5b91a129acf572d6661fa1cac6ce4fb181b005883b38ca600e9004244fb6ff13cde1939c54583a3dc284cd82a6f77ee171a7b7423b040fc6a65070a6ff98a8b45fd3b1de8c325e6ec00c18d077ea6442b9b134fb9d515ea51427ef8dc43bb524c0a2e6958092186e1e3ae6058b114a5d7abfd7056e55596336f9191269731b71c240e1a449b4a83094fe5d5fe2143bcb19a0f913fb4a836f317a32cf74f91b1091b1c16644b39e0ec4dbfc6ec31f9a1da6c2e6c457e976e709b68c921f630fda53185ddc5c9454a63966b5982bc0905a84f134ee7e6187b9e2cd63b4a0fb174bf626c62400517cfb6121df951b3e0e895c1c2c1bd20dc73231f91e2d692c38d2f02f91158c824104c148d08c0ac2e363d7811d964a5fa6415a477e9ac2b304b51e66c52d7ec5d3214bd5f96044a0b96fe6e29a76b2e7818a41ff50db3ebc11eade7089e03237fcb913b17c5ff6de04278ffd7754c62951e493b4044ee916dce246898724a1306c6eae97a689dc9df3f69b42aae6071b00140a8a5d09e67b732c5f093eefc7ca719a7a6d3e5f53f9a36f8a4c9a9e28d19854559f911e1b42ef66ec1a5126ee2adb1d14dc10504a6c00063babea88c1c2b6e97581f771a099388a12d1050a6fe26cba538517195ed399053bd29467422064d8f6dd0661efa9e08f432c0f8ecf42bc589fa357547dc9313da0b172514d4aa102b8a6e01f0205e3c36db2102a7788924d6d314beff379c55d9dc433520355947f4da74038b4f263d74629cac1fa1248b4a89ced59a9005b667f3923b28bb80081429baf8a2748f3f84f31213b660046c22329cf1d3de4f2636be1257c0c8de15cc945f901db2243192802c92162fffef4eee3d4f5aeb9228291d6b89df6ef7c495f9041c65e386a8d77d3ba4b6bc19f0d049d07a49ca95deac3242d0ae8f643df4c65eae119f73516da42e17f8a06b9ea17e1bf248a50b57b870be2cf2269314534a17e77fc0266e05651169a0be11328371dd426d72cb51fa7e1ab5f75f55c0db9453824eeaaa1e156b5c0e0ba27e1f2f99b0733b2b6f004f8dd9f41321b6c24d36ccda327cadc85d97132878c40bb03252cb0"
user.cephos.lfn3@1="__head_5939415B__f"
user.cephos.spill_out=0sMQA=

root@185node:/data/osd.0/current/15.15b_head# 

```
注意该短文件名对应的文件有扩展属性信息：

* user.cephos.lfn3
* user.cephos.lfn3@1

ceph将object 所有需要的信息都存放在 user.cephos.lfn$INDEX_VERSION 这个扩展属性里面。 但是为什么冒出来个user.cephos.lfn3@1，
这就是chain_xattr的含义了。2个Linux 扩展属性信息存放的是一笔扩展属性，仅仅是因为EXT4这个本地文件系统能存放的数据非常有限 2K，没有办法将value存放在单个key对应的 扩展属性里面，所以使用多个key来描述一个属性。这就是chain_xattr中chain的含义。


即如果你希望存放一个key value到Linux文件系统的某个文件的扩展属性中，受限于扩展属性能容纳的value长度有限，你不得不这么存放：

key key@1 key@2 key@3 



OK,都讲完了，还是有一些代码需要梳理，先到此处吧。我也累了。


