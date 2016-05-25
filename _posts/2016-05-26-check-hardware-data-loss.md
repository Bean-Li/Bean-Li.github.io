---
layout: post
title: Can hardware withstand Power Failure without data loss
date: 2016-05-26 22:33:40
categories: linux
tag: linux
excerpt: 提供一种检查硬件配置或硬件是否存在数据丢失风险的方法
---

## 前言

做存储的而言，最担心的情况就是数据丢失。但是在客户现场，异常掉电是可能的，但是纵然异常掉电，也不应该丢失数据，因此从RAID的层面来讲，硬件必须要有BBU，这种情况下使用Write Back的配置才会安全。 之前也曾提到过，Disk cache Policy应该是Disable，否则，纵然有BBU也会data loss。

这只是一种情况，事实上存储硬件的形态有很多中，单盘，JBOD，外接盘阵，很多种情况，我们创建集群之后，我们可能很关心的一点是，我们的硬件（包括硬件配置）是否是可靠的？ 异常掉电是否会data loss？

本文提供一种方法来简单的监测硬件的可靠行。当然这个方法并非是我发明的，当初我为了证明 disk cache policy Eanble是不安全的，我也想设计一套方案，但是被我从网上找到了这个： [diskchecker.pl](http://brad.livejournal.com/2116715.html)

我不知道如果我找不到这片文章，是否能够想出来这个方案，但是应该设计的没有这么好。不过看到这篇文章，我立刻放弃了自己动手的想法，站在巨人的肩上并不可耻，解决问题才是王道。No problem should ever have to be solved twice。 当然了，光荣属于前辈。

## 原理

这个脚本监测的目标是，当用户掉用了sync/fsync/syncfs等接口之后，是否真正写入了持久化设备，哪怕掉电，也不会丢失sync过的数据。

表面看，这个要求好像很低，事实上并非如此，很多硬件如果配置不当，sync/fsync/syncfs/并不能确保数据能够扛住异常掉电。

该脚本设计的非常精巧，它为了测试目的节点的硬件是否有数据丢失的风险，它引入了仲裁节点。所谓仲裁节点是另外一台机器，测试脚本写入每次写入16KB，写入之前都会通知仲裁节点写入的内容。 由于写入的16KB是单字节的重复，因此，不需要传输16KB的内容到仲裁节点，只需要告知仲裁节点，页面num和要写入的字节，当然了每一次下入的字节内容是随机的。当写入目标之后，会掉用fsync，sync写入的内容，sync成功之后，再次通知仲裁节点。

可以预设写入文件的大小，反正文件是16KB 16KB的写入，然后在任何时间点，拔电源，测试异常掉电，重启之后，对文件的内容进行监测，通过和仲裁节点通信判断sync过的数据是否真正地写入了持久化设备。

这个脚本太优美了，设计太精巧了，我忍不住把全文贴在此处，这个脚本的作者是Brad Fitzpatrick，向他致敬，可以从[github](https://gist.github.com/bradfitz/3172656#file-diskchecker-pl)上获取该脚本

```
#!/usr/bin/perl
#
# Brad's el-ghetto do-our-storage-stacks-lie?-script
#

sub usage {
    die <<'END';
Usage: diskchecker.pl -s <server[:port]> verify <file>
       diskchecker.pl -s <server[:port]> create <file> <size_in_MB>
       diskchecker.pl -l [port]
END
}

use strict;
use IO::Socket::INET;
use IO::Handle;
use Getopt::Long;
use Socket qw(IPPROTO_TCP TCP_NODELAY);

my $server;
my $listen;
usage() unless GetOptions('server=s' => \$server,
                          'listen:5400' => \$listen);
usage() unless $server || $listen;
usage() if     $server && $listen;

# LISTEN MODE:
listen_mode($listen) if $listen;

# CLIENT MODE:
my $LEN = 16 * 1024;   # 16kB (same as InnoDB page)
my $mode = shift;
usage() unless $mode =~ /^verify|create$/;

my $file = shift or usage();
my $size;
if ($mode eq "create") {
    $size = shift or usage();
}

$server .= ":5400" unless $server =~ /:/;

my $sock = IO::Socket::INET->new(PeerAddr => $server)
    or die "Couldn't connect to host:port of '$server'\n";

setsockopt($sock, IPPROTO_TCP, TCP_NODELAY, pack("l", 1)) or die;

create() if $mode eq "create";
verify() if $mode eq "verify";
exit 0;

sub verify {
    sendmsg($sock, "read");

    my $error_ct = 0;
    my %error_ct;

    my $size = -s $file;
    my $max_pages = int($size / $LEN);

    my $percent;
    my $last_dump = 0;
    my $show_percent = sub {
	printf " verifying: %.02f%%\n", $percent;
    };

    open (F, $file) or die "Couldn't open file $file for read\n";

    while (<$sock>) {
	chomp;
	my ($page, $good, $val, $ago) = split(/\t/, $_);
	$percent = 100 * $page / ($max_pages || 1);

	my $now = time;
	if ($last_dump != $now) {
	    $last_dump = $now;
	    $show_percent->();
	}

	next unless $good;

	my $offset = $page * $LEN;
	sysseek F, $offset, 0;
	my $buf;
	my $rv = sysread(F, $buf, $LEN);
	my $tobe = sprintf("%08x", $val) x ($LEN / 8);
	substr($tobe, $LEN-1, 1) = "\n";

	unless ($buf eq $tobe) {
	    $error_ct{$ago}++;
	    $error_ct++;
	    print "  Error at page $page, $ago seconds before end.\n";
	}
    }
    $show_percent->();

    print "Total errors: $error_ct\n";
    if ($error_ct) {
	print "Histogram of seconds before end:\n";
	foreach (sort { $a <=> $b } keys %error_ct) {
	    printf "  %4d %4d\n", $_, $error_ct{$_};
	}
    }
}

sub create {
    open (F, ">$file") or die "Couldn't open file $file\n";

    my $ioh = IO::Handle->new_from_fd(fileno(F), "w")
	or die;

    my $pages = int( ($size * 1024 * 1024) / $LEN );  # 50 MiB of 16k pages (3200 pages)

    my %page_hit;
    my $pages_hit = 0;
    my $uniq_pages_hit = 0;
    my $start = time();
    my $last_dump = $start;

    while (1) {
	my $rand = int rand 2000000;
	my $buf = sprintf("%08x", $rand) x ($LEN / 8);
	substr($buf, $LEN-1, 1) = "\n";

	my $pagenum = int rand $pages;
	my $offset = $pagenum * $LEN;

        sendmsg($sock, "pre\t$pagenum\t$rand");

        # now wait for acknowledgement
        my $ok = readmsg($sock);
        die "didn't get 'ok' from server ($pagenum $rand), msg=[$ok] = $!" unless $ok eq "ok";

	sysseek F,$offset,0;
	my $wv = syswrite(F, $buf, $LEN);
	die "return value wasn't $LEN\n" unless $wv == $LEN;
	$ioh->sync or die "couldn't do IO::Handle::sync";  # does fsync

	sendmsg($sock, "post\t$pagenum\t$rand");

        $pages_hit++;
        unless ($page_hit{$pagenum}++) {
            $uniq_pages_hit++;
        }

        my $now = time;
        if ($now != $last_dump) {
            $last_dump = $now;
            my $runtime = $now - $start;
            printf("  diskchecker: running %d sec, %.02f%% coverage of %d MB (%d writes; %d/s)\n",
                   $runtime,
                   (100 * $uniq_pages_hit / $pages),
                   $size,
                   $pages_hit,
                   $pages_hit / $runtime,
                   );
        }

    }
}

sub readmsg {
    my $sock = shift;
    my $len;
    my $rv = sysread($sock, $len, 1);
    return undef unless $rv == 1;
    my $msg;
    $rv = sysread($sock, $msg, ord($len));
    return $msg;
}

sub sendmsg {
    my ($sock, $msg) = @_;
    my $rv = syswrite($sock, chr(length($msg)) . $msg);
    my $expect = length($msg) + 1;
    die "sendmsg failed rv=$rv, expect=$expect" unless $rv == $expect;
    return 1;
}

sub listen_mode {
    my $port = shift;
    my $server = IO::Socket::INET->new(ReuseAddr => 1,
                                       Listen => 1,
                                       LocalPort => $port)
        or die "couldn't make server socket\n";

    while (1) {
        print "[server] diskchecker.pl: waiting for connection...\n";
        my $sock = $server->accept()
            or die "  die: no connection?";
        setsockopt($sock, IPPROTO_TCP, TCP_NODELAY, pack("l", 1)) or die;

        fork and next;
        process_incoming_conn($sock);
        exit 0;
    }
}

sub process_incoming_conn {
    my $sock = shift;
    my $peername = getpeername($sock) or
        die "connection not there?\n";
    my ($port, $iaddr) = sockaddr_in($peername);
    my $ip = inet_ntoa($iaddr);

    my $file = "/tmp/$ip.diskchecker";
    die "[$ip] $file is a symlink" if -l $file;

    print "[$ip] New connection\n";

    my $lines = 0;
    my %state;
    my $end;

    while (1) {
        if ($lines) {
            last unless wait_for_readability(fileno($sock), 3);
        }
        my $line = readmsg($sock);
        last unless $line;

        if ($line eq "read") {
            print "[$ip] Sending state info from ${ip}'s last create.\n";
            open (S, "$file") or die "Couldn't open $file for reading.";
            while (<S>) {
                print $sock $_;
            }
            close S;
            print "[$ip] Done.\n";
            exit 0;
        }

        $lines++;
        my $now = time;
        $end = $now;
        my ($state, $pagenum, $rand) = split(/\t/, $line);
        if ($state eq "pre") {
            $state{$pagenum} = [ 0, $rand+0, $now ];
            sendmsg($sock, "ok");
        } elsif ($state eq "post") {
            $state{$pagenum} = [ 1, $rand+0, $now ];
        }
        print "[$ip] $lines writes\n" if $lines % 1000 == 0;
    }

    print "[$ip] Writing state file...\n";
    open (S, ">$file") or die "Couldn't open $file for writing.";
    foreach (sort { $a <=> $b } keys %state) {
        my $v = $state{$_};
        my $before_end = $end - $v->[2];
        print S "$_\t$v->[0]\t$v->[1]\t$before_end\n";
    }
    print "[$ip] Done.\n";
}

sub wait_for_readability {
    my ($fileno, $timeout) = @_;
    return 0 unless $fileno && $timeout;

    my $rin;
    vec($rin, $fileno, 1) = 1;
    my $nfound = select($rin, undef, undef, $timeout);

    return 0 unless defined $nfound;
    return $nfound ? 1 : 0;
}

```

## 应用

我们考虑下ceph，由于ceph是一种软件定义存储，后台的OSD对应的存储设备可能是单盘，可能是RAID，可能是JBOD，可能是外接盘阵，必须有一种方法来检查硬件能否扛住异常掉电，那么毫无疑问，上面的方法非常重要。

有很多人觉得，既然执行过sync，数据一定是安全的，一定可以挡住异常掉电，其实这种想法是一厢情愿。以RAID为例，如果存在BBU，并且采用Write Back的方法，只要Disk Cache Policy是Enabled，数据依然是不安全的，异常掉电，几乎总是会丢失数据，甚至造成文件系统损坏。

我们下面详细介绍如何使用diskchecker.pl来判断硬件是否有data loss的风险。


具体实施的方法如下：

0 将diskchecker.pl上传到两个存储节点上的/root目录,chmod a+x /root/diskchecker.pl

1 在多个节点上任意选一台作为仲裁节点，比如本实验中的186，在186节点执行如下命令：

```
/root/diskchecker.pl -l
```

其输出如下：

```
root@node186:~# /root/diskchecker.pl -l
[server] diskchecker.pl: waiting for connection...

```


2  在测试的目标节点上，执行写入动作，其命令如下：
   当然，因为我们要测试OSD的安全性，因此，需要在/data/osd.X下写入内容。

```
root@node188:/data/osd.4# /root/diskchecker.pl -s 10.16.172.186  create  osd_test 500M
  diskchecker: running 1 sec, 3.28% coverage of 500 MB (1068 writes; 1068/s)
  diskchecker: running 2 sec, 10.16% coverage of 500 MB (3428 writes; 1714/s)
  diskchecker: running 3 sec, 11.41% coverage of 500 MB (3868 writes; 1289/s)
  diskchecker: running 4 sec, 11.76% coverage of 500 MB (3997 writes; 999/s)
  diskchecker: running 5 sec, 13.69% coverage of 500 MB (4712 writes; 942/s)
  diskchecker: running 6 sec, 15.91% coverage of 500 MB (5550 writes; 925/s)
  diskchecker: running 7 sec, 19.82% coverage of 500 MB (7051 writes; 1007/s)
  diskchecker: running 8 sec, 22.23% coverage of 500 MB (8033 writes; 1004/s)
  diskchecker: running 9 sec, 24.82% coverage of 500 MB (9144 writes; 1016/s)
  diskchecker: running 10 sec, 26.92% coverage of 500 MB (10051 writes; 1005/s)
  diskchecker: running 11 sec, 28.87% coverage of 500 MB (10920 writes; 992/s)
  diskchecker: running 12 sec, 31.89% coverage of 500 MB (12271 writes; 1022/s)
  diskchecker: running 13 sec, 33.91% coverage of 500 MB (13246 writes; 1018/s)
  diskchecker: running 14 sec, 36.08% coverage of 500 MB (14345 writes; 1024/s)
  diskchecker: running 15 sec, 37.68% coverage of 500 MB (15189 writes; 1012/s)
  diskchecker: running 16 sec, 39.80% coverage of 500 MB (16263 writes; 1016/s)
  diskchecker: running 17 sec, 42.17% coverage of 500 MB (17600 writes; 1035/s)
  diskchecker: running 18 sec, 43.89% coverage of 500 MB (18608 writes; 1033/s)
  diskchecker: running 19 sec, 45.60% coverage of 500 MB (19598 writes; 1031/s)
  diskchecker: running 20 sec, 46.88% coverage of 500 MB (20353 writes; 1017/s)
  diskchecker: running 21 sec, 48.61% coverage of 500 MB (21383 writes; 1018/s)
  diskchecker: running 22 sec, 51.20% coverage of 500 MB (23081 writes; 1049/s)
```

4 写入一部分内容后，对目标节点强制断电，可以通过IPMI的 Power Reset，或者在机房拔电源或者长按关机键。注意不必等到写入100%，写入60%左右即可强制断电。

5 重启目标机器，并验证数据的完整性,同样在/data/osd.X目录下执行如下命令：

```
/root/diskchecker.pl -s 10.16.172.186 verify osd_test
```

如果你看到如下输出（关键是Total errors:0）,表示数据足够安全：

```
root@node188:/data/osd.4# /root/diskchecker.pl -s 10.16.172.186 verify osd_test
 verifying: 0.00%
 verifying: 16.52%
 verifying: 50.35%
 verifying: 81.77%
 verifying: 100.00%
Total errors: 0
```

如果很不幸，你看到了如下输出：

```
  Error at page 31271, 138 seconds before end.
  Error at page 31272, 5 seconds before end.
  Error at page 31273, 121 seconds before end.
  Error at page 31274, 88 seconds before end.
  Error at page 31275, 121 seconds before end.
  Error at page 31276, 26 seconds before end.
  Error at page 31277, 74 seconds before end.
  Error at page 31278, 11 seconds before end.
  Error at page 31279, 7 seconds before end.
  Error at page 31280, 17 seconds before end.
  Error at page 31281, 24 seconds before end.
  Error at page 31282, 76 seconds before end.
  Error at page 31283, 6 seconds before end.
```

那么，很显然，你的硬件并不安全，异常掉电的情况下，丢失数据几乎是必然。

## 一些测试结果



| hardware type              | reboot                         |      shutdown   | 长按关机键      |   拔电源        |
| -------------------------- | ------------------------------ |-----------------|---------------|-----------------|
| `RAID + DISK Cache Enable`      |  OK OK OK OK | OK OK         | OK ERR ERR  | ERR ERR
| `RAID + DISK Cache Disable`   | OK OK OK OK    | OK OK         | OK OK OK    | OK OK
| `DISK + Write Cache Enable`   | OK OK OK OK    | OK OK         | OK OK OK    | OK OK
| `DISK + Write Cache Disable`   | OK OK OK OK    | OK OK         | OK OK OK    | OK OK


从测试结果上看，

1. Shutdown 和reboot 总是安全的，这也符合我们的预期，正常shutdown和reboot 也丢失数据，那硬件确实太坑爹了。
2. RAID（BBU ＋ Write Back）只要打开Disk Cache，异常掉电（包括长按关机键和拔电源）丢失数据的概率非常的大，试验了5次，有4次丢失数据。
3. 对于单盘，无论Write Cache是否是Enable，都不会丢失数据，这个行为与操作系统版本是有关系的。

对于第二点，RAID配置，一定要设置Disk Cache Policy为disable，这个已经在[前面文章](https://bean-li.github.io/disk-cache-policy/)介绍过了。

对于第三点，ceph的代码中有如下内容

```
void FileJournal::_check_disk_write_cache() const
{
    ...
    // is our kernel new enough?
    int ver = get_linux_version();
    if (ver == 0) {
      dout(10) << "_check_disk_write_cache: get_linux_version failed" << dendl;
    } else if (ver >= KERNEL_VERSION(2, 6, 33)) {
      dout(20) << "_check_disk_write_cache: disk write cache is on, but your "
	       << "kernel is new enough to handle it correctly. (fn:"
	       << fn << ")" << dendl;
      break;
    }
    ...
}
```

含义很明确了，只要内核版本大于2.6.33 ，fsync,fdatasync 都会flush disk write cache，无需担心异常掉电会引起sync过的数据丢失。
这一点Sage Weil大神已经在[ Re: write cache disabling recommendations for journal and storage disks ?](https://www.spinics.net/lists/ceph-devel/msg06173.html)已经分析过了。

## 尾声

事实上，还有一些工作需要做，比如内核的2.6.33版本如何做到fsync会冲刷 disk write cache，我希望从内核代码层面找到实现方法。虽然手里也攒了一些文章，和这个话题有关系，但是现在还没有研究透彻。

第二是其他硬件，JBOD／外接盘阵等硬件形式，我还没机会测试和研究。事实上，外接盘阵一样有丢失数据的风险，只不过目前我还没有研究透彻，希望后面琢磨透了之后，可以分享心得。




