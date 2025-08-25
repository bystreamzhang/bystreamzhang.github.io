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

对于基本的6个workload, 设定的每个插入数据大小为:`Default data size: 1 KB records (10 fields, 100 bytes each, plus key)`

实际打印可以发现,key是`userxxx`形式,其中xxx部分就是20个数字(也存在19的情况,不知道为什么),合起来key长度是24, 加上10个100bytes的fields就是1KB.

再具体打印fields或者分析将values转为data的`SerializeRow`函数,可以发现每个fields还有一个string类型的name属性, 内容就是`"fieldsX"`, X从0-9.

每个fields的name和其size会被放入data, 而key不会. 并且name和size的长度会以4字节放入data, 结构如下图:

![insert_input and converted data structure](data.drawio.png)

所以data的实际长度不是1024, 而是1000 + 10*6 + 10*4*2 = 1140.

这里的分析意义在于自行设计数据生成程序并"欺骗"YCSB, 使其能正常改用我们指定的数据进行测试而非默认的随机数据.

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
EXTRA_CXXFLAGS ?= -I/home/ROCKSDB_PATH/rocksdb/include
EXTRA_LDFLAGS ?= -L/home/ROCKSDB_PATH/rocksdb  -lsnappy -lzstd -lbz2 -llz4  -lgflags -u zenfs_filesystem_reg -lzbd
```

接着修改YCSB-cpp的 rocksdb/rocksdb_db.cc文件  
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
rocksdb.fs_uri=zenfs://dev:nvme0n1
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

结合前面基本知识处的分析, 我不能简单取消原先的insert函数, 我生成的数据必须是完全符合insert函数内部接受的数据格式的, 更底层的东西我不想改变.

至于key, 本来我生成数据时key是从0开始递增, 但分析rocksdb_db.cc代码后, 我认为insert的key和后续run阶段使用的key是对应的, 所以我们必须使用insert函数本来会使用的key.

每次insert的key都是独一无二的, 所以load阶段我不需要考虑insert接受的key重复的问题. 我可以直接按顺序来, 第一个接受的key在我这里对应1, 然后取出指定数据中的第一个1000B作为value.
而对于run阶段的insert, 输入的key基本也都是独一无二的, 可能存在某个insert后的key被delete然后又insert的情况, 但应该比较少.
总的来说, 我的想法是, load和run阶段分别有一个insert计数器, 每次insert时根据计数器取出输入数据对应位置的1000B数据, 其他就还是和原insert函数做法一致. 所以key我是不动的, 只是替换value.

这样的一个好处是, 事实上我根本不需要做数据转换了, 原生的文本文件就够用, 每次取1000B就行了. 不过如果完全用原生数据, 程序内部要每次取10次100B来构成data, 这个额外时间可能影响测试结果, 所以这个转换成1140B的步骤就直接在外部的`convert_to_ycsb.py`做了, 也不会很复杂, 按data结构生成就行.

这个预处理最麻烦的是data结构中两个length部分, 因为是实际占了4个字节, 源码中用了一种奇怪的方式, 本质上就是把数字对应的4个字节存到字符数组里并且顺序是小端序, 低字节在前. 比如长度是100的时候, 是 64 00 00 00, 如果单字节打印, 很可能后面3个字节是显示不了的异常字节(并且打印字符串时遇到它们会直接视为打印结束,类似换行符), 第一个字节对应的字符是d所以会显示一个d.

python中要实现这个效果, 也需要一些额外操作, 具体见代码.

我的这个数据生成和使用的总思路存在几个可能问题:1. 输入文件大小如果不够怎么办? 那就取模.如果大小过大, 超出部分就用不到. 2. run阶段的insert也会重头开始取输入数据, 这可能和实际测试存在差异, 本来这个insert结果不应该和load阶段的insert相同. 不过这种相同应该对测试效果无影响, 毕竟数据本身就是我们决定的, 插入什么也是我们定的. 这个问题其实也可以改正不过先就这样. 3. 我预处理出data, 相对来说可能还使insert时需要的时间变短了, 不过也有额外读取文件的耗时, 感觉影响应该不大.

### 切换到新的rocksdb和zenfs版本(并且也得能切换回去)

之前做FlexZNS和CCZNS测试时琢磨的，不一定正确。

假设上面的已经成功, 目前有新的rocksdb和zenfs文件夹.

首先需要把zenfs配置到rocksdb, 然后编译rocksdb, 参考<https://github.com/westerndigitalcorporation/zenfs>

应该可以为不同版本rocksdb设定不同的路径, make时修改`PREFIX`指定路径而不是都安装到`user/local`. 测试时选择不同的路径即可选定不同的测试版本.

具体来说(下面针对FlexZNS, 其他ZNS如果也修改了rocksdb和zenfs, 大体流程应该相同但是细节可能不同):

- 将zenfs代码根目录放到rocksdb的plugin文件夹
- 在rocksdb的代码根目录, 修改Makefile:

```Makefile
#PREFIX ?= /usr/local
PREFIX ?= /home/zwl/FlexZNS-ICCD23-EA/Softwares/rocksdb/base
```

编译安装rocksdb:

```bash
DEBUG_LEVEL=0 ROCKSDB_PLUGINS=zenfs USE_RTTI=1 LDFLAGS="-lz" make -j$(nproc) librocksdb.a
```

如果遇到报错:`plugin/zenfs/zenfs.mk:37: *** Generating ZenFS version failed.  Stop.`则在zenfs文件夹中, 将`generate-version.sh`中的`VERSION`自定义为一个定值.

在`plugin/zenfs/util`文件夹也做`make`.

然后修改下面的文件:

- YCSB-cpp的Makefile: `EXTRA_CXXFLAGS`和`EXTRA_LDFLAGS`改为新rocksdb的对应路径. 并且最好加上`-lz`(为了避免后面链接不到对应的库, 不知为何我不加会找不到某些函数)
- 测试脚本my_test.sh中的zenfs路径要改为指向新rocksdb中的plugin中的zenfs

然后在YCSB-cpp文件夹做make. 注意先做`make clean`.

注意: rocksdb的make可能比较耗时, 并且生成内容比较大(几个G), 所以注意存储空间. 后续YCSB-cpp编译出ycsb执行程序后, rocksdb可以执行`make clean`来释放空间.

这里可以编译出ycsb, 但是测试可能出现`Caught exception: RocksDB CreateFromUri: Invalid argument: Corruption: ZenFS Superblock: Error: block size missmatch: zenfs://dev:nvme0n1`的报错,
分析认为应该是FlexZNS对rocksdb或zenfs的修改导致. 因为这个流程对于旧rocksdb是成立的.

