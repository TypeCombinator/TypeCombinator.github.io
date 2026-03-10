---
title: 平衡树之巅，超越FreeBSD Weak AVL
subtitle:
date: 2026-03-09T11:28:51+08:00
slug: beyond_freebsd_weak_avl
draft: false
author:
  name: TypeCombinator
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - blog
categories:
  - algorithm
  - c++
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---
在之前那篇关于WBT（Weight-Balanced Tree）的文章中，我们看到FreeBSD的Weak AVL表现出了比Linux的Red-Black Tree更好的元素插入性能，但直觉告诉我这种性能差异可能是具体实现造成的。由于初学红黑树时给我留下了实现复杂的坏印象，很长一段时间里我都在刻意避开它，好在最近我遇到了[Revisiting 2-3 Red-Black Trees with a Pedagogically Sound yet Efficient
Deletion Algorithm: The Parity-Seeking Delete Algorithm](https://arxiv.org/pdf/2004.04344)这篇论文（**推荐直接看原论文**），让笔者顺利捡起了这些旧知识。
<!--more-->
## 更好懂的红黑树删除算法：Parity-Seeking

红黑树的复杂主要体现在节点删除算法，因此本文选择跳过节点插入算法。红黑树有诸多变体，如果将红节点的约束从强到弱排序有：左倾红黑树`==`AA树`>`2-3红黑树`>`2-3-4红黑树，为方便描述，本文后续内容限定在2-3红黑树和2-3-4红黑树，这两种红黑树的删除算法是一样的，这也是本文将要介绍的主角：**Parity-Seeking**，如果要翻译为中文，笔者认为"**找平**"一词可能比较贴切。

###  简单情形

当需要删除红黑树中的任一节点时，通过用前驱或者后继节点替换待删除节点，最终都可以转换为删除一个度小于等于1的节点。当删除一个度为1的节点，根据红黑树的性质，该节点必然是只有一个红色叶子节点的黑色节点，当删除该节点，直接用叶子替换即可，如**Figure11**所示。

![](/blog/beyond_freebsd_weak_avl/figure11.png)

<center><b>Figure11</b></center>

当删除一个叶子节点，即度为0的节点，如果是删除的红色叶子节点，由于不会改变黑高，直接删除即可，如**Figure12**所示。

![](/blog/beyond_freebsd_weak_avl/figure12.png)

<center><b>Figure12</b></center>

### 一般情形

需要进一步处理的是删除黑色叶子节点，为了解决该问题，需先定义**缺陷子树**：以节点`x`为根的缺陷子树，该子树的黑高比`x`兄弟节点`y`表示的子树黑高少1。而**Parity-Seeking**删除算法大体思路为：要么修复缺陷子树`x`，使其黑高加1；要么降低兄弟子树`y`的黑高，将缺陷传递到父节点。这里有3种情形：
1. `x`是红色。此时，只需要将`x`染黑就能修复缺陷，如**Figure13a**所示。
![](/blog/beyond_freebsd_weak_avl/figure13a.png)

<center><b>Figure13a</b></center>

2. `x`和`y`都是黑色。此时，将`y`染为红色，子树`x`和`y`的黑高一致，缺陷子树将转移到父节点，并将原来的`y`重新标记为`z`，如**Figure13b**所示。这可能违反红黑树的定义，因为`z`可能有红色的子节点，此时需要一些**修复规则**。
![](/blog/beyond_freebsd_weak_avl/figure13b.png)

<center><b>Figure13b</b></center>

3. `x`是黑色，而`y`是红色。此时，需要进行一次单旋，当`x`是左子节点，那么对`x`的父节点做一次左旋，否则，对`x`的父节点进行右旋，如**Figure13c**所示，注意旋转之后`x`和`y`的位置，代表`x`的节点没有改变。
![](/blog/beyond_freebsd_weak_avl/figure13c.png)

<center><b>Figure13c</b></center>

### 修复规则

对于**情形2**，缺陷子树的兄弟节点染为了红色，而兄弟节点有红色的子节点时将违反红黑树的定义，此时将兄弟节点记为 `z`，缺陷子树已转移到父节点，记为`x`，这需要一些旋转操作，如**Figure15a**和**Figure15b**所示。
![](/blog/beyond_freebsd_weak_avl/figure15a.png)

<center><b>Figure15a</b></center>

![](/blog/beyond_freebsd_weak_avl/figure15b.png)

<center><b>Figure15b</b></center>

以上的旋转规则和插入的旋转规则是一致的，当`z`的红子节点和`z`一样都是左子节点或都是右子节点，则进行一次**Figure15b**所示的单旋，否则，先进行一次**Figure15a**的单旋，将其转换为**Figure15b**的情形，这对应了双旋。经过**Figure15a**和**Figure15b**的旋转处理，缺陷子树`x`将有2个红色子节点。此时，只需要将这2个红子节点染黑，缺陷子树`x`的黑高加1，即完成修复。

![](/blog/beyond_freebsd_weak_avl/figure15c.png)

<center><b>Figure15c</b></center>

到此，我们几乎已经覆盖了删除所有情形，那么如果缺陷子树`x`传递到了根节点呢？显然，此时什么都不必做，这代表着整个红黑树的黑高减少了1。

### 小结

**Parity-Seeking**用配平黑高这一思路贯穿始终，让删除算法变得更易理解了。该算法的关键不同在于**情形2**中的将缺陷子树的黑色兄弟节点染红这一步骤，这让后续的旋转操作变得合理起来。

## 实践

严格来讲，**Parity-Seeking**删除算法也许并不能称之为一种新的算法，其实现代码在适当优化调整之后，与经典的红黑树删除算法其实是完全等价的。这并不是在否定其价值，当再看经典的红黑树删除算法时，笔者的脑海自然就带入了**Parity-Seeking**的思路，如果教科书上的删除算法一开始就是如此，也许笔者也不会有长时间的坏印象。论文为了实现简洁，使用了哨兵节点`nilSentinel`这一技巧，这与笔者先前在WBT的实现中对哨兵节点的用法不同，该`nilSentinel`不止用于叶子节点，还用在了根节点上，这要求`nilSentinel`是动态的，即每颗树都要分配一个哨兵节点，即使相同类型的树也不能共用。经过权衡，笔者并没有采用任何哨兵节点的优化，只用传统的方式将2-3红黑树和2-3-4红黑树重新实现了一遍。

### 元素插入

笔者在AMD64的机器上，使用GCC 14.2（补充一下，之前的那篇关于WBT的文章用的GCC 11.4），用Google Benchmark进行性能测试，随机元素插入性能如下图，纵轴用总时间（cpu_time）除以元素个数得到该数据规模下每个元素的平均消耗时间（单位是纳秒），再将该时间取以10为底的对数。

![](/blog/beyond_freebsd_weak_avl/insert_bench.png)

Google Benchmark输出的原始数据如下表：

| scale | freebsd irbt | 2-3 irbt | 2-3-4 irbt | linux irbt |
| ---- | ---- | ---- | ---- | ---- |
|1024| 33038 | 22540 | 25807 | 36232 |
|2048| 90293 | 69279 | 72434 | 95542 |
|4096| 201734 | 174475 | 179115 | 242430 |
|8192| 475654 | 427997 | 428936 | 541894 |
|16384| 1190172 | 1024084 | 1065317 | 1321475 |
|32768| 3024638 | 2522696 | 2756564 | 3308750 |
|65536| 7517172 | 6094355 | 6228935 | 7766550 |
|131072| 17042586 | 14716441 | 14788657 | 19982307 |
|262144| 43322081 | 39009192 | 38203664 | 48119577 |
|524288| 135570023 | 118641589 | 129199589 | 142302555 |
|1048576| 506609803 | 401441028 | 450359840 | 433564527 |
|2097152| 1220509049 | 1171674691 | 1209221666 | 1246632229 |

以上可以看到，2-3红黑树和2-3-4红黑树的元素插入性能显著优于FreeBSD Weak AVL，特别是在数据规模小于100万的时候，而2-3红黑树的性能是最佳的。那么如果用有序数据进行测试呢？

![](/blog/beyond_freebsd_weak_avl/ordered_insert_bench.png)

如上图所示，在有序数据的场景下，笔者实现的2-3红黑树和2-3-4红黑树的性能彻底垮掉了，为什么呢？原因是笔者使用了cmov指令对一些分支进行了优化，而有序数据会让分支预测器非常Happy，cmov在此场景下则没有优势。解决的办法也很简单，用一个**模板参数**控制，随机数据用cmov，有序数据就用分支跳转。当使用分支跳转时，笔者实现的红黑树与Linux的红黑树性能基本一致。

### 元素删除

随机元素删除性能测试如下图所示：

![](/blog/beyond_freebsd_weak_avl/remove_bench.png)

Google Benchmark输出的原始数据如下表：

| scale | freebsd irbt | 2-3 irbt | 2-3-4 irbt | linux irbt |
| ---- | ---- | ---- | ---- | ---- |
|1024| 19097 | 13704 | 14004 | 13611 |
|2048| 49802 | 35885 | 35030 | 39257 |
|4096| 108776 | 87209 | 77850 | 89718 |
|8192| 227691 | 191707 | 166581 | 202077 |
|16384| 517042 | 423716 | 375053 | 440918 |
|32768| 1100035 | 953313 | 842491 | 953575 |
|65536| 2388056 | 2066778 | 1837172 | 2033688 |
|131072| 5418714 | 4456451 | 3912994 | 4308811 |
|262144| 13342784 | 12142352 | 11036054 | 11426325 |
|524288| 50113463 | 53031105 | 45086169 | 40338798 |
|1048576| 152065989 | 161203959 | 153232832 | 135551903 |
|2097152| 405734969 | 439995440 | 416865849 | 360217372 |

以上可以看到，2-3-4红黑树的删除性能在小于50万数据规模的场景下性能是最好的，在更大的数据规模时，性能有一定的劣化，此处笔者未再深究。

## 总结

**Parity-Seeking**让红黑树删除算法变得更直观易懂，其本质与经典删除算法等价，并不会带来实际性能的提升。虽然红黑树和Weak AVL在不同的实现下表现出一定的性能差异，但笔者认为二者仍然在伯仲之间，当需要极致的元素增删性能时，可任选其一。