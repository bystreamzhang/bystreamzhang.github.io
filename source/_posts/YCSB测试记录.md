---
title: YCSB测试记录
date: 2025-02-26 10:00:56
tags:
    - YCSB
    - ZNS
    - FEMU
categories: Study
---

# 说明

记录自己进行YCSB测试的流程和注意事项.

我是为Balloon-ZNS优化项目做YCSB测试,需要在有FEMU模拟的ZNS SSD设备的QEMU虚拟机上做测试,并且YCSB需要支持以特定数据为输入而不是简单指定workload.

YCSB官方文档: [YCSB wiki](https://github.com/brianfrankcooper/YCSB/wiki)

我修改的YCSB-CPP代码仓库: [YCSB-cpp-modification](https://github.com/bystreamzhang/YCSB-cpp-modification)

## 一些基本知识

随时补充

负载有两个阶段: loading和transactions

- loading决定了插入数据内容
- transactions决定了对数据内容执行的操作

测试时用-load和-run指定不同的阶段

根据我对6个workload进行的测试, load阶段就进行一种组合操作:

- InsertSingle + SerializeRow

而run阶段可能进行以下组合操作:

1. UpdateSingle + DeserializeRow + SerializeRow 或者其他顺序, 比如U S D
2. ReadSingle + DeserializeRow
3. ScanSingle + 众多DeserializeRow调用(可能约1:50)
4. InsertSingle + SerializeRow run阶段也可以有插入,与insertproportion参数设置有关,比如workloadd

不同组合穿插出现, 没什么规律.

可能还可以进行其他操作,代码中还有scan, merge, delete等,还得测试

对于workloada,有以下设定:

```properties
recordcount=100000
operationcount=100000

workload=com.yahoo.ycsb.workloads.CoreWorkload

readallfields=true

readproportion=0.5
updateproportion=0.5
scanproportion=0
insertproportion=0

requestdistribution=zipfian
```

这里可以看到操作类型,可能就说明了为什么没有scan操作.

统计来看,中间输出结果中insert操作组合确实刚好进行了100000次.也可以从最终输出结果看到.

对于同一个负载,如果两次运行,分别带有-load和-run参数,他们执行的程序是不同的,程序内的一些变量值会初始化.不过load会往设备写入数据, 我这里配置后应该会往模拟的zns ssd设备nvme0n1写入数据, 然后run阶段会读取这些数据并执行操作.

## 原生femu测试

我打算先学会用ycsb测原生femu，balloon-zns的测试可能会有额外问题。

### 基本测试环境搭建

首先参考[【续】ZNS SSD模拟器进行YCSB测试_编译zenfs报错-CSDN博客](https://blog.csdn.net/yumWant2debug/article/details/136376402) ，使用YSCB-cpp。注意，这里要按博客做几个文件配置否则make会失败。（如果先前make失败了，后续重新make要先make clean）

不过还是有问题：

我现在使用YSCB测试femu模拟的zns ssd，按文档和网上教程搭建了zenfs环境，然后做了如下步骤：

```bash
git clone https://github.com/ls4154/YCSB-cpp.git
cd YCSB-cpp
git submodule update --init
make
```

进行YCSB-cpp的安装，但由于ZenFS非标准[POSIX](https://so.csdn.net/so/search?q=POSIX&spm=1001.2101.3001.7020)接口文件系统，需要修改代码进行文件系统适配。  
修改Makefile如下：

```makefile
# Extra options
DEBUG_BUILD ?=
EXTRA_CXXFLAGS ?= -I/home/spdk/Desktop/rocksdb/include
EXTRA_LDFLAGS ?= -L/home/spdk/Desktop/rocksdb  -lsnappy -lzstd -lbz2 -llz4  -lgflags -u zenfs_filesystem_reg -lzbd
```

接着修改 rocksdb/rocksdb_db.cc文件  
在RocksdbDB::GetOptions定义处加入如下代码

```c
if (!env_uri.empty() || !fs_uri.empty()) {
rocksdb::Status s = rocksdb::Env::CreateFromUri(rocksdb::ConfigOptions(), env_uri, fs_uri, &env, &env_guard);
if (!s.ok()) {
throw utils::Exception(std::string("RocksDB CreateFromUri: ") + s.ToString());
}
//加入这行代码
printf("env is nullptr ? %s %p filesystem %p\n",env ? "no" : "yes",env,env->GetFileSystem().get());
//
opt->env = env;
}
```

此外还要修改rocksdb.properties来指定fs_uri参数，我的文件参考如下

```properties
#rocksdb.dbname=/tmp/ycsb-rocksdb
//这里修改是因为创建ZenFS时会指定目录
rocksdb.dbname=/
rocksdb.format=single
rocksdb.destroy=false
rocksdb.fs_uri=zenfs://dev:nvme4n1
# Load options from file
#rocksdb.optionsfile=

# Below options are ignored if options file is used
rocksdb.compression=snappy
rocksdb.max_background_jobs=2
rocksdb.target_file_size_base=67108864
rocksdb.target_file_size_multiplier=1
rocksdb.max_bytes_for_level_base=268435456
rocksdb.write_buffer_size=67108864
rocksdb.max_open_files=-1
rocksdb.max_write_buffer_number=2
rocksdb.use_direct_io_for_flush_compaction=false
rocksdb.use_direct_reads=false
rocksdb.allow_mmap_writes=false
rocksdb.allow_mmap_reads=false
rocksdb.cache_size=8388608
rocksdb.bloom_bits=0

# deprecated since rocksdb 8.0
rocksdb.compressed_cache_size=0

rocksdb.increase_parallelism=false
rocksdb.optimize_level_style_compaction=false
```

修改完毕后再make. 注意需要提前安装cmake.

然后运行`./ycsb -load -run -db rocksdb -P workloads/workloadb -P rocksdb/rocksdb.properties -s`
报错：Unknown database name rocksdb

这个的解决方法应该是，在makefile中修改BIND_ROCKSDB ?= 1。

然后还有报错：`Caught exception: RocksDB CreateFromUri: Invalid argument: IO error: Invalid argument: Current ZBD scheduler is not mq-deadline, set it to mq-deadline.: zenfs://dev:nvme0n1`

这个要求和rocksdb里面一样，需要运行`echo deadline > /sys/class/block/nvme0n1/queue/scheduler`。**这个每次开终端都要运行**

然后，还有报错：`Caught exception: RocksDB CreateFromUri: Invalid argument: IO error: Not implemented: To few zones on zoned backend (32 required): zenfs://dev:nvme0n1`

说明需要32个zone，和rocksdb一样

还有报错：`Caught exception: RocksDB CreateFromUri: Invalid argument: NotFound: No valid superblock found: zenfs://dev:nvme0n1`

参考之前rocksdb的脚本，可能需要做一下zenfs的mkfs

我自己也写了一个脚本my_test.sh，运行即可。

然后还可能有报错：`Caught exception: RocksDB Open: Invalid argument: Compression type Snappy is not linked with the binary.`

rocksdb的测试脚本是有 `--compression_type=none`的。根据rocksdb_db.cc，要把properties中的rocksdb.compression设置为no，才算取消压缩。另外我也将--use_direct_io_for_flush_and_compaction设置为true了。

### 生成数据并使用指定数据

现在的问题是, 测试时需要以6个不同压缩率的文本文件作为输入. 而YCSB测试时输入文本有特定格式. 我编写了一个将文本文件转换格式的python程序

由于YCSB-CPP没有自带的指定输入文本的参数,我在rocksdb_db.cc文件中自行添加了LoadFromFile函数并做了一些修改.

根据前面的分析, 如果要使用指定数据, 就要用其内容替换load阶段本来要insert的内容. 而run阶段不需要执行我定义的LoadFromFile函数, 可以通过不传递datapath来阻止.

对于run阶段存在insert的情况,我暂时不考虑改动,就插入默认生成的数据吧.

### 错误记录

1. 对于`requestdistribution=zipfian`的几个workload,run阶段的READ都变成READ-FAILED, UPDATE都变成UPDATE-FAILED, 测试结果自然也比较奇怪, 同workload不同测试数据差距可能很大,但是对于没有内置压缩的原生femu不应该这样.

原因分析和解决:



