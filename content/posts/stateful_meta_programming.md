---
title: 简单点，元编程的方式简单点
subtitle:
date: 2024-11-06T15:08:01+08:00
slug: stateful_meta_programming
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
一直以来，C\+\+模板元编程的代码都极不直观，直到我看到了这样一段代码：

```c++
template<class... Ts>
struct example {
  mp::apply_t<std::variant,
      std::array{mp::meta<Ts>...}
    | std::views::drop(1)
    | std::views::reverse
    | std::views::filter([](auto m) { return mp::invoke<std::is_integral>(m); })
    | std::views::transform([](auto m) { return mp::invoke<std::add_const>(m); })
    | std::views::take(2)
    | std::ranges::to<mp::vector<mp::info>>()
  > v;
};

static_assert(
  typeid(std::variant<const int, const short>)
  ==
  typeid(example<double, void, const short, int>::v)
);
```
<!--more-->
这不就是我的梦中情码么？类型计算和值计算完全一致！好奇心驱使下看了下代码实现，还是有点小小的失望，作者使用了有状态模板元编程，写到这里感觉要开始部分劝退了，不过在C\+\+20的concept加持下，有状态模板元编程也得到了大幅简化，既然是有状态，那么就需要对状态进行读写：

- 写：例化友元函数
- 读：通过`requires`检测友元函数是否例化

先讲一下这个代码实现的基本思路，要让类型计算变成值计算，那么就要建立类型和值的双向映射，可以通过`mp::meta<T>`将类型T映射为一个值，然后可以通过`mp::type_of<meta<T>>`将值还原为类型T。作者利用有状态元编程的手段实现了一个计数器，每次例化`mp::meta<T>`时，对应的计数器加1。原始的代码做了编译时间优化，里面的一些命名也有些误导，不易理解，这里贴一个我改写的非优化的版本，代码如下：

```c++
#define MP_SIZE 16u

enum class id_t : size_t {
};

template <id_t>
struct info {
    constexpr auto friend get(info);
};

template <class T>
struct meta_impl {
    using value_type = T;

    template <size_t n = MP_SIZE - 1u>
    static constexpr auto gen() -> size_t {
        if constexpr (n == 0) {
            return 0;
        } else if constexpr (requires { get(info<id_t{n - 1}>{}); }) {
            return n + requires { get(info<id_t{n}>{}); };
        } else {
            return gen<n - 1u>();
        }
    }

    static constexpr auto id = id_t{gen()};

    constexpr auto friend get(info<id>) {
        return meta_impl{};
    }
};
```

可以看到gen是从大往小（这是为了与作者的代码对应）去搜索计算的，当检测到`get(info<n - 1>)`已经例化时那么就返回`n - 1 + 1`，即`n`，否则通过`gen<n - 1>()`继续搜索，对于边界情形`n == 0`，则直接返回0，以上代码测试如下：

```c++
static_assert(static_cast<std::size_t>(meta_impl<char>::id) == 0);

static_assert(static_cast<std::size_t>(meta_impl<unsigned char>::id) == 1);

static_assert(static_cast<std::size_t>(meta_impl<short>::id) == 2);

static_assert(static_cast<std::size_t>(meta_impl<unsigned short>::id) == 3);
```

可以看到类型已经映射为了值，且这个值对应了`meta_impl<T>`的例化顺序，实际使用可以再用模板变量进行简化，代码如下：

```c++
template <class T>
inline constexpr id_t meta = meta_impl<T>::id;
```

每次例化`meta_impl<T>`，都会例化一个与之对应的`get(info<id>)`，其返回类型是`meta_impl<T>`，因此通过值还原类型的代码如下：

```c++
template <id_t meta>
using type_of = typename decltype(get(info<meta>{}))::value_type;
```

前面也提到作者的原始代码做了编译时间的优化，其实也就是使用了二分法，代码如下：

```c++
template <size_t left = 0u, size_t right = MP_SIZE - 1u>
static constexpr auto gen() -> size_t {
    if constexpr (left >= right) {
        return left + requires { get(info<id_t{left}>{}); };
    } else if constexpr (constexpr auto mid = left + (right - left) / 2u;
                         requires { get(info<id_t{mid}>{}); }) {
        return gen<mid + 1u, right>();
    } else {
        return gen<left, mid - 1u>();
    }
}
```

这样，`gen`的时间复杂度从O(n)降低为O(log(n))，作者也做了编译时间的性能测试，这个库和[mp11](https://github.com/boostorg/mp11)性能相近，在各个编译器上比拼互有胜负，最后作者也提到，由于这个库是值计算，如果将来编译器加入了完整`constexpr`的JIT实现，那么这个库是极具潜力的。

## 总结

上面提到的库是[**mp**](https://github.com/qlibs/mp)，库作者是**Kris Jusiak**，cppcon上的常客了，在cppcon2024上，作者对该库做了讲解，讲解的最后，作者给出了编译时间性能测试，这也是让我印象深刻的地方。当然，考虑到有状态元编程的种种问题，该库的学习意义可能大于实践意义，将其作为**P2996**开胃菜也是一个不错的选择。该作者还有其他很多有意思的库，见[**qlibs**](https://github.com/qlibs)。

本文中出现的测试代码放在了我的github仓库[**eespace**](https://github.com/TypeCombinator/eespace/blob/main/examples/smp/main.cpp)中，以后，我也会把平时的一些实验代码放到这个仓库中。

最后，我想抛两个问题：

- 不同编译单元中使用该库是否会违反ODR？
- 非优化版的`gen`能否简化如下：

```c++
template <size_t n = MP_SIZE - 1u>
static constexpr auto gen() -> size_t {
    if constexpr (n == 0 || requires { get(info<id_t{n - 1}>{}); }) {
        return n + requires { get(info<id_t{n}>{}); };
    } else {
        return gen<n - 1u>();
    }
}
```