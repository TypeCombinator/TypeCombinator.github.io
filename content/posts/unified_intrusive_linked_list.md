---
title: 统一的侵入式链表
subtitle:
date: 2025-09-13T23:28:11+08:00
slug: unified_intrusive_linked_list
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
  - log
categories:
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
**「完全拒绝UB是缺乏工程实践的表现」** —— *TypeCombinator*

## 前言

去年，我在写下了在purecpp上的第一篇文章——[《统一的侵入式链表》](http://purecpp.cn/detail?id=2445)，文章里最初的原型是几年前是使用C语言实现的，一直躺在我的私人仓库里吃灰。直到去年我在做调度器优化的相关工作的时候我才回想起来，于是将其拿出来用C\+\+改写，并做了一些尝试和探索（见GitHub仓库[**uit**](https://github.com/TypeCombinator/uit)）。通过这次分享和提问，我得到了ykiko迅速且精准的回答，也意识到了这个idea和C\+\+标准之间的矛盾，遂弃坑了一段时间。但是，终极问题的种子一直深埋脑海——**如何让这个idea能够work呢**？

在后续的日常开发中，我又有了新的思路，实践之后，先前无法通过的测试均能在三大编译器上通过。当然其中过程也并非一帆风顺，哄了**Clang**和**GCC**好几小时，这两位才让我构造的测试通过，至于**MSVC**嘛，向来情绪稳定，毫无意外地通过了测试。**需要说明的是**新方案不仅在用法上更灵活，也没有引入任何额外分支或其他消耗，且不需要加任何额外的编译选项或是在源码中加一些属性，在最高优化等级下，三大编译器均可通过所有测试！

<!--more-->
不过，新方案的代码暂时没有公开，原因之一是它仍不完美。总体上，UB只是减少了，而不是完全消失，未来随着测试项新增或编译器更新仍有可能出现测试不通过的情况。不得不说，对于新的方案，我还是**很有信心**的，即使真的出现问题，我想我还是有能力修复的。

GitHub没法在公开的仓库上建立私有分支，新方案的代码最初躺在专门用于做各种实验的私人仓库中，但这个仓库不方便公开，在知乎上更新[**自答**](
https://www.zhihu.com/question/1247137402/answer/11745522389)时，我建了一个私人仓库[**uit_ng**](https://github.com/TypeCombinator/uit_ng)，待到时机合适再公开吧，如果有足够的人感兴趣话，这也是我的考量之一。

## 回顾

统一的侵入式链表使用了一种名为**mock_head**的技巧，通过该技巧可以将4种不同的链表统一起来。不过随着后来我在仓库中加入的侵入式平衡二叉树——SBT和WBT，现在，我有点后悔取**mock_head**这个名了，改名为**mock_sentinel**会契合得多！不过这不是重点，purecpp上的[文章](http://purecpp.cn/detail?id=2445)由于没放图片，可能不易于理解，这里再结合图片重新介绍一下。

先用一句话解释**mock_head**，即*复用头节点空间去模拟哨兵节点，统一头节点和数据节点的访问*。为说明这一技巧，我将以最容易的单链表开始，示意图如下：

![](/blog/uit/slist.png)

如上图所示，**head**减去数据节点中`right`字段的偏移即可得到**mock_head**，这样**mock_head**与数据节点能以相同的方式访问`right`字段，我们获得了统一访问的能力！毫无疑问，这种操作是UB，且会引入多种违反，但是我想说的是**完全拒绝UB是缺乏工程实践的表现**，为了极致性能必须要有所权衡，没有一种模型能够应对所有情形，C/C\+\+的*某些*极致性能其实是用UB换来的！

字段减偏移的操作不就是Linux内核源码中赫赫有名的`container_of`吗？没错！不同的是Linux内核源码是将`container_of`作用于数据节点，而本文提到的**mock_head**则是将`container_of`作用于头节点。为了简化讨论，我将用C语言单链表来描述，代码（完整的放在[compiler explorer](https://godbolt.org/z/c66dj6YE7)上）如下：

```c
#include <stdint.h>
#include <stdio.h>
#include <stddef.h>
struct node {
    int sn;
    struct node *next;
};
static struct node *head = NULL;

void push_front(struct node *n) {
    n->next = head;
    head = n;
}

#define container_of(_field_, _type_, _member_) \
    ((_type_ *) ((size_t) (_field_) -offsetof(_type_, _member_)))

#define MOCK_HEAD(_field_, _type_, _member_) container_of(_field_, _type_, _member_)

struct node *remove_with_mock_head(struct node *n) {
    struct node *prev = MOCK_HEAD(&head, struct node, next);
    for (struct node *cur = prev->next; cur != NULL;) {
        if (cur == n) {
            prev->next = cur->next;
            return cur;
        }
        prev = cur;
        cur = prev->next;
    }
    return NULL;
}
```

以上代码可以看出，**mock_head**能直接使用`container_of`实现，以上节点删除代码与Linux内核源码中使用二级指针删除节点的代码在形式上是一致的，这里再列出使用二级指针删除节点的代码作为对比，帮助理解，代码如下：

```c
struct node *remove_with_in_linux(struct node *n) {
    struct node **prev = &head;
    for (struct node *cur = *prev; cur != NULL;) {
        if (cur == n) {
            *prev = cur->next;
            return cur;
        }
        prev = &cur->next;
        cur = *prev;
    }
    return NULL;
}
```

可以看到，使用二级指针也能将头和数据节点的访问统一起来，对于单链表，使用**mock_head**相对于二级指针没有代码复杂度和性能上的优势，但是，如果将**mock_head**应用到其他形式的链表，并将语言切换到C++呢？像[**stdexec**](https://github.com/NVIDIA/stdexec)这种方式实现的[intrusive_queue](https://github.com/NVIDIA/stdexec/blob/main/include/stdexec/__detail/__intrusive_queue.hpp)，由于头节点和数据节点访问不统一，需要对边界情形做额外判断，摘取其`push_back`代码如下：

```c++
void push_back(_Item* __item) noexcept {
    STDEXEC_ASSERT(__item != nullptr);
    __item->*_Next = nullptr;
    if (__tail_ == nullptr) {
        __head_ = __item;
    } else {
        __tail_->*_Next = __item;
    }
    __tail_ = __item;
}
```

如果以上代码改用**mock_head**实现呢？摘取与之等价的[**uit**](https://github.com/TypeCombinator/uit)的[idslist](https://github.com/TypeCombinator/uit/blob/main/include/uit/idslist.hpp)的`push_back`实现如下：

```c++
void push_back(T *node) noexcept {
    (node->*M).right = nullptr;
    (head.left->*M).right = node;
    head.left = node;
}
```

可以看到，使用**mock_head**之后，以上代码没有任何分支！为什么可以这样呢？核心原因在于链表为空的处理不一样，示意图如下：

![](/blog/uit/dslist_empty.png)

链表为空时，`left`并没有指向空，而是指向了**mock_head**，这样，在链表为空时也能通过`left`字段去访问到`right`字段，所以，空和非空`idslist`的`push_back`处理被统一了，对于非空的`idslist`示意图如下：

![](/blog/uit/dslist.png)

需要说明的是[**stdexec**](https://github.com/NVIDIA/stdexec)的[intrusive_queue](https://github.com/NVIDIA/stdexec/blob/main/include/stdexec/__detail/__intrusive_queue.hpp)的`pop_front`方法调用是需要在外部判空的，而[**uit**](https://github.com/TypeCombinator/uit)的是在内部判空的，这里**不要混淆**！ `idlist`（双链表）也采用了类似的技巧，示意图如下：

![](/blog/uit/dlist_empty.png)

![](/blog/uit/dlist.png)

链表为空时，`right`和`left`都指向了**mock_head**，通过这种技巧，`idlist`能获得和Linux内核链表形式相同的处理代码，比如`insert`和`remove`方法，代码实现如下：

```c++
static void insert(T *node, T *left, T *right) noexcept {
    (node->*M).right = right;
    (node->*M).left = left;

    (left->*M).right = node;
    (right->*M).left = node;
}

static void remove(T *left, T *right) noexcept {
    (left->*M).right = right;
    (right->*M).left = left;
}
```

同样的，不需要使用任何分支！更为关键的是，由于这些维护链接关系的指针是直接指向节点的，用户不再需要像Linux内核链表那样用`container_of`去还原数据节点，这消除了对用户代码的侵入！

C\+\+侵入式链表的各种实现已经很多了，广为人知的boost的[**intrusive**](https://github.com/boostorg/intrusive)不必多说，[**nodejs**](https://github.com/nodejs/node)也用到了侵入式链表，位于[src/util.h](https://github.com/nodejs/node/blob/main/src/util.h)文件中，不过里面只实现了一个双链表，其实现思路与Linux的内核链表是一致的，所以也使用了会在C\+\+中引入UB的`container_of`。那么，要在C\+\+中实现Linux那种形式的侵入式链表，用继承和`static_cast`代替`container_of`是否可行呢？当然可行！但问题是，**继承真的很不好用**！虽然C\+\+的很多技法都和继承绑定了，但实践开发中，我的看法是能不用继承就不要用！如果抛弃以上方案，改用stdexec的方式实现呢？虽然不会引入UB，但实现起来需要做额外的边界条件判断，这必然引入额外开销！

最后，还剩下一个`isdlist`没有介绍，这等价于Linux中的`hlist`，该链表主要用于实现哈希表，当持有节点需要删除时，如果使用单链表，则需要遍历拿到前驱才能删除，但如果改用双链表，则头节点需要两个指针的空间，而使用`isdlist`则只需要占用一个指针的空间，`isdlist`     的示意图如下：

![](/blog/uit/sdlist.png)

Linux中的`hlist`的数据节点的前驱是一个二级指针，指向的是前驱的后继指针，特殊的首数据节点，则其前驱指针则指向了链表的头节点；`isdlist`由于使用了**mock_head**，二级指针不再被需要，前驱指针直接指向前驱节点，特殊的首数据节点，则其前驱指针指向了**mock_head**。

至此，4种侵入式链表（`islist`、`idslist`、`idlist`和`isdlist`）已回顾完毕！其**优缺点**总结如下：

- **优点**
  - 简单易懂的代码
  - 更少的分支和更好的性能，比如，`idslist`的`push_back`方法没有任何分支
  - 二级指针不再被需要，`isdlist`也不例外
  - 对于`idlist`和`isdlist`，节点删除不需要持有原链表
  - 真正的零成本抽象
  - 没有使用宏

- **缺点**
  - 会引入UB，且有多种违反
  - `isdlist`、`idslist`和`idlist`是自引用类型，`isdlist`和`idlist`是move-only的（这不是大问题，可以用额外的模板参数控制是否启用**mock_head**，以避免自引用和move-only带来的不便）

## 另外

当谈到链表，我们不应当只想到狭义的单链表和双链表，本文的标题为《统一的侵入式链表》，所以，仅仅统一线性链表并非我的终极目标，链式实现的树或图数据结构也可以被纳入进来，在原有的基础上，还加入平衡树相关的侵入式数据结构，这也是为什么我一开始就将链表的后继和前驱命名为`right`和`left`的原因。当前，平衡树的实现包括了SBT、WBT和堆，其中带`parent`指针的侵入式二叉堆的代码贡献给了[stdexec](https://github.com/NVIDIA/stdexec/pull/1674)，里面用到的相关优化思路我未曾搜索到有任何相似的实现，应该也是**个人首创**吧，但我懒得写文章介绍了，等有能看懂的有缘人吧！

## 总结

当前，公开的仓库[**uit**](https://github.com/TypeCombinator/uit)中的线性链表的实现只是展示**mock_head**这一技巧的实验代码，非`-O0`优化等级下，单元测试不能完全通过，不具备实用条件，而可实用的仓库[**uit_ng**](https://github.com/TypeCombinator/uit_ng)现在还没有公开，虽然暂时未公开，但是传达出**mock_head**是可以行得通的技巧这一信息，我想在现阶段已经足够！

使用**mock_head**是可以带来性能提升的，虽然不多，且会引入UB，但**权衡**之下，我认为这是值得的，毕竟，有得必有失！从我研读过的诸多C++开源项目来看，追求性能而有意引入UB的项目也并不少，正如我前面所说，**完全拒绝UB是缺乏工程实践的表现**，C\+\+的模型是有边界，有极限的，总有一些情形是照顾不到的，部分情形可能随着C\+\+语言版本更新被所纳入的新特性所覆盖，但有些情形则是永远不会被C\+\+标准委员会所采纳的，对于这种情况，**如果收益值得**，只要有足够的测试覆盖，是不需要有过多担心的。

C\+\+已经能满足我的一切所想，虽然特性复杂，偶尔让人在一些细节上耗费时间和精力，但每次都是能带来收获的，这也是**C\+\+的魅力**所在，我再也无需纠结C\+\+侵入式链表的性能问题了！