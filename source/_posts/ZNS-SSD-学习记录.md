---
title: ZNS SSD 学习记录
date: 2025-02-27 22:55:46
tags:
    - ZNS
    - SSD
    - FlexZNS
categories: Study
---

# 说明

记录一些基本知识和学习心得. 目前可能比较零散化, 不一定有前后完整的逻辑, 觉得有必要的时候会进行系统整理, 届时逻辑会完整, 不会突然出现新概念

## 论文

ZNS SSD核心论文: [ZNS: Avoiding the Block Interface Tax for Flash-based SSDs](https://www.usenix.org/conference/atc21/presentation/bjorling) 

并行性相关:

- [What you cant forget: Exploiting parallelism for zoned namespaces](https://yonsei.elsevierpure.com/en/publications/what-you-cant-forget-exploiting-parallelism-for-zoned-namespaces#:~:text=This%20paper%20discusses%20the%20main%20benefits%20of%20ZNS,internal%20parallelism%20when%20downsizing%20its%20zone%20writable%20capacity.)
- [Accelerating RocksDB for small-zone ZNS SSDs by parallel I/O mechanism](https://dl.acm.org/doi/abs/10.1145/3564695.3564774#:~:text=In%20this%20paper%2C%20we%20propose%20two%20parallel%20I%2FO,distributing%20I%2FO%20data%20to%20multiple%20zones%20in%20parallel.)

FlexZNS:

- [FlexZNS: Building High-Performance ZNS SSDs with Size-Flexible and Parity-Protected Zones](https://ieeexplore.ieee.org/document/10361036)

这些论文的背景部分都很值得看, 可以帮助了解和巩固基础知识.



## zone

### zone 状态

每个Zone有以下基本状态：

- EMPTY（空状态）
- OPEN（开放状态）
- CLOSED（关闭状态）
- FULL（满状态）

Zone初始处于EMPTY状态，通过写入操作过渡到OPEN状态，最终在完全写满后进入FULL状态。

设备一般会有一个**开放区域限制**(open zone limit), 限制总共处于OPEN状态的zone数量. 如果达到此限制时主机尝试写入新zone, 则必须将另一个OPEN状态zone转为CLOSED状态.

转为CLOSED状态会实际释放一些该zone本来占据的设备资源, 比如写缓存, 这样新zone才有的用, 开放区域限制才有意义.

zone转为CLOSED时其本身数据不受影响. CLOSED状态的Zone仍可写入，但必须重新转为OPEN状态才能继续写入(就从写缓存角度, 自己关闭时写缓存释放掉了自然没法再写入)

FULL状态的zone同样不能写入, 应该也是不占据设备资源(如写缓存)的, 所以不会计入开放区域限制. 进入FULL状态可能也会释放一些设备资源.
不过FULL状态和CLOSED状态的zone肯定还是需要维护一些元数据, 比如LBA映射. 毕竟还可以读取.

写入指针（Write Pointer）指向Zone内下一个可写入的LBA（逻辑块地址），仅在EMPTY和OPEN状态有效。每次成功写入后，指针会更新。若主机发出的写入命令：

1. 未从当前写入指针位置开始
2. 尝试写入FULL状态的Zone

则写入操作将失败.

通过重置Zone命令（Reset Zone），Zone会回到EMPTY状态，写入指针重置为首个LBA，原数据不可访问。

总结:

- 状态切换由写入操作、资源限制、重置命令触发
- 严格顺序写入是ZNS的核心特性，确保高性能和低延迟（避免随机写入开销）
- 开放区域限制需主机合理管理Zone的打开/关闭，避免资源争用
- 重置操作是回收Zone的唯一方式，需谨慎使用（可能影响数据持久性）

我自制的zone状态图如下:

![zone state machine](zone_state_machine.drawio.png)

### zone的update

zone不支持随机写入, 那么如果要更新zone的某部分数据应该怎么办? 难道只能清空zone并重写其所有数据吗?
这当然是一种方案, 不过由于zone可能非常大(比如几个GB), 这样做导致大量的写入放大.

当然有更好的方案. 一种简单的想法就是, 顺序写入新内容, 更新L&P映射表(将要更新的逻辑块(LBA)的映射改变到新内容的物理地址),
旧数据所在的物理块标记为无效, 后续通过zone reset回收. 这样就无需擦除整个zone.

### 超级块

超级块是ZNS SSD引入的设计.

在FlexZNS论文的引言里用了FBG(flash block group)的说法, 更好理解, 就是一群闪存块的组.
本质上和超级块是一个东西, 不过FlexZNS论文里偏向于用FBG表示一个组的概念而超级块是一个整体, 称zone映射到超级块.

FBG是空间分配和垃圾回收的基本单元.

为了利用并行性和增强可靠性，每个FBG包含跨许多die的具有相同偏移的闪存块(就可以并行)，并构成具有奇偶校验保护的die级RAID条带.
(RAID条带（strip）是把连续的数据分割成相同大小的数据块，把每段数据分别写入到阵列中的不同磁盘上的方法)

目前的ZNS SSD产品通常将一个区域映射到这样的FBG上，因此区域大小达到GB大小，例如西部数据ZN540 SSD为2GB，浪潮NS8600 SSD为1.44GB，
FTL只需要维护一个区域级映射表，其大小为每1TB存储容量几个KB.

从主机端的角度来看，ZNS接口将GC责任从SSD FTL转移到主机软件。

zone是空闲空间分配和GC的基本单位。这和超级块FBG是一样的. 对于ZNS SSD, 就可以认为zone和FBG是一一对应的, 只是后者偏向物理层面, 前者是逻辑层面.

主机软件能够控制zone上的数据放置，其中可以利用丰富的主机端语义来减少区域GC开销，
最近的研究表明，这种软件可定义的能力可以显著提高存储性能和WA/RocksDB（一种流行的键值存储）应用场景中的生命周期



### 并行性



## FlexZNS

FlexZNS主要是指出较大的区域的一些缺点, 以及小区域的一些优点, 然后主张应扩展当前NVMe ZNS协议支持大小可配置区域.

论文代码github: [FlexZNS-ICCD23-EA](https://github.com/ywang-wnlo/FlexZNS-ICCD23-EA)

补充一句, 这个仓库用的是子模块加patch, 直接clone下来子模块文件夹是没内容的, 需要`git submodule update --init --force FEMU/base`,
然后将补丁移动到base文件夹并用`git apply FlexZNS.patch`打补丁才能看到实际代码以及历史提交记录.
FlexZNS不仅修改了FEMU的代码, 还修改了内核代码以及zenfs, rocksdb, f2fs这些测试用工具环境的代码, 来实现其功能. 都是子模块加补丁的形式.
如果我没理解错, zenfs, rocksdb, f2fs这些测试用工具环境都是需要在虚拟机(如qemu)上配置运行, 只是都统一放到一个github仓库里了.

### 问题背景

大区域在哪些地方不如小区域?

ZNS SSD并没有消除垃圾回收, 而是把责任从FTL转移给主机软件.

在论文环境中, 当空闲存储空间耗尽时, F2FS或ZenFS将触发区域GC操作. 迁移将牺牲的区域中的有效数据页面, 然后将区域重置为空闲

为了减少GC开销，通常将具有不同生命周期（或写热度）的数据存储在不同的GC单元中。
然而，较大的区域大小会妨碍准确的数据分离，并导致区域GC期间大量的有效数据迁移。

此外, 较大的区域需要更多硬件资源, 则可以同时打开的区域数量(OPEN_ZONE_LIMIT)会更少, 这会减少写并发性和多租户共享能力.

其次, 在多租户场景中, 使用小区域有助于租户之间的性能隔离.
由于小区域映射到的FBG会跨越更少的闪存die, 不同租户的区域就能被放置到单独的闪存die中来实现硬件隔离.

如果能支持大小可配置区域, 能提供更好的软件可定义能力, 可以扩大应用场景.
例如, 应用程序可以选择小区域来存储一小组写热数据以减少GC开销, 而大区域可以用于写冷数据;
在一些常见的场景中, 多个应用程序共享同一个ZNS SSD, 它们可以根据他们自己的数据访问模式为各自的区域配置不同的大小.

但这样做是有挑战的, 大概有如下方面:

- zone越多, RAID保护(比如奇偶校验存储)实现起来开销可能越大或者说实现形式会需要改变
- 如果允许应用程序配置可变区域大小, 应用程序自己管理存储空间也会变得困难, 因为每个zone的用户可写容量会随区域大小配置而改变, 如下面表格所示

| RAID Stripe | Zone size | Zone capacity | Utilization |
| ----------- | --------- | ------------- | ----------- |
| 63+1p       | 2048 MB   | 2016 MB       | 98.44%      |
| 31+1p       | 1024 MB   | 992 MB        | 96.88%      |
| 15+1p       | 512 MB    | 480 MB        | 93.75%      |
| 7+1p        | 256 MB    | 224 MB        | 87.50%      |
| 3+1p        | 128 MB    | 96 MB         | 75%         |

也可以看下图:

![Mapping of zones in different size configurations.](zones_in_diff_sizes.drawio.png)

如果把zone空间设置太小, 也未必有利于减少GC开销, 虽然GC时数据迁移开销会减少, 但GC频率可能增加, 每次GC释放的空闲空间量会减少.

关于不同zone大小下的性能, GC在写放大中的占比, FlexZNS论文有做测试.
结果显示, 除了不存在GC的seq-read等数据集是zone越大越快, 大部分数据集下, zone大小是过大过小都会使性能变慢
GC也确实是写放大的主要贡献者, zone过小或者过大时GC开销都会增加

### 解决方案

FlexZNS的核心思路是, 要保证用户体验, 用户在设置可变区域大小后自己不需要额外做管理.
看到zone多大就是多大, 不会因为设置而改变. 即要隐藏zone容量损失, 或者说用户不需要知道容量损失.

具体来说, 把区域内的奇偶检验存储分离出去. 将闪存存储分为数据区(data area)和奇偶校验区(parity area).

将zone映射到的块的数量缩减, 称每个zone映射到的块的集合为超级块(也称为该zone的FBG), 这些超级块位于data area.
根据FlexZNS论文的一些描述, 该论文中每个超级块是不可再分的, 就是包含所有die的一个同编号的块, 不过zone和FBG是可以小于超级块的.

每个zone的奇偶校验数据存储在专门的奇偶校验超级块(parity superblocks)中, 位于一个没有和zone的FBG重叠的die.

大的zone可以和小zone并存, 达成一种平衡.

FlexZNS的结构图如下:

![Design overview of FlexZNS](FlexZNS.png)

为了保证可靠性，每个zone被映射到一个跨越多个die的闪存块组（FBG），并由die级RAID奇偶校验保护.

#### zone 容量管理

下面用W表示FlexZNS做的工作, E表示解释, S表示样例.

W: FlexZNS为主机软件提供了一个API，用于通过自定义的NVMe命令（命名为zone format）配置分区大小（zone size list，zone num list）

W: 基于区域大小配置，FlexZNS计算容纳区域的总奇偶校验所需的闪存存储量

E: 奇偶检验存储量的计算方式, 公式我就不记录了, 知道以下几点就好:

- 每个奇偶检验块的大小是一样的;
- 每个zone都有一个奇偶校验块, 且如果大小等于超级块, 则有两个奇偶校验块

W: 如果计算后发现用户配置会导致数据区和奇偶校验区的总大小超过闪存的物理容量, 则报错并要求用户干预.

S: 
用论文的例子. 假设具有1TB物理容量的SSD被格式化为大小为512MB和2GB的两组区域。每组分别具有64和480个区域. 奇偶校验块大小为32MB.
则数据区(data area)大小为512MB × 64+2GB × 480 = 992GB，奇偶校验区(parity area)大小可以直接计算, 为32MB（480+64+480）= 32GB。

E: 减少奇偶检验的一种简单方法是构建RAID条带, 在一个超级块内跨越多个zone, 这样奇偶校验可以被这些zone共享.
不过这样一旦条带上的zone被重置并且写入新数据, 闪存上的奇偶校验块就必须被更新, 会导致更高的性能和WA开销.
所以FlexZNS不采用这种方法, 他们认为当配置小区域时, 奇偶校验存储开销可以得到补偿, 并且总存储成本可以降低.

#### zone映射

W: FlexZNS提出一种新的zone映射机制去分别管理数据和奇偶校验.

如FlexZNS的结构图所示, zone映射到数据区域中的FBG, 而其奇偶校验块位于奇偶校验区.

FlexZNS的FTL维护zone和FBG及其奇偶校验块之间的映射表. 这个表平时放在DRAM里, 会被周期性地存入闪存, 在关机等特殊情况也会写入闪存 (DRAM是易失性存储, 闪存是非易失).

S: 对于1TB的SSD, 如果全部分别由512MB或1GB或2GB大小的zone组成, 则表大小为32KB, 16KB, 或12KB(假设超级块大小为2GB, 因此2GB的zone需要两个奇偶校验块).

W:
数据映射: FlexZNS将zone映射到闪存上的一个FBG, 其由跨越多个die的闪存block组成, 也就是一个超级块的一部分.
FlexZNS在data area分配一定数量的超级块来容纳区域, 并将每个超级块拆分为多个适合zone大小的FBG.
当一个zone被打开用于写入时, 具有相同大小的空闲FBG被分配和引用.

E: 这里, 区域是被动态地映射到FBG的, 所以可以自行选择映射到哪一个FBG, 这样可以自定义策略来改善闪存die之间的负载均衡/ 减少IO干扰/ 执行超级块之间的磨损均衡.

W: FlexZNS当前实现的分配策略是为每组相同大小的区域维护一个空闲FBG列表, 并优先选择位于"包含开放区域/FBGs最少的die"上的FBG来分配. 这样应该比随机分配效果好, 比如有利于改善die之间的负载均衡.

W:
奇偶校验映射: 与动态数据映射不同, 奇偶校验映射是静态的.
当SSD被格式化, zone布局被给出的时候, FlexZNS静态分配位于奇偶校验area的闪存块 来存储每个FBG的奇偶校验chunk(论文提及奇偶校验块时用的都是chunk而非block).
当一个FBG被一个zone映射时, 这个zone的奇偶校验数据就被写入预分配好的闪存块.

E: 注意, RAID保护是在die级别被执行, 因为die是独立执行闪存指令的基本单元并且可能意味故障所以需要保护.
为了保证RAID效果, 存放奇偶校验块的die不应该与FBG本身的dies重合(假设FBG有一个3号die存放数据同时存放奇偶校验块, 那么当3号die故障时奇偶校验块的数据也会不可信, 就难以用来做数据恢复).

这也解释了为什么对于映射到跨越所有die的超级块的zone, 会需要两个重复的奇偶校验块. 这是为了加强保护, 如果有其中一个die有问题, 至少大概率还可以依靠另一个(这里都只考虑一个die故障的情况).

W: FlexZNS用一个算法来描述奇偶校验.

E:
核心目标是优化奇偶校验的空间率, 或者减少分配奇偶校验块导致的空间损耗.

这里为什么会有空间损耗? 奇偶校验块的数量不是确定的吗?
问题是, 前面说过存放奇偶校验块的die是有约束的.
称存放奇偶校验块的超级块为奇偶校验超级块(大小仍然是超级块大小).
奇偶校验超级块中未必所有die都能用来存放所有FBG的奇偶校验块.

这就是一个die的分配问题, 满足约束的前提下, 合理分配die, 尽可能减少奇偶校验超级块的数量.

W:
FlexZNS的算法是, 首先处理不是超级块大小的FBG, 按FBG大小从大到小来处理, 每次通过`GetDie`函数从一个未满的奇偶校验超级块获取一个满足约束的die,
如果当前所有超级块都没有符合条件的die, 就分配一个新的超级块并获取die, 新超级块一定会有符合条件的die.
找到die以后就调用`MapParity`函数进行FBG到die的映射.

然后处理超级块大小的FBG, 他们只需要每次取两个不同的die然后做映射就行, 不需要满足什么约束.

E: 这个算法其实就是先把最难处理的FBG处理掉, 因为FBG越大能满足其约束的die越少, 所以在可用的die比较多的时候就先处理掉.
越小的FBG越好处理. 而大小等于超级块大小的FBG是最好处理的所以最后做.

### 评估测试

E: FlexZNS在FEMU上实现. FEMU是一种广泛使用的NVMe SSD模拟器, 允许修改SSD固件并进行全栈硬件-软件研究.

W: FlexZNS使用F2FS文件系统和RocksDB数据库评估, 并且修改了ZenFS(ZNS SSD上运行RocksDB所需的发展中文件系统插件),
支持灵活配置zone大小, ZenFS的zone分配和GC策略也进行了优化, 避免运行时以外崩溃, 提高性能(这些说明有待检验).



### FlexZNS代码分析

FlexZNS有提供论文源码, 我进行了一些分析.

下面用Q表示疑惑, A表示解答, N表示要注意的地方或者说笔记. Q和A的编号有对应关系, 如果Q没有紧跟相同编号的A, 说明这个疑惑还没有得到解决.

Q1: 将lpn转为ppa的`zns_ftl.c::zns_lpn2ppa`函数:

获得zoneid的方式是直接`zone_id = lpn / zspp->pgs_per_zone;`, 我有点疑惑, 每个zone的大小不是应该不固定吗?

N1: `zns-ftl.c::zns_ssd_init_params`, zns ssd的初始化函数中, 对每个层次的硬件数量做了规定, 比如section大小是512, 每个页有8个sec所以是4KB, 然后每个块有2048个页所以是8M. 每个plane有128个块, 每个lun(die)有4个pl, 每个channel有8个lun, 总共8个ch.

然后在注释里说, superblock 2G. 如果这样认为, 则代码中实际上是考虑plane一层的, 即**代码中的超级块不是在每个die里面取一个同偏移的块, 而是在每个plane里取一个同偏移的块**. 因为plane的数量是256, 而die数量是64.

N2: 代码中没有名称包含zone的结构体, 不过`zns-ftl.h`有一个`zns_maptbl`. 本质上就是zone的元数据, 可以通过zone_id得到其逻辑id/起始lba/包含block数量等诸多信息, 不过却命名为maptbl, 我觉得这可能因为zone是逻辑概念所以就称为映射表吧, 或者是习惯性或无知不觉地和映射表概念搞混了. 我倾向于后者并认为这是错误, 总不能说我可以映射到我自己的身高吧.

