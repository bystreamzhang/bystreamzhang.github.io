---
title: FEMU模拟的ZNS源码分析以及Balloon-ZNS复现优化记录
date: 2025-02-23 17:12:58
tags:
---
<link rel="stylesheet" type="text/css" href="../../css/auto-number-title.css" />
<link rel="stylesheet" type="text/css" href="../css/auto-number-title.css" />
<link rel="stylesheet" type="text/css" href="/css/auto-number-title.css" />

# 说明

本文为做毕设项目时的笔记。

[FEMU官方文档](https://github.com/MoatLab/FEMU)
主体代码在hw/femu。
代码分析可参考[0 FEMU 简介 | ["FEMU闪存仿真平台介绍及使用文档"]](https://nnss-hascode.github.io/FEMU-Doc/FEMU-Doc.html)

[FEMU论文](https://www.usenix.org/conference/fast18/presentation/li)  
[Balloon-ZNS论文](https://dl.acm.org/doi/10.1145/3649329.3657368)

主要分析代码：hw/femu/zns下的代码。

暂不分析所有源码，主要是对Balloon-ZNS在ZNS上有所改动的地方进行分析说明，记录我当前的改动方法。（后续可能修改）

有两个读写、延迟计算相关的核心函数需要特别关注：`zns.c::zns_nvme_rw`, `zftl.c::zns_wc_flush`。

## 核心函数说明

这里先只说明femu模拟的原版zns。

### zns.c::zns_nvme_rw

此函数的工作主要分以下几步：

1. 从传入的请求req中获取start logical block address(slba)以及number of logical block(nlb)，以及请求是读or写。

> 关于lb：逻辑块是比逻辑页还小的单位，一个lb为512B。lb是`zone->w_ptr`、`zone->d.wp`、`n->zone_size`等变量的单位。
>
> lba不是以字节为单位，而是以lb为单位。slba=524288代表第524288个lb。可以通过函数转为具体字节地址。

2. 根据slba获得对应zone指针(使用`zns_get_zone_by_slba`函数)。
3. 通过`zns_check_zone_write`等函数初步检查这个请求是否合法，比如zone是否装得下nlb个lb。
4. 通过slba获得具体要读/写的内存逻辑地址。
5. 从req获取QEMUSGList信息，包括每个sg的长度和总数量。
6. 打包既有信息调用`backend_rw(n->mbe, &req->qsg, &data_offset, req->is_write);`函数进行实际的读/写。
7. 对于写操作，检查zns的新状态。
8. 更改`n->zns->active_zone`为slba对应的zone的编号。

### zftl.c::zns_wc_flush

此函数的工作主要分以下几步：

1. 

## 子超级块相关

### 宏

这里模拟器设定die数量为4，我规定一个超级块有4个子超级块。

```c
#define SUPERBLOCK_TO_SUBSUPERBLOCK_RATIO 4
```

### 结构体

此处代码均在zns.h。

超级块本身没有结构体，zns的写缓存会记录当前超级块的id。

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

原版就是获取zns的当前写指针，从中获取物理页的各种信息，包括channel、flash chip(die)、block等。这里并没有设定好ppa的所有信息，比如没有设定plane、page、subpage，这些会在后续设定 (见`zftl.c::zns_wc_flush`)。

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

balloon-zns版本如下。

```c
// add by znbc
static struct ppa get_new_page_bz(struct zns_ssd *zns, FemuCtrl *n)
{
    struct write_pointer_bz *wpp = &zns->wp_bz;
    NvmeZone *zone = &n->zone_array[zns->active_zone];
    u_int64_t ssblk = zone->ssblk[zone->ssblk_idx];
    struct ppa ppa;
    femu_debug("[znbc] zftl.c::get_new_page_bz : active_zone=%d zone->ssblk_idx=%ld, zns->ssblk[%ld].write_pointer=%ld zns->ssblk_size_limit=%ld\n",\
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

本来，只需要wp和active_zone的信息就可以完成get_new_page函数，程序运行时写指针wp就不断前进并在不同的ch和lun(die)跳跃，模拟并行，
而blk就是zns的当前活跃zone编号。

现在引入子超级块以后，一个zone的block未必覆盖所有die所以wp不能顺序跳跃，zone和block也不再对应，die0的block0和die2的block0可能属于不同的zone。
所以现在需要知道子超级块编号才能知道die和blk的值，才能完成get_new_page函数。

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

原本就是先枚举channel然后die，现在先枚举channel以后按道理要在当前子超级块继续枚举，可是无法判断当前子超级块是否满。

调用这个函数的是`zns_wc_flush`函数，其中原本也不会判定当前zone是否满，就是不断调用该前进函数。
**所以如何在这个flush函数的某个位置判断当前子超级块是否满**？

逻辑上，用户发出请求时zone一定是没满的，按原本的做法每个blk使用一个页后轮到下一个blk，如果某个blk满了，访问下一个blk，如果全都满了，
这个zone就是FULL状态，不可能被访问了。所以可以假设flush时zone就是没满的。

我为子超级块设定写指针，记录用了多少。单位是什么？每次get_page会取出一个物理页，就让写指针单位是物理页。
超级块有`ch*lun*plane*blk*num_page`个物理页，子超级块要除以4，当写指针等于这个值时，
说明当前活跃zone的当前子超级块满了，轮到下一个子超级块(没有就要分配一个)。



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

这里，子超级块的`write_pointer`属性是用来判断是否用满的，
每次`get_new_page_bz`获取新页就会加1，如果达到`zns->ssblk_size_limit`就认为满了，就分配一个新的子超级块。

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

方法就是枚举zns的所有子超级块并选择一个未使用的分配。如果都使用了则直接报错。

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
            femu_debug("[znbc] zns.c::zns_clear_zone : reset ssblk(%d).\n", i);
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

##### zns.c::zns_nvme_rw

子超级块设计没有改变这里的代码。

##### zftl.c::zns_wc_flush

