---
title: FEMU模拟的ZNS源码分析以及Balloon-ZNS复现记录
date: 2025-02-23 17:12:58
tags:
    - FEMU
    - ZNS
    - SSD
    - Balloon-ZNS
categories: Study
---

# 说明

本文为做毕设项目时的笔记.

毕设项目概括:

Balloon-ZNS论文要在ZNS SSD中加入内置压缩, 使用有FEMU模拟的ZNS SSD硬件的QEMU虚拟机, 代码主体是FEMU, 其中包含对zns ssd的软硬件模拟代码.
使用fio, rocksdb, YCSB, zenfs等工具环境进行测试.
Balloon-ZNS虽然加入了内置压缩, 但实际存在一些问题, 比如其提出的方法会显著降低压缩效果, 影响性能.
内置压缩会破环逻辑页和物理页的对齐, 大小不同的压缩数据段很能存储, Balloon-ZNS提出的Slot层次会存在要处理溢出残量的问题,
Balloon-ZNS对残量的处理也存在许多优化空间.
我的目标是复现并测试Balloon-ZNS, 分析其中存在的问题, 然后提出一些优化方案并进行实现和测试.

[FEMU官方文档](https://github.com/MoatLab/FEMU)
主体代码在hw/femu.
代码分析可参考[0 FEMU 简介 | ["FEMU闪存仿真平台介绍及使用文档"]](https://nnss-hascode.github.io/FEMU-Doc/FEMU-Doc.html)

[FEMU论文](https://www.usenix.org/conference/fast18/presentation/li)  
[Balloon-ZNS论文](https://dl.acm.org/doi/10.1145/3649329.3657368)

主要分析代码：hw/femu/zns下的代码.

暂不分析所有源码, 主要是对Balloon-ZNS在ZNS上有所改动的地方进行分析说明, 记录我当前的改动方法.（后续可能修改）

有两个读写、延迟计算相关的核心函数需要特别关注：`zns.c::zns_nvme_rw`, `zftl.c::zns_wc_flush`.

## 核心函数说明

这里先只说明femu模拟的原版zns.

### zns.c::zns_nvme_rw

此函数的工作主要分以下几步：

1. 从传入的请求req中获取start logical block address(slba)以及number of logical block(nlb), 以及请求是读or写.

> 关于lb: 逻辑块是比逻辑页还小的单位, 一个lb为512B.lb是`zone->w_ptr`、`zone->d.wp`、`n->zone_size`等变量的单位.
>
> lba不是以字节为单位, 而是以lb为单位.slba=524288代表第524288个lb.可以通过函数转为具体字节地址.
> lba是相对整个zone空间的而不是某个zone. 所以0号zone的slba是0, 1号zone的slba可能是524288(假设一个zone有524288个lba).

2. 根据slba获得对应zone指针(使用`zns_get_zone_by_slba`函数).
3. 通过`zns_check_zone_write`等函数初步检查这个请求是否合法, 比如zone是否装得下nlb个lb.
4. 通过slba获得具体要读/写的内存逻辑地址.
5. 从req获取QEMUSGList信息, 包括每个sg的长度和总数量.
6. 打包既有信息调用`backend_rw(n->mbe, &req->qsg, &data_offset, req->is_write);`函数进行实际的读/写.
7. 对于写操作, 检查zns的新状态.
8. 更改`n->zns->active_zone`为slba对应的zone的编号.

### zftl.c::zns_wc_flush

此函数会在某个写缓存满的时候调用.调用时会传入一个wcidx并且整个函数就是处理这个写缓存.

此函数的工作主要分以下几步：

1. 枚举写缓存wcidx的逻辑页编号i, 进入i循环, 执行下面的2~13
2. 枚举zns的plane编号p, 进入p循环, 执行下面的3~12
3. 调用`get_new_page`函数获取一个物理页地址ppa(此时其信息取决于zns的当前写指针wp以及active zone的值.wp会决定物理页的channel和lun, active zone的值会给予blk).将ppa的plane设为p.
4. 根据闪存类型(SLC1, MLC2, TLC3, QLC4,femu设置是4)进入j循环, 重复j次下面的操作5~11
5. ppa的page设为`get_blk(zns,&ppa)->page_wp`, 也就是获取其blk的page_wp.然后其page_wp++.
6. 根据`ZNS_PAGE_SIZE/LOGICAL_PAGE_SIZE`的值枚举子页subpage, 进入subpage循环, 以下7~10为循环内容
7. 根据wcidx, i, subpage获得lpn
8. 通过lpn, 调用`get_maptbl_ent`函数获得oldppa, 检查这个oldppa是否映射过, 如果映射过可能需要额外处理
9. 将ppa的spg设为subpage
10. 调用`set_maptbl_ent(zns, lpn, &ppa)`, 建立lpn和ppa之间的映射.(对于femu模拟器代码, 内部就是执行`zns->maptbl[lpn] = *ppa;`)
11. 退出subpage循环后, i自增`ZNS_PAGE_SIZE/LOGICAL_PAGE_SIZE`
12. 退出j循环后, 根据ppa调用`zns_advance_status`计算sublat, maxlat来更新模拟的延迟值
13. 退出p循环后, 调用`zns_advance_write_pointer`使zns的写指针前进
14. 退出i循环后, 重设写缓存wcidx的used值为0, 返回maxlat

细节说明：

枚举的i是逻辑页编号, 对应逻辑页大小是`LOGICAL_PAGE_SIZE`(4KB).通过get_new_page获得的ppa是物理页地址, 对应物理页大小是`ZNS_PAGE_SIZE`(16KB).

假设写缓存有100个, num_plane=2, flash_type=4, 物理页大小是逻辑页大小的4倍也就是说subpage=4, 那么首先, 对于第0个逻辑页, 获取一个新物理页ppa.
ppa的plane设定为0, 然后ppa的pg变成其blk的page_wp, 然后其page_wp加1,
然后对于这第0个以及后面的总共4个逻辑页, 取出lpn, ppa记录当前逻辑页的subpage编号, 然后将该逻辑页lpn映射到当前ppa.
然后逻辑页编号变成4. 4-7号逻辑页同样会使ppa的pg改变并使对应blk的page_wp加1, 从映射结果上看4-7号逻辑页也是映射到与之前相同的物理页ppa.8-11和12-15应该也是.

也就是说通过`get_new_page`获取的一个物理页在这种假设下可以容纳`4*4=16`个逻辑页, 其pg和spg属性分别有4种值构成了这16种情况(pg的4个值是ppa的blk本身page_wp往后的4个值,而spg的4个值就是0-3).

在ppa的结构体中(见zns.h),各个属性的域都是规定好的,比如spg就只有2位,表示范围是0-3. 如果物理页大小是逻辑页的16倍,spg就应该有4位.

然后对于i=16-31, 轮到下一个plane的一个新物理页ppa.所有plane做完以后, 使zns的wp前进, 改变channel或lun.然后又回到plane0.

这里的做法应该是为了利用plane的并行性, 这种写法下, 每次都是从plane0开始, 处理完所有plane再进行wp的前进, 获取的物理页ppa的属性只和zns的wp和active zone有关,
代码应该是保证了每个plane获取的ppa都属于同一个channel、die并且有一样的blk, 这样不同plane之间就可以并行, 因为同编号的channel, die和blk可以并行.
这里的并行是模拟的,代码中体现为,每个plane有一个独立的`next_plane_avail_time`,类似计时器,会分别记录延迟,并且取最大值作为要输出的延迟maxlat.
如果串行则他们的延迟就要累加而非取最值.
可参考 [SSD基础架构与NAND IO并发问题探讨](https://blog.csdn.net/zhuzongpeng/article/details/134915562)  和
 [论文阅读：SSD内部多级并行性的探索和利用（TOC 2013）](http://cighao.com/2016/07/20/paper-reading-02-the-multilevel-parallelism-inside-SSDs/)

femu模拟的zns代码中, 除了plane以外的blk,fc,ch等层次的结构体也都有一个计时器,但是只有plane的计时器被使用了,所以其他层次的并行性体现在哪里? 后面分析.

## 延迟模拟

简单说一下延迟模拟, 不做详细解释 (自己也还没弄清全貌).

FEMU通过延迟模拟来模拟真实SSD的性能, 代码中会对操作计算延迟并传递给用户, 或者说测试工具.

比如, 在`zftl.c::zns_write`中, 最后会返回一个计算后的延迟maxlat来反映这次写入的延迟, 如果我们自行给它加上一个大数, 其他都不变,
使用fio, YCSB等测试工具测试时也可以看到测试时间会明显延长, 说明这个延迟不仅仅是数值, 也会实际影响运行时间, 大概就是模拟器会强制停顿这段时间来影响模拟硬件的性能表现.

每个plane都有一个延迟计时器, 但并不是直接根据其总延迟来获得每次读写的延迟值. 下面具体说明延迟计算.

### 延迟计算说明

每一次延迟计算是针对一次req进行的, 目标是计算一个lat并执行`req->reqlat = lat;` 和 `req->expire_time += lat;`. 暂不考虑后续如何使用这个值, 但假设lat会影响最终模拟的性能.

一次req包含的基本信息就是slba和nlb, 说明从slba开始读或写nlb个lb. 以及stime, 表示请求开始时间. 延迟是相对这个时间的.

一次req的lat计算发生的背景函数是`zftl.c::zns_write`, 简单说就是处理slba和nlb对应的所有lpn.
如果是读取则直接映射到ppa然后通过计算lat的核心函数`zns_advance_status`计算lat, 如果是写则放入写缓存, 当写缓存满时调用`zns_flush`.
`zns_flush`中, 分配ppa来处理每一个lpn, 并对每个ppa调用`zns_advance_status`计算lat.

一次req的lat计算过程中, stime是不变的, 而每个plane的计时器会变. 如果小于stime则会先同步到stime再增加. lat是取他们的差值.

不管是对于读或写, `zns_advance_status`计算的lat仅是sublat, 会取最值得到maxlat来返回. 也就是说, 每个req中的所有lpn对应的ppa, 只取计算出的lat的最大值.

也就是说对于一次请求req, 所有空闲plane(计时器小于stime)都是从stime这同一起跑线开始计算延迟, 最终使用的lat是所有plane计算的延迟的最大值.

而req的大小取决于slba和nlb. 如果req很小, 涉及的ppa也会很少, 那么即使分配的plane很少也可能足够使得他们的延迟最值很小.
换句话说, 这种情况下即使分配plane特别多, 也未必能有什么优化效果.

S: 假设一个写req总共涉及16个ppa, 刚好有16个plane可以并行处理, 然后刚好均摊分配这些plane, 此时每个ppa调用的`zns_advance_status`计算的lat就是一次写入默认的写延迟(如12196000).
然后如果改成有64个plane, 多余的plane并不能减少延迟, 最终的lat不会变化.

所以plane数量增多带来的并行性提升未必能提升模拟的性能效果, 非常依赖于:

1. 每次req涉及的ppa数量
2. 涉及的ppa的plane的分布(是集中于某些plane还是均摊分布)

## 并行性

前面讨论`zns_wc_flush`函数时说明了plane级别的并行性体现. 这里讨论其他层次的并行性.

这里放一个参考网站的图片:

![NAND结构示例](nand_structure.png)

我一开始的理解太过简略了, 我以为plane的num是2表示总共只有2个plane, 其实不然, 每个die都有2个plane,
而计算延迟的函数`zns_advance_status`中, 用`get_plane(zns, ppa)`获取的plane地址是和plane的上级即fc(lun/die)和ch有关的,
沿着调用路径追溯就可以发现这一点. 见下面代码:

```c
static inline struct zns_ch *get_ch(struct zns_ssd *zns, struct ppa *ppa)
{
    return &(zns->ch[ppa->g.ch]);
}

static inline struct zns_fc *get_fc(struct zns_ssd *zns, struct ppa *ppa)
{
    struct zns_ch *ch = get_ch(zns, ppa);
    return &(ch->fc[ppa->g.fc]);
}

static inline struct zns_plane *get_plane(struct zns_ssd *zns, struct ppa *ppa)
{
    struct zns_fc *fc = get_fc(zns, ppa);
    return &(fc->plane[ppa->g.pl]);
}
```

总结来说, 取用的plane是哪一个, 将同时取决于ppa的ch, fc 和plane, 而不只是plane值. 

打印中间结果也可以发现, 当ppa的ch, fc和plane值一样时, 取出的pl的地址绝对是一样的.

所以ch, fc级别的并行性也在这里体现了.femu这里设定ch数量为2, fc为4, 则不同plane的数量是2x4x2=16个.这16个plane之间是并行性的, 有独立的延迟计时器.

合理地将写入操作分摊到这些可以并行的plane才可以提升效率. 而如果操作总是集中在某几个plane, 计算出的延迟就会增加.

另外补充一个重要知识: **Plane是并行操作的最小单元**.
执行多Plane操作时有一些限制. 例如, 进行多Plane读/写操作的Page必须位于具有相同芯片、芯片Die、Block和Page地址的Plane上（仅Plane不同）

所以不需要考虑block层次的并行性, 是不存在的.

## 硬件设置

femu模拟器的硬件设置方式是, 在编译生成得到的运行脚本中(或者在`femu-scripts`中)修改诸如`SSD_SIZE_MB`等变量, 将作为参数传递.

不过, 在代码内部也要匹配外部设置. 比如物理页ppa的各级位表示, 在代码中默认是给channel 2位, 这会导致代码内认定的物理页所在channel最多就是2种编号, 理论上会影响结果. 可以看我下面Balloon-ZNS复现中的一些记录.

## Balloon-ZNS思想

这里简要说明一下Balloon-ZNS的核心思想.

就是在ZNS SSD加入内置数据压缩. 数据页在被写入闪存前在线进行压缩, 读取时解压.

因为传统SSD支持设备内压缩, 所以这里假设Balloon-ZNS的SSD控制器内有一个硬件解压缩引擎(论文使用Intel QAT). 这个引擎通常可以在几ms内解压缩一个页所以不会成为性能瓶颈.

内置压缩是放在核心函数`zns_nvme_rw`中, 因为这里做了实际的内存写入. 写入前我们对数据进行压缩然后再写入.

关键挑战是压缩后要保持ZNS SSD的逻辑页物理页地址对齐. Balloon-ZNS对此提出一个具有压缩率适应性的槽对齐存储布局.

这里我直接描述一下Balloon-ZNS论文提到的所有核心设计思路.

### 子超级块设计

这里先说一下Balloon-ZNS里面子超级块的设计.

对于ZNS SSD, 将写入zone时, 其会先被打开, 并且SSD会将其映射到一个闪存超级块, 或者说一个GC单元.

**zone和超级块的映射**: 代码中, 超级块sblk被记录在`zns->cache.write_cache`结构体中. 在`zftl.c::zns_write`中, 通过`zns_get_wcidx(zns)`获取写缓存编号cidx,
这个函数内部会返回sblk等于zns当前活跃zone的写缓存. 如果没有则返回-1. 如果结果是-1, 则后续会找到或腾出一个空的写缓存并将其sblk设为zns当前活跃zone.

这样就完成了论文所说的映射. 注意是一一映射.

一般来说, 一个超级块由一系列闪存块blk组成, 他们位于许多或全部的并行fc(die)中, 并且有相同的offset. 比如, 0号超级块会包含所有die中的0号block. 这也是一个zone会映射到的内容.

而Balloon-ZNS首先将闪存超级块分割成更小的子超级块(sub-superblock), **这将是zone映射的基本单元**.

zone会在首次写入时被映射到一个子超级块, 后续如若需要还可以分配更多来满足空间需求.

这个映射应该可以和超级块映射类似, 存储在写缓存中, 并且取代原本的超级块映射.

子超级块对并行性的影响: 看子超级块具体如何分配.

最小可以并行的单位是plane. 设ch, die, pl三者构成三元组(c,d,p), 总共就是`c * d * p`个可并行单位, 记为cdp.

一个超级块会有cdp个可并行的block. 划分子超级块就是划分这cdp个block.

假设把超级块按die拆分其block, 可以拆出d个, 那么就是影响die级别并行性, 并行性降低d倍. 按plane拆分就是影响plane级别并行性.

不过一般似乎就是对die拆分, 这可能存在习惯问题, 也可能是因为plane的数量一般比较少, 而die比较多,
而子超级块的核心意义就是把超级块拆成一个明显更小一级的粒度, 所以在die的级别拆比较合适.

### slot相关设计

参考下图中的 `Slot-alogned Data Layout` 部分:

![Balloon-ZNS结构示意图](overview_bz.png)

核心就是引入一个新的比page更小的粒度: slot.
page压缩后会放入一个slot, 一个子超级块内slot大小是统一计算好的, 当压缩后page比slot大, 残量(residue) 会被独立保存.

事实上, residue的存在是Balloon-ZNS方案最大的问题. 不可能设定一个slot使得完全没有残量, 毕竟总会有特例, 而slot设大了也会拉低压缩效果.

所以要在slot设为一个能保证压缩效果的值的情况下, 合理处理残量, 降低其负面影响.

### 主子超级块和额外子超级块

为了存放残量, 对子超级块做一个分类: 存放slot的是主子超级块, 记录残量的是额外子超级块.

如前面子超级块部分所说, 每个主子超级块会被专门映射到一个zone, 而额外子超级块则不同, Balloon-ZNS中规定,它是可以被多个zone共享的, 不过一次只能由一个zone写入**直到这个zone关闭**.

这里请保证自己对"zone关闭"理解到位.(我一开始没在意, 其实对后续的代码理解和实现很重要)

关于zone的多种状态, 可参考[ZNS: Avoiding the Block Interface Tax for Flash-based SSDs](https://www.usenix.org/conference/atc21/presentation/bjorling) 或者阅读我ZNS SSD的学习记录.

额外子超级块作为超级块的子类, 也是要顺序写入的.
而一个zone关闭之前都会一直使用一个额外子超级块. 当zone关闭时, 其会释放设备资源, 比如写缓存(ZNS SSD论文特意提到), 但实际数据不会清除, 残量作为实际数据的一部分也理应不动.

根据Balloon-ZNS, zone关闭时其额外子超级块将可以被分配给另一个新打开的zone来append其残量.
通过这样做, 每个zone的残量都记录在连续的物理空间中, 即一组连续且独立的闪存页.

我不太懂论文这里的意思. 我知道一个额外子超级块内可以有来自多个zone的残量并且同一时刻只有其中一个zone是open的.
不过其中同一个zone的残量未必是连续的才对, 假设zone0关闭以后, zone1开启并往其额外子超级块写入了残量,
然后zone0有开启并关闭zone1, 继续写入残量, 则此时zone0的残量并不连续.

(除非每个zone记录自己在额外子超级块的首个残量地址以及总的连续残量长度, 然后当关闭又重新打开时,
将自己之前的所有残量append到当前使用的额外子超级块的末尾, 这样可以永远保证zone残量连续性, 但额外开销量就不好说了.)

不过这个连不连续的问题可能并不影响后续做法吧.

### 残量数据页读取

一个有残量的数据页被存放在两个分割的闪存物理页中.
第一个闪存页可以通过zone级别的映射表获取. 具体就是找到对应主子超级块, 然后根据逻辑地址找到物理页以及对应slot. 因为我们保持了L&P对齐所以这里很快.

至于定位残量, Balloon-ZNS的方法是将其物理地址和长度作为管理元数据插入数据页的第一部分闪存页.
残量数据本身存储在额外子超级块的空间, 并且根据论文的说法, 既然记录了其实际物理地址和长度, 额外子超级块内的残量应该可以紧密存放来提高存储效率.
不过论文后面提到, 为了简化存储管理, residue大小要与256B对齐.

这里就是这个管理元数据实现起来可能麻烦点. 破环写入数据感觉倒没事, 本身写入的就是压缩后的数据, 可以在解压缩时顺便把管理元数据读取出来, 返回的数据不包括就行.

但, 管理元数据只能是在zns.c中写入部分做, 但是此函数在模拟器里和写缓存是分割的, 想确定一个额外子超级块只能在zftl.c做.
所以换个方法. 核心是记录page级别的一个数据, 目标是尽可能减少存储量.
我们还有什么其他page级别的存储变量吗? 有, 通过逻辑地址得到物理地址的映射.
做一个类似的数组存储每个逻辑地址的残量信息. 最简单的方法就是这里直接记录物理地址和长度, 和论文一致.
本质上占用的额外空间和论文一致, 只是位置可能不同

补充, 后续为了解决残量回收问题, zone的残量可能需要移动, 所以这里记录的元数据的地址不应该是固定的物理地址而应该是偏移, 相对于zone的连续残量起点地址的偏移,
所以每个zone还需要记录其额外子超级块编号 & 连续残量的起点地址, 并且如我前面所说, 对于关闭又重新打开的zone, 其连续残量还要移动到末尾以保证后续插入也能连续.

### 压缩性可适应槽

Balloon-ZNS的一个假设是, 数据存在压缩率局部性, 即连续数据往往有类似的压缩率.

在此基础上, 对于slot大小, Balloon-ZNS给出一种动态计算方案: 压缩性可适应槽(Compressibility-Adaptive Slotting)

引入新的**分析窗口**粒度. 一个逻辑zone被分成多个固定大小的分析窗口.

因为zone只能顺序写, 分析窗口也只能被连续写入.

Balloon-ZNS会给每个分析窗口计算一个slot大小, 根据上一个窗口的数据压缩性分析结果来计算.

具体地, 记录分析窗口内每个压缩后数据页的大小, 维护CR(Compression Rate, 压缩率)分布(通过一个简单的桶排序算法).
当窗口初次被写入时, 取出上一个窗口的CR分布中的一个指定的百分位对应的值作为当前窗口的slot大小.
zone的首个窗口的slot大小是预设的, 比如可以设为80%逻辑页大小来尽可能规避residue但也会使得这个窗口的压缩效果减少.
取用CR分布的百分位(percentile)也是预设的, 论文中有30%和70%两种取法.

分析窗口的大小也是可以预设的, 论文设为zone的1/8, 和其子超级块大小一致. 不过似乎两者不强求一致.

论文说, 为了实现分析窗口, 需要扩展zone级别映射表, 我觉得意思就是需要在zone结构体(zns.h::NvmeZone)存储分析窗口信息.
zone里面的映射表应该就是逻辑页到物理页的映射表, 即代码中的maptbl. 其他映射关系, 比如sblk和zone的映射, 是小且临时的.
(论文说法可能有问题, 似乎是把zone的元数据结构体NvmeZone错误地称为zone级别映射表了.)

按我的理解, 分析窗口就是用来计算合适的slot大小的, 计算完就没什么意义了.
分析窗口信息需要在获得slot大小时就进行更新, 因为我们不想额外存储每个slot的具体大小信息. 所以这一步在核心函数`zns_nvme_rw`进行.

### 残量空间回收

考虑一种情况: 主机需要reset一个zone, 本来对于ZNS SSD来说, 只需要无效化其所有数据页即可, 因为都是连续存放在超级块的, 不夹杂其他zone的内容, 不需要GC.

而对于Balloon-ZNS, 主子超级块的内容是可以直接无效化, 但额外子超级块会包含其他zone内容, 就必须有残量空间的回收操作.

论文的思想就是, 把额外子超级块中来自其他zone的有效残量移动到另一个子超级块. 这样做需要我们记录残量时就记录相对其所在zone连续残量起点地址的偏移而非固定地址.

问题是, 我们前面没说过对于额外子超级块本身我们会记录什么信息. 如果拿到一个额外子超级块, 我们如何获知其中数据分别属于哪个zone?
不过这个问题还好, 因为zone数量一般很少(比如16个或32个), 可以枚举每个zone检查哪个zone的额外子超级块是这一个, 是的话就做处理.

论文说残量导致的这种影响很小, 我对此表示怀疑, 论文图4中就显示对于6个真实数据, 残量存在的情况平均可以达到20%~40%,
如果对于更复杂的数据, 即使是微小的垃圾数据堆叠起来也可能导致不可忽视的开销.
不过在FEMU模拟器中, 对于这部分延迟的模拟是个问题. 我甚至不确定论文自己是否模拟并测试了残量垃圾回收导致的开销量.

### 不可压缩数据流的判定处理

子超级块相比超级块, 超级块能提供完整的并行性因此有更高的读写性能.

为了避免不必要的性能开销, Balloon-ZNS可以设置为压缩性分析设置一个较低的CR水位线
(比如0.9, 如果4000b数据只能压缩为3600b, 算上残量可能基本就没压缩效果, 没必要强行压缩).
如果数据CR低于该水位线, 则该zone视为不可压缩, 直接像传统ZNS SSD那样映射到超级块.

这也意味着超级块和子超级块可以共存, 并且应该可以灵活地切换两种模式.

### 扩展内容

这里是我觉得相对没那么重要的一些内容.

#### 残量迁移到主子超级块

论文提到, "一个区域的压缩数据页可能只占用映射到该区域的主子超级块的一部分.
为了提高存储效率，Balloon ZNS将一个区域的剩余部分从额外的子超级块迁移到该区域的部分填充的主子超级块，前提是逻辑区域处于写满状态时有足够的可用空间。"

这个说法, 一方面表明主机可能是不知道zone有内置压缩的, 否则肯定会把zone的主子超级块写满为止. 所以还是按zone原来的大小进行写入.
确实我们都无法预测zone在内置压缩后可能有多大的实际空间占用,所以不能强行说一定有多少扩展空间.

但是问题是, 一个主子超级块只是超级块是1/8, 一个zone可能用到多个, 所以前面用的几个应该还是尽可能用满的. 只有最后一个可能有剩.

另一方面, 这里意思是可以将额外子超级块的连续残量内容迁移到主子超级块, 这样做的话这个主子超级块后续肯定是不能支持写入了.
所以只能是最后一个有剩余的主子超级块来做这个步骤. 所以这里的步骤只能在zone达到FULL状态时进行.

**但是zone什么时候算FULL呢?**

代码中, 用`zone->d.wp == zns_zone_wr_boundary(zone)`来判断是否FULL, 函数其实就是返回`zone->d.zslba + zone->d.zcap`.

所以就是wp到界限了就认为达到FULL了. 而wp这个东西在代码中是逻辑概念, 写入多少lb就增加多少.

所以如果不改变代码, 尽管做了内置压缩, 实际能支持写入的lb数量是不变的. FULL状态的判定也不变.

但如果改变代码, 希望在本来是FULL的时候不判定为FULL, 肯定需要改变wp相关逻辑, 感觉会比较麻烦.

对于"残量迁移到主子超级块"的这个设计, 倒不需要改变代码原有的逻辑, 最后一个主子超级块确实可能会有空余.
当zone进入FULL状态时, 将其残量移动过去即可.

#### 容量管理

Balloon-ZNS论文说, 加入内置压缩后SSD可以向主机展示更大的逻辑地址空间.
不过Balloon-ZNS给出的方案是让用户来确定一个容量扩增比率.
比如, 当扩增比率设置为2, 拥有2TB实际容量的Balloon-ZNS SSD 可以展现4TB用户可见容量.
不过如果这个扩增比率设置得比实际数据压缩比率高, SSD可能在逻辑空间耗尽前就耗尽闪存存储空间.
这种情况下, Balloon-ZNS将向操作系统报告一个out-of-space错误并要求用户干预, 这类似于现有块设备的行为.

#### 软件可扩展性

Balloon-ZNS论文说, 他们的这种压缩能力除了可以被主机以透明方式使用, 也可以被用作一个有用的功能或者一种高效的软件定义方式.
主机可以通过了解数据压缩性来管理写流。将具有相似压缩性的数据页写入同一区域可以最大限度地减少插槽对齐造成的空间浪费和读取截断数据页的开销。
此外，如果数据流的压缩性较差或读取关键，主机可以显式禁用某个区域的压缩（例如，通过提示的区域打开命令），并将该区域映射到闪存超级块。
不过这些都是后话. 关键是Balloon-ZNS本身要有一个好的效果.

## 初步的层级分析以及子超级块相关实现

子超级块是超级块的划分. 具体如何划分?

超级块是所有同编号的块的集合. 比如超级块0包含所有的channel中的所有die中的所有plane中的所有block0.

以Balloon-ZNS论文的环境为0, 其中有8个channel, 每个ch有8个die(共64个), 每个die有128个闪存块, 每个块有8192个4KB大小的物理页. 没有提及plane层级.
假设plane数量为1, 那么此时一个超级块就包含8x8x1=64个block, 也就是在每个plane中占据一个同编号的block. 看似64个块很少, 实际上大小有64x8192x4KB = 2GB.
zone的大小默认和超级块相同, 也是2GB. 子超级块大小被设置为1/8超级块, 论文没有给出原因.
关于论文中没有提到的, 超级块和zone的数量, 如果论文环境中没有plane这一层或者plane就1个那就是128.
否则我认为应该取决于每个plane拥有的block数量, 比如如果有2个plane, 那就是64.

至于FEMU模拟的ZNS SSD, 这些硬件的数量在编译生成的运行脚本中设置, 比如`run-zns.sh`.
默认是8个ch, 每个ch有4个die, 每个die有2个plane, 每个die有32个block. 这里应该是脚本本身说错了, 32是每个plane的blk数量.

写指针会按这个设定前进.
不过FEMU的ZNS SSD中, ppa的结构体设置里, 只分配给channel 1个bit, 就是只支持0和1号ch.
这样不太合理, 相当于就是2个ch.
我试着完全按Balloon-ZNS论文去配置内外的参数.

论文里面SSD有256GB, 非常大, 而femu默认的ZNS SSD设置才4GB. 我觉得页大小和块数量不用一致, 因为不影响并行性.
其他方面我想改成和论文一致. 不过仔细分析发现, 论文没有提到plane, 而由于每个超级块有64个块, 总共也就64个die, 所以论文里可以认为plane就1个.
FEMU默认配置是die4个pl2个, 论文是die8个pl1个, 那么其实是匹配的. 所以也就不用改.

不过后面做rocksdb测试时, 由于zone数量需要32个, 我改了`SSD_SIZE_MB`为8192MB并将代码内的`num_pages`锁死为256, 这样可以使zone数量固定为32.
代码中, ppa结构体我也改成符合外部设置的数量. 具体就是改zns.h的宏定义, 诸如`CH_BITS`等.

> 这里, zwl师姐说, 她试过把ch_bits设置为3，再次测试发现写性能显著提高，然而读性能会不知原因地下降至与写性能一致。另外一个奇怪的点是，在ch_bits=3时，内置压缩后写性能似乎也并不会再提升了。她在femu的仓库里提了issue，作者承认是代码有bug，但他们不知道bug在哪。

我自己在修改后测试原生femu的结果是, fio的顺序写测试速度暴增, 大概就是4倍, 符合预期,而YCSB的所有负载和数据集测试都没什么变化.
我不禁怀疑femu代码的并行性实现是在fio测试下才可以体现, 或者至少这个CH_BITS对并行性的影响是这样, 而YCSB的测试就不太能体现.
Q: 为什么呢?

A: 我对CH_BITS=3和1的情况分别做YCSB测试, 额外统计每个pl全程的总延迟.
测试时使用了六个负载和Balloon-ZNS的数据集, 结果发现对于同一个plane, 比如`ch=0 fc=0 pl=0`对应的plane, ch2个时的总延迟和写延迟大概就是ch8个时的4倍, 读延迟大概是2倍.
不过偶然发现这个新增加的plane延迟计时器在多次测试中不会初始化, 我怀疑之前没做好测试间的初始化, 之后会弄清楚, 这里不提(**TODO**).

主要是, 现在确定了YCSB的延迟计算是会受到CH_BITS影响的, 但不知道为什么对最终的测试结果没什么影响.
我怀疑, 用来模拟的延迟并不是按我之前的理解--即, 并不是完全看plane的总延迟, 并不是plane数量翻倍性能就翻倍.
代码中真正传递的延迟是`lat`(可能是读或写延迟), 其并不反映历史总延迟. 一次写入的默认延迟是12196000, ch8时lat最大就是12196000, 不过ch2时存在lat为24392000甚至更大的情况,
此时应该才真正影响性能, 不过这种情况并不算多, 数万次lat计算中只有972项, 而lat为36588000的有876项, 为48784000的有752项, 为60980000的情况就没有了. 所以总影响就很小.
而对于效果显著的FIO测试, ch2的情况下, 12196000总共6124项, 而36588000可以有5024项, 48784000可以有6547项; 换到ch8情况, 12196000项数就大于2万了, 说明此时优化效果确实明显.

Q: 但是为什么实际传递的lat往往很小呢? 中间存在某种刷新重置的机制吗? plane数量对于延迟计算的影响依赖于什么因素? 这部分的分析我放到前面关于延迟模拟的部分了.

### 子超级块设计思路

回到子超级块设计.

子超级块是超级块的下一级, 核心目的是用来区分出主子超级块和额外子超级块, 从而可以处理残量.

那么首先分析代码中的超级块是如何实现的.

代码中, 每个写缓存有一个sblk属性表示超级块. 总共`ZNS_DEFAULT_NUM_WRITE_CACHE`个写缓存, 默认为3个(不知道这个数字是否需要改变, 后面再分析),
每个写缓存的容量是`id_zns->stripe_unit/LOGICAL_PAGE_SIZE`,
这里`stripe_unit`代表每个不同plane取出一个物理页构成的大小(这里的物理页是考虑flash_type的, 所以在代码中1个顶16个逻辑页).
(参考`zns.c::zns_init_params`)
所以写缓存和超级块的大小是对应的. 写缓存容量cap就是一个超级块的物理大小可以容纳的逻辑页lpn的数量.

写缓存的sblk是可变的, 具体如何取值呢? 写缓存仅在写操作使用, 在`zns_write`函数中, 会假设已经设定好当前写操作的当前活跃zone(active zone), 并根据这个值找到或腾出一个写缓存(如果有写缓存的sblk==active_zone就直接使用, 如果没有, 则调用`zns_flush`清空相对最满的一个写缓存并将其sblk设置为active_zone).

active_zone是在`zns.c::zns_nvme_rw`的末尾根据slba和zone大小设定的, 也就是在数据读写操作结束后.   - 
 


子超级块0必然不能包含所有block0. 



### 宏

这里模拟器设定die数量为4, 我规定一个超级块有4个子超级块.

```c
#define SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO 4
```

### 结构体

此处代码均在zns.h.

超级块本身没有结构体, zns的写缓存会记录当前超级块的id.

```c 
struct zns_write_cache{
    uint64_t sblk; //idx of corresponding superblock
    uint64_t used; 
    uint64_t cap;
    uint64_t* lpns; //identify the cached data
};
```

#### 子超级块

```c 
/*added by znbc: sub-superblock. Give up lun level parallelism*/
struct sub_superblock{
    bool used;
    uint32_t to_zone;
    uint64_t lun;
    uint64_t blk;
    uint64_t write_pointer; // record the number of physical pages written
    bool is_ext;
};
```

我在zns的结构体中记录子超级块相关的一些信息：

```c
struct zns_ssd{
    ...
    /* added by znbc: for balloon-zns */
    struct write_pointer_bz wp_bz;
    struct sub_superblock * ssblk; // this records every sub-superblock
    uint64_t num_ssblk;
    uint64_t ssblk_size_limit; // size limit of sub-superblock 
    uint64_t profiling_window_size;
    struct slot_bz *slots;
    #ifdef BALLOON_ZNS_RESIDUE
    uint64_t now_exssblk; // the extra-superblock in use. Enumerate from back to front!(eg. 63,62,...)
    #endif
    ...
}

...

typedef struct NvmeZone {
    NvmeZoneDescr   d;
    uint64_t        w_ptr;
    QTAILQ_ENTRY(NvmeZone) entry;
    #ifdef BALLOON_ZNS
    // add by znbc
    uint64_t *ssblk;   // the idx-array of sub-superblocks mapped by this zone
    uint64_t num_ssblk; // the number of sub-superblocks mapped by this zone
    uint64_t ssblk_idx; // active sub-superblock
    ProfilingWindow *pfwd; // profiling window
    #ifdef BALLOON_ZNS_RESIDUE
    uint64_t *exssblk;  // the idx-array of extra-sub-superblocks mapped by this zone
    uint64_t num_exssblk;
    uint64_t exssblk_idx;
    #endif
    #endif
} NvmeZone;
```

### 相关函数

#### zftl.c

##### 获取物理页

原版就是获取zns的当前写指针, 从中获取物理页的各种信息, 包括channel、flash chip(die)、block等.这里并没有设定好ppa的所有信息, 比如没有设定plane、page、subpage, 这些会在后续设定 (见`zftl.c::zns_wc_flush`).

```c
static struct ppa get_new_page(struct zns_ssd *zns)
{
    struct write_pointer *wpp = &zns->wp;
    struct ppa ppa;
    ppa.ppa = 0;
    ppa.g.ch = wpp->ch;
    ppa.g.fc = wpp->lun;
    ppa.g.blk = zns->active_zone;
    ppa.g.V = 1; //not padding page
    if(!valid_ppa(zns,&ppa))
    {
        ftl_err("[Misao] invalid ppa: ch %u lun %u pl %u blk %u pg %u subpg  %u \n",ppa.g.ch,ppa.g.fc,ppa.g.pl,ppa.g.blk,ppa.g.pg,ppa.g.spg);
        ppa.ppa = UNMAPPED_PPA;
    }
    return ppa;
}
```

balloon-zns版本如下.

```c
// add by znbc
static struct ppa get_new_page_bz(struct zns_ssd *zns, FemuCtrl *n)
{
    struct write_pointer_bz *wpp = &zns->wp_bz;
    NvmeZone *zone = &n->zone_array[zns->active_zone];
    u_int64_t ssblk = zone->ssblk[zone->ssblk_idx];
    struct ppa ppa;

    zns->active_zone, zone->ssblk_idx, ssblk, zns->ssblk[ssblk].write_pointer, zns->ssblk_size_limit);

    if(zns->ssblk[ssblk].write_pointer >= zns->ssblk_size_limit){
        zone->ssblk_idx++;
        if(zone->ssblk_idx >= SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO){
            ftl_err("zftl.c::get_new_page_bz: a zone have too many sub-superblocks! Should not happen.\n");
            zone->ssblk_idx = 0;
        }
    }
    ppa.ppa = 0;
    ppa.g.ch = wpp->ch;
    ppa.g.fc = zns->ssblk[ssblk].lun;
    ppa.g.blk = zns->ssblk[ssblk].blk;
    ppa.g.V = 1; //not padding page
    zns->ssblk[ssblk].write_pointer++;
    if(!valid_ppa(zns,&ppa))
    {
        ftl_err("[Misao] invalid ppa: ch %u lun %u pl %u blk %u pg %u subpg  %u \n",ppa.g.ch,ppa.g.fc,ppa.g.pl,ppa.g.blk,ppa.g.pg,ppa.g.spg);
        ppa.ppa = UNMAPPED_PPA;
    }
    return ppa;
}
```

结合下面的写指针代码进行说明：

```c
#ifndef BALLOON_ZNS
//no use in Balloon-ZNS
struct write_pointer {
    uint64_t ch;
    uint64_t lun;
};

#else
// added by znbc: write pointer for Balloon-ZNS
struct write_pointer_bz {
    //uint64_t ssblk_idx;
    uint64_t ch;
};
#endif
```

本来, 只需要wp和active_zone的信息就可以完成get_new_page函数, 程序运行时写指针wp就不断前进并在不同的ch和lun(die)跳跃, 模拟并行, 
而blk就是zns的当前活跃zone编号.

现在引入子超级块以后, 一个zone的block未必覆盖所有die所以wp不能顺序跳跃, zone和block也不再对应, die0的block0和die2的block0可能属于不同的zone.
所以现在需要知道子超级块编号才能知道die和blk的值, 才能完成get_new_page函数.

wp的变动主要发生在`zftl.c::zns_advance_write_pointer`,下面展示修改前后的结果：

修改前：
```c
static void zns_advance_write_pointer(struct zns_ssd *zns)
{
    struct write_pointer *wpp = &zns->wp;

    check_addr(wpp->ch, zns->num_ch);
    wpp->ch++;
    if (wpp->ch == zns->num_ch){
        wpp->ch = 0;
        check_addr(wpp->lun, zns->num_lun);
        wpp->lun++;
        // in this case, we should go to next lun //
        if (wpp->lun == zns->num_lun){
            wpp->lun = 0;
        }
    }
}
```

修改后：

```c
static void zns_advance_write_pointer_bz(struct zns_ssd *zns)
{
    struct write_pointer_bz *wpp = &zns->wp_bz;

    check_addr(wpp->ch, zns->num_ch);
    wpp->ch++;
    if (wpp->ch == zns->num_ch){
        wpp->ch = 0;
    }
}
```

原本就是先枚举channel然后die, 现在先枚举channel以后按道理要在当前子超级块继续枚举, 可是无法判断当前子超级块是否满.

调用这个函数的是`zns_wc_flush`函数, 其中原本也不会判定当前zone是否满, 就是不断调用该前进函数.
**所以如何在这个flush函数的某个位置判断当前子超级块是否满**？

逻辑上, 用户发出请求时zone一定是没满的, 按原本的做法每个blk使用一个页后轮到下一个blk, 如果某个blk满了, 访问下一个blk, 如果全都满了, 
这个zone就是FULL状态, 不可能被访问了.所以可以假设flush时zone就是没满的.

我为子超级块设定写指针, 记录用了多少.单位是什么？每次get_page会取出一个物理页, 就让写指针单位是物理页.
超级块有`ch*lun*plane*blk*num_page`个物理页, 子超级块要除以4, 当写指针等于这个值时, 
说明当前活跃zone的当前子超级块满了, 轮到下一个子超级块(没有就要分配一个).



#### zns.c

##### 初始化

zns初始化：

```c
static void zns_init_params(FemuCtrl *n)
{                                                                                   
    ...
    #ifndef BALLOON_ZNS
    id_zns->wp.ch = 0;
    id_zns->wp.lun = 0;
    #else
    //added by znbc: init zns for balloon-zns
    id_zns->wp_bz.ch = 0;
    id_zns->ssblk = g_malloc(sizeof(struct sub_superblock) * id_zns->num_blk * SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO);
    id_zns->num_ssblk = id_zns->num_blk * SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO;
    id_zns->ssblk_size_limit = id_zns->num_ch * id_zns->num_lun * id_zns->num_plane * id_zns->num_page \
    / id_zns->flash_type / SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO;
    for (i = 0; i < id_zns->num_ssblk; i++) {
        id_zns->ssblk[i].used = 0;
        id_zns->ssblk[i].to_zone = 0;
        id_zns->ssblk[i].is_ext = 0;
        id_zns->ssblk[i].lun = i % SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO;
        id_zns->ssblk[i].blk = i / SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO;
        id_zns->ssblk[i].write_pointer = 0;
    }
    id_zns->profiling_window_size = id_zns->num_ch * id_zns->num_lun * id_zns->num_plane * id_zns->num_page \
    * ZNS_PAGE_SIZE / ZONE_SIZE_TO_PROFILING_WINDOW_SIZE_RATIO;
    #ifdef BALLOON_ZNS_RESIDUE
    id_zns->now_exssblk = id_zns->num_ssblk - 1;
    #endif
    
    #endif
    ...
}
```

这里, 子超级块的`write_pointer`属性是用来判断是否用满的, 
每次`get_new_page_bz`获取新页就会加1, 如果达到`zns->ssblk_size_limit`就认为满了, 就分配一个新的子超级块.

zone初始化：
```c
static void zns_init_zoned_state(NvmeNamespace *ns)
{
    ...
    zone = n->zone_array;
    for (i = 0; i < n->num_zones; i++, zone++) {
        if (start + zone_size > capacity) {
            zone_size = capacity - start;
        }
        zone->d.zt = NVME_ZONE_TYPE_SEQ_WRITE;
        zns_set_zone_state(zone, NVME_ZONE_STATE_EMPTY);
        zone->d.za = 0;
        zone->d.zcap = n->zone_capacity;
        zone->d.zslba = start;
        zone->d.wp = start;
        zone->w_ptr = start;
        start += zone_size;
        
        #ifdef BALLOON_ZNS
        // added by znbc, init the sub-superblocks mapped by each zone
        zone->num_ssblk = 0;
        zone->ssblk = g_malloc0(sizeof(u_int64_t) * SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO);
        zone->ssblk_idx = 0;
        zone->pfwd = g_malloc(sizeof(ProfilingWindow) * (ZONE_SIZE_TO_PROFILING_WINDOW_SIZE_RATIO + 1));
        for(int j = 0; j < ZONE_SIZE_TO_PROFILING_WINDOW_SIZE_RATIO; j++){
            memset(zone->pfwd[j].percentile_cnt, 0, sizeof(zone->pfwd[j].percentile_cnt));
            zone->pfwd[j].len = 0;
            zone->pfwd[j].slot_size_percentile = INITIAL_SLOT_SIZE_TO_PAGE_SIZE_PERCENTILE;
            zone->pfwd[j].slot_size_bs = UPPER(LOGICAL_PAGE_SIZE * INITIAL_SLOT_SIZE_TO_PAGE_SIZE_PERCENTILE / 100, SLOT_SIZE_BASE);
        }
        #ifdef BALLOON_ZNS_RESIDUE
        zone->num_exssblk = 0;
        zone->exssblk = g_malloc0(sizeof(u_int64_t) * SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO);
        zone->exssblk_idx = 0;
        #endif
        #endif
    }
    ...
}
```

##### 获取子超级块

```c
// added by znbc, get a free sub-superblock
static uint64_t get_subsuperblock(FemuCtrl *n, uint32_t zone_idx){
    struct zns_ssd *zns = n->zns;
    for(int i = 0; i < zns->num_ssblk; i++){
        if(! zns->ssblk[i].used){
            zns->ssblk[i].used = 1;
            zns->ssblk[i].to_zone = zone_idx;
            femu_debug("[znbc] zns.c::get_subsuperblock : give ssblk(%d) to zone(%d).\n", i, zns->ssblk[i].to_zone);
            return i;
        }
    }
    femu_err("ERROR [znbc] zns.c::get_subsuperblock : Cannot get a free sub-superblock for zone(%d)!\n", zone_idx);
    for(int i = 0; i < zns->num_ssblk; i++){
        if(zns->ssblk[i].used){
            femu_debug("%d,", zns->ssblk[i].to_zone);
        }
    }
    femu_debug("\n[znbc]  zns->num_ssblk = %ld\n.", zns->num_ssblk);
    assert(0);
    return -1;
}
```

方法就是枚举zns的所有子超级块并选择一个未使用的分配.如果都使用了则直接报错.

##### zone清空

```c
static void zns_clear_zone(NvmeNamespace *ns, NvmeZone *zone)
{
    FemuCtrl *n = ns->ctrl;
    uint8_t state;

    zone->w_ptr = zone->d.wp;

    #ifdef BALLOON_ZNS
    // added by znbc
    struct zns_ssd *zns = n->zns;
    for(int i = 0; i < zns->num_ssblk; i++){
        if(zns->ssblk[i].used){
            zns->ssblk[i].used = 0;
            zns->ssblk[i].to_zone = 0;
            zns->ssblk[i].is_ext = 0;
        }
    }
    zone->num_ssblk = 0;
    zone->ssblk_idx = 0;
    #ifdef BALLOON_ZNS_RESIDUE
    zone->num_exssblk = 0;
    zone->exssblk_idx = 0;
    #endif
    #endif
    ...
}
```

#### 核心函数的改动

##### 子超级块对zns_nvme_rw的改动

子超级块设计没有改变这里的代码.

##### 子超级块对zns_wc_flush的改动

此函数本身并没有子超级块,但是其中的子函数`get_new_page_bz`, `zns_advance_write_pointer`在加入子超级块后都有比较大的改动,见上面代码.

## slot相关

### 分析窗口 profiling windows 以及slot大小计算

在`zns.c::zns_nvme_rw`函数中，需要加入分析窗口，压缩过程每次压缩一个分析窗口，然后计算插槽大小，并用于下一个分析窗口的计算。为此还需要维护一个记录CR分布的桶数组。

顺序写的测试发现，设定bs为128k时，一次`zns_nvme_rw`的sg的len是4096，单位是字节，然后nsg是32，乘起来就是128k。如果设定bs=256k，则nsg变成64，设定512k则变成128。这是为了保证`len*nsg`的值等于bs大小。

分析窗口的大小，按论文来设定的话是33554432B。假设bs=256k，则一个分析窗口会包含128次对`zns_nvme_rw`函数的调用(33554432/256k=128)。分析窗口数量不多，并且同一个zone的不同分析窗口可能会有不同的插槽大小，后续都会用到，所以选择在每个zone里都记录这个zone的所有分析窗口结构体，结构体中有窗口插槽大小以及一个记录CR分布的桶数组。

分析窗口结构体我定义如下:

```c
// added by znbc, profiling window for Balloon-ZNS
typedef struct ProfilingWindow {
    uint64_t len; // current length of this profiling window
    uint64_t percentile_cnt[101]; // counts of different compressed-page-size to page-size percentile
    uint32_t slot_size_percentile;
    uint32_t slot_size_bs; // bytes. base is 256B (SLOT_SIZE_BASE in zns.h)
}ProfilingWindow;
```

我写了一个`zns_get_pfwd_id_by_slba`函数可以通过slba获取对应的profiling window的id，就不需要在zone内记录当前分析窗口。

```c
// added by znbc, get profiling window id by start lba
static inline uint32_t zns_get_pfwd_id_by_slba(NvmeNamespace *ns, uint64_t slba)
{
    FemuCtrl *n = ns->ctrl;
    uint32_t zone_idx = zns_zone_idx(ns, slba);
    uint64_t zslba = zone_slba(n, zone_idx);
    return (slba - zslba) / (n->zone_size / ZONE_SIZE_TO_PROFILING_WINDOW_SIZE_RATIO);
}
```

**插槽大小计算**。这个应该比较简单，我思路是记录每个百分比的数量，然后最后扫一遍取出我们想要的那个百分比。而且这个插槽大小本身不需要完全精确，分析窗口也不需要刚好被用完，只需要保证每个分析窗口都有一个插槽大小即可，如果当前写入数据量已经超出当前分析窗口，不需要中途切换到下一个（不过其实中途切换也挺简单，编号加1就行，但我想假设整个一次`zns_nvme_rw`写入期间分析窗口编号不变）。

相关代码都体现在zns.c::zns_nvme_rw函数:

```c
/*Misao: backend read/write without latency emulation*/
static uint16_t zns_nvme_rw(FemuCtrl *n, NvmeNamespace *ns, NvmeCmd *cmd,
                           NvmeRequest *req,bool append)
{
    ...
    // 初始化阶段,获取分析窗口
    NvmeZone *zone;
    zone = zns_get_zone_by_slba(ns, slba);

    // added by znbc
    uint32_t pfwd_id = zns_get_pfwd_id_by_slba(ns, slba);

    ...
    // 完成写入和压缩后
    // added by znbc, update the profiling window
    for(int j = 0; j < nsg; j++){
        zone->pfwd[pfwd_id].percentile_cnt[req->compressed_size[j] * 100 / len] ++;
    }
    zone->pfwd[pfwd_id].len += len * nsg;
    if(zone->pfwd[pfwd_id].len >= n->zns->profiling_window_size){
        // calculate the slot size
        uint64_t cnt = 0, tot = n->zns->profiling_window_size / len;

        for(int k = 0; k <= 100; k++){
            cnt += zone->pfwd[pfwd_id].percentile_cnt[k];
            if(cnt * 100 >= tot * CR_VALUE_PERCENTILE){

                if(!k) continue;
                // update the next profiling window
                zone->pfwd[pfwd_id + 1].slot_size_percentile = k;
                
                zone->pfwd[pfwd_id + 1].slot_size_bs = UPPER((LOGICAL_PAGE_SIZE * k / 100), SLOT_SIZE_BASE); // same as Balloon-ZNS paper
                if(zone->pfwd[pfwd_id + 1].slot_size_bs > LOGICAL_PAGE_SIZE) zone->pfwd[pfwd_id + 1].slot_size_bs = LOGICAL_PAGE_SIZE;

                break;
            }
        }
    }

    assert(len == LOGICAL_PAGE_SIZE); // So that the nsg is the number of logical pages.(Not necessary. If not, the codes below may need change).
    uint64_t secs_per_pg = LOGICAL_PAGE_SIZE/n->zns->lbasz;
    uint64_t start_lpn = slba / secs_per_pg;
    uint64_t end_lpn = (slba + nlb - 1) / secs_per_pg;

    for (uint64_t lpn = start_lpn, i = 0; lpn <= end_lpn; lpn++, i++){
        struct slot_bz *slot = &n->zns->slots[lpn];
        slot->slot_size_bs = zone->pfwd[pfwd_id].slot_size_bs;
        slot->have_residue = (req->compressed_size[i] > zone->pfwd[pfwd_id].slot_size_bs) ? true :false;
    }
    ...

}
```

省略了一些中间结果输出语句.

### slot大小的使用

知道插槽大小百分比后应该怎么做？

Balloon-ZNS论文:

>为了保持L&Paddr对齐，Balloon-ZNS将连续的闪存页划分为更小的插槽。槽位大小根据数据的可压缩性（参见后续的可压缩性-自适应槽位技术）进行配置。例如，如果插槽大小是页面大小的1/3或2/3，则将一个或两个flash页面分别分区到三个插槽中。每个压缩数据页适合并与一个槽对齐。如果压缩页的内存占用小于插槽，则会浪费插槽中的一些空间。
>如果内存占用比插槽大，页面将被截断，超过插槽的剩余部分将在日志中单独维护（例如，图3中的数据页C）。

>截断的数据页存储在两个单独的flash页中。通过区域级映射表可以找到包含主导部分的第一个flash页面。假设一个flash页面被分成N个槽位，zone0中的一个逻辑地址为L的截断的数据页可以在映射到Zone 0的主子超级块中的第（L/N）个物理页的第（L%N）个槽中找到。为了定位驻留在额外子超级块中的余量，在写入数据页时，Balloon-ZNS将其物理地址和长度作为管理元数据嵌入到前导部分。因此，可以通过按顺序读取两个闪存页来获得截断的数据页。虽然如果维护独立的页级映射元数据以跟踪残留，则可以并行读取两个闪存页，但由于ZNS ssd对RAM的要求过高，因此不考虑这种方法。

fio(其实就是host端任何应用)都不知道我们有内置压缩，所以对于应用，slb和nlb这些东西都是和没有压缩的时候一样。backend_rw写入的时候也是正常的写入。参考师姐建议, 选择在zftl.c中修改zns_write，因为这个是将逻辑页中的数据下刷到实际的物理页中，并更新逻辑页-物理页的映射表。我们就在这里增加逻辑页-slot的映射。

模拟器中对于请求的处理逻辑如下：

**先调用一次`zns.c::zns_advance_zone_wp`然后调用一次`zftl.c::zns_write`，当写入字节数达到写缓存大小(`1024*4KiB`, 是8192个lba大小)时后者会调用`zftl.c::zns_wc_flush`并轮到一系列的`zftl.c::get_new_page_bz`操作，然后又继续回到`zns_advance_zone_wp`，以此往复**

画图展示就是这样:

![请求处理逻辑图](2.drawio.png)

实际的写入发生在zns.c::zns_nvme_rw，写入的地址就是请求中slba对应的字节地址加上控制器后端逻辑地址mbe。当写入字节数达到写缓存大小时才发生flush并设定映射，flush中是不进行实际的写入或拷贝的。

我的实现思路是用类似于lpn-ppa的映射表去记录lpn和slot的对应关系.

在zns_nvme_rw函数就处理slot的大小并且在zns里记录一个lpn到slot的映射表数组.

做flush的时候，通过lpn获取对应slot进而获取大小，然后每次填充分配的一个空闲物理页.注意, 论文中3.1节提到slot需要和256B对齐,为了简化存储管理.

比如一个设定好pg和spg的ppa是4KB，假设slot百分比是67%，就用67%乘4KB，小数部分向上取整并与256对齐，得到一个字节数，填充该物理页，然后这个物理页剩余的空间直接用于下一个物理页。剩余空间量累积起来就可能超过一个slot大小从而可以直接容纳下一个slot，从而可以节省本应分配的物理页，减少延迟（分析`zns_advance_status`可以得知，同一个plane每次调用该函数都会积累延迟，最终的延迟会取每个plane延迟的最大值，所以减少该函数调用次数就可以减少延迟）。

原有的lpn-ppa映射表的含义需要改变，因为一个ppa可能包含多个slot，不过这个表应该仍然可以存在，同时作为lpn-ppa和slot-ppa的映射表。

**关于读取, 暂时不做考虑.**

我的这种方法和论文的实现方法有所不同,我的可能更简单暴力一点,如果存在问题后续再分析改正.

我的具体代码实现思路：

1. 在zns.c::zns_nvme_rw，先assert(sg.len == LOGICAL_PAGE_SIZE)，对于简单的fio测试是成立的,不成立时再讨论。然后每次就是nsg个逻辑页，即nsg个slot，每个slot需要有一个编号，我希望slot和lpn一一对应，所以当场根据slba求出每个slot的lpn。在zns结构体加一个数组，记录每个slot的residue有几个slot大小（residue具体怎么处理目前不管），这个数组可以类比maptbl，元素个数应该是一样的。另外还要在这里处理slot大小，以宏定义的256B为单位向上取整，记录在分析窗口结构体内。
2. 在zftl.c::zns_flush里，做的事情就是我上面说的，为了进一步简化代码，我直接在slot里记录当前slot大小。那么写缓存满的时候未必真的写了`1024*4KB`. 核心目标就是减少物理页的获取从而减少延迟。对于跨两个ppa的slot，我让它映射到第一个ppa。

以上只是实现最简单最基本的功能。做完之后理论上应该可以降低写延迟。读取方面后续再考虑。

`zftl.c::zns_flush`的相关代码如下:

```c
static uint64_t zns_wc_flush(struct zns_ssd* zns, int wcidx, int type,uint64_t stime, FemuCtrl *n)
{
    ...
    for(j = 0; j < flash_type ;j++)
    {
        ppa.g.pg = get_blk(zns,&ppa)->page_wp;
        get_blk(zns,&ppa)->page_wp++;
        for(subpage = 0;subpage < ZNS_PAGE_SIZE/LOGICAL_PAGE_SIZE;subpage++)
        {
            if(i >= zns->cache.write_cache[wcidx].used)
            {
                //No need to write an invalid page
                break;
            }
            ppa.g.spg = subpage;
            lpn = zns->cache.write_cache[wcidx].lpns[i]; // this is also a slot-id
            bool map_last_ppa = (ppa_res > 0)? true: false;
            ppa_res += LOGICAL_PAGE_SIZE;
            while(ppa_res >= zns->slots[lpn].slot_size_bs){
                ppa_res -= zns->slots[lpn].slot_size_bs;
                oldppa = get_maptbl_ent(zns, lpn);
                if (mapped_ppa(&oldppa)) {
                    /* FIXME: Misao: update old page information*/
                }
                if(map_last_ppa == true){
                    set_maptbl_ent(zns, lpn, &last_ppa);
                    map_last_ppa = false;
                }
                else
                    set_maptbl_ent(zns, lpn, &ppa);
                #ifdef BALLOON_ZNS_RESIDUE
                if(zns->slots[lpn].have_residue){
                    struct nand_cmd swr;
                    swr.type = type;
                    swr.cmd = NAND_WRITE;
                    swr.stime = stime;
                    /* get latency statistics */
                    struct ppa ppa_r = get_residue_page(zns);
                    sublat = zns_advance_status(zns, &ppa_r, &swr);
                    maxlat = (sublat > maxlat) ? sublat : maxlat;
                }
                #endif
                i++;
                if(i >= zns->cache.write_cache[wcidx].used){
                    //No need to write an invalid page
                    break;
                }
                lpn = zns->cache.write_cache[wcidx].lpns[i];
            }
            last_ppa = ppa;
        }
    }
    ...
}
```

## residue

>如果内存占用比插槽大，页面将被截断，超过插槽的剩余部分将在日志中单独维护.我们将分槽的子超级块称为**主子超级块**，将用于记录超大页残余的子超级块称为**额外的子超级块**。主子超级块专门映射到每个区域，而额外的子超级块可以由多个区域共享。一个额外的子超级块被一个开放区域拥有并且只能由它写入，直到它关闭。然后，可以将额外子超级块分配到另一个新开放的区域，来append其残基。通过这样做，每个区域的残数被记录在连续的物理空间中，即连续独立的一组flash页。
>截断的数据页存储在两个单独的flash页中。通过区域级映射表可以找到包含主导部分的第一个flash页面。假设一个flash页面被分成N个槽位，zone0中的一个逻辑地址为L的截断的数据页可以在映射到Zone 0的主子超级块中的第（L/N）个物理页的第（L%N）个槽中找到。为了定位驻留在额外子超级块中的余量，在写入数据页时，Balloon-ZNS将其物理地址和长度作为管理元数据嵌入到前导部分。因此，可以通过按顺序读取两个闪存页来获得截断的数据页。虽然如果维护独立的页级映射元数据以跟踪残留，则可以并行读取两个闪存页，但由于ZNS ssd对RAM的要求过高，因此不考虑这种方法。

目前还是不做读优化，先只考虑写。

一切从简单出发。我的想法就是：

1. zns有一个ssblk数组，这个数组从末尾往前分配额外子超级块，zns要记录当前可用额外子超级块编号并记得更新。
2. 每个zone记录其额外子超级块的编号。这些信息足够为residue调用`get_new_page`。考虑到理论上可能一个zone有多个额外子超级块，我也记录总数量和当前id。规定最大数量和ssblk数量一样，这里是4，就是说占的额外子超级块再大也不至于比zone还大吧。
3. 然后在flush里面，每次取出一个物理页ppa然后处理slot的时候，对于每个有residue的slot，额外创建一个临时的ppa_r，然后根据已有信息调用对应额外子超级块的一个物理页。
4. 这里需要为此设计一个新函数，可以对比`get_new_page`。如果满了，就从zns的倒数下一个额外子超级块进行取用。
5. 然后在flush中，使用新取出的物理页更新延迟。至于这个residue的数据是否真的写入这个物理页，我只能说并没有，目前这里只是起到一个模拟计算延迟的效果。

关于论文中说的，“额外子超级块被一个开放区域拥有并且只能由它写入，直到它关闭。然后，可以将额外子超级块分配到另一个新开放的区域。”这种灵活的设计就先不做了，现在就先假设额外子超级块一旦分配就永远属于某个zone。有什么问题以后再改。

这里的一个问题是，`get_new_page`每次取出物理页会赋予ch、lun、blk，而只有ssblk信息的话ch是不知道的，ch是根据zns内部的写指针获得的，每次每次完所有plane后都会更新。exssblk作为ssblk也是有ch级别的并行性的，所以虽然说是连续写入，实际上应该也是可以多个ch轮着写入。不过每次具体使用哪个ch呢？我这里直接也使用zns的写指针了，只是可以一定程度体现ch级别的并行性。不过这样的话同一个ch下可能有一些连续的slot不能体现ch并行性（本来可以体现），但反正这里plane就2个，一个ch对应的物理页也就两个，影响应该不大。

不过这里flush函数中zone的编号不太好获取。所以感觉这个额外子超级块也可以直接与zone无关，就是根据当前zns的额外子超级块编号获取exssblk。这个做法和上面的做法的实际效果应该类似，因为上面也是从后往前分配额外子超级块，实际使用的额外子超级块应该是一样的。如果以后真的要读取residue的实际内容，这部分肯定要改，关键是slot如何得到其residue的位置信息，现在不谈。

相关代码:

获取残留页:

```c
static struct ppa get_residue_page(struct zns_ssd *zns){
    struct ppa ppa;
    struct write_pointer_bz *wpp = &zns->wp_bz;
    u_int64_t exssblk = zns->now_exssblk;
    //femu_debug("zftl.c::get_residue_page: exssblk=%lu\n", exssblk);
    if(zns->ssblk[exssblk].write_pointer >= zns->ssblk_size_limit){
        if(zns->now_exssblk == 0 || zns->ssblk[ zns->now_exssblk -1].used == true){
            ftl_debug("[Error] zftl.c::get_residue_page: Do not have more sub-superblocks for residue! Will cause wrong results!\n");
            if(zns->now_exssblk == 0) zns->now_exssblk = 1;
        }
        zns->now_exssblk--;
    }
    ppa.ppa = 0;
    ppa.g.ch = wpp->ch;
    ppa.g.fc = zns->ssblk[exssblk].lun;
    ppa.g.blk = zns->ssblk[exssblk].blk;
    ppa.g.V = 1; //not padding page
    zns->ssblk[exssblk].write_pointer++;
    if(!valid_ppa(zns,&ppa))
    {
        ftl_err("[Misao] invalid ppa: ch %u lun %u pl %u blk %u pg %u subpg  %u \n",ppa.g.ch,ppa.g.fc,ppa.g.pl,ppa.g.blk,ppa.g.pg,ppa.g.spg);
        ppa.ppa = UNMAPPED_PPA;
    }
    return ppa;
}
```

`zns_wc_flush`函数中的额外处理在前面slot部分有展示.

residue总的来说占比不大,对结果的影响也应该不大.不过如果测试数据变复杂, residue导致的空间浪费和垃圾回收开销可能就不容小视了. 

