---
title: ZNS SSD 学习记录2
date: 2025-06-15 20:55:46
tags:
    - ZNS
    - SSD
categories: Study
---
# 说明

ZNS SSD的第二篇笔记，第一篇是比较初期写的，认识比较浅，并且其中提到的FlexZNS暂时不打算研究了。

这一篇希望从头介绍ZNS SSD的知识，然后重点学习ZenFS和RocksDB在ZNS SSD中扮演的角色。

## 论文

ZNS SSD核心论文: [ZNS: Avoiding the Block Interface Tax for Flash-based SSDs](https://www.usenix.org/conference/atc21/presentation/bjorling)

这篇论文有多重要呢？Linux中对ZNS SSD的支持、ZenFS、fio对ZNS SSD的支持等，都属于本论文的工作内容，在论文里是有介绍的。所以必学。

## 标准

NVMe有一个ZNS指令集标准，可在[NVMe](https://nvmexpress.org/specifications/)下载

西部数据有一个关于Zone存储的网站：[zonedstorage.io](https://zonedstorage.io/docs/introduction)，提供分区存储原则、底层存储设备技术等信息以及Linux生态系统相关支持的概述

我感觉前者就是只从ZNS的角度进行说明，目的是规范ZNS接口。后者是说ZNS如何用于存储，相关的各种知识都会讲。

## ZNS 核心论文学习

### 背景

1. 块接口：把（）以（）形式暴露出来，好处是（）。但问题是（）

## NVMe ZNS命令集规范学习

### 说明

本规范定义了特定于分区存储空间命令集的要求和行为。通常适用于NVMe或适用于多个I/O命令集的功能在NVM Express基本规范中定义。

> 不同的NVMe规范是有优先级的，规范发生冲突时，最底层、编号最小的最优先。底层规范应该都适用于上层规范，所以ZNS SSD除了遵守ZNS接口规范也要遵守底层规范，比如NVM Express Base Specification。规范中在1.4节也说明了一些定义是来自底层规范还是本规范特定。

### 定义

就解释几个我一开始没懂的，或者比较重要的。

#### Address-Specific Write Command

**与特定逻辑块地址（LBA）绑定的写入类命令**。这类命令会改变指定 LBA 地址上的数据状态。

* **包括的命令** （规范明确列出）：
  * `Write`：向指定 LBA 写入数据。
  * `Write Uncorrectable`：将指定 LBA 标记为“不可纠正错误”（模拟物理损坏）。
  * `Write Zeroes`：将指定 LBA 写入全 0 数据。
  * `Copy`：将源 LBA 的数据复制到目标 LBA（本质是写入目标 LBA）。

注意：zone append command不属于此类，虽然也属于写指令。

#### zone

作为一个单元管理的连续的逻辑块地址范围。

#### Zone Descriptor

包含区域信息的数据结构

#### Zone Descriptor Extension

由**主机**定义的区域关联数据

#### zoned namespace

一个命名空间，被划分为多个区域，并与ZNS命令集相关联。
