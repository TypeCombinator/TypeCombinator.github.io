---
title: 浅谈C++最佳异步范式
subtitle:
date: 2025-09-18T09:30:00+08:00
slug: best_async_paradigm
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
*「C\+\+老仙，法力无边，攻无不克，战无不胜！」*—— 来自一名C\+\+杂兵

## CPS+Monad

- future/promise+then链：过去式，eager模式，类型擦除
- ` std::execution`：未来式，lazy模式，类型安全
<!--more-->

### 类型安全的收益

保留了完整的编译期信息，利于优化，函数容易内联，可以避免动态内存分配，性能上限高。

### 类型安全的代价

Monad会导致产生大量复杂的中间类型，类型嵌套可能非常深，这些类型是又广又深，人和编译器都表示压力很大啊！实践起来还是需要一些类型擦除！

## 能不能避免CPS或Monad？！

**协程！启动！**

- 有栈协程：不同指令集的平台需要适配，大多有现成，可以直接copy，栈大小需要权衡，太大了浪费内存，太小了容易爆栈，函数调用照常，协程切换稍慢，适合CPU密集任务
- 无栈协程：类型擦除，默认动态分配，链式维护父子调用，原始调用栈丢失，调试麻烦，链式返回比普通函数返回慢，且调用开销比普通函数大，调用链越长则开销越多，但是协程切换快，适合IO密集任务

### 无栈协程有传染性？

先手动狗头，TCP还会粘包呢！协程调用普通函数，再想通过该普通函数再调协程？可是普通函数不能直接暂停和恢复啊，CPS和协程总要选一个吧？再换个角度，普通函数的栈在哪里，有栈协程怎么就没有这个问题？那是因为用有栈协程，普通函数就已经获得了随时暂停和恢复的能力。

`std::execution`可以把有栈和无栈协程也纳入进去啊？可是……实现太复杂了，只想用协程！可是……单一种类的协程不够用？那把有栈和无栈协程混合起来！可是……fork和join不好用，想要when_all和when_any之类的组合子怎么办？**Monad又回来了**！

## 能统一吗？

就像物理学家想找到统一万物的理论一样，程序员也在不断地尝试找到各种问题的统一方案，但是统一和谐的外表下通常隐藏着复杂和冲突，你以为你开的是一辆汽车，可是某一天他开始变身，成了一个变形金刚，此刻才发觉自己竟从未驾驭！试问，万物统一理论会是弦论吗？异步的最佳范式会是`std::execution`吗？

异步的最佳范式是什么？协程也不是不能用！

运行时多态的最佳范式是什么？虚函数也不是不能用！

错误处理的最佳范式是什么？错误码也不是不能用！

……

现代C\+\+已经被函数式玩坏了，CPS和Monad这些函数式语言的基础操作，怎么到了C\+\+里面实现起来就这么复杂？因为C\+\+的元编程是被发现而不是设计好的，几乎没法打直球，所有的代码生成和类型计算都需要以一种**间接且蹩脚的函数式编程**来实现，函数式再叠函数式可不得让人抓狂！

Monad需要用到product type、sum type等基础设施，以`std::variant`为例，试问有办法不通过递归union得到一个支持任意个数类型且能编译期使用的实现吗？线性递归union导致模板类实例化太深！那改成二叉树形式递归能降低实例化深度吗？于是，绞尽脑汁，写了一段难以维护的代码，所幸，还是收获到了编译速度的提升，输入更多的类型编译器也不罢工了，此时，你获得了短暂的颅内高潮，再点上一只烟，贤者真身现！编译期有那么重要吗？写个只支持运行时的丐版也不是不能用！

## C\+\+26能带来希望的曙光吗？

也许吧！到时候，`std::execution`会获得代码的简化，编译速度的提升，但是会好多少，不能确定，让我们拭目以待！但无论如何，Monad带来的问题都不会消失！C\+\+的类型系统非常强大，但又十分复杂，先借用《SICP》里的一句话：

*「The evaluator, which determines the meaning of expressions in a programming language, is just another program.」*

**解释器或编译器也不过是另外一种程序**，再借用《黑客与画家》的“格林斯潘第十定律”：

*「任何C或Fortran程序复杂到一定程度之后，都会包含一个临时开发的、只有一半功能的、不完全符合规格的、到处都是bug的、运行速度很慢的Common Lisp实现。」*

在C\+\+里面使用元编程，首先的体验就是编译缓慢，然后容易遇到的现象就是，当前编译器版本有Bug，需要升级到更新的版本，或是遇到3大编译器表现不一致，一个能编译通过，另外一个又编译报错，此时不得不尝试规避，奇迹淫巧又增加了！有时候不禁让人联想，**在C\+\+编译器里面塞一个优化好的Lisp解释器做编译期计算能不能行？**

## 追寻各自的光

`std::execution`在大多数人的手里都是变形金刚，很酷，但难以驾驭，在少数人手里是高达，能运用自如。我们必须考虑当下，考虑语言和编译器的限制，考虑自己的使用场景，大多数时候，ASIO或者Folly够用了，再想简化，自制协程库也不是不行，毕竟C\+\+里面的每个语言特性，以及每个库，不理清其实现原理，不尝试copy一遍，用出问题也很正常！然后，有的人骑自行车，有的人开高达，合适的才是最佳的，在C\+\+的世界里，分裂才是常态，但我们都在**追寻各自的光**！