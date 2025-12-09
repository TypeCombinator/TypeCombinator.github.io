---
title: 用Zig语言实现mock_head
subtitle:
date: 2025-09-14T09:21:22+08:00
slug: mock_head_for_zig
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
  - zig
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
## 前言

周末闲来无事，将之前的Zig代码整理记录出来吧！对于Zig语言，我已经关注了相当长的一段时间，作为一门新语言，它有着相当多的良好设计，最值得一提的是`compiletime`，并将类型作为值，这样在编译期对类型的计算和写运行时代码基本无异，而在C\+\+中，与此特性对等的是C\+\+26的[**P2996**](https://wg21.link/p2996)，不过C\+\+类型系统相当复杂，当前的[P2996实现](https://github.com/bloomberg/clang-p2996)只能算是功能验证，编译还很缓慢，而Zig从设计之初就考虑到了编译速度，这是一个相当大的优势。
顺带一提，**Kris Jusiak**的元编程库[mp](https://github.com/qlibs/mp)可以作为P2996的开胃菜，这在我之前的[文章](https://typecombinator.github.io/posts/unified_intrusive_linked_list)有解读。
<!--more-->
可是，我还是太懒了，虽然关注了Zig语言这么久，却一直没有动力去学习它，前段时间我找到一个切入点，尝试用Zig实现**mock_head**，为此我花了周末的一天时间学习并实践。使用Zig尝试的另一个原因是我了解到，Zig语言看起来没有C\+\+的生命周期和严格别名的这样的限制，但是对于是否有对象的边界检查以及指针的其他限制，我还不是太清楚，不过既然决定了，还是要**付诸实践**。

## 实现
我在4种侵入式链表中挑选了`idslist`作为首次尝试，这里先贴一下`idslist`使用**mock_head**的示意图，如下：
![](/blog/uit/dslist_empty.png)
![](/blog/uit/dslist.png)

实现起来基本思路与C++是一致的，将数据节点类型和`right`成员名称作为`compiletime`参数，再结合`builtin`函数`@offsetOf`，即可获得**mock_head**，代码（完整代码见[compiler explorer](https://zig.godbolt.org/z/Mz6s18af3)）如下：

```zig
pub fn idslist(comptime T: type, comptime right_field_name: []const u8) type {
    return extern struct {
        const Self = @This();

        m_right: ?*T = undefined,
        m_left: *T = undefined,

        fn mock_head(self: *Self) *T {
            return @ptrFromInt(@intFromPtr(self) - @offsetOf(T, right_field_name));
        }
    };
}
```
减去成员偏移获取对象指针，这与`container_of`的实现是一致的，这里需要提一下，为什么我没有使用`@fieldParentPtr`这个`builtin`函数呢？是因为在其官方文档中，我看到了如下描述：

> "If `field_ptr` does not point to the `field_name` field of an instance of the result type, and the result type has ill-defined layout, invokes unchecked [Illegal Behavior](https://ziglang.org/documentation/master/#Illegal-Behavior)."

`@fieldParentPtr`函数的声明如下：

```zig
@fieldParentPtr(comptime field_name: []const u8, field_ptr: *T) anytype
```

Zig使用的是IB（[Illegal-Behavior](https://ziglang.org/documentation/master/#Illegal-Behavior)）而不是UB，由于**mock_head**并不是一个实际存在的对象实例，我认为这会导致Zig语言所描述的IB，所以，我最终采用了`@offsetOf`去实现，拿到了**mock_head**之后，一切都变得简单了，链表初始化为空的代码如下：

```zig
pub fn init(self: *Self) void {
    self.m_right = null;
    self.m_left = self.mock_head();
}
```

链表为空时，`m_left`指向的是`mock_head`而不是空，而`push_front`、`push_back`和`pop_front`代码如下：

```zig
pub fn push_front(self: *Self, node: *T) void {
    if (self.m_right == null) {
        self.m_left = node;
    }
    @field(node, right_field_name) = self.m_right;
    self.m_right = node;
}

pub fn push_back(self: *Self, node: *T) void {
    @field(node, right_field_name) = null;
    @field(self.m_left, right_field_name) = node;
    self.m_left = node;
}

pub fn pop_front(self: *Self) ?*T {
    const first: ?*T = self.right;
    if (first == null) {
        return null;
    }
    const first_right: ?*T = @field(first.?, right_field_name);
    // Tail?
    if (first_right == null) {
        self.m_left = self.mock_head();
    }
    self.m_right = first_right;
    return first;
}
```

可以看到`push_back`是不需要分支的，这是使用**mock_head**所带来的收益！最后，我把完整代码（也放在了[compiler explorer](https://zig.godbolt.org/z/Mz6s18af3)）贴出，如下：

```zig
const std = @import("std");

pub fn idslist(comptime T: type, comptime right_field_name: []const u8) type {
    return extern struct {
        const Self = @This();

        m_right: ?*T = undefined,
        m_left: *T = undefined,

        pub fn init(self: *Self) void {
            self.m_right = null;
            self.m_left = self.mock_head();
        }

        pub fn empty(self: *Self) bool {
            return self.m_right == null;
        }

        pub fn clear(self: *Self) void {
            self.m_right = null;
            self.m_left = self.mock_head();
        }

        pub fn front(self: *Self) ?*T {
            return self.m_right;
        }

        pub fn back(self: *Self) *T {
            return self.m_left;
        }

        pub fn push_front(self: *Self, node: *T) void {
            if (self.m_right == null) {
                self.m_left = node;
            }
            @field(node, right_field_name) = self.m_right;
            self.m_right = node;
        }

        pub fn push_back(self: *Self, node: *T) void {
            @field(node, right_field_name) = null;
            @field(self.m_left, right_field_name) = node;
            self.m_left = node;
        }

        pub fn pop_front(self: *Self) ?*T {
            const first: ?*T = self.right;
            if (first == null) {
                return null;
            }
            const first_right: ?*T = @field(first.?, right_field_name);
            // Tail?
            if (first_right == null) {
                self.m_left = self.mock_head();
            }
            self.m_right = first_right;
            return first;
        }

        pub fn remove(self: *Self, node: *T) ?*T {
            var left: *T = self.mock_head();
            // var right_opt: ?*T = self.m_right;
            var right_opt: ?*T = @field(left, right_field_name);
            while (right_opt) |right| {
                if (right == node) {
                    right_opt = @field(right, right_field_name);
                    @field(left, right_field_name) = right_opt;
                    // Tail?
                    if (right_opt == null) {
                        self.m_left = left;
                    }
                    return node;
                }
                left = right;
                right_opt = @field(left, right_field_name);
            }
            return null;
        }

        fn mock_head(self: *Self) *T {
            // Will this cause potential Illegal Behavior?
            return @ptrFromInt(@intFromPtr(self) - @offsetOf(T, right_field_name));
        }

        pub const Iterator = struct {
            current: ?*T,

            pub fn next(self: *Iterator) ?*T {
                if (self.current) |node| {
                    self.current = @field(node, right_field_name);
                    return node;
                }
                return null;
            }
        };

        pub fn iterator(self: *Self) Iterator {
            return Iterator{
                .current = self.m_right,
            };
        }
    };
}

const Apple = extern struct { sn: i64, right: ?*Apple = undefined };

const AppleList = idslist(Apple, "right");

test "push front" {
    var a0: Apple = .{ .sn = 0 };
    var list: AppleList = .{};
    list.init();

    list.push_front(&a0);

    try std.testing.expect(!list.empty());
    try std.testing.expectEqual(list.front(), &a0);
    try std.testing.expectEqual(list.back(), &a0);

    var a1: Apple = .{ .sn = 1 };
    list.push_front(&a1);
    try std.testing.expect(!list.empty());
    try std.testing.expectEqual(list.front(), &a1);
    try std.testing.expectEqual(list.back(), &a0);
}

test "push back" {
    var a0: Apple = .{ .sn = 0 };
    var list: AppleList = .{};
    list.init();
    try std.testing.expect(list.empty());

    list.push_back(&a0);

    try std.testing.expect(!list.empty());
    try std.testing.expectEqual(list.front(), &a0);
    try std.testing.expectEqual(list.back(), &a0);

    var a1: Apple = .{ .sn = 1 };
    list.push_back(&a1);
    try std.testing.expect(!list.empty());
    try std.testing.expectEqual(list.front(), &a0);
    try std.testing.expectEqual(list.back(), &a1);
}

test "remove" {
    var a0: Apple = .{ .sn = 0 };
    var a1: Apple = .{ .sn = 1 };
    var a2: Apple = .{ .sn = 2 };
    var list: AppleList = .{};
    list.init();
    list.push_back(&a0);
    list.push_back(&a1);
    list.push_back(&a2);

    try std.testing.expect(!list.empty());

    try std.testing.expectEqual(list.remove(&a1), &a1);
    try std.testing.expectEqual(list.front(), &a0);
    try std.testing.expectEqual(list.back(), &a2);

    try std.testing.expectEqual(list.remove(&a0), &a0);
    try std.testing.expectEqual(list.front(), &a2);
    try std.testing.expectEqual(list.back(), &a2);

    try std.testing.expectEqual(list.remove(&a2), &a2);
    try std.testing.expect(list.empty());
}

pub fn main() !void {
    std.debug.print("Try mock_head!\n", .{});
    std.debug.print("offsetof: {}\n", .{@offsetOf(Apple, "right")});
}
```

## 结果


不幸的是，Zig语言也表现出只能在Debug模式下通过测试，**Release模式则测试失败**的现象！正如文章开头所提，虽然Zig语言看起来没有C\+\+的生命周期和严格别名的这样的限制，但是对于是否有对象的边界检查以及指针的其他限制，我还不是太清楚，毕竟我只学了一天！从结果来看，可能也是违反了Zig语言的某些规则，有人推测是因为Release模式使用了LLVM后端的缘故，当前Zig开发的后端只用在了Debug模式，等Zig彻底去除了LLVM后端再测试吧！不过，LLVM后端累积那么多年的PASS，支持那么多平台，想要替代，也许得等到**猴年马月**了！

如果不用**mock_head**，将头节点的`m_left`指针指向头节点的`m_right`成员是否可行呢？我想这应该是个可行的方案，但是这脱离了本文的目的了！

## 总结

这只是对Zig语言的一次粗浅的尝试，当出现这种Release模式下测试失败的问题，我暂时不能像使用C\+\+那样想到对应的解决办法，毕竟，我对Zig的底层细节也不熟悉，有时间再去研究和折腾吧！Zig语言作为后继者，其实也并没有完全踩中先辈踏过的台阶，比如Zig语言的无栈协程设计，就走了相当长的一个弯路，只能说人类的进步是螺旋上升的！当前，Zig语言距离成熟还需要一个漫长的时间，在此期间我也会持续关注，现阶段对我来说，还是**C\+\+大法好**！