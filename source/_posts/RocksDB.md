---
title: RocksDB
date: 2025-11-05 21:13:23
tags:
    - RocksDB
    - SSD
categories: Study
---

# 说明

## 编译

我在ZNS SSD模拟器上，本机docker镜像中都做过rocksdb配置。

编译其实很简单，因为官方有说明，网上也很多参考。

具体就是，clone RocksDB仓库，在其中的[INSTALL.md](https://github.com/facebook/rocksdb/blob/main/INSTALL.md)，
可以看到关于编译和依赖项的介绍。
在examples文件夹还有一些写好的样例可以参考。

对于docker场景，下面的DockerFile就包含了RocksDB动态库的安装：

```Dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

# 环境配置基于LearnedKV文档

RUN apt-get update && apt-get install -y sudo libhiredis-dev \
build-essential git cmake pkg-config curl wget unzip python3 python3-pip \
libtbb-dev libsnappy-dev libbz2-dev liblz4-dev libzstd-dev \
libleveldb-dev libgmp-dev libeigen3-dev \
libaio-dev uuid-dev zlib1g-dev \
&& rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir numpy pandas matplotlib seaborn

# 安装RocksDB (在/opt文件夹)

WORKDIR /opt
RUN git clone https://github.com/facebook/rocksdb.git && \
cd rocksdb && \
git fetch --all --tags && \
git checkout v9.3.1 && \
make -j"$(nproc)" shared_lib && \
    make install -j"$(nproc)"

# 运行时能找到动态库

ENV LD_LIBRARY_PATH=/usr/local/lib

# 将 /usr/local/lib 加入动态链接缓存（可选但更稳妥）

RUN echo "/usr/local/lib" > /etc/ld.so.conf.d/local.conf && ldconfig

# 测试工具如db_bench相关配置(如果不使用，可注释)

WORKDIR /opt/rocksdb
RUN apt-get update && \
    apt-get install -y libgflags-dev locales && \
    locale-gen en_US.UTF-8 && \
    update-locale LANG=en_US.UTF-8 && \
    export LANG=en_US.UTF-8
RUN make clean
RUN CXXFLAGS="-fPIC" make db_bench -j"$(nproc)"

# 工作目录

WORKDIR /work

RUN echo "RocksDB在/opt文件夹"
```

RocksDB自带测试工具db_bench的编译参考了[db_bench教程](https://www.codeleading.com/article/16653770496/)
-fPIC参数不加有问题，问ai后加的，不清楚具体为什么

## ZNS相关测试

现在要在ZNS SSD上做RocksDB相关的测试，做些记录。

需要做下面的测试：

1. 逐层搜SSD的开销.最坏情况下，最长延迟，平均延迟99.9%尾延迟；
2. 单个SST中，检索具体kv对的开销；
3. 模型训练时间中，用新数据训练的时间和垃圾回收训练时间。

旧的测试脚本：

```sh
#!/bin/bash

rm -rf /home/zwl/dbtest

echo deadline > /sys/class/block/nvme0n1/queue/scheduler

./plugin/zenfs/util/zenfs mkfs --zbd=nvme0n1 --aux_path=/home/zwl/dbtest

./db_bench --fs_uri=zenfs://dev:nvme0n1 --benchmarks="fillseq,stats,sstables,levelstats" --num=4000000 --value_size=800 --key_size=16 --min_write_buffer_number_to_merge=2 --max_write_buffer_number=6 --write_buffer_size=67108864 --target_file_size_base=67108864 --max_background_flushes=4 --use_direct_io_for_flush_and_compaction --compression_type=none > dbtest_output.txt 2>&1

```

部分测试结果：

```bash
RocksDB:    version 8.9.2
Date:       Wed Nov  5 13:19:19 2025
CPU:        4 * Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz
CPUCache:   16384 KB
Set seed to 1762348759367326 because --seed was 0
Initializing RocksDB Options from the specified file
Initializing RocksDB Options from command-line flags
Integrated BlobDB: blob cache disabled
Keys:       16 bytes each (+ 0 bytes user-defined timestamp)
Values:     800 bytes each (400 bytes after compression)
Entries:    2000000
Prefix:    0 bytes
Keys per prefix:    0
RawSize:    1556.4 MB (estimated)
FileSize:   793.5 MB (estimated)
Write rate: 0 bytes/second
Read rate: 0 ops/second
Compression: NoCompression
Compression sampling rate: 0
Memtablerep: SkipListFactory
Perf Level: 1
------------------------------------------------
Initializing RocksDB Options from the specified file
Initializing RocksDB Options from command-line flags
Integrated BlobDB: blob cache disabled
DB path: [rocksdbtest/dbbench]
fillseq      :      20.732 micros/op 48233 ops/sec 41.465 seconds 2000000 operations;   37.5 MB/s


** Compaction Stats [default] **
Level    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) CompMergeCPU(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop Rblob(GB) Wblob(GB)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  L0      2/0   248.88 MB   1.0      0.0     0.0      0.0       1.3      1.3       0.0   1.0      0.0     35.8     38.26              2.74        11    3.479       0      0       0.0       0.0
  L1      1/0   124.44 MB   0.5      0.0     0.0      0.0       0.0      0.0       1.1   0.0      0.0      0.0      0.00              0.00         0    0.000       0      0       0.0       0.0
  L2      8/0   995.53 MB   0.4      0.0     0.0      0.0       0.0      0.0       1.0   0.0      0.0      0.0      0.00              0.00         0    0.000       0      0       0.0       0.0
 Sum     11/0    1.34 GB   0.0      0.0     0.0      0.0       1.3      1.3       2.1   1.0      0.0     35.8     38.26              2.74        11    3.479       0      0       0.0       0.0
 Int      0/0    0.00 KB   0.0      0.0     0.0      0.0       1.3      1.3       2.1   1.0      0.0     35.8     38.26              2.74        11    3.479       0      0       0.0       0.0

** Compaction Stats [default] **
Priority    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) CompMergeCPU(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop Rblob(GB) Wblob(GB)
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
High      0/0    0.00 KB   0.0      0.0     0.0      0.0       1.3      1.3       0.0   0.0      0.0     35.8     38.26              2.74        11    3.479       0      0       0.0       0.0

Blob file count: 0, total size: 0.0 GB, garbage size: 0.0 GB, space amp: 0.0

Uptime(secs): 41.5 total, 38.5 interval
Flush(GB): cumulative 1.337, interval 1.337
AddFile(GB): cumulative 0.000, interval 0.000
AddFile(Total Files): cumulative 0, interval 0
AddFile(L0 Files): cumulative 0, interval 0
AddFile(Keys): cumulative 0, interval 0
Cumulative compaction: 1.34 GB write, 32.99 MB/s write, 0.00 GB read, 0.00 MB/s read, 38.3 seconds
Interval compaction: 1.34 GB write, 35.59 MB/s write, 0.00 GB read, 0.00 MB/s read, 38.3 seconds
Write Stall (count): cf-l0-file-count-limit-delays-with-ongoing-compaction: 0, cf-l0-file-count-limit-stops-with-ongoing-compaction: 0, l0-file-count-limit-delays: 0, l0-file-count-limit-stops: 0, memtable-limit-delays: 0, memtable-limit-stops: 0, pending-compaction-bytes-delays: 0, pending-compaction-bytes-stops: 0, total-delays: 0, total-stops: 0
Block cache LRUCache@0x55cdce1943c0#3300 capacity: 32.00 MB seed: 1271250590 usage: 0.09 KB table_size: 1024 occupancy: 1 collections: 1 last_copies: 1 last_secs: 0.000178 secs_since: 41
Block cache entry stats(count,size,portion): Misc(1,0.00 KB,0%)

** File Read Latency Histogram By Level [default] **

** DB Stats **
Uptime(secs): 41.5 total, 38.5 interval
Cumulative writes: 2000K writes, 2000K keys, 2000K commit groups, 1.0 writes per commit group, ingest: 1.55 GB, 38.24 MB/s
Cumulative WAL: 2000K writes, 0 syncs, 2000000.00 writes per sync, written: 1.55 GB, 38.24 MB/s
Cumulative stall: 00:00:0.000 H:M:S, 0.0 percent
Interval writes: 1822K writes, 1822K keys, 1822K commit groups, 1.0 writes per commit group, ingest: 1446.16 MB, 37.60 MB/s
Interval WAL: 1822K writes, 0 syncs, 1822608.00 writes per sync, written: 1.41 GB, 37.60 MB/s
Interval stall: 00:00:0.000 H:M:S, 0.0 percent
Write Stall (count): write-buffer-manager-limit-stops: 0


--- level 0 --- version# 20 ---
 40:130478850[1574006 .. 1731397]['00000000001804753030303030303030' seq:1574006, type:1 .. '00000000001A6B443030303030303030' seq:1731397, type:1](0)
 37:130486320[1416605 .. 1574005]['0000000000159D9C3030303030303030' seq:1416605, type:1 .. '00000000001804743030303030303030' seq:1574005, type:1](0)
--- level 1 --- version# 20 ---
 34:130483816[1259207 .. 1416604]['00000000001336C63030303030303030' seq:1259207, type:1 .. '0000000000159D9B3030303030303030' seq:1416604, type:1](0)
--- level 2 --- version# 20 ---
 10:130485461[1 .. 157400]['00000000000000003030303030303030' seq:1, type:1 .. '00000000000266D73030303030303030' seq:157400, type:1](0)
 13:130493750[157401 .. 314810]['00000000000266D83030303030303030' seq:157401, type:1 .. '000000000004CDB93030303030303030' seq:314810, type:1](0)
 16:130487962[314811 .. 472213]['000000000004CDBA3030303030303030' seq:314811, type:1 .. '00000000000734943030303030303030' seq:472213, type:1](0)
 19:130487140[472214 .. 629615]['00000000000734953030303030303030' seq:472214, type:1 .. '0000000000099B6E3030303030303030' seq:629615, type:1](0)
 22:130476348[629616 .. 787004]['0000000000099B6F3030303030303030' seq:629616, type:1 .. '00000000000C023B3030303030303030' seq:787004, type:1](0)
 25:130483816[787005 .. 944402]['00000000000C023C3030303030303030' seq:787005, type:1 .. '00000000000E69113030303030303030' seq:944402, type:1](0)
 28:130483817[944403 .. 1101800]['00000000000E69123030303030303030' seq:944403, type:1 .. '000000000010CFE73030303030303030' seq:1101800, type:1](0)
 31:130490467[1101801 .. 1259206]['000000000010CFE83030303030303030' seq:1101801, type:1 .. '00000000001336C53030303030303030' seq:1259206, type:1](0)
--- level 3 --- version# 20 ---
--- level 4 --- version# 20 ---
--- level 5 --- version# 20 ---
--- level 6 --- version# 20 ---


Level Files Size(MB)
--------------------
  0        2      249
  1        1      124
  2        8      996
  3        0        0
  4        0        0
  5        0        0
  6        0        0
```

如果把entry数量乘2，发生如下改变：

```bash
** Compaction Stats [default] **
Level    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) CompMergeCPU(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop Rblob(GB) Wblob(GB)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  L0      0/0    0.00 KB   0.0      0.0     0.0      0.0       2.9      2.9       0.0   1.0      0.0     38.4     77.71              5.98        24    3.238       0      0       0.0       0.0
  L2     20/0    2.43 GB   1.0      0.0     0.0      0.0       0.0      0.0       2.9   0.0      0.0      0.0      0.00              0.00         0    0.000       0      0       0.0       0.0
  L3      4/0   497.74 MB   0.0      0.0     0.0      0.0       0.0      0.0       0.5   0.0      0.0      0.0      0.00              0.00         0    0.000       0      0       0.0       0.0
 Sum     24/0    2.92 GB   0.0      0.0     0.0      0.0       2.9      2.9       3.4   1.0      0.0     38.4     77.71              5.98        24    3.238       0      0       0.0       0.0
 Int      0/0    0.00 KB   0.0      0.0     0.0      0.0       2.9      2.9       3.4   1.0      0.0     38.4     77.71              5.98        24    3.238       0      0       0.0       0.0

** Compaction Stats [default] **
Priority    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) CompMergeCPU(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop Rblob(GB) Wblob(GB)
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
High      0/0    0.00 KB   0.0      0.0     0.0      0.0       2.9      2.9       0.0   0.0      0.0     38.4     77.71              5.98        24    3.238       0      0       0.0       0.0

Blob file count: 0, total size: 0.0 GB, garbage size: 0.0 GB, space amp: 0.0

Uptime(secs): 80.2 total, 77.1 interval
Flush(GB): cumulative 2.917, interval 2.917
AddFile(GB): cumulative 0.000, interval 0.000
AddFile(Total Files): cumulative 0, interval 0
AddFile(L0 Files): cumulative 0, interval 0
AddFile(Keys): cumulative 0, interval 0
Cumulative compaction: 2.92 GB write, 37.26 MB/s write, 0.00 GB read, 0.00 MB/s read, 77.7 seconds
Interval compaction: 2.92 GB write, 38.72 MB/s write, 0.00 GB read, 0.00 MB/s read, 77.7 seconds
Write Stall (count): cf-l0-file-count-limit-delays-with-ongoing-compaction: 0, cf-l0-file-count-limit-stops-with-ongoing-compaction: 0, l0-file-count-limit-delays: 0, l0-file-count-limit-stops: 0, memtable-limit-delays: 1, memtable-limit-stops: 0, pending-compaction-bytes-delays: 0, pending-compaction-bytes-stops: 0, total-delays: 1, total-stops: 0
interval: 1 total count
Block cache LRUCache@0x564de5fad3c0#3725 capacity: 32.00 MB seed: 374113911 usage: 0.09 KB table_size: 1024 occupancy: 1 collections: 1 last_copies: 1 last_secs: 0.000165 secs_since: 80
Block cache entry stats(count,size,portion): Misc(1,0.00 KB,0%)

** File Read Latency Histogram By Level [default] **

** DB Stats **
Uptime(secs): 80.2 total, 77.1 interval
Cumulative writes: 4000K writes, 4000K keys, 4000K commit groups, 1.0 writes per commit group, ingest: 3.10 GB, 39.59 MB/s
Cumulative WAL: 4000K writes, 0 syncs, 4000000.00 writes per sync, written: 3.10 GB, 39.59 MB/s
Cumulative stall: 00:00:0.005 H:M:S, 0.0 percent
Interval writes: 3811K writes, 3811K keys, 3811K commit groups, 1.0 writes per commit group, ingest: 3024.15 MB, 39.21 MB/s
Interval WAL: 3811K writes, 0 syncs, 3811365.00 writes per sync, written: 2.95 GB, 39.21 MB/s
Interval stall: 00:00:0.005 H:M:S, 0.0 percent
Write Stall (count): write-buffer-manager-limit-stops: 0


--- level 0 --- version# 44 ---
--- level 1 --- version# 44 ---
--- level 2 --- version# 44 ---
 22:130486320[629577 .. 786977]['0000000000099B483030303030303030' seq:629577, type:1 .. '00000000000C02203030303030303030' seq:786977, type:1](0)
 25:130480493[786978 .. 944371]['00000000000C02213030303030303030' seq:786978, type:1 .. '00000000000E68F23030303030303030' seq:944371, type:1](0)
 28:130482995[944372 .. 1101768]['00000000000E68F33030303030303030' seq:944372, type:1 .. '000000000010CFC73030303030303030' seq:1101768, type:1](0)
 31:130492928[1101769 .. 1259177]['000000000010CFC83030303030303030' seq:1101769, type:1 .. '00000000001336A83030303030303030' seq:1259177, type:1](0)
 34:130482176[1259178 .. 1416573]['00000000001336A93030303030303030' seq:1259178, type:1 .. '0000000000159D7C3030303030303030' seq:1416573, type:1](0)
 37:130478851[1416574 .. 1573965]['0000000000159D7D3030303030303030' seq:1416574, type:1 .. '000000000018044C3030303030303030' seq:1573965, type:1](0)
 40:130487961[1573966 .. 1731368]['000000000018044D3030303030303030' seq:1573966, type:1 .. '00000000001A6B273030303030303030' seq:1731368, type:1](0)
 43:130483817[1731369 .. 1888766]['00000000001A6B283030303030303030' seq:1731369, type:1 .. '00000000001CD1FD3030303030303030' seq:1888766, type:1](0)
 46:130485459[1888767 .. 2046166]['00000000001CD1FE3030303030303030' seq:1888767, type:1 .. '00000000001F38D53030303030303030' seq:2046166, type:1](0)
 49:130485460[2046167 .. 2203566]['00000000001F38D63030303030303030' seq:2046167, type:1 .. '0000000000219FAD3030303030303030' seq:2203566, type:1](0)
 52:130485459[2203567 .. 2360966]['0000000000219FAE3030303030303030' seq:2203567, type:1 .. '00000000002406853030303030303030' seq:2360966, type:1](0)
 55:130488784[2360967 .. 2518370]['00000000002406863030303030303030' seq:2360967, type:1 .. '0000000000266D613030303030303030' seq:2518370, type:1](0)
 58:130491285[2518371 .. 2675777]['0000000000266D623030303030303030' seq:2518371, type:1 .. '000000000028D4403030303030303030' seq:2675777, type:1](0)
 61:130480493[2675778 .. 2833171]['000000000028D4413030303030303030' seq:2675778, type:1 .. '00000000002B3B123030303030303030' seq:2833171, type:1](0)
 64:130478030[2833172 .. 2990562]['00000000002B3B133030303030303030' seq:2833172, type:1 .. '00000000002DA1E13030303030303030' seq:2990562, type:1](0)
 67:130492928[2990563 .. 3147971]['00000000002DA1E23030303030303030' seq:2990563, type:1 .. '00000000003008C23030303030303030' seq:3147971, type:1](0)
 70:130483818[3147972 .. 3305369]['00000000003008C33030303030303030' seq:3147972, type:1 .. '0000000000326F983030303030303030' seq:3305369, type:1](0)
 73:130488784[3305370 .. 3462773]['0000000000326F993030303030303030' seq:3305370, type:1 .. '000000000034D6743030303030303030' seq:3462773, type:1](0)
 76:130490465[3462774 .. 3620179]['000000000034D6753030303030303030' seq:3462774, type:1 .. '0000000000373D523030303030303030' seq:3620179, type:1](0)
 79:130488784[3620180 .. 3777583]['0000000000373D533030303030303030' seq:3620180, type:1 .. '000000000039A42E3030303030303030' seq:3777583, type:1](0)
--- level 3 --- version# 44 ---
 10:130483819[1 .. 157398]['00000000000000003030303030303030' seq:1, type:1 .. '00000000000266D53030303030303030' seq:157398, type:1](0)
 13:130480492[157399 .. 314792]['00000000000266D63030303030303030' seq:157399, type:1 .. '000000000004CDA73030303030303030' seq:314792, type:1](0)
 16:130478030[314793 .. 472183]['000000000004CDA83030303030303030' seq:314793, type:1 .. '00000000000734763030303030303030' seq:472183, type:1](0)
 19:130479671[472184 .. 629576]['00000000000734773030303030303030' seq:472184, type:1 .. '0000000000099B473030303030303030' seq:629576, type:1](0)
--- level 4 --- version# 44 ---
--- level 5 --- version# 44 ---
--- level 6 --- version# 44 ---


Level Files Size(MB)
--------------------
  0        0        0
  1        0        0
  2       20     2489
  3        4      498
  4        0        0
  5        0        0
  6        0        0
```

我在服务器本机也做了测试(一般的SSD。在docker中的rocksdb测试)。

```sh
#!/bin/bash
set -euo pipefail
DB_DIR=${DB_DIR:-/opt/rocksdb/ZNBC/dbtest}
WAL_DIR=${WAL_DIR:-/opt/rocksdb/ZNBC/dbtest/wal}
mkdir -p "$DB_DIR" "$WAL_DIR"
./db_bench \
  --db="$DB_DIR" \
  --wal_dir="$WAL_DIR" \
  --benchmarks="fillseq,stats,sstables,levelstats" \
  --num=2000000 \
  --value_size=800 \
  --key_size=16 \
  --min_write_buffer_number_to_merge=2 \
  --max_write_buffer_number=6 \
  --write_buffer_size=67108864 \
  --target_file_size_base=67108864 \
  --max_background_flushes=4 \
  --use_direct_io_for_flush_and_compaction \
  --compression_type=none \
  > dbtest_output.txt 2>&1
```

## 代码分析和修改
