---
layout: post
title: ceph 写流程（1）
date: 2016-10-06 22:21:40
categories: ceph-internal
tag: ceph
excerpt: 介绍ceph 写流程
---

# 前言

前面花了两篇博客的篇幅介绍了读流程。写流程和读流程相比，有大量的流程走向是公用的，我将这些公共的流程总结如下：

![](/assets/ceph_internals/read_workflow_enter_queue.png)

![](/assets/ceph_internals/read_write_workflow_common.png)

对于读流程而言，相对比较简单，这也是我们先介绍读流程的原因。先设置一个小目标，达成之后，对总体的大目标也有好处，相当于将大任务分解成了几个小任务。

从 execute_ctx 开始，读写流程开始严重的分叉，下图是读流程的流程图

![](/assets/ceph_internals/read_flow_execute_ctx.png)



# 写流程的三个侧面

写流程之所以比读流程复杂，原因在于读流程只需要去Primary OSD读取响应的Object即可，而写流程牵扯到多个OSD。下图来自Ceph的官方文档：

![](/assets/ceph_internals/write_3_replica.png)

写流程之所以比读流程复杂源于多个方面，

* 牵扯多个OSD的写入，如何确保多副本之间一致性  （PGLog）
* 对于单个OSD的写入，如何确保最终的一致性      （Journal and FileStore）
* 多个副本所在的OSD，如果状态不是active ＋ clean 

多种因素造成了写流程的异常复杂。本人功力有限，所以先从主干流程介绍起。这个写流程，打算至少从三个侧重点分别介绍：

* 第一个篇侧重在Primary OSD和Secondary OSD的交互流程，即Primary 如何将写相关的任务发送给Secondary OSD，Secondary OSD又如何发送自己的完成进度给Primary OSD， Primary OSD收到相关的进度，又采取了什么样的行动，以及Primary如何给Client发送响应

* 第二篇文章侧重于数据部分，即各个OSD 收到写入的请求之后，从filestore层面做了哪些的事情，在不同的完成阶段，会做哪些事情

* 第三篇文章会侧重于PGLog，为了确保各个副本的一致性，出了写入数据，事务操作也会纪录PGLog，一旦某个出现异常，可以根据PGLog的信息，确定哪个OSD的数据是最可靠的，从发起数据的同步。

因为写入的流程异常的复杂，因此，介绍A侧面的时候，尽量不涉及B和C侧面，否则所有细节纠缠在一起，就会将设计思想淹没在无数的细节之中，这并不利于我们理解写入流程。

# Primary和Secondary的交互

ceph的读和写显著的不同在于读基本上只需要从Primary OSD中读取（offset，length）指定部分的内容即可，不牵扯到多个OSD之间的交互，而且读并没有对存储作出改变。 而写则不然，首先，ceph支持多副本，也支持纠删码（本系列暂不考虑纠删码），写入本身就牵扯到多个OSD之间的互动。其次，正常情况自不必说，但是多个副本之间的写入可能会在某个副本出现问题，副本之间需要能够确定哪个副本的数据已经正确写入，哪个副本的数据还未写入完毕，这就加剧了ceph写入的复杂程度。

本文只介绍Primary 和Secondary之间的消息交互，作为整个写入过程的整体框架。

