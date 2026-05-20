---
title: "The Hidden Gem of Rank-Balanced Trees: Weak AVL"
subtitle:
date: 2026-03-31T14:20:25+08:00
slug: rank_balanced_tree_weak_avl
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
In the paper [Rank-Balanced Trees](https://sidsen.azurewebsites.net/papers/rb-trees-talg.pdf), the authors unify AVL trees and Red-Black Trees under the concept of **rank**-balanced trees, and also propose the WAVL (Weak AVL) tree. Compared to Red-Black Trees, where node deletion can require up to 3 rotations (when the sibling of the **deficient subtree** is red), WAVL tree deletion requires at most 2 rotations. Like Red-Black Trees, for a WAVL tree with `n` nodes, the maximum tree height is `2 lg n`.
<!--more-->

## Rank-Balanced Tree

In a rank-balanced tree, each node is labeled with a rank. The rank of node `x` is denoted `r(x)`. If we denote the parent of `x` as `x.P`, then the parent's rank is `r(x.P)`, and the **rank difference** of node `x` is `i = r(x.P) - r(x)`. A node with rank difference `i` is called an `i-child`. If a node has two children (which may be `nil`) with rank differences `i` and `j` respectively, the node is called an `i,j-node` (here `i` and `j` are not distinguished by left or right). In particular, a leaf node can **also** be called an `i,j-leaf`. The original paper assigns a rank of -1 to `nil` nodes and 0 to leaf nodes, which I think is not a good idea. This article assigns a rank of 0 to `nil` nodes and 1 to leaf nodes, which brings several benefits. For instance, in an AVL tree, the rank of any node equals the height of the subtree rooted at that node, whereas under the original paper's convention, the rank would be 1 less than the height. Using the concepts of **rank** and **rank difference**, AVL trees and Red-Black Trees can be described as follows:

- For an AVL tree, rank is the tree height. All nodes are either `1-child` or `2-child`, but `2,2-node` is not allowed;

- For a Red-Black Tree, rank is the black height. All nodes are either `0-child` (red) or `1-child` (black), and no parent-child pair may both be `0-child` (red). The various Red-Black Tree variants only differ in how they constrain `0-child` nodes.

If the paper merely unified AVL trees and Red-Black Trees under a single framework, it would be unremarkable. The paper relaxed the balance condition of AVL trees and proposed the WAVL (Weak AVL) tree. In a WAVL tree, all nodes are either `1-child` or `2-child`, but leaf nodes are not allowed to be `2,2-node`. A WAVL tree with no deletions is simply an AVL tree; deleting nodes from a WAVL tree may produce `2,2-node`s, so **an AVL tree is a subset of the WAVL tree**. The original paper already provides clear, diagram-rich descriptions of the balance repair for insertion and deletion, so I'll skip those here (laziness, admittedly). As usual, **I recommend reading the [original paper](https://sidsen.azurewebsites.net/papers/rb-trees-talg.pdf) directly**.

## Implementation Approaches

Before writing this article, I found [MaskRay](https://github.com/MaskRay)'s post titled [Weak AVL Tree](https://maskray.me/blog/2025-12-14-weak-avl-tree). With such excellent work already out there, some parts of this section are excerpted from that article. A WAVL tree only has two kinds of nodes: `1-child` and `2-child`, which can be distinguished with just 1 bit. This leads to several flexible implementation approaches:

1. Each node stores 2 bits of rank difference, with 1 bit each for the left and right child rank differences. A 0 bit means rank difference is 1, and a 1 means rank difference is 2;

2. Each node stores 1 bit of `rank parity`. A node whose `parity` matches its parent's is a `2-child`; different parity means `1-child`;

3. Each node stores 1 bit of rank difference. 0 means the node is a `1-child`, 1 means `2-child`;

4. Each node directly stores an absolute rank; 8 bits are more than sufficient.

In WAVL tree implementations, FreeBSD and LLVM libc use approach 1, while Fuchsia uses approach 2. Clearly, these four approaches are not limited to WAVL; they apply to other Rank-Balanced Trees as well, including AVL trees and Red-Black Trees. AVL trees typically use approach 4, and Red-Black Tree implementations use approach 3. The color of the root node in a Red-Black Tree is not important, coloring the root black when using approach 3 simply saves some handling or checks, for instance, if the parent is red, then the grandparent must exist. If a Red-Black Tree were re-implemented using approach 1, the insignificance of the root color would become even more apparent. The paper [Succinct Encodings of Binary Trees with Application to AVL Trees](https://arxiv.org/pdf/2311.15511) proved that each node in an AVL tree only needs approximately 0.938 bits of information.

## Performance Differences

In a previous article, I mentioned that FreeBSD's WAVL tree implementation demonstrates superior node insertion performance due to its specific implementation details. When there are no deletions, a WAVL tree is just an AVL tree, so the node insertion performance of a WAVL tree should be exactly the same as an AVL tree's. Under the same implementation approach, AVL tree insertion performance is usually somewhat worse than that of a Red-Black Tree (specifically 2-3 or 2-3-4), despite the AVL tree having a lower average height. FreeBSD's WAVL tree uses an array for node link pointers, with the following code:

```c++
#define RB_ENTRY(type)                     \
    struct {                               \
        struct type *rbe_link[3];          \
    }

#define _RB_LINK(elm, dir, field)	(elm)->field.rbe_link[dir]
#define _RB_UP(elm, field)		_RB_LINK(elm, 0, field)
#define _RB_L				((__uintptr_t)1)
#define _RB_R				((__uintptr_t)2)
#define _RB_LR				((__uintptr_t)3)
```

This approach not only cuts the balance repair code in half, generating fewer instructions, but more importantly reduces branching. When implementing my WAVL tree, I also used 2 bits per node to store the left and right child rank differences, which is the same approach as FreeBSD's WAVL tree. However, I chose not to aggregate the link pointers into an array. Instead, I kept the link pointers and rank information independent of each other, for the following reasons:

- Link pointers should not be aggregated using structs or arrays. A few bits of rank information would cause memory padding, make it inconvenient for users to adjust memory layout, and weaken composability. If link pointers are kept separate and reused, migrating nodes across different types of intrusive containers becomes very easy;
- For C++, there's no need for the trick of storing rank information in unused bits of pointers, because the rank information member can be independent. Nodes typically have some unused bits, and the user can decide which bits of which member to use.

The performance test for random data node insertion (on AMD64, using GCC 14.2) is as follows:

![](/blog/rank_balanced_tree_weak_avl/insert_bench.png)

| scale | freebsd irbt | linux irbt | iwavl |
| ---- | ---- | ---- | ---- |
|1024| 34294 | 36524 | 46748 |
|2048| 88903 | 100393 | 107477 |
|4096| 201539 | 234922 | 244127 |
|8192| 469216 | 553990 | 563755 |
|16384| 1216399 | 1304529 | 1425013 |
|32768| 3065210 | 3208797 | 3517131 |
|65536| 7606062 | 8059004 | 8074589 |
|131072| 17262033 | 18361193 | 19153061 |
|262144| 43454573 | 53017281 | 48495796 |
|524288| 146204107 | 139397586 | 164274886 |
|1048576| 450734218 | 423810850 | 471755907 |
|2097152| 1204783879 | 1245952545 | 1242640883 |

We can see that my WAVL tree implementation's random data insertion performance is significantly worse than FreeBSD's WAVL tree. But what about the ordered data insertion scenario? The performance test is as follows:

![](/blog/rank_balanced_tree_weak_avl/ordered_insert_bench.png)

| scale | freebsd irbt | linux irbt | iwavl |
| ---- | ---- | ---- | ---- |
|1024| 19663 | 16298 | 15295 |
|2048| 44925 | 37863 | 32076 |
|4096| 96822 | 80177 | 68706 |
|8192| 212104 | 176478 | 155490 |
|16384| 479272 | 446065 | 318969 |
|32768| 1012017 | 1219457 | 723401 |
|65536| 2228758 | 3324682 | 1453124 |
|131072| 4887640 | 8461540 | 3201482 |
|262144| 15315245 | 20760413 | 9942619 |
|524288| 32797817 | 46391989 | 26196780 |
|1048576| 102695146 | 105690405 | 61572883 |
|2097152| 226944816 | 238706167 | 134984623 |

In the ordered data scenario, my WAVL tree implementation becomes the best performer, showing that **using an array for link pointers does not always yield a net performance gain**. Although FreeBSD's implementation cuts the balance maintenance code in half, it also makes that half of the code somewhat slower. Moreover, ordered data performance testing is not unrealistic, since scenarios where data is approximately ordered within a time window are not uncommon.

Finally, the random node deletion performance test is as follows:

![](/blog/rank_balanced_tree_weak_avl/remove_bench.png)

| scale | freebsd irbt | linux irbt | iwavl |
| ---- | ---- | ---- | ---- |
|1024| 19724 | 13465 | 17333 |
|2048| 50641 | 38997 | 50958 |
|4096| 107180 | 92152 | 112187 |
|8192| 236067 | 204360 | 241209 |
|16384| 512442 | 442631 | 534209 |
|32768| 1109353 | 960895 | 1132581 |
|65536| 2412255 | 2033348 | 2428146 |
|131072| 5141650 | 4344579 | 5112267 |
|262144| 13088146 | 11387562 | 13182215 |
|524288| 46844235 | 40135798 | 45369560 |
|1048576| 148720348 | 132877141 | 147022911 |
|2097152| 413715335 | 357234772 | 388865411 |

In the random node deletion scenario, FreeBSD's and my WAVL tree implementations perform similarly, but both are somewhat worse than Linux's Red-Black Tree. However, this still doesn't prove that WAVL deletion performance is worse than Red-Black Trees. It's worth noting that my WAVL implementation did not use any of the special optimizations (branch-free, sentinel nodes, or others) I previously applied in my Red-Black Tree implementations, as those optimizations are general-purpose. During node deletion, one could also choose the replacement node from the side of the subtree with a higher rank, though this introduces extra branches and makes the code a bit more complex.

The above test results are for reference only. When revisiting FreeBSD's WAVL tree implementation, I found there is still room for improvement. In summary, the specific performance of these balanced trees depends not only on factors like tree height, the number of bottom-up maintenance levels, and rotation counts, but also on the concrete code implementation and usage scenarios. The WAVL tree implementation is highly flexible, especially in how node ranks are handled after rotations in the deletion algorithm. Specifically, a non-leaf node that is both a `1,1-node` and a `2-child` can be **mutually converted** with a non-leaf node that is both a `2,2-node` and a `1-child`. Different choices not only lead to differences in code and performance, but also produce different WAVL trees, even when the tree undergoes the same sequence of insertions and deletions.

## One That Slipped Through the Net

**WBT (Weight-Balanced Tree) is also a Rank-Balanced Tree**. The so-called **rank** can be seen as an estimate of tree height. We can take the base-2 logarithm of a node's Weight in a WBT as its rank. In a WBT, for node `v`, denote its weight as `v.S`, and its left and right children as `v.L` and `v.R`, respectively. For ease of description, unlike the earlier article, leaf node weights here are set to 2. If a WBT is balanced, all subtrees satisfy the following conditions:

- `v.L.S * ∆ ≥ v.R.S`
- `v.R.S * ∆ ≥ v.L.S`

The root node's Weight can be obtained by summing the Weights of its left and right children. Substituting `v.S = v.L.S + v.R.S` and taking the logarithm yields the following inequalities:

- `lg(∆ + 1) ≥ lg(v.S) - lg(v.L.S)`
- `lg(∆ + 1) ≥ lg(v.S) - lg(v.R.S)`

We denote `lg(v.S)` as the node's rank, i.e., `rank(v) = lg(v.S)`, which ultimately simplifies to:

- `lg(∆ + 1) ≥ rank(v) - min(rank(v.L), rank(v.R))`

The balance condition of WBT can be described as: the **maximum rank difference** is less than or equal to `lg(∆ + 1)`. When `∆ = 3`, the corresponding maximum rank difference is 2. What distinguishes WBT from other Rank-Balanced Trees is that the rank or rank difference of nodes in a WBT can be **fractional**.

## Top-Down

A key reason for WBT's underwhelming performance is that, with a bottom-up balance maintenance strategy, the Weight must always be updated all the way from the leaf to the root during node insertion and deletion. Adopting a top-down balance maintenance strategy, which only requires a single traversal, significantly improves WBT performance. For Red-Black Trees, AVL trees, or WAVL trees, bottom-up maintenance to the root is only needed in a minority of cases, so the gains from switching these to a top-down strategy are limited. Moreover, top-down maintenance has its own limitations, for example, it is not well-suited to scenarios where duplicate keys are disallowed.

All Rank-Balanced Trees support top-down maintenance, but **a top-down maintenance strategy is not always a good choice**. For AVL trees or WAVL trees, top-down element insertion requires a 5-node look-ahead, and the more look-ahead is needed, the more complex the implementation becomes. Therefore, the practical value of top-down AVL or WAVL trees may be quite limited. For concurrent scenarios, if you want to implement a lock-free balanced binary tree, a top-down Red-Black Tree is worth considering. And in scenarios where querying node rank or the k-th largest element is needed, WBT is a more suitable choice.

## Mountains Are Mountains Again, Waters Are Waters Again

When a Rank-Balanced Tree becomes unbalanced, two techniques are used to restore balance:

- The first is **rank balancing**, specifically promotion or demotion, which corresponds to recoloring in red-black trees. This ensures not only that the absolute ranks computed from the left and right subtrees are equal, but also that the rank differences satisfy the balance constraints.
- The second is **rotation**. A subtree with a larger rank is taller, so among sibling nodes, the one with the smaller rank difference is taller. Rotation essentially adjusts the taller subtree toward the shorter sibling, selecting a node closer to the binary split as the root, making the tree more balanced.
