# 前言

上篇博客介绍了iostat的一些输出，这篇介绍blktrace这个神器。上一节介绍iostat的时候，我们心心念念希望得到块设备处理io的service time，而不是service time + wait time，因为对于评估一个磁盘或者云磁盘而言，service time才是衡量磁盘性能的核心指标和直接指标。很不幸iostat无法提供这个指标，但是blktrace可以。

blktrace是一柄神器，很多工具都是基于该神器的：ioprof，seekwatcher，iowatcher，这个工具基本可以满足我们的对块设备请求的所有了解。

# blktrace的原理

一个I/O请求，从应用层到底层块设备，路径如下图所示：

![](/assets/IO/Linux-storage-stack-diagram_v4.0.png)

从上图可以看出IO路径是很复杂的。这么复杂的IO路径我们是无法用短短一篇小博文介绍清楚的。我们将IO路径简化一下：

![](/assets/IO/io_path_simple.png)



一个I/O请求进入block layer之后，可能会经历下面的过程：

- Remap: 可能被DM(Device Mapper)或MD(Multiple Device, Software RAID) remap到其它设备
- Split: 可能会因为I/O请求与扇区边界未对齐、或者size太大而被分拆(split)成多个物理I/O
- Merge: 可能会因为与其它I/O请求的物理位置相邻而合并(merge)成一个I/O
- 被IO Scheduler依照调度策略发送给driver
- 被driver提交给硬件，经过HBA、电缆（光纤、网线等）、交换机（SAN或网络）、最后到达存储设备，设备完成IO请求之后再把结果发回。

blktrace 能够记录下IO所经历的各个步骤:

![](/assets/IO/blktrace_architecture.png)

我们一起看下blktrace的输出长什么样子：

![](/assets/IO/blktrace_out.png)



* 第一个字段：8,0  这个字段是设备号 major device ID和minor device ID。
* 第二个字段：3  表示CPU
* 第三个字段：11 序列号
* 第四个字段：0.009507758  Time Stamp是时间偏移
* 第五个字段：PID 本次IO对应的进程ID
* 第六个字段：Event，这个字段非常重要，反映了IO进行到了那一步
* 第七个字段：R表示 Read， W是Write，D表示block，B表示Barrier Operation
* 第八个字段：223490+56，表示的是起始block number 和 number of blocks，即我们常说的Offset 和 Size
* 第九个字段： 进程名

其中第六个字段非常有用：每一个字母都代表了IO请求所经历的某个阶段。

Q – 即将生成IO请求
|
G – IO请求生成
|
I – IO请求进入IO Scheduler队列
|
D – IO请求进入driver
|
C – IO请求执行完毕

注意，整个IO路径，分成很多段，每一段开始的时候，都会有一个时间戳，根据上一段开始的时间和下一段开始的时间，就可以得到IO 路径各段花费的时间。

注意，我们心心念念的service time，也就是反应块设备处理能力的指标，就是从D到C所花费的时间，简称D2C。

而iostat输出中的await，即整个IO从生成请求到IO请求执行完毕，即从Q到C所花费的时间，我们简称Q2C。

我们知道Linux 有I/O scheduler，调度器的效率如何，I2D是重要的指标。

注意，这只是blktrace输出的一个部分，很明显，我们还能拿到offset和size，根据offset，我们能拿到某一段时间里，应用程序都访问了整个块设备的那些block，从而绘制出块设备访问轨迹图。

另外还有size和第七个字段（Read or Write），我们可以知道IO size的分布直方图。对于本文来讲，我们就是要根据blktrace来获取这些信息。



# blktrace、blkparse和btt

我们接下来简单介绍这些工具的使用，其中这三个命令都是属于blktrace这个包的，他们是一家人。

首先通过如下命令，可以查看磁盘上的实时信息：

```
 blktrace -d /dev/sdb -o – | blkparse -i –
```

这个命令会连绵不绝地出现很多输出，当你输入ctrl＋C的时候，会停止。

当然了，你也可以先用如下命令采集信息，待所有信息采集完毕后，统一分析所有采集到的数据。搜集信息的命令如下：

```
blktrace -d /dev/sdb
```

注意，这个命令并不是只输出一个文件，他会根据CPU的个数上，每一个CPU都会输出一个文件，如下所示：

```
-rw-r--r-- 1 manu manu  26K Jul  3 02:04 sdb.blktrace.0
-rw-r--r-- 1 manu manu  13K Jul  3 02:04 sdb.blktrace.1
-rw-r--r-- 1 manu manu 3.0K Jul  3 02:04 sdb.blktrace.10
-rw-r--r-- 1 manu manu 5.7K Jul  3 02:04 sdb.blktrace.11
-rw-r--r-- 1 manu manu  16K Jul  3 02:04 sdb.blktrace.12
-rw-r--r-- 1 manu manu  12K Jul  3 02:04 sdb.blktrace.13
-rw-r--r-- 1 manu manu 6.3K Jul  3 02:04 sdb.blktrace.14
-rw-r--r-- 1 manu manu  12K Jul  3 02:04 sdb.blktrace.15
-rw-r--r-- 1 manu manu  12K Jul  3 02:04 sdb.blktrace.16
-rw-r--r-- 1 manu manu 4.6K Jul  3 02:04 sdb.blktrace.17
-rw-r--r-- 1 manu manu  476 Jul  3 02:04 sdb.blktrace.18
-rw-r--r-- 1 manu manu  220 Jul  3 02:04 sdb.blktrace.19
-rw-r--r-- 1 manu manu  22K Jul  3 02:04 sdb.blktrace.2
-rw-r--r-- 1 manu manu  320 Jul  3 02:04 sdb.blktrace.20
-rw-r--r-- 1 manu manu  320 Jul  3 02:04 sdb.blktrace.21
-rw-r--r-- 1 manu manu 1.1K Jul  3 02:05 sdb.blktrace.22
-rw-r--r-- 1 manu manu  696 Jul  3 02:04 sdb.blktrace.23
-rw-r--r-- 1 manu manu  14K Jul  3 02:04 sdb.blktrace.3
-rw-r--r-- 1 manu manu  13K Jul  3 02:04 sdb.blktrace.4
-rw-r--r-- 1 manu manu 7.9K Jul  3 02:04 sdb.blktrace.5
-rw-r--r-- 1 manu manu 2.4K Jul  3 02:04 sdb.blktrace.6
-rw-r--r-- 1 manu manu 6.2K Jul  3 02:04 sdb.blktrace.7
-rw-r--r-- 1 manu manu 6.0K Jul  3 02:04 sdb.blktrace.8
-rw-r--r-- 1 manu manu 4.2K Jul  3 02:05 sdb.blktrace.9
```

有了输出，我们可以通过blkparse  -i sdb来分析采集的数据：

```
 8,16   1        1     0.000000000 897886  Q   R 0 + 1 [dd]
  8,16   1        2     0.000002510 897886  G   R 0 + 1 [dd]
  8,16   1        3     0.000003649 897886  P   N [dd]
  8,16   1        4     0.000005329 897886  I   R 0 + 1 [dd]
  8,16   1        5     0.000006234 897886  U   N [dd] 1
  8,16   1        6     0.000007286 897886  D   R 0 + 1 [dd]
  8,16   1        7     0.000168995     0  C   R 0 + 1 [0]
  8,16   1        8     0.000192641 680104  G   N [kworker/1:0]
  8,16   1        9     0.000193318 680104  I   N 0 (00 ..) [kworker/1:0]
  8,16   1       10     0.000193637 680104  D   N 0 (00 ..) [kworker/1:0]
  8,16   0        1     0.000206557 897879  C   N (00 ..) [0]
  8,16   3        1     0.100477153 53656  Q FWS [eziscsid.py]
  8,16   3        2     0.100479799 53656  G FWS [eziscsid.py]
  8,16   3        3     0.100481319 53656  I FWS [eziscsid.py]
  8,16   3        4     0.100499251    24  C  WS 0 [0]
  8,16   4        1     0.100533738 849600  G   N [kworker/4:0]
  8,16   4        2     0.100534608 849600  I   N 0 (00 ..) [kworker/4:0]
  8,16   4        3     0.100535119 849600  D   N 0 (00 ..) [kworker/4:0]
  8,16   0        2     0.100574116     0  C   N (00 ..) [0]
  8,16   0        3     0.103764700 823362  G   N [kworker/0:0]
  8,16   0        4     0.103765221 823362  I   N 0 (00 ..) [kworker/0:0]
  8,16   0        5     0.103765744 823362  D   N 0 (00 ..) [kworker/0:0]
  8,16   0        6     0.103778570 897905  C   N (00 ..) [0]
  8,16   0        7     0.104790659 897907  G   N [scsi_id]
```

注意，blkparse仅仅是将blktrace输出的信息转化成人可以阅读和理解的输出，但是，信息太多，太杂，人完全没法得到关键信息。
这时候btt就横空出世了，这个工具可以将blktrace采集回来的数据，进行分析，得到对人更有用的信息。事实上，btt也是我们的终点。



接下来，我们要利用blktrace blkparse 以及btt来采集和分析单块磁盘的的性能，最终我会生成一个pdf的文档。步骤如下：

```
输入： blktrace采集到的原始数据
输出： 使用btt，blkparse还有自己写的一些bash脚本和python脚本，输出出pdf格式的report
```

* 通过各种工具，生成原始的分析结果，以及绘制对应的PNG图片：
* 将分析结果以表格和图片的方式，写入markdown文本
* 将markdown 文本通过pandoc转换成pdf文档。


# 获取个阶段的延迟信息

注意，btt已经可以很自如地生成这部分统计信息，我们可以很容易得到如下的表格：

![](/assets/IO/latency_distribution_table.png)

方法如下：

首先blkparse可以将对应不同cpu的多个文件聚合成一个文件：

```
blkparse -i sdb -d sdb.blktrace.bin
```
然后btt就可以分析这个sdb.blktrace.bin了：

```
==================== All Devices ====================                                                                                               

            ALL           MIN           AVG           MAX           N   
--------------- ------------- ------------- ------------- -----------

Q2Q               0.000003604   0.150467580   5.459567832         307 
Q2G               0.000000466   0.000001259   0.000010933         308 
G2I               0.000001152   0.000033227   0.000225327         308 
I2D               0.000000890   0.000009647   0.000031208         308 
D2C               0.000039838   0.000159417   0.006600934         308 
Q2C               0.000071104   0.000203550   0.006605087         308 

==================== Device Overhead ====================

       DEV |       Q2G       G2I       Q2M       I2D       D2C 
---------- | --------- --------- --------- --------- ---------
| (  8, 16) | 0.6186%  16.3239%   0.0000%   4.7391%  78.3183% |
| --------- | ---------------------------------------- |
| Overall   | 0.6186%  16.3239%   0.0000%   4.7391%  78.3183% |

==================== Device Merge Information ====================

       DEV |       #Q       #D   Ratio |   BLKmin   BLKavg   BLKmax    Total
---------- | -------- -------- ------- | -------- -------- -------- --------
 (  8, 16) |      308      308     1.0 |        1      416      512   128297

==================== Device Q2Q Seek Information ====================

       DEV |          NSEEKS            MEAN          MEDIAN | MODE               
---------- | --------------- --------------- --------------- | ---------------
 (  8, 16) |             308     165293887.1               0 | 0(209)

---------- | --------------- --------------- --------------- | ---------------                                                                      
   Overall |          NSEEKS            MEAN          MEDIAN | MODE
   Average |             308     165293887.1               0 | 0(209)

```

注意： D2C和Q2C，一个是表征块设备性能的关键指标，另一个是客户发起请求到收到响应的时间，我们可以看出，

D2C 平均在0.000159417 秒，即0.159毫秒
Q2C 平均在0.000203550 秒，即0.204毫秒，

无论是service time 还是客户感知到的await time，都是非常短的，表现非常不俗。

而最大值部分，D2C最大值为0.006600934，这个值是6.6毫秒，也并不差。


# IOPS 和 MBPS

从btt出发，我们分析出来采样时间内，整个块设备的IOPS：

![](/assets/IO/sdb_iops.png)
![](/assets/IO/sdb_mbps.png)

获取方法如下：

* blkparse -i sdb -d sdb.blktrace.bin
* btt -i sdb.blktrace.bin -q sdb.q2c_latency

注意，这一步之后，我们会得到如下文件：

* sdb.q2c_latency_8,16_q2c,dat
* sys_iops_fp.dat
* sys_mbps_fp.dat
* 8,16_iops_fp.dat
* 8,17_mbps_fp.dat

注意，如果我们blktrace －d sdb，只关注sdb的时候，我们可以通过sys_iops_fp.dat和sys_mbps_fp.dat获取对应的IOPS和MBPS信息：

```
cat sys_iops_fp.dat 
0 2
1 1
3 2
7 1
8 1
13 1
15 1
16 2
17 1
22 30
23 40
25 1
26 1
29 18
30 97
31 10
34 1
37 15
38 55
39 14
42 2
43 1
45 1
46 10
```



# IO Size Historgram

我们很关心，在采样的时间内，IO size的分布情况，因为这个可以得到，过去的时间里，我们是大IO居多还是小IO居多：

![](/assets/IO/sdb_iosize_hist.png)

步骤如下：

* blkparse -i  sdb -d sub.blktrace.bin
* btt -i sdb.blktrace.bin -B sdb.offset

这个步骤之后会生成三个文件：

* sdb.offset\_8,16\_r.dat
* sdb.offset\_8,16\_w.dat
* sdb.offset\_8,16\_c.dat

其中r表示读操作的offset和size信息，w表示写操作的offset和size信息，c表示读＋写。

其输出格式如下：

```
 cat sdb.offset_8,16_w.dat
   22.894414021 78127616 78128128
   22.894423001 78128128 78128640
   22.894425645 78128640 78128896
   22.946692637 78134528 78135040
   22.946704256 78135040 78135552
   22.946708799 78135552 78136064
   22.946712643 78136064 78136576
   22.947038705 78139392 78139904
   22.947044009 78139904 78140416
   22.947046356 78140416 78140928
   22.947332137 78145536 78146048
   22.947336606 78146048 78146560
   22.947338683 78146560 78147072
   22.956552855 78155520 78156032
   22.956559841 78156032 78156544
   22.956564151 78156544 78156800
```

注意，第一个字段是时间，第二个字段是开始扇区即offset，第三个字段为结束扇区。不难根据第二个字段和第三个字段算出来size。当然了单位为扇区。



# 访问轨迹图

注意上小节，可以拿到不同时间里，访问磁盘的位置以及访问扇区的个数，如果不考虑访问扇区的个数，我们可以得到一张访问轨迹2D图：

![](/assets/IO/sdb_offset.png)

如果把访问扇区的个数作为第三个维度，可以得到一张三维图

![](/assets/IO/sdb_offset_pattern.png)





