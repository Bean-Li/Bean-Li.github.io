---
layout: post
title:  Radosgw上传对象与multisite相关的逻辑
date: 2018-12-16 13:20:40
categories: ceph-internal
tag: ceph-internal
excerpt: 本文介绍上传对象中和multisite数据同步相关的部分逻辑
---
# 前言

本文对MultiSite内部数据结构和流程做一些梳理，加深对RadosGW内部流程的理解。因为MultiSite如何搭建，网上有较多的资料，因此不再赘述。

本文中创建的zonegroup为xxxx，两个zone：

* master
* secondary

zonegroup相关的信息如下：

```bash
{
    "id": "9908295f-d8f5-4ac3-acd7-c955a177bd09",
    "name": "xxxx",
    "api_name": "",
    "is_master": "true",
    "endpoints": [
        "http:\/\/s3.246.com\/"
    ],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "8aa27332-01da-486a-994c-1ce527fa2fd7",
    "zones": [
        {
            "id": "484742ba-f8b7-4681-8411-af96ac778150",
            "name": "secondary",
            "endpoints": [
                "http:\/\/s3.243.com\/"
            ],
            "log_meta": "false",
            "log_data": "true",
            "bucket_index_max_shards": 0,
            "read_only": "false"
        },
        {
            "id": "8aa27332-01da-486a-994c-1ce527fa2fd7",
            "name": "master",
            "endpoints": [
                "http:\/\/s3.246.com\/"
            ],
            "log_meta": "false",
            "log_data": "true",
            "bucket_index_max_shards": 0,
            "read_only": "false"
        }
    ],
    "placement_targets": [
        {
            "name": "default-placement",
            "tags": []
        }
    ],
    "default_placement": "default-placement",
    "realm_id": "0c4b59a1-e1e7-4367-9b65-af238a2f145b"
}
```



# 相关的pool

## 数据池(data pool)和索引池(index pool)

首当其中的pool是数据pool，即用户上传的对象数据，最终存放的地点：

```shell
root@NODE-246:/var/log/ceph# radosgw-admin zone get 
{
    "id": "8aa27332-01da-486a-994c-1ce527fa2fd7",
    "name": "master",
    "domain_root": "default.rgw.data.root",
    "control_pool": "default.rgw.control",
    "gc_pool": "default.rgw.gc",
    "log_pool": "default.rgw.log",
    "intent_log_pool": "default.rgw.intent-log",
    "usage_log_pool": "default.rgw.usage",
    "user_keys_pool": "default.rgw.users.keys",
    "user_email_pool": "default.rgw.users.email",
    "user_swift_pool": "default.rgw.users.swift",
    "user_uid_pool": "default.rgw.users.uid",
    "system_key": {
        "access_key": "B9494C9XE7L7N50E9K2V",
        "secret_key": "O8e3IYV0gxHOwy61Og5ep4f7vQWPPFPhqRXjJrYT"
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "default.rgw.buckets.index",
                "data_pool": "default.rgw.buckets.data",
                "data_extra_pool": "default.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "metadata_heap": "",
    "realm_id": "0c4b59a1-e1e7-4367-9b65-af238a2f145b"
}
```

从上面可以看出，master zone的default-placement中

| 作用              | pool name                  |
| --------------- | -------------------------- |
| data pool       | default.rgw.buckets.data   |
| index pool      | default.rgw.buckets.index  |
| data extra pool | default.rgw.buckets.non-ec |

测试版本是Jewel，尚不支持index 动态shard，我们选择index max shards=8，即每个bucket 有8个分片。

```
rgw_override_bucket_index_max_shards = 8 
```

通过如下指令可以看到我们当前集群的bucket信息：

```
root@NODE-246:/var/log/ceph# radosgw-admin bucket stats
[
    {
        "bucket": "segtest2",
        "pool": "default.rgw.buckets.data",
        "index_pool": "default.rgw.buckets.index",
        "id": "8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769",
        "marker": "8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769",
        "owner": "segs3account",
        ...
    },
    {
        "bucket": "segtest1",
        "pool": "default.rgw.buckets.data",
        "index_pool": "default.rgw.buckets.index",
        "id": "8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768",
        "marker": "8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768",
        "owner": "segs3account",
        ...
    }
}
```

从上图可以看到，一共有两个bucket，bucket id分别是：

| bucket name | bucket id                                |
| ----------- | ---------------------------------------- |
| segtest1    | 8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768 |
| segtest2    | 8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769 |

每个bucket有8个index shards，共有16个对象。

```
root@NODE-246:/var/log/ceph# rados -p default.rgw.buckets.index ls
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.7
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.0
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.2
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.5
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.6
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.3
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.4
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.3
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.0
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.1
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.6
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.5
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.2
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.1
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.769.4
.dir.8aa27332-01da-486a-994c-1ce527fa2fd7.4641.768.7
```



##  default.rgw.log pool

log pool记录的是各种日志信息，对于MultiSite这种使用场景，我们可以从default.rgw.log pool中找到这种命名的对象：

```bash
root@NODE-246:~# rados -p default.rgw.log ls  |grep data_log 
data_log.0
data_log.11
data_log.12
data_log.8
data_log.14
data_log.13
data_log.10
data_log.9
data_log.7
```

一般来讲这种命名风格的对象最多有rgw_data_log_num_shards，对于我们的场景：

```
OPTION(rgw_data_log_num_shards, OPT_INT, 128) 
```

rgw_bucket.h中可以看到如下代码：

```cpp
    num_shards = cct->_conf->rgw_data_log_num_shards;
    oids = new string[num_shards];
    string prefix = cct->_conf->rgw_data_log_obj_prefix;
    if (prefix.empty()) {
      prefix = "data_log";
    }   
    for (int i = 0; i < num_shards; i++) {
      char buf[16];
      snprintf(buf, sizeof(buf), "%s.%d", prefix.c_str(), i); 
      oids[i] = buf;
    }   
    renew_thread = new ChangesRenewThread(cct, this);
    renew_thread->create("rgw_dt_lg_renew")
```

一般来讲，该对象内容为空，相关有用的信息，都记录在omap中：

```bash
root@NODE-246:~# rados -p default.rgw.log stat data_log.61
default.rgw.log/data_log.61 mtime 2018-12-10 14:39:38.000000, size 0
root@NODE-246:~# rados -p default.rgw.log listomapkeys data_log.61
1_1544421980.298394_2914.1
1_1544422002.458109_2939.1
...
1_1544423969.748641_4486.1
1_1544423978.090683_4495.1
1_1544424000.286801_4507.1
```

# 写入对象

## 概述

宏观上讲，上传一个对象到bucket中，需要写多个地方，如果同时打开了bi log和data log的话。

* default.rgw.buckets.data : 将真实数据写入此pool，一般来讲新增一个<bucket_id>_<key>的对象
* default.rgw.buckets.index: 当数据写入完成之后，在该对象对应的bucket index shard的omap中增加该对象的信息
* bucket index 对象的omap中增加bi log
* default.rgw.log pool中的data_log对象的omap中增加data log

## bi log

当上传对象完毕之后，我们查看bucket index shard，可以看到如下内容：

````bash
root@node247:/var/log/ceph# rados -p default.rgw.buckets.index listomapkeys .dir.19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.1.7 
oem.tar.bz2
0_00000000001.1.2
0_00000000002.2.3
````

其中oem.tar.bz2文件是我们上传的对象，略过不提，除此意外还有两个0_00000000001.1.2和0_00000000002.2.3对象。

```
key (18 bytes):
00000000  80 30 5f 30 30 30 30 30  30 30 30 30 30 31 2e 31  |.0_00000000001.1|
00000010  2e 32                                             |.2|
00000012

value (133 bytes) :
00000000  03 01 7f 00 00 00 0f 00  00 00 30 30 30 30 30 30  |..........000000|
00000010  30 30 30 30 31 2e 31 2e  32 0b 00 00 00 6f 65 6d  |00001.1.2....oem|
00000020  2e 74 61 72 2e 62 7a 32  00 00 00 00 00 00 00 00  |.tar.bz2........|
00000030  01 01 0a 00 00 00 88 ff  ff ff ff ff ff ff ff 00  |................|
00000040  30 00 00 00 31 39 63 62  66 32 35 30 2d 62 62 33  |0...19cbf250-bb3|
00000050  65 2d 34 62 38 63 2d 62  35 62 66 2d 31 61 34 30  |e-4b8c-b5bf-1a40|
00000060  64 61 36 36 31 30 66 65  2e 31 35 30 38 33 2e 36  |da6610fe.15083.6|
00000070  34 32 31 30 00 00 01 00  00 00 00 00 00 00 00 00  |4210............|
00000080  00 00 00 00 00                                    |.....|
00000085

key (18 bytes):
00000000  80 30 5f 30 30 30 30 30  30 30 30 30 30 32 2e 32  |.0_00000000002.2|
00000010  2e 33                                             |.3|
00000012

value (125 bytes) :
00000000  03 01 77 00 00 00 0f 00  00 00 30 30 30 30 30 30  |..w.......000000|
00000010  30 30 30 30 32 2e 32 2e  33 0b 00 00 00 6f 65 6d  |00002.2.3....oem|
00000020  2e 74 61 72 2e 62 7a 32  e7 b2 14 5c 20 8e a4 04  |.tar.bz2...\ ...|
00000030  01 01 02 00 00 00 03 01  30 00 00 00 31 39 63 62  |........0...19cb|
00000040  66 32 35 30 2d 62 62 33  65 2d 34 62 38 63 2d 62  |f250-bb3e-4b8c-b|
00000050  35 62 66 2d 31 61 34 30  64 61 36 36 31 30 66 65  |5bf-1a40da6610fe|
00000060  2e 31 35 30 38 33 2e 36  34 32 31 30 00 01 02 00  |.15083.64210....|
00000070  00 00 00 00 00 00 00 00  00 00 00 00 00           |.............|
0000007d
```

为什么PUT对象之后，在.dir.19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.1.7 的omap中还有两个key-value对，它们是干什么用的？

我们打开所有OSD的debug-objclass，查看下

```
ceph tell osd.\* injectargs --debug-objclass 20
```

我们在日志中可以看到如下内容：

```bash 
ceph-client.radosgw.0的日志：
--------------------------------------
2018-12-15 15:53:11.079498 7f45723c7700 10 moving default.rgw.data.root+.bucket.meta.bucket_0:19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.1 to cache LRU end
2018-12-15 15:53:11.079530 7f45723c7700 20  bucket index object: .dir.19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.1.7
2018-12-15 15:53:11.083307 7f45723c7700 20 RGWDataChangesLog::add_entry() bucket.name=bucket_0 shard_id=7 now=2018-12-15 15:53:11.0.083306s cur_expiration=1970-01-01 08:00:00.000000s
2018-12-15 15:53:11.083351 7f45723c7700 20 RGWDataChangesLog::add_entry() sending update with now=2018-12-15 15:53:11.0.083306s cur_expiration=2018-12-15 15:53:41.0.083306s
2018-12-15 15:53:11.085002 7f45723c7700  2 req 64210:0.012000:s3:PUT /bucket_0/oem.tar.bz2:put_obj:completing
2018-12-15 15:53:11.085140 7f45723c7700  2 req 64210:0.012139:s3:PUT /bucket_0/oem.tar.bz2:put_obj:op status=0
2018-12-15 15:53:11.085148 7f45723c7700  2 req 64210:0.012147:s3:PUT /bucket_0/oem.tar.bz2:put_obj:http status=200
2018-12-15 15:53:11.085159 7f45723c7700  1 ====== req done req=0x7f45723c1750 op status=0 http_status=200 ======

ceph-osd.0.log 
----------------
2018-12-15 15:53:11.080017 7faaa6fb3700  1 <cls> cls/rgw/cls_rgw.cc:689: rgw_bucket_prepare_op(): request: op=0 name=oem.tar.bz2 instance= tag=19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.64210

2018-12-15 15:53:11.083526 7faaa6fb3700  1 <cls> cls/rgw/cls_rgw.cc:830: rgw_bucket_complete_op(): request: op=0 name=oem.tar.bz2 instance= ver=3:1 tag=19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.64210

2018-12-15 15:53:11.083592 7faaa6fb3700  1 <cls> cls/rgw/cls_rgw.cc:753: read_index_entry(): existing entry: ver=-1:0 name=oem.tar.bz2 instance= locator=

2018-12-15 15:53:11.083639 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:949: rgw_bucket_complete_op(): remove_objs.size()=0

2018-12-15 15:53:12.142564 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:470: start_key=oem.tar.bz2 len=11
2018-12-15 15:53:12.142584 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:487: got entry oem.tar.bz2[] m.size()=0

2018-12-15 15:53:12.170787 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:470: start_key=oem.tar.bz2 len=11
2018-12-15 15:53:12.170799 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:487: got entry oem.tar.bz2[] m.size()=0

2018-12-15 15:53:12.194152 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:470: start_key=oem.tar.bz2 len=11
2018-12-15 15:53:12.194167 7faaa6fb3700 20 <cls> cls/rgw/cls_rgw.cc:487: got entry oem.tar.bz2[] m.size()=1

2018-12-15 15:53:12.256510 7faaa6fb3700 10 <cls> cls/rgw/cls_rgw.cc:2591: bi_log_iterate_range
2018-12-15 15:53:12.256523 7faaa6fb3700  0 <cls> cls/rgw/cls_rgw.cc:2621: bi_log_iterate_entries start_key=<80>0_00000000002.2.3 end_key=<80>1000_
```

从上面的日志可以看出：

* rgw_bucket_prepare_op
* rgw_bucket_complete_op
* RGWDataChangesLog::add_entry() 

在void RGWPutObj::execute()函数的最后，会调用proecssor->complete 函数：

```
  op_ret = processor->complete(etag, &mtime, real_time(), attrs,
                               (delete_at ? *delete_at : real_time()), if_match, if_nomatch,
                               (user_data.empty() ? nullptr : &user_data));  
```

complete函数会调用do_complete函数，因此直接看do_complete函数：

```cpp
int RGWPutObjProcessor_Atomic::do_complete(string& etag, real_time *mtime, real_time set_mtime,
                                           map<string, bufferlist>& attrs, real_time delete_at,
                                           const char *if_match,
                                           const char *if_nomatch, const string *user_data) {
  //等待该rgw对象的所有异步写完成
  int r = complete_writing_data();                                              
  if (r < 0)
    return r;
  //标识该对象为Atomic类型的对象
  obj_ctx.set_atomic(head_obj);
  // 将该rgw对象的attrs写入head对象的xattr中
  RGWRados::Object op_target(store, bucket_info, obj_ctx, head_obj);
  /* some object types shouldn't be versioned, e.g., multipart parts */
  op_target.set_versioning_disabled(!versioned_object);

  RGWRados::Object::Write obj_op(&op_target);

  obj_op.meta.data = &first_chunk;
  obj_op.meta.manifest = &manifest;
  obj_op.meta.ptag = &unique_tag; /* use req_id as operation tag */
  obj_op.meta.if_match = if_match;
  obj_op.meta.if_nomatch = if_nomatch;
  obj_op.meta.mtime = mtime;
  obj_op.meta.set_mtime = set_mtime;
  obj_op.meta.owner = bucket_info.owner;
  obj_op.meta.flags = PUT_OBJ_CREATE;
  obj_op.meta.olh_epoch = olh_epoch;
  obj_op.meta.delete_at = delete_at;
  obj_op.meta.user_data = user_data;

  /* write_meta是一个综合操作，是我们下面分析的重点 */
  r = obj_op.write_meta(obj_len, attrs);
  if (r < 0) {
    return r;
  }
  canceled = obj_op.meta.canceled;
  return 0;                                                     
}
```

要探究bucket index shard中 omap中的0_00000000001.1.2和0_00000000002.2.3到底是什么，需要进入write_meta函数：

```cpp
int RGWRados::Object::Write::write_meta(uint64_t size,
                  map<string, bufferlist>& attrs)
{
  int r = 0;
  RGWRados *store = target->get_store();
  if ((r = this->_write_meta(store, size, attrs, true)) == -ENOTSUP) {
    ldout(store->ctx(), 0) << "WARNING: " << __func__
      << "(): got ENOSUP, retry w/o store pg ver" << dendl;
    r = this->_write_meta(store, size, attrs, false);      
  }
  return r;
}


int RGWRados::Object::Write::_write_meta(RGWRados *store, uint64_t size,
                  map<string, bufferlist>& attrs, bool store_pg_ver)
{
  ...
  r = index_op.prepare(CLS_RGW_OP_ADD);
  if (r < 0)
    return r;

  r = ref.ioctx.operate(ref.oid, &op); 
  if (r < 0) { /* we can expect to get -ECANCELED if object was replaced under,
                or -ENOENT if was removed, or -EEXIST if it did not exist
                before and now it does */
    goto done_cancel;
  }

  epoch = ref.ioctx.get_last_version();
  poolid = ref.ioctx.get_id();

  r = target->complete_atomic_modification();
  if (r < 0) {
    ldout(store->ctx(), 0) << "ERROR: complete_atomic_modification returned r=" << r << dendl;
  }
  r = index_op.complete(poolid, epoch, size, 
                        meta.set_mtime, etag, content_type, &acl_bl,
                        meta.category, meta.remove_objs, meta.user_data);

  ...    
}
```

### RGWRados::Bucket::UpdateIndex::prepare

在index_op.prepare 操作中，bucket index shard 中0_00000000001.1.2 该key-value对写入。

```cpp
int RGWRados::Bucket::UpdateIndex::prepare(RGWModifyOp op)
{
  if (blind) {
    return 0;
  }
  RGWRados *store = target->get_store();
  BucketShard *bs;
  int ret = get_bucket_shard(&bs);
  if (ret < 0) {
    ldout(store->ctx(), 5) << "failed to get BucketShard object: ret=" << ret << dendl;
    return ret;
  }
  if (obj_state && obj_state->write_tag.length()) {
    optag = string(obj_state->write_tag.c_str(), obj_state->write_tag.length());
  } else {
    if (optag.empty()) {
      append_rand_alpha(store->ctx(), optag, optag, 32);
    }
  }
  ret = store->cls_obj_prepare_op(*bs, op, optag, obj, bilog_flags);
  return ret;
}
int RGWRados::cls_obj_prepare_op(BucketShard& bs, RGWModifyOp op, string& tag, 
                                 rgw_obj& obj, uint16_t bilog_flags)
{
  ObjectWriteOperation o;
  cls_rgw_obj_key key(obj.get_index_key_name(), obj.get_instance());
  cls_rgw_bucket_prepare_op(o, op, tag, key, obj.get_loc(), get_zone().log_data, bilog_flags);
  int flags = librados::OPERATION_FULL_TRY;
  int r = bs.index_ctx.operate(bs.bucket_obj, &o, flags);
  return r;
}
void cls_rgw_bucket_prepare_op(ObjectWriteOperation& o, RGWModifyOp op, string& tag,
                               const cls_rgw_obj_key& key, const string& locator, bool log_op,
                               uint16_t bilog_flags)
{
  struct rgw_cls_obj_prepare_op call;
  call.op = op; 
  call.tag = tag;
  call.key = key;
  call.locator = locator;
  call.log_op = log_op;
  call.bilog_flags = bilog_flags;
  bufferlist in; 
  ::encode(call, in);
  o.exec("rgw", "bucket_prepare_op", in);
} 

cls/rgw/cls_rgw.cc
----------------------
void __cls_init()
{
    ...
   cls_register_cxx_method(h_class, "bucket_prepare_op", CLS_METHOD_RD | CLS_METHOD_WR, rgw_bucket_prepare_op, &h_rgw_bucket_prepare_op); 
    ...
}

int rgw_bucket_prepare_op(cls_method_context_t hctx, bufferlist *in, bufferlist *out)
{
  ...
  CLS_LOG(1, "rgw_bucket_prepare_op(): request: op=%d name=%s instance=%s tag=%s\n",
          op.op, op.key.name.c_str(), op.key.instance.c_str(), op.tag.c_str());

...
      // fill in proper state
  struct rgw_bucket_pending_info info;
  info.timestamp = real_clock::now();
  info.state = CLS_RGW_STATE_PENDING_MODIFY;
  info.op = op.op;
  entry.pending_map.insert(pair<string, rgw_bucket_pending_info>(op.tag, info));

  struct rgw_bucket_dir_header header;
  rc = read_bucket_header(hctx, &header);
  if (rc < 0) {
    CLS_LOG(1, "ERROR: rgw_bucket_complete_op(): failed to read header\n");
    return rc;
  }

  if (op.log_op) {
    //產生出0_00000000001.1.2
    rc = log_index_operation(hctx, op.key, op.op, op.tag, entry.meta.mtime,
                             entry.ver, info.state, header.ver, header.max_marker, op.bilog_flags, NULL, NULL);
    if (rc < 0)
      return rc;
  }

  // write out new key to disk
  bufferlist info_bl;
  ::encode(entry, info_bl);
  rc = cls_cxx_map_set_val(hctx, idx, &info_bl);
  if (rc < 0)
    return rc; 
  return write_bucket_header(hctx, &header);
}
```

注意上面的log_index_operation函數，我們的第一個0_00000000001.1.2對象即由該函數產生。

```cpp
static void bi_log_prefix(string& key)
{
  key = BI_PREFIX_CHAR;
  key.append(bucket_index_prefixes[BI_BUCKET_LOG_INDEX]);
}

static void bi_log_index_key(cls_method_context_t hctx, string& key, string& id, uint64_t index_ver)                                                   
{
  bi_log_prefix(key);
  get_index_ver_key(hctx, index_ver, &id);
  key.append(id);
}
#define BI_PREFIX_CHAR 0x80    
#define BI_BUCKET_OBJS_INDEX          0
#define BI_BUCKET_LOG_INDEX           1
#define BI_BUCKET_OBJ_INSTANCE_INDEX  2
#define BI_BUCKET_OLH_DATA_INDEX      3
#define BI_BUCKET_LAST_INDEX          4
static string bucket_index_prefixes[] = { "", /* special handling for the objs list index */
                                          "0_",     /* bucket log index */
                                          "1000_",  /* obj instance index */
                                          "1001_",  /* olh data index */
                                          /* this must be the last index */
                                          "9999_",};

```

我们可以看到，这种bi log都是以字符0x80开始，后面跟着'0_':

```
key (18 bytes):
00000000  80 30 5f 30 30 30 30 30  30 30 30 30 30 32 2e 32  |.0_00000000002.2|
00000010  2e 33                                             |.3|
00000012
```

我们可以通过radosgw-admin bilog list查看相应的bilog：

```bash
    {
        "op_id": "7#00000000001.1.2",
        "op_tag": "19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.64210",
        "op": "write",
        "object": "oem.tar.bz2",
        "instance": "",
        "state": "pending",
        "index_ver": 1,
        "timestamp": "0.000000",
        "ver": {
            "pool": -1,
            "epoch": 0
        },
        "bilog_flags": 0,
        "versioned": false,
        "owner": "",
        "owner_display_name": ""
    },
```

### RGWRados::Bucket::UpdateIndex::complete

介绍完UpdateIndex的prepare阶段，该介绍complete阶段了

```c++
int RGWRados::Bucket::UpdateIndex::complete(int64_t poolid, uint64_t epoch, uint64_t size, 
                                    ceph::real_time& ut, string& etag, string& content_type,                                           bufferlist *acl_bl, RGWObjCategory category,
                                    list<rgw_obj_key> *remove_objs, const string *user_data)
```

在该函数的末尾：

```c++
  ret = store->cls_obj_complete_add(*bs, optag, poolid, epoch, ent, category, remove_objs, bilog_flags);
  int r = store->data_log->add_entry(bs->bucket, bs->shard_id);
  if (r < 0) {
    lderr(store->ctx()) << "ERROR: failed writing data log" << dendl;
  }
  return ret;
```

其中store->cls_obj_complete_add这个函数：

```cpp
ret = store->cls_obj_complete_add(*bs, optag, poolid, epoch, ent, category, remove_objs, bilog_flags); 
int RGWRados::cls_obj_complete_add(BucketShard& bs, string& tag,
                                   int64_t pool, uint64_t epoch,
                                   RGWObjEnt& ent, RGWObjCategory category,
                                   list<rgw_obj_key> *remove_objs, uint16_t bilog_flags)
{
  return cls_obj_complete_op(bs, CLS_RGW_OP_ADD, tag, pool, epoch, ent, category, remove_objs, bilog_flags);
}

int RGWRados::cls_obj_complete_op(BucketShard& bs, RGWModifyOp op, string& tag,
                                  int64_t pool, uint64_t epoch,
                                  RGWObjEnt& ent, RGWObjCategory category,
                                  list<rgw_obj_key> *remove_objs, uint16_t bilog_flags)
{
      ...
      cls_rgw_bucket_complete_op(o, op, tag, ver, key, dir_meta, pro,  
                             get_zone().log_data, bilog_flags);
      ...
}
void cls_rgw_bucket_complete_op(ObjectWriteOperation& o, RGWModifyOp op, string& tag,
                                rgw_bucket_entry_ver& ver,
                                const cls_rgw_obj_key& key,
                                rgw_bucket_dir_entry_meta& dir_meta,
                                list<cls_rgw_obj_key> *remove_objs, bool log_op,
                                uint16_t bilog_flags)
{

  bufferlist in;
  struct rgw_cls_obj_complete_op call;
  call.op = op;
  call.tag = tag;
  call.key = key;
  call.ver = ver;
  call.meta = dir_meta;
  call.log_op = log_op;
  call.bilog_flags = bilog_flags;
  if (remove_objs)
    call.remove_objs = *remove_objs;
  ::encode(call, in);
  o.exec("rgw", "bucket_complete_op", in);
}

cls/rgw/cls_rgw.cc
-------------------
int rgw_bucket_complete_op(cls_method_context_t hctx, bufferlist *in, bufferlist *out)
{
   ...
    case CLS_RGW_OP_ADD:
    {
      struct rgw_bucket_dir_entry_meta& meta = op.meta;
      struct rgw_bucket_category_stats& stats = header.stats[meta.category];
      entry.meta = meta;
      entry.key = op.key;
      entry.exists = true;
      entry.tag = op.tag;
      stats.num_entries++;
      stats.total_size += meta.accounted_size;
      stats.total_size_rounded += cls_rgw_get_rounded_size(meta.accounted_size);
      bufferlist new_key_bl;
      ::encode(entry, new_key_bl);
      int ret = cls_cxx_map_set_val(hctx, idx, &new_key_bl);
      if (ret < 0)
        return ret;
    }
    break;
  }

  if (op.log_op) {
    rc = log_index_operation(hctx, op.key, op.op, op.tag, entry.meta.mtime, entry.ver,
                             CLS_RGW_STATE_COMPLETE, header.ver, header.max_marker, op.bilog_flags, NULL, NULL);
    if (rc < 0)
      return rc;                                               
 }
}
```

可以看到log_index_operation 函数，这个函数是第二个 bi log

```bash
key (18 bytes):
00000000  80 30 5f 30 30 30 30 30  30 30 30 30 30 32 2e 32  |.0_00000000002.2|
00000010  2e 33                                             |.3|
00000012
```

同样，我们可以通过radosgw-admin命令查看bilog

````bash
   radosgw-admin bilog list  --bucket bucket_0
   {
        "op_id": "7#00000000002.2.3",
        "op_tag": "19cbf250-bb3e-4b8c-b5bf-1a40da6610fe.15083.64210",
        "op": "write",
        "object": "oem.tar.bz2",
        "instance": "",
        "state": "complete",
        "index_ver": 2,
        "timestamp": "2018-12-15 07:53:11.077893152Z",
        "ver": {
            "pool": 3,
            "epoch": 1
        },
        "bilog_flags": 0,
        "versioned": false,
        "owner": "",
        "owner_display_name": ""
    },

````

至此，当上传对象的时候，两条bi log都介绍完了，值得注意的是key中的数值，

```cpp
static void bi_log_index_key(cls_method_context_t hctx, string& key, string& id, uint64_t index_ver)
{
  bi_log_prefix(key);
  get_index_ver_key(hctx, index_ver, &id);
  key.append(id);
}
static void get_index_ver_key(cls_method_context_t hctx, uint64_t index_ver, string *key)
{
  char buf[48];
  snprintf(buf, sizeof(buf), "%011llu.%llu.%d", (unsigned long long)index_ver,
           (unsigned long long)cls_current_version(hctx),
           cls_current_subop_num(hctx));                                               
  *key = buf;
} 
uint64_t cls_current_version(cls_method_context_t hctx)  
{ 
  ReplicatedPG::OpContext *ctx = *(ReplicatedPG::OpContext **)hctx;

  return ctx->pg->info.last_user_version;
}
int cls_current_subop_num(cls_method_context_t hctx)
{ 
  ReplicatedPG::OpContext *ctx = *(ReplicatedPG::OpContext **)hctx;

  return ctx->current_osd_subop_num;
}
```

Ceph保证了后面的序列部分单调递增。这个单调性对于multisite增量同步比较重要。

## data_log 

在UpdateIndex::complete 函数中，有如下内容：

```
  ret = store->cls_obj_complete_add(*bs, optag, poolid, epoch, ent, category, remove_objs, bilog_flags);
  int r = store->data_log->add_entry(bs->bucket, bs->shard_id);
  if (r < 0) {
    lderr(store->ctx()) << "ERROR: failed writing data log" << dendl;
  }
  return ret;
```

其中store->data_log->add_entry 即为往default.rgw.log 对应的data_log条目中增加log的部分。

当对bucket进行对象操作时，会在omap上新建一条"1_"+<timestamp> 开头的日志，表明这个bucket被修改过，增量同步时会根据这些日志判断出哪些bucket被更改过，进而再针对每个bucket进行同步。

### bucket 与 data_log.X的映射

以我们的Jewel 版本为例，bucket的shard个数为 8， 而default.rgw.log 中data_log.X 对象的个数为rgw_data_log_num_shards，即128个，RGW提供了两者的映射关系：

```cpp
int RGWDataChangesLog::choose_oid(const rgw_bucket_shard& bs) {
    const string& name = bs.bucket.name;
    int shard_shift = (bs.shard_id > 0 ? bs.shard_id : 0);
    uint32_t r = (ceph_str_hash_linux(name.c_str(), name.size()) + shard_shift) % num_shards; 
    return (int)r;
}
```

加入我们有N个bucket，每个bucket shards是8，也就是将8*N个对象通过choose_oid映射到128个data_log.X对象。

上传一个对象之后，我们可以从default.rgw.log 的data_log.X的omap信息中得到一笔新的key-value信息：

```
root@NODE-246:/var/log# rados -p default.rgw.log ls |grep data_log  |xargs -I {} rados -p default.rgw.log listomapvals {} 
1_1544942616.469385_1491.1
value (185 bytes) :
00000000  02 01 b3 00 00 00 00 00  00 00 37 00 00 00 62 75  |..........7...bu|
00000010  63 6b 65 74 5f 30 3a 31  39 63 62 66 32 35 30 2d  |cket_0:19cbf250-|
00000020  62 62 33 65 2d 34 62 38  63 2d 62 35 62 66 2d 31  |bb3e-4b8c-b5bf-1|
00000030  61 34 30 64 61 36 36 31  30 66 65 2e 31 35 30 38  |a40da6610fe.1508|
00000040  33 2e 31 3a 37 18 f4 15  5c 44 40 fa 1b 4a 00 00  |3.1:7...\D@..J..|
00000050  00 01 01 44 00 00 00 01  37 00 00 00 62 75 63 6b  |...D....7...buck|
00000060  65 74 5f 30 3a 31 39 63  62 66 32 35 30 2d 62 62  |et_0:19cbf250-bb|
00000070  33 65 2d 34 62 38 63 2d  62 35 62 66 2d 31 61 34  |3e-4b8c-b5bf-1a4|
00000080  30 64 61 36 36 31 30 66  65 2e 31 35 30 38 33 2e  |0da6610fe.15083.|
00000090  31 3a 37 18 f4 15 5c 44  40 fa 1b 1a 00 00 00 31  |1:7...\D@......1|
000000a0  5f 31 35 34 34 39 34 32  36 31 36 2e 34 36 39 33  |_1544942616.4693|
000000b0  38 35 5f 31 34 39 31 2e  31                       |85_1491.1|
000000b9
```

这个键值命名的规范是如何的？

```cpp
cls/log/cls_log.cc
-----------------------
static string log_index_prefix = "1_"; 
static void get_index(cls_method_context_t hctx, utime_t& ts, string& index)
{
  get_index_time_prefix(ts, index);   
  string unique_id;
  cls_cxx_subop_version(hctx, &unique_id);
  index.append(unique_id);
}
static void get_index_time_prefix(utime_t& ts, string& index)
{
  char buf[32];
  snprintf(buf, sizeof(buf), "%010ld.%06ld_", (long)ts.sec(), (long)ts.usec());
  index = log_index_prefix + buf;
}
uint64_t cls_current_version(cls_method_context_t hctx)
{
  ReplicatedPG::OpContext *ctx = *(ReplicatedPG::OpContext **)hctx;

  return ctx->pg->info.last_user_version;
}
int cls_current_subop_num(cls_method_context_t hctx)
{
  ReplicatedPG::OpContext *ctx = *(ReplicatedPG::OpContext **)hctx;
  return ctx->current_osd_subop_num;
}
void cls_cxx_subop_version(cls_method_context_t hctx, string *s) 
{
  if (!s)
    return;
  char buf[32];
  uint64_t ver = cls_current_version(hctx);
  int subop_num = cls_current_subop_num(hctx);
  snprintf(buf, sizeof(buf), "%lld.%d", (long long)ver, subop_num);
  *s = buf;
}
```

1_1544942616.469385_1491.1这个键值也是一样，ceph保证其单调递增的特性。当multisite同步的时候，这个特性很重要。
