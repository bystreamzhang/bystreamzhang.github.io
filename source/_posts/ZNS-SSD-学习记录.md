---
title: ZNS SSD 学习记录
date: 2025-02-27 22:55:46
tags:
    - ZNS
    - SSD
---

# 说明

参考ZNS SSD核心论文: [ZNS: Avoiding the Block Interface Tax for Flash-based SSDs](https://www.usenix.org/conference/atc21/presentation/bjorling) 

记录一些基本知识和学习心得. 目前可能比较零散化, 不一定有前后完整的逻辑, 觉得有必要的时候会进行系统整理, 届时逻辑会完整, 不会突然出现新概念

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
