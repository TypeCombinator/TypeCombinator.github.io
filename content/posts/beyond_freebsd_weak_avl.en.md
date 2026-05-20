---
title: Beyond FreeBSD Weak AVL
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
In a previous article about WBT (Weight-Balanced Trees), we saw that FreeBSD's Weak AVL demonstrated better element insertion performance than Linux's Red-Black Tree. However, my intuition told me that this performance difference might stem from specific implementations. Since learning Red-Black Trees left me with a bad impression of their complexity, I deliberately avoided them for a long time. Fortunately, I recently came across the paper [Revisiting 2-3 Red-Black Trees with a Pedagogically Sound yet Efficient Deletion Algorithm: The Parity-Seeking Delete Algorithm](https://arxiv.org/pdf/2004.04344) (**I recommend reading the original paper directly**), which helped me smoothly pick up this old knowledge.
<!--more-->
## A More Understandable Red-Black Tree Deletion Algorithm: Parity-Seeking

The complexity of Red-Black Trees is mainly concentrated in the node deletion algorithm, so this article skips the node insertion algorithm. There are many variants of Red-Black Trees. If we order the constraints on red nodes from strongest to weakest: left-leaning red-black tree `==` AA tree `>` 2-3 red-black tree `>` 2-3-4 red-black tree. For clarity, the subsequent content of this article is limited to 2-3 red-black trees and 2-3-4 red-black trees, both of which share the same deletion algorithm — the protagonist of this article: **Parity-Seeking**. If I were to translate it into Chinese, I think "**找平**" (level-seeking) might be a fitting term.

### Simple Cases

When we need to delete any node in a Red-Black Tree, by replacing the target node with its predecessor or successor, the problem ultimately reduces to deleting a node with degree ≤ 1. When deleting a node of degree 1, by the properties of a Red-Black Tree, this node must be a black node with exactly one red leaf child. Deleting it simply requires replacing it with that leaf, as shown in **Figure11**.

![](/blog/beyond_freebsd_weak_avl/figure11.png)

<center><b>Figure11</b></center>

When deleting a leaf node (degree 0), if it's a red leaf, deletion does not change the black height, so we can simply remove it, as shown in **Figure12**.

![](/blog/beyond_freebsd_weak_avl/figure12.png)

<center><b>Figure12</b></center>

### General Cases

What requires further handling is the deletion of a black leaf node. To address this, we must first define a **deficient subtree**: a subtree rooted at node `x` whose black height is 1 less than the black height of the subtree rooted at its sibling `y`. The general idea of the **Parity-Seeking** deletion algorithm is: either repair the deficient subtree `x` by increasing its black height by 1, or lower the black height of the sibling subtree `y`, passing the deficiency upward to the parent. There are three cases:

1. `x` is red. At this point, simply coloring `x` black repairs the deficiency, as shown in **Figure13a**.
![](/blog/beyond_freebsd_weak_avl/figure13a.png)

<center><b>Figure13a</b></center>

2. Both `x` and `y` are black. At this point, color `y` red. The black heights of subtrees `x` and `y` become equal, the deficient subtree transfers to the parent, and the original `y` is relabeled as `z`, as shown in **Figure13b**. This may violate the Red-Black Tree definition because `z` may have red children, requiring some **repair rules**.
![](/blog/beyond_freebsd_weak_avl/figure13b.png)

<center><b>Figure13b</b></center>

3. `x` is black and `y` is red. At this point, a single rotation is needed. If `x` is a left child, perform a left rotation on `x`'s parent; otherwise, perform a right rotation on `x`'s parent, as shown in **Figure13c**. Note the positions of `x` and `y` after the rotation — the node representing `x` has not changed.
![](/blog/beyond_freebsd_weak_avl/figure13c.png)

<center><b>Figure13c</b></center>

### Repair Rules

For **Case 2**, the sibling of the deficient subtree has been colored red. If the sibling has red children, this will violate the Red-Black Tree definition. At this point, denote the sibling as `z`; the deficiency has already been transferred to the parent, denoted as `x`. This requires some rotation operations, as shown in **Figure15a** and **Figure15b**.
![](/blog/beyond_freebsd_weak_avl/figure15a.png)

<center><b>Figure15a</b></center>

![](/blog/beyond_freebsd_weak_avl/figure15b.png)

<center><b>Figure15b</b></center>

The rotation rules above are consistent with the insertion rotation rules. When `z`'s red child and `z` are both left children or both right children, perform a single rotation as shown in **Figure15b**. Otherwise, first perform a single rotation as shown in **Figure15a** to transform it into the **Figure15b** case — this corresponds to a double rotation. After the rotations in **Figure15a** and **Figure15b**, the deficient subtree `x` will have 2 red children. At this point, simply coloring these 2 red children black increases the black height of the deficient subtree `x` by 1, completing the repair.

![](/blog/beyond_freebsd_weak_avl/figure15c.png)

<center><b>Figure15c</b></center>

At this point, we have almost covered all deletion cases. But what if the deficient subtree `x` propagates all the way to the root? Obviously, nothing needs to be done — this simply means the black height of the entire Red-Black Tree has decreased by 1.

### Summary

**Parity-Seeking** uses the idea of balancing black heights as its unifying thread, making the deletion algorithm much easier to understand. The key difference of this algorithm lies in **Case 2**, where coloring the black sibling of the deficient subtree red makes the subsequent rotation operations justified.

## Practice

Strictly speaking, the **Parity-Seeking** deletion algorithm may not truly qualify as a new algorithm. After appropriate optimization and adjustments, its implementation code is in fact completely equivalent to the classic Red-Black Tree deletion algorithm. This is not to diminish its value — when I revisit the classic deletion algorithm now, my mind naturally applies the **Parity-Seeking** way of thinking. If textbooks had presented the deletion algorithm this way from the beginning, perhaps I wouldn't have held a bad impression for so long. For implementation simplicity, the paper uses a sentinel node `nilSentinel`, which differs from how I previously used sentinel nodes in my WBT implementation. This `nilSentinel` is used not only for leaf nodes but also for the root node, which requires `nilSentinel` to be dynamic — each tree must allocate its own sentinel node, and even trees of the same type cannot share one. After weighing the trade-offs, I chose not to adopt any sentinel node optimization and simply re-implemented both the 2-3 and 2-3-4 Red-Black Trees in the traditional way.

### Element Insertion

I ran performance tests using Google Benchmark on an AMD64 machine with GCC 14.2 (as a note, the earlier WBT article used GCC 11.4). The random element insertion performance is shown in the figure below. The vertical axis represents the average time per element (in nanoseconds) at each data scale, obtained by dividing total time (cpu_time) by the number of elements, then taking the base-10 logarithm.

![](/blog/beyond_freebsd_weak_avl/insert_bench.png)

The raw data from Google Benchmark is shown in the table below:

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

As can be seen above, the element insertion performance of the 2-3 and 2-3-4 Red-Black Trees significantly outperforms FreeBSD Weak AVL, especially at data scales below 1 million, with the 2-3 Red-Black Tree delivering the best performance. But what about testing with ordered data?

![](/blog/beyond_freebsd_weak_avl/ordered_insert_bench.png)

As shown above, in the ordered data scenario, the performance of my 2-3 and 2-3-4 Red-Black Tree implementations completely collapsed. Why? It's because I used `cmov` instructions to optimize some branches, and ordered data makes the branch predictor very happy — `cmov` has no advantage in this scenario. The solution is simple: control this with a **template parameter** — use `cmov` for random data and branch jumps for ordered data. When using branch jumps, the performance of my Red-Black Tree implementations is essentially on par with Linux's Red-Black Tree.

### Element Deletion

The random element deletion performance test results are shown below:

![](/blog/beyond_freebsd_weak_avl/remove_bench.png)

The raw data from Google Benchmark is shown in the table below:

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

As can be seen above, the deletion performance of the 2-3-4 Red-Black Tree is the best at data scales below 500,000. At larger scales, there is some performance degradation, which I didn't investigate further.

## Conclusion

**Parity-Seeking** makes the Red-Black Tree deletion algorithm more intuitive and understandable. Its essence is equivalent to the classic deletion algorithm and does not bring actual performance improvements. Although Red-Black Trees and Weak AVL show some performance differences under different implementations, I believe they remain in the same league. When ultimate element insertion and deletion performance is needed, either one is a fine choice.
