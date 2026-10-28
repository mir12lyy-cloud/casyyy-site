+++
title = "128位整数 C++ 中的一些事"
description = "一些 128 位整数的变化" 
date = 2026-10-28T10:58:23+08:00
image = "ichika.png"
math = false
license = "CC BY-NC-SA 4.0"
comments = true
draft = false
toc = true
categories = ["知识"]
tags = ["C/C++"]
slug = "integer128"
+++

## 文章背景

现有的关于 C++ 的 128 位整数还是挺多的。这篇文章主要讲述 C++ 的 128 位整数在三大编译器的表现，C++ 新标准（C++20）下的新变化。

## 编译器支持

我们的固有印象是 128 位整数的编译器支持局限于 clang, gcc。MSVC 对 128 位的整数似乎是缺位的。

不过，MSVC 事实上具有库模拟的 128 位整数。存放在 `<__msvc_int128.h>` 中，我们可以如下使用。

```C++
#include <__msvc_int128.hpp>

int main() {
    using i128 = std::_Signed128;
    using u128 = std::_Unsigned128;

    i128 a = 32;
    u128 b = 223;

    auto c = a * b;
    // 其他事……
}
```

不过效率上就不如编译器内置支持的了。也不一定稳定。

接下来讲述的有关 128 位整数的特殊变化基于 clang 和 gcc。MSVC 不再过多讲述。

## 关于IO

在一些常用场景，诸如算法竞赛中，我们需要使用 128 位整数存储一些大数。但是 128 位整数通常没法直接输入打印。

```C++
#include <cstdio>
#include <iostream>
using i128 = __int128;
using u128 = unsigned __int128;

int main() {
    i128 a = 0;
    scanf("%lld", &a); // 通常不起效。
    std::cin >> a;     // 没有针对128位整数的重载。

    printf("%lld", a); // 同理。
    std::cout << a;    // 同理
}
```

### 一般应对

为了应对这种情况，通常我们需要手写以下的 IO，这里举一个现成例子（摘录自：[Codeforces关于128位整数的tip](https://codeforces.com/blog/entry/75044)）：

```C++
__int128 read() {
    __int128 x = 0, f = 1;
    char ch = getchar();
    while (ch < '0' || ch > '9') {
        if (ch == '-') f = -1;
        ch = getchar();
    }
    while (ch >= '0' && ch <= '9') {
        x = x * 10 + ch - '0';
        ch = getchar();
    }
    return x * f;
}
void print(__int128 x) {
    if (x < 0) {
        putchar('-');
        x = -x;
    }
    if (x > 9) print(x / 10);
    putchar(x % 10 + '0');
}
```

对于算法竞赛选手已经成为了标配。当然，如果你用的是 boost 库的 128 位整数，那倒是可以直接进行输入输出。

```C++
#include <iostream>
#include <boost/multiprecision/cpp_int.hpp>

int main() {
    using namespace boost::multiprecision;
    uint128_t a = 0;
    std::cin >> a;
    std::cout << a;
    return 0;
}
```

### 新变化

接下来讲述输入输出在 C++20 的变化。

对于输入，C++20 的输入形式没有太大变化。

对于输出，C++20 引入了 `<format>` ，同时 C++23 引入了 `<print>` 。使用 fmt 式的语法和格式控制符简化输出。

```C++
#include <format>
#include <iostream>
#include <print>

int main() {
    int a = 32;
    std::cout << std::format("{}\n", a);   // 默认输出
    std::cout << std::format("{:x}\n", a); // 16进制输出
    std::print("{}\n", a);
    std::print("{:x}\n", a);
}
```

然后有趣的地方就来了。clang 和 gcc 的编译器内置的 128 位整数中，也可以像以下方式输出：

```C++
#include <format>
#include <iostream>
#include <print>

using i128 = __int128;

int main() {
    i128 a = 32;
    std::cout << std::format("{}\n", a);   // 默认输出
    std::cout << std::format("{:x}\n", a); // 16进制输出
    std::print("{}\n", a);
    std::print("{:x}\n", a);
}
```

### 原理

事实上并不复杂，基于 `<format>` 的输出的核心是一个 `std::formatter` 的结构体，在 gcc 的 libstdc++ 中，被以下定义：

```C++
  template<typename _Tp, typename _CharT>
    struct formatter
    {
      formatter() = delete; // No std::formatter specialization for this type.
      formatter(const formatter&) = delete;
      formatter& operator=(const formatter&) = delete;
    };
```

默认的 `std::formatter` 无法被实例化，防止意外的打印。

libstdc++ 通过控制一个 constexpr 变量来决定一个整数是否能被输出。默认的形式是：

```C++
template<typename _Tp>
constexpr bool __is_formattable_integer = __is_integer<_Tp>::__value;

template<typename _Tp>
concept __formattable_integer = __is_formattable_integer<_Tp>;
```

然后在以下中被检查：

```C++
template<__format::__formattable_integer _Tp, __format::__char _CharT>
struct formatter<_Tp, _CharT>;
```

`__is_integer` 是 libstdc++ 内部用于判断一个类型是否位整型的一个 trait。事实上，针对非整数，会被如下定义：

```C++
template<typename _Tp>
    struct __is_integer
    {
      enum { __value = 0 };
      typedef __false_type __type;
    };
```

整数则是把对应的 `__value` 设置成 0，`__false_type` 设置为 `__true_type`。

以及针对扩展整数的生成宏：

```C++

#define __INT_N(TYPE)    \
  __extension__    \
  template<>    \
    struct __is_integer<TYPE>  \
    {     \
      enum { __value = 1 };  \
      typedef __true_type __type; \
    };     \
  __extension__    \
  template<>    \
    struct __is_integer<unsigned TYPE> \
    {     \
      enum { __value = 1 };  \
      typedef __true_type __type; \
    };
```

libstdc++ 对 128 位整数，会通过这样的宏生成对应的 traits，然后被检测到，`__is_integer` 对于 128 位整数是 `true`，同时还做了一个偏特化进行强调：

```C++
#if defined __SIZEOF_INT128__
  template<> inline constexpr bool __is_formattable_integer<__int128>  = true;
  template<> inline constexpr bool __is_formattable_integer<unsigned __int128> = true;
#endif
```

这样子，`__formattable_integer` 对于 128 位整数会被认为是 `true` 。然后就可以使用 libstdc++ 针对整数的 `std::formatter` 了。`std::format` 和 `std::print` 都基于 `std::formatter`，因此就能正常输出了。

而 clang 在使用自己的 libc++ 时，则是通过如下方式进行支持：

```C++
template <__fmt_char_type _CharT>
struct formatter<__int128_t, _CharT> : public __formatter_integer<_CharT> {};
```

直接给内置 128 位整数偏特化 `std::formatter` ，都采用内部的 `__formatter_integer` 进行统一实现。

然后在 `__formatter_integer` 中，有一个这样的类型别名：

```C++
using _Type = __make_32_64_or_128_bit_t<_Tp>;
static_assert(!is_void<_Type>::value, "unsupported integral type used in __formatter_integer::__format");
```

定义在内部的 `<__type_traits/make_32_64_or_128_bit.h>` 中，定义如下：

```C++
using __make_32_64_or_128_bit_t _LIBCPP_NODEBUG =
    __copy_unsigned_t<_Tp,
        __conditional_t<sizeof(_Tp) <= sizeof(int32_t),    int32_t,
        __conditional_t<sizeof(_Tp) <= sizeof(int64_t),    int64_t,
#if _LIBCPP_HAS_INT128
        __conditional_t<sizeof(_Tp) <= sizeof(__int128_t), __int128_t,
        /* else */                                         void>
#else
        /* else */                                         void
#endif
    > > >;
```

在 clang 内部有 128 位整数下，实例化结果会是 128 位整数，然后就可以正常进行格式化了。

**注意：如果 clang 使用的是 MSVC STL，即使 128 位仍然存在，但此时是无法输出的。MSVC STL没有针对 128 位整数进行特化。**

### 总结

在 C++20 之后，C++ 对于 128 位整数的输入没有太大变化，集中在输出。对于主要使用 gcc 或非 MSVC STL 的 clang，可以免去冗余的辅助函数：

```C++
#include <print>
int main() {
    using i128 = __int128;

    i128 a = 32;
    std::println("{}", a);
}
```

## 关于位操作

C++20 还有一个重要特性是：`<bit>` 头文件针对了旧有的位操作提供了支持。

C++20 中支持的有关位运算操作如下：

- `std::has_single_bit` 检查一个数是否是 2 的整数次幂。
- `std::bit_ceil` / `std::bit_floor` 查找不小于给定值的最小/最大 2 的整数幂。
- `std::bit_width` 查找表示给定值所需的最小位数，类似于 log2(n)
- `std::rotl` / `std::rotr` 计算按位左旋/右旋的结果
- `std::countl_zero` / `std::countl_one` / `std::countr_zero` / `std::countr_one` 计算从最高/最低有效位开始连续 0/1 的数量。
- `std::popcount` 计算无符号整数中 1 位的数量。

然后，这些操作在 gcc 的 libstdc++ 中都支持 128 位。

```C++
#include <print>
#include <bit>

int main() {
    using u128 = unsigned __int128;
    u128 a = 3311123123;
    std::println("{}", std::bit_width(a));
    std::println("{}", std::bit_ceil(a));
    std::println("{}", std::bit_floor(a));
    std::println("{}", std::has_single_bit(a));
    std::println("{}", std::rotl(a, 3));
    std::println("{}", std::rotr(a, 3));
    std::println("{}", std::countl_zero(a));
    std::println("{}", std::countl_one(a));
    std::println("{}", std::countr_zero(a));
    std::println("{}", std::countr_one(a));
    std::println("{}", std::popcount(a));
}
```

非 MSVC STL 的 clang 也可以如此使用。

## 结语

总之，128 位整数作为扩展整数，在 C++20 后变得更好的和标准库进行结合。减少了使用 128 位整数的一些负担。

虽然不一定能直接用上，但知晓后也许可以有新的解法。
