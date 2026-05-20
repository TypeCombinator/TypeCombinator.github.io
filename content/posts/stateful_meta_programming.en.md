---
title: Keep It Simple, the Metaprogramming Way
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
For a long time, C++ template metaprogramming code has been extremely unintuitive, until I came across this snippet:

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
Isn't this exactly the kind of code I've been dreaming of? Type computation and value computation are completely unified! Driven by curiosity, I looked into the implementation, and was slightly disappointed: the author used stateful template metaprogramming. I feel like I'm about to start losing readers here, but with the help of C++20 concepts, stateful metaprogramming has also been greatly simplified. Since it's stateful, we need to read and write state:

- Write: instantiate a friend function
- Read: detect whether the friend function is instantiated via `requires`

First, let's go over the basic idea behind this implementation. To turn type computation into value computation, we need a bidirectional mapping between types and values. We can use `mp::meta<T>` to map type T to a value, and then use `mp::type_of<meta<T>>` to restore the value back to type T. The author implements a counter using stateful metaprogramming: each time `mp::meta<T>` is instantiated, the corresponding counter increments by 1. The original code is optimized for compilation time, and some of the naming is a bit misleading and hard to follow. Here's a rewritten, non-optimized version:

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

As you can see, `gen` searches from largest to smallest (this is to match the author's code). When it detects that `get(info<n - 1>)` has been instantiated, it returns `n - 1 + 1`, i.e. `n`; otherwise it continues searching via `gen<n - 1>()`. For the boundary case `n == 0`, it directly returns 0. The above code is tested as follows:

```c++
static_assert(static_cast<std::size_t>(meta_impl<char>::id) == 0);

static_assert(static_cast<std::size_t>(meta_impl<unsigned char>::id) == 1);

static_assert(static_cast<std::size_t>(meta_impl<short>::id) == 2);

static_assert(static_cast<std::size_t>(meta_impl<unsigned short>::id) == 3);
```

We can see that types have been mapped to values, and the value corresponds to the instantiation order of `meta_impl<T>`. In practice, we can simplify further using a template variable:

```c++
template <class T>
inline constexpr id_t meta = meta_impl<T>::id;
```

Each time `meta_impl<T>` is instantiated, a corresponding `get(info<id>)` is instantiated, with a return type of `meta_impl<T>`. Thus the code for restoring a value back to a type is:

```c++
template <id_t meta>
using type_of = typename decltype(get(info<meta>{}))::value_type;
```

As mentioned earlier, the author's original code includes a compilation-time optimization, which essentially uses binary search:

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

This reduces the time complexity of `gen` from O(n) to O(log(n)). The author also ran compilation-time benchmarks, and this library performs comparably to [mp11](https://github.com/boostorg/mp11), with wins and losses trading off across different compilers. Finally, the author noted that since this library operates on values, if compilers ever gain full `constexpr` JIT support, the library has tremendous potential.

## Conclusion

The library mentioned above is [**mp**](https://github.com/qlibs/mp), by **Kris Jusiak**, a regular at CppCon. At CppCon 2024, the author gave a talk on this library, and what left a deep impression on me was the compilation-time benchmark he presented at the end. Of course, given the various issues with stateful metaprogramming, this library probably has more educational value than practical value. It also makes a nice appetizer before diving into **P2996**. The author has many other interesting libraries as well, check out [**qlibs**](https://github.com/qlibs).

The test code appearing in this article is in my GitHub repository [**eespace**](https://github.com/TypeCombinator/eespace/blob/main/examples/smp/main.cpp). Going forward, I'll be putting my experimental code into this repo as well.

Finally, I'd like to pose two questions:

- Would using this library across different translation units violate the ODR?
- Can the non-optimized `gen` be simplified to the following:

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
