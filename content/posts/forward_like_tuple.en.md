---
title: forward_like_tuple
subtitle:
date: 2025-09-05T21:41:02+08:00
slug: forward_like_tuple
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
*"Transfer the warmth of one person to another's chest"*  
-- Eason Chan, *Love Transfer*

## Preface

C++23 core language updates were too few, so I never felt motivated to upgrade. But considering that next year is 2026, using C++23 in '26 seems reasonable, right? Among all the C++23 features that can dramatically simplify code, the first to come to mind is probably **deducing this**. So I set out to refactor my codebase with this feature first. Finally, no more writing `&`, `const &`, `&&`, and `const &&` versions of member functions over and over again. Paired with `forward_like`, a single member function covers all cases. Initially things went smoothly, but when refactoring `tuple`, I hit a unit test compilation error. Upon digging deeper, I found that `forward_like` simply doesn't work for `tuple`'s `get` method.
<!--more-->
## The Problem

For `std::tuple`, the `get` function is designed to expose access to the tuple's member elements. To simplify, let's use a `struct` for discussion:

```c++
struct foo {
    int &lvref_m;
};
void test() {
    int number = 1;
    const auto &obj = foo{number};
    obj.lvref_m = 2; // Okay!
}
```

As you can see, when `obj` is of type `const foo &`, the type of the member `lvref_m` that needs to be exposed should be `int &`. If we implement the `get` method using `forward_like`, its return type would be `const int &`, because `forward_like` applies the `const` qualifier to the member type. There are many other scenarios where it doesn't work either, which I won't belabor here. Here's the `forward_like` implementation taken from **cppreference**:

```c++
template <class T, class U>
constexpr auto&& forward_like(U&& x) noexcept {
    constexpr bool is_adding_const = std::is_const_v<std::remove_reference_t<T>>;
    if constexpr (std::is_lvalue_reference_v<T&&>) {
        if constexpr (is_adding_const) {
            return std::as_const(x);
        } else {
            return static_cast<U&>(x);
        }
    } else {
        if constexpr (is_adding_const) {
            return std::move(std::as_const(x));
        } else {
            return std::move(x);
        }
    }
}
```

This implementation is very intuitive, which left me wondering: does `tuple` really need special handling? Only upon closer inspection did I notice this description on cppreference:

*"The main scenario that std::forward_like caters to is adapting "far" objects. Neither the tuple nor the language scenarios do the right thing for that main use-case, so the merge model is used for std::forward_like."*

Turns out I was too focused on the code! `std::forward_like` indeed doesn't work for `tuple`; this is just an implementation tailored for the main use case. Proposal **P2445** mentions three models: `merge`, `tuple`, and `language`, and `std::forward_like` uses the `merge` model. The specific differences between these three models, taken from the proposal's table:

| Owner      | Member     | 'merge'    | 'tuple'    | 'language' |
| ---------- | ---------- | ---------- | ---------- | ---------- |
|            | `&`        | `&&`       | `&`        | `&`        |
| `&&`       | `&`        | `&&`       | `&`        | `&`        |
| `const`    | `&`        | `const &&` | `&`        | `&`        |
| `const &`  | `&`        | `const &`  | `&`        | `&`        |
| `const &&` | `&`        | `const &&` | `&`        | `&`        |
|            | `&&`       | `&&`       | `&&`       | `&`        |
| `&&`       | `&&`       | `&&`       | `&&`       | `&`        |
| `const`    | `&&`       | `const &&` | `&&`       | `&`        |
| `const &`  | `&&`       | `const &`  | `&`        | `&`        |
| `const &&` | `&&`       | `const &&` | `&&`       | `&`        |
|            | `const &`  | `const &&` | `const &`  | `const &`  |
| `&&`       | `const &`  | `const &&` | `const &`  | `const &`  |
| `const`    | `const &`  | `const &&` | `const &`  | `const &`  |
| `const &&` | `const &`  | `const &&` | `const &`  | `const &`  |
|            | `const &&` | `const &&` | `const &&` | `const &`  |
| `&&`       | `const &&` | `const &&` | `const &&` | `const &`  |
| `const`    | `const &&` | `const &&` | `const &&` | `const &`  |
| `const &&` | `const &&` | `const &&` | `const &&` | `const &`  |

## Implementation

For the `tuple` model, the value categories of **Owner** and **Member** need to be collapsed, while the `const` qualifier is inherited from the member. I have to say, when I first saw the original code in the proposal, I even doubted it was a C++23 proposal. But considering it's all a labor of love, the flaws don't overshadow the merits! Here's a more modern and simpler implementation of `forward_like_tuple`:

```c++
template <typename T, typename U>
constexpr auto &&forward_like_tuple(auto &&u) noexcept {
    constexpr auto is_const_this = std::is_const_v<std::remove_reference_t<T>>;
    if constexpr (std::is_lvalue_reference_v<T>) {
        if constexpr (is_const_this) {
            return static_cast<const U &>(u);
        } else {
            return static_cast<U &>(u);
        }
    } else {
        if constexpr (is_const_this) {
            return static_cast<const U &&>(u);
        } else {
            return static_cast<U &&>(u);
        }
    }
}
```

In practice, I didn't use the four-branch version shown above, but rather a simplified three-branch version. However, to stay consistent with the proposal and avoid misleading readers, there's no need to show it here. Of course, the above code still isn't rigorous enough; it lacks constraints. The deduced type of `u` and the template type `U` need to be *similar*, meaning they are the same type after removing `cvref`:

```c++
template <typename T, typename U>
concept is_similar = std::is_same_v<std::remove_cvref_t<T>, std::remove_cvref_t<U>>;
```

The reason I didn't directly add the constraint is that its use case is quite narrow. The user should know what they're doing. Moreover, the constraint requires traits like `std::remove_cvref_t`, which are implemented using template classes. For the compiler, template class instantiation is heavier than function overloading. While simplifying code with **deducing this**, it's also crucial to avoid increasing compilation time as much as possible.

## Testing

The proposal already provides test code, sparing me the effort of duplicating work. The test code includes `std::tuple` as a control comparison. After excerpting and modifying, the complete test code is available on [compiler explorer](https://godbolt.org/z/86e4an4ne). A portion of the code:

```c++
template <typename T, typename U>
using _copy_ref_t = std::conditional_t<
    std::is_rvalue_reference_v<T>,
    U &&,
    std::conditional_t<std::is_lvalue_reference_v<T>, U &, U>>;

template <typename T, typename U>
using _copy_const_t = std::conditional_t<
    std::is_const_v<std::remove_reference_t<T>>,
    _copy_ref_t<U, std::remove_reference_t<U> const>,
    U>;

template <typename T, typename U>
using _copy_cvref_t = _copy_ref_t<T &&, _copy_const_t<T, U>>;

struct probe { };

template <typename M>
struct S {
    M m;
    using value_type = M;
};

template <typename T, typename, typename Tuple, typename>
void test() noexcept {
    using value_type = typename std::remove_cvref_t<T>::value_type;

    using tpl_model =
        decltype(std::get<0>(std::declval<_copy_cvref_t<T, std::tuple<value_type>>>()));
    using tpl = decltype(forward_like_tuple<T, value_type>(std::declval<value_type>()));

    static_assert(std::is_same_v<Tuple, tpl>);
    // sanity checks
    static_assert(std::is_same_v<Tuple, tpl_model>);
}

void test() noexcept {
    using p = probe;
    // clang-format off
    //   TEST TYPE             ,'merge'    ,'tuple'    ,'language'
    test<S<p         >         , p &&      , p &&      , p &&      >();
    test<S<p         > &       , p &       , p &       , p &       >();
    test<S<p         > &&      , p &&      , p &&      , p &&      >();
    test<S<p         > const   , p const &&, p const &&, p const &&>();
    test<S<p         > const & , p const & , p const & , p const & >();
    test<S<p         > const &&, p const &&, p const &&, p const &&>();
    test<S<p const   >         , p const &&, p const &&, p const &&>();
    test<S<p const   > &       , p const & , p const & , p const & >();
    test<S<p const   > &&      , p const &&, p const &&, p const &&>();
    test<S<p const   > const   , p const &&, p const &&, p const &&>();
    test<S<p const   > const & , p const & , p const & , p const & >();
    test<S<p const   > const &&, p const &&, p const &&, p const &&>();
    test<S<p &       > &       , p &       , p &       , p &       >();
    test<S<p &&      > &       , p &       , p &       , p &       >();
    test<S<p const & > &       , p const & , p const & , p const & >();
    test<S<p const &&> &       , p const & , p const & , p const & >();
    test<S<p const & > const & , p const & , p const & , p const & >();
    test<S<p const &&> const & , p const & , p const & , p const & >();

    test<S<p &       >         , p &&      , p &       , p &       >();
    test<S<p &       > &&      , p &&      , p &       , p &       >();
    test<S<p &       > const   , p const &&, p &       , p &       >();
    test<S<p &       > const & , p const & , p &       , p &       >();
    test<S<p &       > const &&, p const &&, p &       , p &       >();
    test<S<p &&      >         , p &&      , p &&      , p &       >();
    test<S<p &&      > &&      , p &&      , p &&      , p &       >();
    test<S<p &&      > const   , p const &&, p &&      , p &       >();
    test<S<p &&      > const & , p const & , p &       , p &       >();
    test<S<p &&      > const &&, p const &&, p &&      , p &       >();
    test<S<p const & >         , p const &&, p const & , p const & >();
    test<S<p const & > &&      , p const &&, p const & , p const & >();
    test<S<p const & > const   , p const &&, p const & , p const & >();
    test<S<p const & > const &&, p const &&, p const & , p const & >();
    test<S<p const &&>         , p const &&, p const &&, p const & >();
    test<S<p const &&> &&      , p const &&, p const &&, p const & >();
    test<S<p const &&> const   , p const &&, p const &&, p const & >();
    test<S<p const &&> const &&, p const &&, p const &&, p const & >();
    // clang-format on
}
```

## Conclusion

After using **deducing this** and `forward_like_tuple`, the `get` method of `tuple` does seem much simpler. But from the compiler's perspective, this merely turns several overloaded `get` methods into a handful of compile-time branches inside `forward_like_tuple`. The same goes for `forward_like`. Moreover, comparing the implementations of `forward_like` and `forward_like_tuple`, the two correspond structurally as well. This article is only meant to spark discussion; proposal **P2445** provides a far more detailed account.
