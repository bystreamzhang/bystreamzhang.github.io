---
title: FEMU模拟的ZNS源码分析以及Balloon-ZNS复现优化记录
date: 2025-02-23 17:12:58
tags:
    - FEMU
    - ZNS
    - Balloon-ZNS
categories: Study
---

# 说明

本文为做毕设项目时的笔记.

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

> 关于lb：逻辑块是比逻辑页还小的单位, 一个lb为512B.lb是`zone->w_ptr`、`zone->d.wp`、`n->zone_size`等变量的单位.
>
> lba不是以字节为单位, 而是以lb为单位.slba=524288代表第524288个lb.可以通过函数转为具体字节地址.

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

femu模拟的zns代码中, 除了plane以外的blk,fc,ch等层次的结构体也都有一个计时器,但是只有plane的计时器被使用了,所以其他层次的并行性体现在哪里?

## 并行性

前面讨论`zns_wc_flush`函数时说明了plane级别的并行性体现. 这里讨论其他层次的并行性.

(还待补充, 目前还没搞清楚其他层次的并行性体现在代码的哪里)

## 子超级块相关

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

