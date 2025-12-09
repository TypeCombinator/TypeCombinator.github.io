---
title: CppCon2025笔记之libc++
subtitle:
date: 2025-09-25T22:52:13+08:00
slug: cppcon2025_huixie90
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

看到华人小哥，必须得关注一下，演讲题目是《Implement Standard Library: Design Decisions, Optimisations and Testing in Implementing Libc\+\+》，作者[**Hui Xie**](https://github.com/huixie90)是libc\+\+的贡献者、BSI成员、WG21成员……谈了一些libc\+\+中相关的设计决策、优化和测试细节。本文结合演讲视频和仓库幻灯片整理而来，任何转述错误和个人观点与原作者无关，**推荐观看原视频**，原作者的讲解更清晰。

<!--more-->

## std::expected

使用`[[no_unique_address]]`复用非C结构体因内存对齐而引入的尾部padding，考虑如下代码：

```c++
class Foo {
    int i;
    char c;
    bool b;
};
enum class ErrCode : int { Err1, Err2, Err3 };
```

由于内存对齐，末尾有2字节的padding（**先记住这一点**），可以得出`sizeof(Foo)`为8，那么`sizeof(std::expected<Foo, ErrCode>)`是多少呢？如果使用libstdc\+\+，结果是12，但是使用libc\+\+，结果则是8，为什么呢？libc\+\+的`std::expected`简化代码如下：

```c++
template <class Val, class Err>
class expected {
    union U {
        [[no_unique_address]] Val val_;
        [[no_unique_address]] Err err_;
    };
    [[no_unique_address]] U u_;
    bool has_val_;
};
```
![](/blog/cppcon2025_huixie90/expected0.png)

因为 *「The attribute-token no_unique_address specifies that a non-static data member is a potentially-overlapping subobject」*，`class Foo`不是C结构体，其尾部padding被复用了。收益是更小的内存占用，更好的缓存局部性，`std::expected`通常用于错误处理，并用于返回值，对于如下代码：

```c++
std::expected<Foo, ErrCode> compute() { return Foo{}; }
```

使用gcc和libstdc++编译输出的汇编代码如下：

```assembly
compute():
        mov     DWORD PTR [rsp-24], 0
        xor     eax, eax
        mov     BYTE PTR [rsp-24], 1
        mov     ecx, DWORD PTR [rsp-24]
        mov     rdx, rcx
        ret
```

而使用clang和libc++编译输出的汇编代码如下：

```assembly
compute():
        movabs  rax, 281474976710656
        ret
```

那么只有收益吗？想象一下，邻居家有一个大院子，一部分用于游泳池，一部分用于花园……还留有一些空地，作为其邻居，如果看到块空地没用，冒然将自己的物品放上去，会发生什么？自然是少不了一番邻里不合的景象了！假如将`std::expected`拷贝构造函数实现如下：

```c++
template <class Val, class Err>
class expected {
  public:
    expected(const expected& other)
        : has_val_(other.has_val_) {
        if (has_val_) {
          std::construct_at(std::addressof(u_.val_), other.u_.val_);
        } else {
          std::construct_at(std::addressof(u_.err_), other.u_.err_);
        }
    }
};
```

乍一看，以上的代码是符合直觉的，但是如果尝试运行如下代码呢？

```c++
int main() {
    std::expected<Foo, int> e1(Foo{});
    std::expected e2(e1);
    assert(e2.has_value());
}
```

断言失败了！因为 *「To zero-initialize an object or reference of type T means: … if T is a (possibly cv-qualified) non-union class type, its padding bits ([basic.types.general]) are initialized to zero bits and …」* ，邻居表示，我清扫自家院子，把空地的杂物扔了，也怪不得我啊！对于`class Foo`，`construct_at`会将padding字段清零，附带着`has_val_`也被清零了，修复该问题的办法就是将`has_val_`的赋值挪到调用`std::construct_at`之后，修改如下：

```c++
template <class Val, class Err>
class expected {
  public:
    expected(const expected& other) {
        if (other.has_val_) {
          std::construct_at(std::addressof(u_.val_), other.u_.val_);
        } else {
          std::construct_at(std::addressof(u_.err_), other.u_.err_);
        }
        has_val_ = other.has_val_;
    }
};
```

那么，问题被终结了吗？考虑`std::expected`套娃，代码如下：

``` c++
std::expected<std::expected<Foo, ErrCode>, ErrCode> e = ...;
e.value().emplace(); // the inner expected construct_at will overwrite outer expected bool
```
此时，`std::expected`的内存布局如下：

![](/blog/cppcon2025_huixie90/expected1_nest0.png)

对内层的`expected`对象调用`emplace`，会调用`construct_at`将padding空间给清零，外层`expected`的`has_val_`也附带着被清零了，这和前面拷贝构造函数的问题类似，更为直观的代码如下：

```c++
struct Bar {
    [[no_unique_address]] std::expected<Foo, ErrCode> e;
    char c;
};

Bar bar = ...;
bar.c = 'c';
bar.e.emplace(); // construct_at will overwrite c
```
`struct Bar`的内存布局如下：

![](/blog/cppcon2025_huixie90/expected1_nest1.png)

成员`char c`复用了前一成员的尾部padding空间，`construct_at`会将成员`c`的内存清零，以上代码中的字符写入成了无效操作！为了解决以上问题，再次修改代码：

```c++
template <class Val, class Err>
class expected {
    struct repr {
        union U {
            [[no_unique_address]] Val val_;
            [[no_unique_address]] Err err_;
        };
        [[no_unique_address]] U u_;
        bool has_val_;
    };
 
    repr repr_; // no [[no_unique_address]] on this member
};
```

关键在于`repr repr_;`这行代码没有加`[[no_unique_address]]`，这打断了`[[no_unique_address]]`的递归，`char c`不会再复用尾部padding，此时，`struct Bar`的内存布局如下：

![](/blog/cppcon2025_huixie90/expected2.png)

那么，经过以上修改，这就是最佳的代码了吗？如果`std::expected`传入的`Val`或`Err`类型没有padding呢？比如`std::expected<unsigned int, int>`，成员`bool has_val_`将没有可以复用的尾部padding，其内存布局如下：

![](/blog/cppcon2025_huixie90/expected3.png)

最终修改代码，添加对此种情况的分支处理，如下：

```c++
template <class Val, class Err>
struct repr {
    union U {
        [[no_unique_address]] Val val_;
        [[no_unique_address]] Err err_;
    };
    [[no_unique_address]] U u_;
    bool has_val_;
};

template <class Val, class Err>
struct expected_base {
    repr<Val, Err> repr_; // no [[no_unique_address]]
};
template <class Val, class Err> requires bool_is_not_in_padding
struct expected_base {
    [[no_unique_address]] repr<Val, Err> repr_;
};

template <class Val, class Err>
class expected : expected_base<Val, Err> {};
```

当成员`bool has_val_`没有复用padding的时候，才添加`[[no_unique_address]]`，即条件属性，此时`repr<Val, Err> repr_`的尾部padding是完全受控的，当前的`expected`不会读写这些尾部padding，因此可以安全地被外部复用。

以下为**个人拙见**，过去要实现potentially-overlapping子对象只能通过继承，现在使用组合加上`[[no_unique_address]]`属性也能实现，这意味着某个类即使带有`final`说明符也阻止不了overlapping，也不能有魔法代码去读写尾部padding，比如使用`std::memset`和`std::memcpy`除了要求 trivially-copyable以外，还要求不能是potentially-overlapping的子对象，否则是UB，比如使用libc++的`std::expected`这样写就会有问题：

```c++
class Foo final {
    int i;
    char c;
    bool b;
   public:
    void clear() noexcept {
        // *this = {}; // Good, but little bit slow.
        std::memset(this, 0, sizeof(*this)); // Bad, but fast.
    }
};

enum class ErrCode : int { Err1, Err2, Err3 };

static_assert(std::is_trivially_copyable_v<Foo>);

int main() {
    std::expected<Foo, ErrCode> e{Foo{}};
    
    static_assert(sizeof(e) == 8);

    e.value().clear();
    assert(e.has_value());
    return 0;
}
// clang++ -std=c++23 -stdlib=libc++
```

以上只是为了说明C\+\+20引入`[[no_unique_address]]`之后的变化，而不是指libc\+\+的`std::expected`实现存在问题。如果有尾部padding，又不想被复用，且不想手动填充对齐呢？标准也没有提供`[[non_overlapping]]`这样的属性啊！ 想来简单的办法就是借用下transparent comparator这种方案，不想被复用的类型就加一行`using non_overlapping = ???;`这样的代码，输入此类型的模板类再使用`is_non_overlapping`检测下做相应处理，或者把逻辑反过来？

```c++
template <typename T>
concept is_non_overlapping = requires { typename T::non_overlapping; };
```
这样把工作转移给库作者了，也不太合适，那么用**匿名结构体**包裹一下呢？又不符合C\+\+标准，还是手动对齐，远离魔法代码吧！

## stop_token

`stop_source`, `stop_token`和`stop_callback`是C\+\+20提供的用于协作式地停止线程的3个组件，其中使用了一个共享的状态，大体实现如下：

```c++
class stop_token {
  std::shared_ptr<__stop_state> state_;
};

class stop_source {
  std::shared_ptr<__stop_state> state_;
};
 
template <class Callback>
class stop_callback {
  [[no_unique_address]] Callback callback_;
  std::shared_ptr<__stop_state> state_;
};
```

其中的`__stop_state`可以实现如下：

```c++
class __stop_state {
  std::atomic<bool> stop_requested_;
  std::atomic<unsigned> stop_source_count_; // for stop_possible()
  std::list<stop_callback*> stop_callbacks_;
  std::mutex list_mutex_;
};
static_assert(sizeof(__stop_state) == 72);
static_assert(sizeof(stop_token) == 16);
```

libc++优化实现大体如下：

```c++
class __stop_state {
  // The "callback list locked" bit implements a 1-bit lock to guard
  // operations on the callback list
  //
  //       31 - 2          |  1                   |    0           |
  //  stop_source counter  | callback list locked | stop_requested |
  atomic<uint32_t> state_ = 0;
 
  // Reference count for stop_token + stop_callback + stop_source
  atomic<uint32_t> ref_count_ = 0;
 
  // Lightweight intrusive non-owning list of callbacks
  // Only stores a pointer to the root node
  __intrusive_list_view<stop_callback_base> callback_list_;
};
static_assert(sizeof(__stop_state) == 16);
static_assert(sizeof(stop_token) == 8);
```

首先是充分利用原子变量的每一个比特，不再使用`std::mutex`，然后是改用侵入式shared_ptr和侵入式链表，`__stop_state`内存占用大幅减小，`stop_token`降低为一个指针大小。

## ranges

### Segmented Iterators

```c++
std::deque<int> d = ...
// 1
for (int& i : d) {
    i = std::clamp(i, 200, 500);
}
// 2
std::ranges::for_each(d, [](int& i) {
    i = std::clamp(i, 200, 500);
});
```

以上代码的`std::ranges::for_each`版本要快得多，原因和`std::deque`的设计有关，该容器是用block组织的，`std::ranges::for_each`采用双层循环，契合了`std::deque`的设计。
![](/blog/cppcon2025_huixie90/deque.png)


### ranges::copy

```c++
std::vector<std::vector<int>> v = ...;
std::vector<int> out;
// 1
out.reserve(total_size);
for (const auto& inner : v) {
  for (int i: inner) {
    out.push_back(i);
  }
}
// 2
out.resize(total_size);
std::ranges::copy(v | std::views::join, out.begin());
```

`std::ranges::copy`要快很多，是因为对于以上情形，其内部调用的是`std::memmove`，代码如下：

```c++
template <class Iter, class OutIter>
  requires is_segmented_iterator<Iter>
pair<Iter, OutIter> __copy(Iter first, Iter last, OutIter result) {
  std::__for_each_segment(first, last, [&](auto inner_first, auto inner_last) {
      result = std::__copy(inner_first, inner_last, std::move(result)).second;
  });
  return std::make_pair(last, std::move(result));
}
template <class In, class Out>
  requires can_lower_copy_assignment_to_memmove<In, Out>
pair<In*, Out*> __copy(In* first, In* last, Out* result) {
  std::memmove(result, first, last - first);
  return std::make_pair(last, result + n);
}
```

### flat_map Insertion

```c++
// flat_map<int, double>
class flat_map {
  std::vector<int> keys_; // always sorted
  std::vector<double> values_;
  [[no_unique_address]] std::less<int> compare_;
};
std::flat_map<int, double> m1 = ...;
std::flat_map<int, double> m2 = ...;
// 1
for (const auto& [key, val] : m2) {
  m1.emplace(key, val);
}
// 2
m1.insert_range(m2);
```

使用`insert_range`更快，代码如下：

```c++
template<container-compatible-range<value_type> R>
constexpr void insert_range(R&& rg) {
  __append(ranges::begin(rg), ranges::end(rg)); // O(M)
 
  auto zv = ranges::views::zip(keys_, values_);
  ranges::sort(zv.begin() + old_size, zv.end()); // O(MLogM)
 
  ranges::inplace_merge(zv.begin(), zv.begin() + old_size, zv.end()); // O(M+N)
 
  auto dup_start = ranges::unique(zv).begin(); // O(M+N)
  __erase(dup_start); // O(M+N)
}

template <class InputIterator, class Sentinel>
void __append(InputIterator first, Sentinel last) {
  for (; first != last; ++first) {
    std::pair<Key, Val> kv = *first;
    keys_.insert(keys_.end(), std::move(kv.first));
    values_.insert(values_.end(), std::move(kv.second));
  }
}
```

以上代码中的`__append`是使用`for`循环一个个插入元素的，还可以更快，作者引入了`product_iterator`这个`concept`，代码如下：

```c++
template <class Iter>
  requires is_product_iterator_of_size<Iter, 2>
void __append(Iter first, Iter last) {
  using Traits = product_iterator_traits<Iter>;

  keys_.insert(keys_.end(),
      Traits::template get_iterator<0>(first),
      Traits::template get_iterator<0>(last));
 
  values_.insert(values_.end(),
      Traits::template get_iterator<1>(first),
      Traits::template get_iterator<1>(last));
}
```

## Testing in libc++

这部分没细看了……对不住了……

## 另外
purecpp每年也有[**大会**](http://purecpp.cn/detail?id=2468)，今年的**正在筹办**，感兴趣的公司或个人可以去支持下！