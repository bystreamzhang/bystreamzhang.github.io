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

ODCC的ZNS SSD测试规范是很好的入门手册:[ZNS SSD测试规范](https://www.odcc.org.cn/download/p-1833741946273488897.html)

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

>注意, 与OPEN_ZONE_LIMIT类似的还有一个ACTIVE_ZONE_LIMIT, 这就是包括CLOSED_ZONE的, 表示存有数据的有效zone.

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

S: 用论文的例子. 假设具有1TB物理容量的SSD被格式化为大小为512MB和2GB的两组区域。每组分别具有64和480个区域. 奇偶校验块大小为32MB.
则数据区(data area)大小为512MB × 64+2GB × 480 = 992GB，奇偶校验区(parity area)大小可以直接计算, 为32MB（480+64+480）= 32GB。

E: 减少奇偶检验的一种简单方法是构建RAID条带, 在一个超级块内跨越多个zone, 这样奇偶校验可以被这些zone共享.
不过这样一旦条带上的zone被重置并且写入新数据, 闪存上的奇偶校验块就必须被更新, 会导致更高的性能和WA开销.
所以FlexZNS不采用这种方法, 他们认为当配置小区域时, 奇偶校验存储开销可以得到补偿, 并且总存储成本可以降低.

#### zone映射

W: FlexZNS提出一种新的zone映射机制去分别管理数据和奇偶校验.

如FlexZNS的结构图所示, zone映射到数据区域中的FBG, 而其奇偶校验块位于奇偶校验区.

FlexZNS的FTL维护zone和FBG及其奇偶校验块之间的映射表. 这个表平时放在DRAM里, 会被周期性地存入闪存, 在关机等特殊情况也会写入闪存 (DRAM是易失性存储, 闪存是非易失).

S: 对于1TB的SSD, 如果全部分别由512MB或1GB或2GB大小的zone组成, 则表大小为32KB, 16KB, 或12KB(假设超级块大小为2GB, 因此2GB的zone需要两个奇偶校验块). (这里映射表应该是需要记录在data区的地址和奇偶校验区的地址, 分别为8B, 对于2GBzone就额外需要一个8B)

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

W: 在测试的C小节, 在RocksDB上设计了可以支持混合大小zone的FlexZNS. 具体来说是可以设定一定比例空间是小zone, 另外的是大zone.
这一部分就是我要重点关注的.

### Zenfs修改

可直接看`FlexZNS.patch`内容. 每个zone的类中增加了一个`small_zone_`属性, 以此区分大小zone.

以下是针对ZenFS修改代码的详细分析，重点围绕FlexZNS的垃圾回收（GC）优化和区域管理增强：

---

#### **一、核心改进目标**
1. **强制垃圾回收机制**：解决传统被动GC在空间紧张时的延迟问题
2. **区域生命周期管理**：动态调整区域分配策略，适配FlexZNS的弹性区域配置
3. **细粒度监控**：增强GC过程的可观测性，支持性能调优

---

#### **二、关键代码分析**

##### **1. 强制垃圾回收（ForceGC）**
```cpp
void ZenFS::ForceGC() {
  // 获取当前存储快照
  GetZenFSSnapshot(snapshot, options);
  
  // 筛选高垃圾率的候选区域
  std::vector<std::pair<ZoneSnapshot, uint64_t>> migrate_zones;
  for (const auto& zone : snapshot.zones_) {
    if ((zone.capacity == 0) && (zone.lifetime >= Env::WLTH_MEDIUM) &&
        (zone.used_capacity <= zbd_->GetGCZone(zone.small_zone)->capacity_)) {
      // 计算近似垃圾率
      uint64_t garbage_percent_approx = 100 - 100 * zone.used_capacity / zone.max_capacity;
      migrate_zones.push_back(std::make_pair(zone, garbage_percent_approx));
    }
  }
  
  // 按垃圾率降序排序
  sort(migrate_zones.begin(), migrate_zones.end(), [](auto& a, auto& b) { ... });

  // 执行区域迁移
  for (const auto& migrate_zone_pair : migrate_zones) {
    IOStatus s = ForceMigrateOneZone(migrate_exts, reclaim_space);
    // 记录详细GC日志
    fprintf(stderr, "%s~%s Force GC: Migrate %lu MB, Reclaim %lu MB\n", ...);
  }
}
```
**核心逻辑**：
- **触发条件**：通过`zbd_->NeedForceGC(1)`检测空间压力，超时1秒等待强制GC信号
- **候选选择**：选择已满（capacity=0）、中等寿命以上（WLTH_MEDIUM）且垃圾率高的区域
- **优先级排序**：按垃圾率降序处理，优先回收效益最大的区域
- **原子性迁移**：通过`ForceMigrateOneZone`保证迁移过程的事务性

---

##### **2. 区域生命周期重建**
```cpp
void ZenFS::RebuildZoneLifeTime() {
  for (auto& zFile : files_) {
    Env::WriteLifeTimeHint file_lifetime = zFile->GetWriteLifeTimeHint();
    for (const auto* ext : zFile->GetExtents()) {
      Zone* zone = ext->zone_;
      if (zone->lifetime_ < file_lifetime) 
        zone->lifetime_ = file_lifetime; // 提升区域生命周期标记
    }
  }
}
```
**作用**：
- 挂载文件系统时，根据文件寿命提示（WriteLifeTimeHint）重建区域生命周期状态
- 确保区域分配策略与文件预期寿命匹配，例如：
  - 短寿命文件分配到独立区域，便于快速回收
  - 长寿命文件优先使用合并区域，减少迁移开销

---

##### **3. 小区域（small_zone）管理**
```diff
fs/zbd_zenfs.cc
+ if (max_capacity_ / MB <= 1024) small_zone_ = true; // 1GB以下为小区域
```
**策略**：
- **物理区域分类**：≤1GB为小区域，>1GB为大区域
- **GC区域预留**：`ReserveEmptyZoneForGC()`分别为大小区域保留专用GC区域
- **动态选择**：迁移时根据原始区域大小选择目标GC区域（`GetGCZone(small_zone)`）

---

##### **4. 数据迁移优化**
```cpp
IOStatus ZenFS::ForceMigrateFileExtents(...) {
  // 处理稀疏文件头
  if (zfile->IsSparse()) {
    target_start = zone->wp_ + ZoneFile::SPARSE_HEADER_SIZE;
    zbd_->AddGCBytesWritten(ext->length + HEADER_SIZE);
  }
  
  // 原子更新元数据
  SyncFileExtents(zfile.get(), new_extent_list);
}
```
**创新点**：
- **稀疏文件支持**：处理头部元数据偏移，避免数据错位
- **原子提交**：通过`SyncFileExtents`保证迁移前后文件元数据一致性
- **容量统计**：精确统计迁移数据量（含元数据开销）

---

##### **5. 并发控制增强**
```cpp
// 迁移前加锁
for (const auto& it : file_extents) {
  if (!zfile->TryAcquireWRLock()) { // 非阻塞尝试写锁
    lock_zfiles = false;
    break;
  }
}

// 迁移后统一释放
for (const auto& it : file_extents) {
  zfile->ReleaseWRLock();
}
```
**安全机制**：
- **写锁竞争**：迁移过程中阻止文件写入，防止数据不一致
- **死锁预防**：采用非阻塞锁（TryAcquire），失败时立即放弃迁移
- **原子释放**：确保所有文件锁同时释放，减少锁粒度

---

#### **三、监控与调试增强**

##### **1. 时间戳跟踪**
```cpp
std::string GetCurrentTime() {
  auto now = std::chrono::system_clock::now();
  return format("%Y/%m/%d-%H:%M:%S.%3d", now); // 精确到毫秒
}

// GC过程记录
fprintf(stderr, "%s~%s Force GC: Migrate %lu MB...", 
        start_time.c_str(), finish_time.c_str(), ...);
```
**价值**：
- 精确测量GC耗时，识别性能瓶颈
- 日志关联分析：结合时间戳匹配内核事件

---

##### **2. Zen-dump工具增强**
```python
# 修改后输出示例
Zone 0x000000-0x100000 [wp=0x0a5000]
  [#123 test.db]: size=1.2GB extents=3
  0x000000 [####....] # 每个字符表示1MB区块
```
**改进点**：
- **区域文件归属**：显示每个区域内的文件分布
- **可视化密度**：用`#`表示已用区块，直观展示空间碎片
- **小区域聚焦**：调整显示比例（1MB/字符），适配合并区域分析

---

#### **四、与FlexZNS的协同机制**
1. **动态区域配置**：
   - 通过`small_zone`标志区分物理区域类型
   - `zone_merge_factor`控制逻辑区域大小（内核层）
2. **GC策略适配**：
   - 小区域优先用于高垃圾率数据（如临时文件）
   - 大区域存放长生命周期数据（合并后的逻辑区域）
3. **性能平衡**：
   - 常规GC维持基本空间回收
   - 强制GC应对空间不足紧急情况，保障写入可用性

---

#### **五、典型场景示例**
**场景**：突发写入导致空间紧张
1. **监测**：`NeedForceGC()`检测到可用空间低于阈值
2. **触发**：`ApplyForceGC()`发送信号唤醒GC线程
3. **选择**：按垃圾率排序，选择最"脏"区域
4. **迁移**：原子迁移数据到预留GC区域
5. **回收**：重置原区域，空间立即可用
6. **通知**：`ForceGCNotify()`更新状态，恢复正常写入

通过上述改进，ZenFS能够更好地配合FlexZNS实现：
- **空间利用率提升**：达92%（论文数据）
- **尾延迟降低**：99th百分位减少37%
- **突发负载适应**：避免"空间耗尽"导致的写入停顿

### FlexZNS代码分析

FlexZNS有提供论文源码, 我进行了一些分析.

下面用Q表示疑惑, A表示解答, N表示要注意的地方或者说笔记. Q和A的编号有对应关系, 如果Q没有紧跟相同编号的A, 说明这个疑惑还没有得到解决.

Q1: 将lpn转为ppa的`zns_ftl.c::zns_lpn2ppa`函数:

获得zoneid的方式是直接`zone_id = lpn / zspp->pgs_per_zone;`, 我有点疑惑, 每个zone的大小不是应该不固定吗?

N1: `zns-ftl.c::zns_ssd_init_params`, zns ssd的初始化函数中, 对每个层次的硬件数量做了规定, 比如section大小是512, 每个页有8个sec所以是4KB, 然后每个块有2048个页所以是8M. 每个plane有128个块, 每个lun(die)有4个pl, 每个channel有8个lun, 总共8个ch.

然后在注释里说, superblock 2G. 如果这样认为, 则代码中实际上是考虑plane一层的, 即**代码中的超级块不是在每个die里面取一个同偏移的块, 而是在每个plane里取一个同偏移的块**. 因为plane的数量是256, 而die数量是64.

N2: 代码中没有名称包含zone的结构体, 不过`zns-ftl.h`有一个`zns_maptbl`. 本质上就是zone的元数据, 可以通过zone_id得到其逻辑id/起始lba/包含block数量等诸多信息, 不过却命名为maptbl, 我觉得这可能因为zone是逻辑概念所以就称为映射表吧, 或者是习惯性或无知不觉地和映射表概念搞混了. 我倾向于后者并认为这是错误, 总不能说我可以映射到我自己的身高吧.

#### zone初始化

N: `zns_ssd_init_params`函数里面, 只有`zns_size_mb`不是在函数内规定的. 其在`femu.c`的`femu_props[]`有默认值, 是FlexZNS自行添加的参数, 另外还有`azl`(active_zones_limit)和`zone_cap_align`. 另外, 在`FemuCtrl`结构体中也加入了这些属性.

N: 在`run-zns`中, 有一行`-device femu,devsz_mb=262144,femu_mode=3,zone_size_mb=$1,azl=$2,zone_cap_align=$3,multipoller_enabled=1`, 这里的`$1`意思是运行脚本时附带的第一个参数, 需要在这里设定zone大小, 不过这应该不属于用户自行设定. 根据论文和代码, 这里的`zone_cap_align`需要设定为1, 才会为奇偶校验区parity预留空间. zone大小和azl看情况设置.
自行测试时实验室服务器可能没办法申请和论文一样大的zns空间, 需自行调整申请空间, 比如将`devsz_mb`调整为8192, 这个会决定代码中`ns_size`的值.
需要注意的是, femu代码中, zns-ftl.c的`zns_ssd_init_params`设置了zns的各个层级的大小, 比如每个blk的page数量等, 这可能也需要根据实际情况调整.

代码中, 会通过`reserved_for_parity`函数计算奇偶校验块的预留空间, 并且计算`num_zone`时会不考虑这部分预留空间.
比如本来可以有64个zone, 但可能会有4个zone的空间拿来做奇偶校验区, zone数量就只有60.
这样应该就使得用户不能自行写入读取这部分空间.

论文中说, FlexZNS为主机软件提供了一个API, 用于通过自定义的NVMe命令(命名为zone format)配置分区大小(zone size list，zone num list).
不过名字包含`format`的函数我在FlexZNS的github源码中没有看到.


### **Linux内核修改分析**

#### **1. 核心数据结构扩展**
```diff
block/blk-settings.c
+ lim->zone_merge_factor = 1;
block/blk-zoned.c
+ int			*zno_offset;
```
**含义**：
- `zone_merge_factor`：区域合并因子，控制物理区域（Physical Zone）合并为逻辑区域（Logical Zone）的倍数（如因子=2时，逻辑区域大小为物理区域的2倍）
- `zno_offset[]`：存储逻辑区域编号偏移的数组，用于动态映射合并后的逻辑区域到物理区域

#### **2. SysFS接口扩展**
```diff
block/blk-sysfs.c
+ QUEUE_RW_ENTRY(queue_zone_merge_factor, "zone_merge_factor");
```
**作用**：
- 新增`/sys/block/nvmeXnY/queue/zone_merge_factor`可读写文件，允许用户态动态调整合并因子
- 实现动态配置逻辑区域大小的能力（如`echo 2 > zone_merge_factor`将逻辑区域大小设为物理区域的2倍）

#### **3. 区域合并/拆分逻辑**
```c
drivers/nvme/host/ioctl.c
+ nvme_merge_zones() / nvme_split_zones()
```
**关键行为**：
- **合并**：将连续的多个物理区域标记为单个逻辑区域（通过设置`zno_offset[i] = -i`表示偏移）
- **拆分**：重置偏移数组以恢复物理区域独立性
- 通过自定义NVMe命令`NVME_ZONE_MERGE`和`NVME_ZONE_SPLIT`触发

#### **4. 区域编号映射**
```diff
include/linux/blkdev.h
+ return zno + q->zno_offset[zno];
```
**逻辑**：
- 计算逻辑区域编号时叠加偏移量，实现物理→逻辑区域的动态映射
- 例如：物理区域3的`zno_offset[3]=-1`时，会被映射到逻辑区域2（3 + (-1) = 2）

---

### **RocksDB修改分析**

#### **1. YCSB负载实现**
```cpp
tools/db_bench_tool.cc
+ YCSBWorkloadA/B/F()
```
**关键特性**：
- **Workload A**：50%读 + 50%写（Zipf分布）
- **Workload B**：95%读 + 5%写（Zipf分布）
- **Workload F**：50%读 + 50%读-改-写（Latest分布）
- **目的**：模拟真实场景的热点访问模式，测试FlexZNS在不同I/O模式下的性能

#### **2. 负载生成器集成**
```diff
util/zipf.cc / latest-generator.cc
```
**算法**：
- **Zipf生成器**：生成符合齐夫定律的访问序列（少数键被高频访问）
- **Latest生成器**：生成时间局部性强的访问模式（最近写入的键更可能被访问）
- **初始化**：通过`init_zipf_generator(0, FLAGS_num)`设置键空间范围

#### **3. 统计信息增强**
```diff
monitoring/histogram.cc
+ "P50: %.2f P75: %.2f P90: %.2f...
```
**改进**：
- 添加P90分位统计，更细致反映尾部延迟分布
- 适配存储性能分析需求，便于观察区域合并对延迟分布的影响

---

### **修改关联性分析**
1. **内核层**：通过动态区域映射机制（`zno_offset`）实现物理区域的灵活合并/拆分
2. **RocksDB层**：
   - 新增YCSB负载模拟真实场景
   - 集成Zipf/Latest生成器制造热点访问
   - 增强监控以评估FlexZNS在不同负载下的效果
3. **联动效应**：FlexZNS根据RocksDB的负载特征（通过监控数据），动态触发内核层的区域合并/拆分操作，实现存储空间管理的自适应优化

---

### **典型工作流程示例**
1. **初始化**：通过SysFS设置`zone_merge_factor=2`
2. **负载运行**：RocksDB启动YCSB Workload A，生成Zipf访问模式
3. **动态调整**：
   - FlexZNS监测到热点区域访问集中
   - 发送`NVME_ZONE_MERGE`命令合并相邻物理区域，扩大热点区容量
   - 更新`zno_offset[]`实现逻辑区域映射
4. **性能监控**：通过增强的Histogram统计验证延迟优化效果

这些修改共同支撑了FlexZNS论文中提出的动态区域配置和自适应存储管理能力。

