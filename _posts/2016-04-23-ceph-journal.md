# Ceph的Journal机制

## Journal简介

ceph的FileStore模式下，使用采用了写日志的方式。最初接触ceph的时候，如果Journal分区和Data分区都在某个RAID上，通过ceph -w看到的写入带宽和底层磁盘的带宽相比较，就会发现有写入放大的问题。


