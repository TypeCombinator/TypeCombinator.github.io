---
title: "Before the Tricks Turn to Ashes: Implementing Reflection Annotations in C++20"
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
It has been a while since reflection was officially included in C++26, and GCC 16 was among the first to provide support for reflection, even though the earliest reflection experiments were carried out on Clang. Before C++26 becomes widespread, I think C++20 can still be given a bit of life support, with a dose of reflection annotations.
<!--more-->

## Reflection Annotations

If you don’t want to read this article, you can jump straight to [kon-qi](https://github.com/TypeCombinator/kon-qi). The proposal [P3394](https://wg21.link/p3394) mentions that annotations prior to C++26 can be implemented with code of the following form:

```c++
template <typename T, auto... Annotations>
using Noted = T;

struct C {
    Noted<int, 1> a;
    Noted<int*, some, thing> b;
};
```
The problem with this approach is that the type of every member has to be wrapped in a template class, which means it also has to be unpacked when used. That is not very satisfactory, so I tried other approaches.

In C++20, an important step in the technique of implementing reflection with structured bindings is to obtain the member pointers. We can store all the member pointers and build a mapping table from each member pointer to its annotations. When we need the annotations of a certain member, we simply look up that mapping table through the member pointer. This is very similar to the technique for implementing tuple, tuple builds a mapping from index to type, whereas when I implemented annotations in `kon::qi`, I built a mapping from member pointer to annotation. Through this approach, `kon::qi` supports three different forms of reflection annotation registration.

The first is the default internal annotation, which can be registered as follows:

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

At first glance this looks a bit cumbersome, so `kon::qi` also supports inline annotations, which keep the annotation and the member on the same line. Here is an example:

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

Considering that we may also need to add annotations to third-party code, `kon::qi` also supports external annotations. Here is an example:

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

## More Tricks

While implementing reflection annotations, I suddenly thought of an article I wrote earlier about stateful metaprogramming, see [Keep It Simple, the Metaprogramming Way](https://typecombinator.github.io/en/posts/stateful_meta_programming/). The stateful metaprogramming mentioned in that article completes the bidirectional mapping between integers and types through a compile time counter implemented via **friend injection**. When implementing reflection with C++20, we can achieve a mapping from pointer to type. What if we further convert the pointer to `void *`? I arrived at the following code:

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

The code above does not use a counter, and it works correctly on all three major compilers, see [https://godbolt.org/z/xWEYxxzKb](https://godbolt.org/z/xWEYxxzKb). At this point, I took a look at C++26 reflection and felt that my collection of useless knowledge had grown once again.

## Optimization

Annotations are a box into which anything can be stuffed. `kon::qi` supports specifying multiple scan ranges in enum type reflection, and aggregate type reflection supports scanning a specified range for the number of members. By specifying scan ranges, users can gain some compilation speedup.

As mentioned earlier, when implementing annotations we need to first store the member pointers. Since these pointers are of different types, they need to be stored in a variadic template class. In fact, we could simply convert these pointers to `void *` and store them in an array, and retrieving them by index would also be more efficient. After all, implementing reflection with C++20 already comes with many limitations, so it is acceptable not to support the case where member objects overlap in memory. However, I have not implemented this optimization yet, so I will just note it here for now.

For more implementation details, see the repository at [https://github.com/TypeCombinator/kon-qi](https://github.com/TypeCombinator/kon-qi).

## A Few Thoughts

The tricks will eventually turn to ashes, and technological progress never stops. We need not mourn the passing of the old feats, nor should we mistake the power of tools for the end of thought.
