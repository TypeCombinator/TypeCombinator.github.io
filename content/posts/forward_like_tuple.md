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
*「把一个人的温暖转移到另一个的胸膛」*  
——陈奕迅《爱情转移》

## 前言

C\+\+23语核更新太少，一直没有动力去升级，不过考虑到明年是2026年了，26年用C\+\+23还算合理吧？C\+\+23中，能大幅简化代码的特性，首当其冲的恐怕就是**deducing this**了，于是先使用该特性对代码进行了一番改造，终于不用再反复写`&`、`const &`、`&&`和`const &&`这几种成员函数了，搭配上`forward_like`，一个成员函数即覆盖所有情形，起初还算顺利，但是改造在`tuple`时却碰到了单元测试编译报错的问题，一探之下，发现`forward_like`并不适用于`tuple`的`get`方法。
<!--more-->
## 问题

对于`std::tuple`，`get`函数的实现意图是为了将`tuple`中的成员的访问能力暴露出来，为了简化，这里用`struct`代替讨论，代码如下：

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

可以看出，当`obj`为`const foo &`类型时，需要暴露的`lvref_m`成员的类型应该是`int &`，如果使用`forward_like`实现`get`方法，其返回类型将是`const int &`，因为`forward_like`会把`const`限定符加到成员类型上，还有许多其他情形也不适用，这里不再赘述。摘取**cppreference**的`forward_like`实现，代码如下：

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

这个实现很符合直觉，于是我心里冒出了疑问，对于`tuple`只能特殊处理了吗？细看之下才发现cppreference有如下描述：

*「The main scenario that std::forward_like caters to is adapting “far” objects. Neither the tuple nor the language scenarios do the right thing for that main use-case, so the merge model is used for std::forward_like.」*

原来是我光顾着看代码了！`std::forward_like`确实不适用于`tuple`，这只是一个适用主要场景的实现，提案**P2445**提到了3种模型分别为`merge`、`tuple`和`language`，而`std::forward_like`使用的是`merge`模型。这3种模型的具体区别，摘取提案表格如下：

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

## 实现

对于`tuple`模型，**Owner**和**Member**值类别需要折叠，而`const`限定符则从成员继承。不得不说，最初看到提案给的原始代码时，我都怀疑这是一个C\+\+23的提案，但考虑到都是为爱发电，瑕不掩瑜吧！这里给出一个更现代且更简单的`forward_like_tuple`的实现，代码如下：

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

在实践时，笔者并没有使用上面给出的4个分支的版本，而是使用的3个分支的简化版本，但为契合提案和避免误导，就不必放出来了。当然，以上代码还不严谨，缺乏约束，`u`的推导类型和模板类型`U`需要相似，即，二者去除`cvref`之后是相同类型，定义如下：

```c++
template <typename T, typename U>
concept is_similar = std::is_same_v<std::remove_cvref_t<T>, std::remove_cvref_t<U>>;
```

之所以没有直接加上约束，是因为其使用场景很窄，使用者应当清楚自己在做什么，而且约束中需要使用`std::remove_cvref_t`这样的`trait`，这种`trait`是使用模板类实现的，对于编译器来说，模板类实例化是要比函数重载更重的，使用**deducing this**简化代码的同时，尽量避免增加编译时间也至关重要。

## 测试

提案已经给了测试代码，避免了重复劳动，该测试代码包括了`std::tuple`作为对照测试，截取并删改后，完整测试代码见[compiler explorer](https://godbolt.org/z/86e4an4ne)，部分代码如下：

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

## 总结

`tuple`的`get` 方法在使用**deducing this**和`forward_like_tuple`之后看上去是简化不少，可从编译器的视角来看，这只是把几个`get`方法函数重载变成了`forward_like_tuple`内部的若干个编译期分支，`forward_like`也是如此，而且对比`forward_like`和`forward_like_tuple`的代码实现，二者从结构上也能形成对应。本文只为抛砖，提案 **P2445**有更为详尽的描述。