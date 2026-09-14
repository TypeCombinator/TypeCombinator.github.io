---
title: "在奇技淫巧灰飞烟灭之前：用C++20实现反射注解"
subtitle:
date: 2026-09-14T21:19:28+08:00
slug: implementing_annotations_for_reflection_in_cpp20
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
距离反射被正式被纳入C++26已经有一段时间了，GCC 16也率先提供了对反射的支持，尽管最初的反射实验是在Clang上进行的。在C++26普及之前，我想C++20还能再抢救一下，给C++20打一剂反射注解的补丁。
<!--more-->

## 反射注解

不想看本文的，可以直接跳转到[kon-qi](https://github.com/TypeCombinator/kon-qi)。提案[P3394](https://wg21.link/p3394)中提到，在C++26之前的注解可以通过如下形式的代码实现：

```c++
template <typename T, auto... Annotations>
using Noted = T;

struct C {
    Noted<int, 1> a;
    Noted<int*, some, thing> b;
};
```
这个方案的问题在于每个成员的类型都要用一个模板类包裹一下，这意味着使用的时候还需要解包。这不太让人满意，因此我尝试了别的方案。

在C++20中，使用结构化绑定实现反射的技巧中的重要一步就是拿到成员的指针。我们可以将所有的成员的指针存起来，并建立每个成员指针到注解的映射表，当需要获取某个成员的注解时，通过成员指针去查询该映射表就可以了。这和实现tuple的技巧非常像，tuple建立的是索引到类型的映射，而我在`kon::qi`中实现注解的时候建立的是成员指针到注解的映射。通过该方案，`kon::qi`支持了3种不同形式的反射注解注册。

首先是默认的内部注解，可以按如下形式注册：

```c++
#include <kon/qi/struct.hpp>

struct point {
    double x;
    double y;
    double z;

    template <typename addon_host_type = point>
    struct addon_register {
        KON_QI_ADDON_M(x, 10, 'X');
        KON_QI_ADDON_M(y, 20);
        // No annotation for the z.

        KON_QI_ADDON_INIT();
    };
};
```

一眼看上去有点繁琐，所以，`kon::qi`还支持了内联注解，让注解和成员在同一行，示例如下：

```c++
#include <kon/qi/struct.hpp>

struct point {
    using addon_host_type = point;

    double KON_QI_IADDON_M(x, 10, 'X');
    double KON_QI_IADDON_M(y, 20);
    // No annotation for the z.
    double z;

    KON_QI_IADDON_INIT();
};
```

考虑到可能还需要对第三方的代码添加注解，`kon::qi`还支持了外部注解，示例如下：

```c++
#include <kon/qi/struct.hpp>

struct point {
    double x;
    double y;
    double z;
};

namespace kon::qi {
template <>
struct addon_register<point> {
    using addon_host_type = point;

    KON_QI_ADDON_M(x, 10, 'X');
    KON_QI_ADDON_M(y, 20);
    // No annotation for the z.

    KON_QI_ADDON_INIT();
};
}
```

## 更多的奇技淫巧

在实现反射注解的过程当中，我突然联想到之前写的一篇关于有状态元编程的文章，见[简单点，元编程的方式简单点](https://typecombinator.github.io/posts/stateful_meta_programming/)。该文章提到的有状态元编程是通过**友元注入**实现的编译期计数器，来完成整数和类型的双向映射。用C++20实现反射的时候，我们能实现指针到类型的映射，如果进一步地将指针转换为`void *`呢？我得到了如下代码：

```c++
#include <type_traits>

template <typename T>
struct meta_wrapper { };

template <typename T>
extern const meta_wrapper<T> ext_decl{};

using meta_id_t = const void *;

template <meta_id_t Id>
struct meta_tag {
    constexpr auto friend get(meta_tag);
};

template <class T>
struct meta_impl {
    using value_type = T;
    static constexpr auto id = meta_id_t{static_cast<meta_id_t>(&ext_decl<T>)};

    constexpr auto friend get(meta_tag<id>) {
        return meta_impl{};
    }
};

template <class T>
constexpr meta_id_t to_meta_id = meta_impl<T>::id;

template <meta_id_t M>
using from_meta_id = typename decltype(get(meta_tag<M>{}))::value_type;

constexpr meta_id_t int_lref_id = to_meta_id<int &>;
constexpr meta_id_t char_id = to_meta_id<char>;
constexpr meta_id_t char_ptr_id = to_meta_id<char *>;

static_assert(std::is_same_v<from_meta_id<int_lref_id>, int &>);
static_assert(std::is_same_v<from_meta_id<char_id>, char>);
static_assert(std::is_same_v<from_meta_id<char_ptr_id>, char *>);
```

以上代码没有使用计数器，在三大编译器上都能正常工作，见[https://godbolt.org/z/xWEYxxzKb](https://godbolt.org/z/xWEYxxzKb)。此时，我看了一眼C++26的反射，感觉自己没用的知识又增加了。

## 优化

注解是个框，啥都能往里装，`kon::qi`支持了在枚举类型反射中指定多段扫描区间的功能，聚合类型反射支持了在指定区间扫描成员数量的功能，用户通过指定扫描区间可以获得一些编译提速。

前面提到实现注解时，需要先将成员指针存起来，由于这些指针是不同的类型，需要存到可变参数的模板类中。其实完全可以将这些指针转换为`void *`，然后存到数组里，通过索引获取也更高效。毕竟用C++20实现反射本来就有诸多限制，成员对象内存重叠的情况不支持也罢。不过，这个优化我还没有实施，暂且记录于此吧。

更多实现细节参见仓库[https://github.com/TypeCombinator/kon-qi](https://github.com/TypeCombinator/kon-qi)。

## 一点感想

奇技淫巧终将灰飞烟灭，技术的进步永不停歇，我们不必为旧日绝活的消逝惋惜，也不该把工具的强大误认成思考的终结。