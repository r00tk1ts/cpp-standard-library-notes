# C++标准库-Utilities篇-元编程

**元程序**一词源于英文***metaprogram***。在英语中，***metaprogram***的意思是***a program about a program***，翻译过来就是**程序的程序**。直白地讲，**元程序**就是**用于操纵代码的程序**。

模板元编程主张效率至上，通过花里胡哨的操作与天书一般的代码将运行时的效率压榨至机制，当然与此同时也带来了高昂的编译期成本（在模板元编程中，一些运行期的工作被转移到了编译期）。Modern C++标准库的实现体几乎全部构建在元编程基础之上。所以，要看懂标准库源代码，首先就要懂一点元编程。相比Old C++晦涩难懂的黑魔法实现技巧，从C++11开始，各种语法糖（主要是模板相关）的引入使得元编程变得更加可读也更为容易编写（甚至能做到Old C++做不到的事儿），自此，Modern C++的元编程无论是可读性还是实现成本都有了大幅度的改良。

在libstdc++标准库手册中，元编程的一些基础建设被归类到了***Utilities/Metaprogramming***目录下，下面我们对这些基础建设中非常经典且有用的部分进行源码级别的讲解，并对各种高阶实现技巧进行诠释，穿插其中。

## 元程序与常量

元程序只能操纵那些在编译期就板上钉钉的物件，不能像普通的程序那样随意玩弄变量、发起函数调用。C++的元程序借用模板和常量（众所周知，C++的`const`不是真const，`constexpr`才是const）完成各种元函数的编写（语言体态生呈现为类模板），本质上做的事儿是编译期类型的计算，而非运行时值的计算。

我们知道，C++的编译期常量可以通过`constexpr`来定义：

```cpp
// 全局的g_value在编译期就已经确定了值
constexpr int g_value = 1;
```

那么元函数究竟长什么样呢？我们举一对例子来辅助理解，下面我们先写一下递归的斐波那契C++普通程序的实现：

```cpp
int fibonacci(int n) {
    if (n == 1 || n == 2) return 1;
    return fibonacci(n-1) + fibonacci(n-2);
}

int main() {
    std::cout << fibonacci(5) << std::endl;
    return 0;
}
```

暂时先不考虑这个经典实现的性能问题，这里不仅使用了参数变量`n`，同时还递归调用了函数本身，而这些在元程序中都是不被允许的，如果使用元编程来实现的话，会变成这个样子：

```cpp
template<int N>
struct fibonacci {
    static constexpr int value = fibonacci<N-1>::value + fibonacci<N-2>::value;
};

// 递归的终止条件
template<>
struct fibonacci<1> {
   static constexpr int value = 1;
};

template<>
struct fibonacci<2> {
   static constexpr int value = 1;
};

int main(){
    std::cout << fibonacci<5>::value << std::endl;
    return 0;
}
```



### `integral_constant`

而在元编程中，仅仅使用编译期常量是不够的，很多场合，我们需要一个能够把具体的某个常量值视为某种独特的类型的转换器，而从C++11开始，标准库为我们定义了`integral_constant`完成了这种转换：

```cpp
  /**
   * @defgroup metaprogramming Metaprogramming
   * @ingroup utilities
   *
   * Template utilities for compile-time introspection and modification,
   * including type classification traits, type property inspection traits
   * and type transformation traits.
   *
   * @since C++11
   *
   * @{
   */

  /// integral_constant
  template<typename _Tp, _Tp __v>
    struct integral_constant
    {
      static constexpr _Tp                  value = __v;
      typedef _Tp                           value_type;
      typedef integral_constant<_Tp, __v>   type;
      // 类型转换和函数调用操作符重载，用于在某些场合反向把值吐出去
      constexpr operator value_type() const noexcept { return value; }
      constexpr value_type operator()() const noexcept { return value; }
    };
```

这里面涉及了元编程中一个非常重要的技巧——"**trait**"，"**trait**"直译为"**特质**"，它用来在编译期取得某种类型信息，再根据这些信息依赖特化与SFINAE去进行元函数分支、循环结构的编译期选择。**"trait"**由于其发音和功效，中译一般称其为**“萃取”**，这一意译可谓惟妙惟肖。

> 是不是现在根本理解不了trait在干啥？这就对了，继续往后看。

从现在起，要牢牢记住`integral_constant`中定义的三个物件：`value`，`value_type`和`type`，这三个东西各有其妙用，字面意义上分别是静态编译期常量（使用非类型模板参数`__v`做值初始化）、模板参数`_Tp`类型、模板类类型。

> 注意`type`是具体的模板类类型，不是类模板，前者是类型，后者是模板，模板类是类模板产生的具体类。

有了这个东西，我们就可以做如下的定义：

```cpp
typedef integral_constant<int, 1> one_type;
typedef integral_constant<int, 2> two_type;
typedef integral_constant<int, 3> three_type;
```

此时，`one_type`，`two_type`，`three_type`各是完全不同的类型，以`one_type`举例来说，它的内部有着静态成员`value`（值为1）且定义了`value_type`和`type`两种类型，前者可以通过`one_type::value`取到常量值，后两者可以通过`typename one_type::value_type`和`typename one_type::type`取到特定类型。

借助`value`，我们可以做编译期断言：

```cpp
static_assert(one_type::value + two_type::value == three_type::value, "1 + 2 != 3");
```

当然，你完全可以这样写来发挥同等效用：

```cpp
static_assert(1 + 2 == 3, "1 + 2 != 3");
```

那么问题来了，这么费劲把自己绕进去有啥用呢？`value_type`和`type`又有啥用呢？让子弹飞一会儿~

### `true_type`,`false_type`

我们知道，C++除了支持整型常量以外，还支持bool型常量，那么自然也就有`true`和`false`对应的两种类型，在标准库中，它们叫作`true_type`和`false_type`(对熟悉**TMP**的同学来说，这个是最熟悉且使用频率最高的物件)：

```cpp
using true_type = integral_constant<bool, true>;
using false_type = integral_constant<bool, false>;
```

C++17还补充定义了更泛用的别名模板：

```cpp
  template<bool __v>
  	using bool_constant = integral_constant<bool, __v>;
```

