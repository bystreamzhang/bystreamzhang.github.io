---
title: Balloon-ZNS优化记录
date: 2025-02-28 18:40:39
tags:
    - FEMU
    - ZNS
    - SSD
    - Balloon-ZNS
categories: Study
---

# 说明

记录对Balloon-ZNS的优化思路和实现.

## 思路1

考虑默认让超级块和额外子超级块共存, slot就存放在超级块, 然后另外将一些超级块分为小的额外子超级块来存放残量. 

zwl师姐建议:

```
这个方法的本质是让zns支持多种并行度的zone。这常见于讨论大zone和小zone的工作中。例如，大zone 2GB，小zone128MB。大zone占据一个superblock，小zone只在一小部分channel上占据一个子superblock。这样不同规格的zone的数量只能在命名空间初始化时就规划好，初始化完成后就不能再更改了。比如说，我们在femu的代码里就写好，占据superblock的zone有x个，占据sub-superblock的zone有y个。然后编译启动femu，进去之后那个设备中的zone就已经按照我们规划的固定好，不能再更改。（关于这个问题，我放了一些论文在下面，你可以参考其中关于大zone小zone的部分。）那么此时有一个问题，我们应该预备多少占据superblock的zone来存储slot？又预备多少sub-superblock的zone来存储residue？如果这两者的zone不够，我们应该如何处理？所以我认为你可以将这个当成一个潜在的优化方向！
```


