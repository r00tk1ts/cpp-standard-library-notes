# C++标准库-Utilities篇-元编程

**元程序**一词源于英文***metaprogram***。在英语中，***metaprogram***的意思是***a program about a program***，翻译过来就是**程序的程序**。直白地讲，**元程序**就是**用于操纵代码的程序**。

模板元编程主张效率至上，通过花里胡哨的操作与天书一般的代码将运行时的效率压榨至极致，当然与此同时也带来了高昂的编译期成本（在模板元编程中，一些运行期的工作被转移到了编译期）。Modern C++标准库的实现体几乎全部构建在元编程基础之上。所以，要看懂标准库源代码，首先就要懂一点元编程。相比Old C++晦涩难懂的黑魔法实现技巧，从C++11开始，各种语法糖（主要是模板相关）的引入使得元编程变得更加可读也更为容易编写（甚至能做到Old C++做不到的事儿），自此，Modern C++的元编程无论是可读性还是实现成本都有了大幅度的改良。

在libstdc++标准库手册中，元编程的一些基础建设被归类到了***Utilities/Metaprogramming***目录下，下面我们对这些基础建设中非常经典且有用的部分进行源码级别的讲解，并对各种高阶实现技巧进行诠释，穿插其中。

## 元程序与常量

元程序只能操纵那些在编译期就板上钉钉的物件，不能像普通的程序那样随意玩弄变量、发起函数调用。C++的元程序借用模板、常量（众所周知，C++的`const`不是真const，`constexpr`才是const）和各种关键字(`sizeof`,`decltype`啥的)完成各种元函数的编写（语言体态上大部分呈现为类模板），本质上做的事儿是编译期类型的计算，而非运行时值的计算。

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

// 利用特化来处理递归的终止条件
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

这里`fibonacci<5>`实际上在编译期就已经算好了。

> 这让我不禁想起了OI里用模板元作弊的那些骚操作。。。

这里的元程序就是借助了C++的模板来实现，`fibonacci`从语言上来看是个类模板，但是在元编程的世界里它是个元函数，模板参数`N`是它的调用参数，与运行时普通的函数调用不同，在元函数里，无论是“实参”还是“形参”它们都是统一的，并且在编译期就已经确定了。

## `integral_constant`

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

在注释的使用说明中提到了元编程中一个非常重要的技巧——"**trait**"，"**trait**"直译为"**特质**"（大抵就是形容某种物件的逼格），它用来在编译期取得某种类型信息。Trait在标准库中大量使用，诸如iterator trait（这东西用于桥接容器和算法，毕竟算法内部不知道传进来的是啥子东西，就只能定义一套接口，即iterator来统一操纵，trait负责获取）、char trait（处理不同字符类型，`basic_string`里满地都是）、type trait（C++11之后不断丰富，抄，就硬抄）等。**"Trait"**由于其发音和功效，中译一般称其为**“特性萃取”**，这一意译可谓惟妙惟肖。

> Bjarne Stroustrup对Trait的解释：
>
> > Think of a trait as a small object whose main purpose is to carry information used by another object or algorithm to determine *policy* or *implementation* details.
> >
> > Trait是种小对象，它的主要目的就是携带被另一个对象或算法所使用的信息，以确定某个policy或是implementation的细节。

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

C++17还补充定义了更泛用的别名模板并对外曝光（实际上C++11时就有了`__bool_constant`，但这双下划线开头的懂得都懂，C++17标准才让这个东西名正言顺）：

```cpp
  template<bool __v>
  	using bool_constant = integral_constant<bool, __v>;
```

这里有人就会问了：bool类型一共不就`true`和`false`两个值吗，定义`true_type`和`false_type`完全够了啊，非要曝光出去这个`bool_constant`干啥？

实际上，我们写元程序时，很多情况都是要依赖于元程序的调用参数来确定`__v`具体是个啥东西，并不是直接传一个`true`或`false`的参数进来（比如`bool_constant<sizeof(xxx) == 1>`，时刻铭记：这里的判断在编译期就确定了）。

## Type traits

C++11标准化的Type traits（中译一般称作**“型别萃取”**，直译的叫**“类型特性”**）大幅度降低了模板元编程的上手门槛。我们前面讲到的`integral_constant`本身不属于type trait，但是它会为各种type trait所用。千言万语，究竟何为Type Traits呢？手册是如下定义的：

> Type traits defines a compile-time template-based interface to query or modify the properties of types.

翻译一下就是在编译期基于模板去查询或修改类型的属性。嗯，这依然是一句黑话，讳莫如深，我们看看它都给了哪些元函数，又分别能做什么。

**Type properties**

| Categories           | Traits                                                       |
| -------------------- | ------------------------------------------------------------ |
| Primary type         | `is_void`,`is_null_pointer`,`is_integral`,`is_floating_point`,`is_array`,`is_enum`,`is_union`,`is_class`,`is_function`,`is_pointer`,`is_lvalue_reference`,`is_rvalue_reference`,`is_member_object_pointer`,`is_member_function_pointer` |
| Composite type       | `is_fundamental`,`is_arithmetic`,`is_scalar`,`is_object`,`is_compound`,`is_reference`,`is_member_pointer` |
| Type properties      | `is_const`,`is_volatile`,`is_trivial`,`is_trivially_copyable`,`is_standard_layout`,`is_pod`... |
| Supported operations | `is_constructible`,`is_default_constructible`,`is_copy_constructible`,`is_assignable`... |
| Property queries     | `alignment_of`,`rank`,`extent`                               |
| Type relationships   | `is_same`,`is_base_of`,`is_convertible`...                   |

**Type modifications**

| Categories                  | Traits                                                       |
| --------------------------- | ------------------------------------------------------------ |
| Const-volatility specifiers | `remove_cv`,`remove_const`,`remove_volatile`,`add_cv`,`add_const`,`add_volatile` |
| References                  | `remove_reference`,`add_lvalue_reference`,`add_rvalue_reference` |
| Pointers                    | `remove_pointer`,`add_pointer`                               |
| Sign modifiers              | `make_signed`,`make_unsigned`                                |
| Arrays                      | `remove_extent`,`remove_all_extents`                         |

**Miscellaneous transformations**

`aligned_storage`,`aligned_union`,`decay`,`remove_cvref`,`enable_if`,`conditional`,`common_type`,`common_reference`,`underlying_type`,`result_of`,`invoke_result`,`void_t`,`type_identity`

**Operations on traits**

`conjunction`,`disjunction`,`negation`

**Helper classes**

`integral_constant`,`bool_constant`,`true_type`,`false_type`

可以看到**Type properties**提供了可供查询的属性，而**Type modifications**则会修改属性，它们分门别类，各有各的妙用，除此之外，还有杂项转换type trait（诸如`enable_if`,`conditonal`这些都可以叫做**helper types**）以及C++17引入的逻辑与或非操作。

而最后的Helper classes大家已经熟悉了，它作为基石游走在各种type trait之间，完成编译期就能处理的骚操作。

### Miscellaneous transformations

挑选几个颇为有用的trait，并详述一下它的设计和使用。

#### `conditional`

先来了解一下`std::conditional`，这东西大家应该都很熟悉，我们先看看[手册](https://en.cppreference.com/w/cpp/types/conditional)对它的解释：

```cpp
template< bool B, class T, class F >
struct conditional;
```

> Provides member typedef `type`, which is defined as `T` if `B` is true at compile time, or as `F` if `B` is false.
>
> The behavior of a program that adds specializations for `conditional` is undefined.

举个例子：

```cpp
// Type1会在编译期确定，其类型为double，因为B是false
// 同理Type2是int
typedef std::conditional<sizeof(int) >= sizeof(double), int, double>::type Type1;
typedef std::conditional<sizeof(int) <= sizeof(double), int, double>::type Type2;
```

是否看这个`::type`非常眼熟，别急，我们先看看手册里给出的一个可行的实现方案：

```cpp
// 用不到的模板参数可以省略name
// 主模板处理B为true的情景
template<bool, class T, class>
struct conditional {
    typedef T type;
};

// 利用偏特化处理B为false的情景
template<class T, class F>
struct conditional<false, T, F> {
    typedef F type;
}
```

实际上这就是libstdc++的实现，只是代码风格更glibc一点：

```cpp
// Primary template.
  /// Define a member typedef @c type to one of two argument types.
  template<bool _Cond, typename _Iftrue, typename _Iffalse>
    struct conditional
    { typedef _Iftrue type; };

  // Partial specialization for false.
  template<typename _Iftrue, typename _Iffalse>
    struct conditional<false, _Iftrue, _Iffalse>
    { typedef _Iffalse type; };
```

#### `enable_if`

再来看看`std::enable_if`:

```cpp
template< bool B, class T = void >
struct enable_if;
```

>If `B` is true, **std::enable_if** has a public member typedef `type`, equal to `T`; otherwise, there is no member typedef.
>
>This metafunction is a convenient way to leverage [SFINAE](https://en.cppreference.com/w/cpp/language/sfinae) to conditionally remove functions from [overload resolution](https://en.cppreference.com/w/cpp/language/overload_resolution) based on type traits and to provide separate function overloads and specializations for different type traits. **std::enable_if** can be used as an additional function argument (not applicable to operator overloads), as a return type (not applicable to constructors and destructors), or as a class template or function template parameter.
>
>The behavior of a program that adds specializations for `enable_if` is undefined.

一个可能的设计：

```cpp
// 主模板处理B为false的情景
// 注意这个类模板给了完整的定义，只是内部没有type这一类型
template<bool B, class T = void>
struct enable_if {};
 
// 偏特化处理B为true的情景
// 此时type类型就是模板参数T的类型
template<class T>
struct enable_if<true, T> { typedef T type; };
```

相比`conditional`，`enable_if`的设计更为简单，但使用机制却更加复杂：对于`B`为`false`的情景，`enable_if<false, XXX>::type`是unresolved symbol，但由于SFINAE机制（前提是你得正确使用），并不会导致编译错误，而是在重载集合中把这一候选踢出去。

> SFINAE是模板元编程的基础，想往下看，这个必须得先搞懂。

C++14后，标准库自己定义了别名模板，来简化`::type`的写法：

```cpp
template< bool B, class T = void >
using enable_if_t = typename enable_if<B,T>::type;
```

> C++14为标准库很多trait都做了别名模板来简化书写，它们都以trait加后缀`_t`来命名。其实相对value还有个`_v`

关于`enable_if`用途，手册里给了一个很经典的正确和错误的例子，以加深印象：

```cpp
/* WRONG */
 
struct T {
    enum { int_t, float_t } type;
    template <typename Integer,
              typename = std::enable_if_t<std::is_integral<Integer>::value>
    >
    T(Integer) : type(int_t) {}
 
    template <typename Floating,
              typename = std::enable_if_t<std::is_floating_point<Floating>::value>
    >
    T(Floating) : type(float_t) {} // error: treated as redefinition
};
 
/* RIGHT */
 
struct T {
    enum { int_t, float_t } type;
    template <typename Integer,
              std::enable_if_t<std::is_integral<Integer>::value, bool> = true
    >
    T(Integer) : type(int_t) {}
 
    template <typename Floating,
              std::enable_if_t<std::is_floating_point<Floating>::value, bool> = true
    >
    T(Floating) : type(float_t) {} // OK
};
```

这里先别管`std::is_integral<Integer>::value`或是`std::is_floating_point<Floating>::value`，它们的实现我们后面会讲，总之二者都会在编译期返回一个bool值（要么是`false`，要么是`true`）。

在编译期处理时，如果拿到的值是`false`，那么由于`::type`不存在，就会触发SFINAE而从候选集中被丢弃。然而，SFINAE这东西理解起来不难，想要用对还是需要经验。这不，上面手册中的例子就是一个标准的错误用法。

上例中的前者，两个构造器之所以被编译器视为重复定义，是因为这两个函数模板从编译器的视角来看是完全等同的(equivalence)，编译器根本不care默认模板实参，`Integer`和`Floating`仅仅是名字不同，本质上都是一个类型参数而已。因此，在编译器眼里，两个重载的构造器都是`template<typename P, typename>T(P)`，你甚至都没走到SFINAE这一步呢，编译期就已经认为这是个重定义给你打回来了。

至于后者就不一样了，这里换成了非类型模板参数，对于两个构造器来说，有且仅有其中的一个可以成为正当候选（对整型来说就是前者，对浮点型来说是后者），因为另一个总会被SFINAE干掉：

```cpp
// 对int来说，能成功实例化的只有这一个候选
template<typename int, bool anonymous=true>
T(int) : type(int_t) {}

// 对float来说，能成功实例化的只有这一个候选
template<typename float, bool anonymous=true>
T(float) : type(float_t) {}
```

至于匿名的第二模板参数，它的作用只是为了通过`enable_if`来做出分支选择，利用SFINAE丢弃错误的“重载”，它的值是什么其实无关紧要，你给`false`也行，它甚至连名字都没有，因为构造函数内部根本用不到这个东西，它的使命只是为了在编译期做分支选择，找到那个最合适的候选人。此外，为啥这里非要给个默认实参呢，这个其实道理就更简单了，因为首先这个东西无法在调用处自动推导，这就意味着需要程序员自己去写`T<float, true>`，这不仅显得多余，还浪费了第一个模板实参`float`的自动推导。因此，我们给它随便设个默认值就行了。

对于初学者来说，经常会把另外的一个例子和上面这个搞混：

```cpp
// primary template
template<typename P, typename Enable = void>
class T{};	

// partial specialization of T
template<typename P>
class T<P, std::enable_if_t<std::is_floating_point<P>::value>> {};
```

像上面的这种是完全可以的，因为下面是对主模板的偏特化，并不是重新定义，偏特化的模板相比主模板更加的特殊，所以一旦参数匹配，它会优先成为候选实例化：

```cpp
T<int>{};		// matches the primary template
T<double>{};	// matches the partial specialization
```

### Type properties

#### `is_same`

现在我们来看类型属性中颇为基础的`is_same` trait：

```cpp
template< class T, class U >
struct is_same;
```

> If `T` and `U` name the same type (taking into account const/volatile qualifications), provides the member constant `value` equal to true. Otherwise `value` is false.
>
> Commutativity is satisfied, i.e. for any two types `T` and `U`, is_same<T, U>::value == true if and only if is_same<U, T>::value == true.

根据[手册](https://en.cppreference.com/w/cpp/types/is_same)的定义，其实我们现在完全可以利用前面所学的知识，自己实现出来，无非就是利用类模板特化来处理分支：

```cpp
// 主模板匹配T和U类型不同的情景
template<typename T, typename U>
struct is_same : public false_type {};

// 偏特化来处理模板参数类型相同的情景
template<typename T>
struct is_same<T,T> : public true_type {};
```

看到没，`false_type`和`true_type`就在这里派上了用场，作为基类，其内部的`value`这一静态常量就暴露了出去，视模板参数的不同而在不同的实例化体中或为`true`，或为`false`。

> 实际上libstdc++的实现如出一辙，只是他有个历史包袱`__is_same`要兼容。

```cpp
constexpr bool a = is_same<int, int32_t>::value;	// true
constexpr bool b = is_same<int, int64_t>::value;	// false
```

标准库在C++17对内嵌的静态常量`value`也定义了别名模板：

```cpp
template< class T, class U >
inline constexpr bool is_same_v = is_same<T, U>::value;
```

其命名方式类似对内置`type`的处理，它是以模板名加`_v`的方式来命名。

> 由于是常量别名模板，为了防止重定义问题要加inline，这是C++17的特性。

于是，前面的例子可以写成：

```cpp
constexpr bool a = is_same_v<int, int32_t>;	// true
```

#### `is_integral`

该trait用于判断模板参数类型是否为整型：

```cpp
template< class T >
struct is_integral;
```

这个东西实现起来其实也很简单，只需要我们枚举基础类型中所有的整型类型，对他们进行特化并继承`true_type`即可，而主模板直接继承`false_type`。原理上非常简单，但是实现起来颇具篇幅。

我们看一下libstdc++的实现：

```cpp
template<typename _Tp>
struct is_integral : public __is_integral_helper<__remove_cv_t<_Tp>>::type {};
```

`__remove_cv_t`用于移除模板参数的CV修饰符（常见的type modification，洗掉CV），由于标准没有把`_t`的别名模板标准化，所以用得是内置的命名法（估计是委员会忘了），实际上本体是`typename remove_cv<_Tp>::type`，这一部分后面会讲到。这里通过继承内部的`__is_integral_helper`又包了一层，因此`__is_integral_helper`才是本体，我们拆开看看：

```cpp
// 主模板果然如此
template<typename>
struct __is_integral_helper : public false_type {};

// 剩下的都是特化
template<>
struct __is_integral_helper<bool> : public true_type {};

template<>
struct __is_integral_helper<char> : public true_type {};

template<>
struct __is_integral_helper<signed char> : public true_type {};

template<>
struct __is_integral_helper<unsigned char> : public true_type {};

template<>
struct __is_integral_helper<short> : public true_type {};

template<>
struct __is_integral_helper<unsigned short> : public true_type {};

template<>
struct __is_integral_helper<int> : public true_type {};

template<>
struct __is_integral_helper<unsigned int> : public true_type {};

...
```

弄懂了`is_integral`，自然也就对`is_floating_point`的实现了然于胸，这里就不做展开了。

#### `is_array`

```cpp
template< class T >
struct is_array;
```

如何判断`T`是个raw array呢？这个东西看起来复杂，实际上判断条件非常简单：

Raw array作为模板参数，无非就两种：定长和不定长（C++的不定长不是真的不定长，只是一种语法糖）。因此，只需要对二者进行特化处理即可。

```cpp
template<class T>
struct is_array : std::false_type {};
 
template<class T>
struct is_array<T[]> : std::true_type {};
 
template<class T, std::size_t N>
struct is_array<T[N]> : std::true_type {};
```

> 注意，`std::is_array`判断的是raw array，传`std::array`进去取到的`value`依然是`false`。

#### `is_class`

```cpp
template< class T >
struct is_class;
```

如何判断`T`是否是一个类呢？这个看起来很复杂，实现起来也有些复杂。我们先看看[手册](https://en.cppreference.com/w/cpp/types/is_class)里给出的可能实现方案：

```cpp
namespace detail {
// 这里声明了两个函数模板，前者的参数为int型类成员变量指针
template <class T>
std::integral_constant<bool, !std::is_union<T>::value> test(int T::*);
 
// 而后者的参数是任意数量、任意类型，...的匹配在编译器眼里是优先级最低的，作为兜底的存在
template <class>
std::false_type test(...);
}
 
template <class T>
struct is_class : decltype(detail::test<T>(nullptr))
{};
```

`is_class`继承`detail::test`返回的类型作为基类（经典元函数调用方式），当我们的`T`是某种类类型时(`struct` or `class`)，基类会优先匹配前者，而如果`T`并不是类类型时，由于`int T::*`这一成员指针无法匹配故触发SFINAE，进而只能退而求其次，选择兜底的后者。如此，`is_class`借助元函数调用（继承）+分支（函数模板重载）就完成了功能判断。

对模板元不熟悉的小伙伴这里肯定会有这样几个疑问：

1. 这不对啊，如果我的`T`中没有定义`int`型成员变量，那不是也匹配不上吗？
2. 就算我定义了`int`型成员变量，元函数调用中对函数模板的调用为啥传了个`nullptr`啊？而且`test`只做了声明，没有定义体啊，这不会编译报错吗？
3. 为啥前者还要对返回类型做SFINAE控制，`std::is_union`是干啥的？

首先先解决一下问题3：因为`union`在`C/C++`中是个特殊的存在，如果单纯用类成员指针来判断，`union`其实也是满足条件的，但实际上我们都知道`union`它不是`class`或是`struct`，差别还是非常大的，因此我们要通过`is_union`这一trait把混入其中的`union`通过SFINAE开除，让他只能match后者。

至于问题1和问题2其实是一个问题，这里面有很多细节，首先大家要了解一个点：当编译器看到`decltype`关键字时，是不会对其内的表达式求值的，只需要在编译期确定这个表达式的返回类型而已。因此，`detail::test<T>(nullptr)`不会真正的发起调用（也就不需要实现体），只是会去从可用候选集中找到匹配度最高的那个（所以当`T`是类类型时，就会找到前者）。而另一方面，即使`T`不曾定义`int`型成员变量也无关紧要，因为只要`T`是类类型，那么`int T::*`在语法上就是一个合法的定义，至于你`T`中究竟有没有`int`类型的成员变量，编译器根本不care（在编译期，那是后面阶段的事儿了，但对这一case来说到这里就戛然而止了，没有之后了），所以，你也完全可以不用`int`，换其他任意合法的定义都行。

> 实际上C++11之前，没有强大的`decltype`，都是通过sizeof操作符配合晦涩的`enum`或`sruct`完成这一系列分支选择，C++11之前的模板元编程可以多看看Loki库的实现，以后会单独讲。

> 而在libstdc++实现中，实际上`is_class`的实现只是简单了做了元函数调用：
>
> ```cpp
> template<typename _Tp>
>     struct is_class
>     : public integral_constant<bool, __is_class(_Tp)>
>     { };
> ```
>
> `__is_class`实际上由编译器自己完成(built-in)，不需要实现。
>
> 其他诸如`__is_union`、`__is_enum`也都类似。

#### `is_function`

```cpp
template< class T >
struct is_function;
```

> Checks whether `T` is a function type. Types like [std::function](http://en.cppreference.com/w/cpp/utility/functional/function), lambdas, classes with overloaded `operator()` and pointers to functions don't count as function types. Provides the member constant `value` which is equal to true, if `T` is a function type. Otherwise, `value` is equal to false.

注意，如[手册](https://en.cppreference.com/w/cpp/types/is_function)所描述，这里的函数类型不包括泛化的诸如`std::function`,`lambda`,重载了函数调用操作符的`class`或是函数指针（前3个都可以叫函数对象、或是可调用对象，C++98前可以叫functor（侯捷译作仿函数））。想判断泛化函数对象的可以用另一个trait，叫`is_invocable`。

那么如何判断类型`T`为函数类型呢？实际上也不难，只是很麻烦，类似`is_integral`那样，通过特化来枚举一遍就行了，比如：

```cpp
// primary template
template<typename>
struct is_function : std::false_type {};

// specialization for regular functions with one argument
template<typename Ret, typename Arg>
struct is_function<Ret(Arg)> : std::true_type {};

// specialization for regular functions with two argument
template<typename Ret, typename Arg1, typename Arg2>
struct is_function<Ret(Arg1, Arg2)> : std::true_type {};

...
```

嗯，我们发现了个问题，不像`is_integral`是固定的单一参数，函数类型的参数可是无穷无尽的，没法一一枚举。不过好在C++11之后，模板语法有了核弹级加强——可变模板参数(variadic template parameters)，因此，上述的写法稍作修改：

```cpp
// primary template
template<typename>
struct is_function : std::false_type {};

// specialization for regular functions
template<typename Ret, typename... Args>
struct is_function<Ret(Args...)> : std::true_type {};
```

如此，通过可变模板参数就能让所有的regular function都精准匹配到这一特化模板了。

> 实际上，C++11之前，很多库诸如chromium base、Loki等对不定参数的处理都是通过上面的笨方法做了一堆定义，然后参数数量会有个上限比如50。

除了regular function，我们还得处理从C那继承来的legacy：variadic function。

> C里面的那些个printf, sprintf啥的，都是variadic function，这些个legacy在C++里恶心至极，每次折腾function都得考虑兼容它们。

```cpp
// specialization for variadic functions such as std::printf
template<class Ret, class... Args>
struct is_function<Ret(Args......)> : std::true_type {};
```

这个`......`就很魔性，前半个`...`是可变模板参数的语法，后半个`...`是C式变参。

对于函数类型来说，不仅要处理本体，因为类成员函数的关系，还得对CV限定符、ref限定符的版本进行特化处理（c++17之后甚至还要修正noexcept）以得以匹配：

```cpp
// specialization for function types that have cv-qualifiers
template<class Ret, class... Args>
struct is_function<Ret(Args...) const> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) volatile> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) const volatile> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) const> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) volatile> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) const volatile> : std::true_type {};

// specialization for function types that have ref-qualifiers
template<class Ret, class... Args>
struct is_function<Ret(Args...) &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) const &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) volatile &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) const volatile &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) const &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) volatile &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) const volatile &> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) const &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) volatile &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args...) const volatile &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) const &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) volatile &&> : std::true_type {};
template<class Ret, class... Args>
struct is_function<Ret(Args......) const volatile &&> : std::true_type {};
```

> 类成员函数的限定符有4种，分别是const、volatile、&和&&并且可以组合使用，所以特化就整了个排列组合。

当然，上述的实现体看起来就非常笨重，实际上有简化的写法，我们看看libstdc++的实现：

```cpp
  template<typename _Tp>
    struct is_function
    : public __bool_constant<!is_const<const _Tp>::value> { };

  template<typename _Tp>
    struct is_function<_Tp&>
    : public false_type { };

  template<typename _Tp>
    struct is_function<_Tp&&>
    : public false_type { };
```

为什么这里的实现这么简单呢，实际上虽然类成员函数可以有CV和ref限定符，但实际上这里对函数的限定符和平时我们对诸如`int`类型做限定：`const int`,`int&`,`int&&`的实际意义是完全不同的，前者实际上并非是对top做adding cv/ref-qualification，而是对类成员函数内部隐藏的对象做限制(比如`const`实际上作用的是隐含的`this`)。

那么话又说回来了，在C++语言中，只有function type和ref这两种物件不能拥有CV限定，因此，在主模板中，`is_const<const _Tp>::value`如果编译期计算的结果是`false`，就意味着top const限定对`_Tp`不起作用，也就意味着`_Tp`要么是引用类型，要么是函数类型，因此，我们只需要把引用类型踢出去就行了，如此才额外定义了两个特化模板，将左值引用和右值引用都通过优先级更高的特化模板筛了出去。

实际上，libc++的实现更为简单，关于ref类型的鉴别，完全可以复用另一个trait:

```cpp
template <class _Tp> 
  struct is_function
    : public _BoolConstant<!(is_reference<_Tp>::value || is_const<const _Tp>::value)> {};
```

看到这里，这个`_BoolConstant`你应该完全知道是个啥东西了，而`is_reference`这一trait要如何实现，你也已经了然于胸了。

#### `is_reference`

```cpp
template< class T >
struct is_reference;
```

嗯，其实怎么实现，在`is_function`的最后已经给出了。我们姑且看看libc++怎么实现的：

```cpp
template <class _Tp> struct is_reference        : public false_type {};
template <class _Tp> struct is_reference<_Tp&>  : public true_type {};
template <class _Tp> struct is_reference<_Tp&&> : public true_type {};
```

如出一辙。

> libstdc++的实现用了or trait，就不用它做例子了。

##### `is_lvalue_reference`,`is_rvalue_reference`

有时候还是需要区分左值引用和右值引用的，单单是`is_reference`粒度是不够的，那么怎么实现呢？这个其实就更显然了，只需要把`is_reference`阉割一下，只处理你关心的那个特化就行了：

```cpp
template <class _Tp> struct is_lvalue_reference       : public false_type {};
template <class _Tp> struct is_lvalue_reference<_Tp&> : public true_type {};

template <class _Tp> struct is_rvalue_reference        : public false_type {};
template <class _Tp> struct is_rvalue_reference<_Tp&&> : public true_type {};
```

#### `is_pointer`

```cpp
template< class T >
struct is_pointer;
```

> Checks whether `T` is a [pointer to object](https://en.cppreference.com/w/cpp/language/pointer) or a pointer to function (but not a pointer to member/member function) or a cv-qualified version thereof. Provides the member constant `value` which is equal to true, if `T` is a object/function pointer type. Otherwise, `value` is equal to false.

如[手册](https://en.cppreference.com/w/cpp/types/is_pointer)描述，这个trait判断的是对象或函数的指针，这不包括类成员函数/变量指针（实际上这个东西根本就不是指针，只是个偏移量）。

实现起来也很简单，也是按特化处理即可。

```cpp
template<typename>
    struct __is_pointer_helper
    : public false_type { };

  template<typename _Tp>
    struct __is_pointer_helper<_Tp*>
    : public true_type { };

  template<typename _Tp>
    struct is_pointer
    : public __is_pointer_helper<__remove_cv_t<_Tp>>::type
    { };
```

注意这里通过`__is_pointer_helper`做了元函数转发，转发前通过内部的trait`__remove_cv_t`对`_Tp`的CV进行了清洗，然后就是经典的特化模板匹配指针类型，主模板处理其他情景。

可能有的小伙伴会问，这里为啥要主动去消除掉CV呢？我们编写`is_reference`的时候可是没有做类似的事儿啊。指针这个东西在C++里是很复杂的，它和引用不同，引用本身是没有所谓的CV限定的，我们平时用的比如`const int&`常量左值引用，这个引用的作用实际上是对类型`int`生效。举个例子，比如在应用于`is_lvalue_reference`时，特化模板在匹配时模板参数`_Tp`被推导成了`const int`，而主模板则将`_Tp`匹配成`const int&`（由于优先级问题它不会被选择实例化）。

然而，对于指针来说，它本身是可以有CV修饰的，这个CV修饰限定的是指针本身，而不是指向的类型。我们以前学const时，经常会弄混这样几个东西：

```cpp
const int *p1;	// p1指向的类型是const int
int const *p2;	// p2指向的类型也是const int
int* const p3;	// p3指向的类型是int，本身是const
```

`p1`和`p2`实际上是同一种类型，指针指向的类型是`const int`，即指向的对象值本身不能修改，但是`p3`却不同，它指向的是个`int`对象，对象值本身可以修改，不能修改的是`p3`所指的内容，即一旦指定就不能再换。

当然，你完全还可以写`const int * const p4;`

在C++语言中，我们将修饰指针的`const`称作**top const**，修饰所指对象类型的称作**bottom const**。

因此，如果不提前洗掉CV，那么仅仅写一个`is_pointer<_Tp*>`的特化体，是不够的，它无法匹配`T *const`（再加上`volatile`的排列组合）。

#### `is_member_pointer`

C++里对有两个独特的类型，它们虽然名叫指针，但却和指针有本质的不同，那就是成员对象指针和成员函数指针。

那么这二者怎么判断呢？回顾一下`is_class` trait实现，实际上就是对类成员变量指针做了特化的判断，如此，只需要如法炮制即可：

```cpp
template< class T >
struct is_member_pointer_helper : std::false_type {};
 
template< class T, class U >
struct is_member_pointer_helper<T U::*> : std::true_type {};
 
template< class T >
struct is_member_pointer : 
    is_member_pointer_helper<typename std::remove_cv<T>::type> {};
```

特化模板中的`T`可以被推导成类成员变量类型或是类成员函数类型。libstdc++的实现与此完全一致。

> 这里需要`remove_cv`的原因与`is_pointer`相同。

##### `is_member_object_pointer`,`is_member_function_pointer`

那么如何区分到底是类成员变量指针还是类成员函数指针呢？`is_member_pointer`并无能力区分二者。

标准库进一步定义了二者：

```cpp
template< class T >
struct is_member_object_pointer;

template< class T >
struct is_member_function_pointer;
```

这两个东西怎么判断呢？其实也很简单，我们先考虑成员函数指针，在`is_member_pointer`的特化模板中，`T`会被推导成某种函数类型，那么我们只需要判断一下这个`T`是不是函数类型即可，而这一功能，我们已经有了`is_function`来实现：

```cpp
template< class T >
struct is_member_function_pointer_helper : std::false_type {};
 
template< class T, class U>
struct is_member_function_pointer_helper<T U::*> : std::is_function<T> {};
 
template< class T >
struct is_member_function_pointer 
  : is_member_function_pointer_helper< typename std::remove_cv<T>::type > {};
```

`is_member_function_pointer_helper`的特化模板中，利用`is_function`做了函数类型的type trait。

那么类成员变量指针怎么判断呢，变量的类型可太多了！实际上，你直接用not逻辑就行：

```cpp
template<class T>
struct is_member_object_pointer : std::integral_constant<
                                      bool,
                                      std::is_member_pointer<T>::value &&
                                      !std::is_member_function_pointer<T>::value
                                  > {};
```

> C++17封装了逻辑与或非的trait，用于简化像上述的元函数调用，libstdc++长成这样：
>
> ```cpp
>   template<typename>
>     struct __is_member_object_pointer_helper
>     : public false_type { };
> 
>   template<typename _Tp, typename _Cp>
>     struct __is_member_object_pointer_helper<_Tp _Cp::*>
>     : public __not_<is_function<_Tp>>::type { };
> 
>   /// is_member_object_pointer
>   template<typename _Tp>
>     struct is_member_object_pointer
>     : public __is_member_object_pointer_helper<__remove_cv_t<_Tp>>::type
>     { };
> ```

#### `is_void`

现在，C++语法中几乎所有的类型我们都有了trait了，但仔细想想，其实还有个最特别的`void`类型没处理：

```cpp
template< class T >
struct is_void;
```

思考一下，这个奇怪的类型要怎么判定呢？

其实也很简单，你只需要对`void`本身做特化就行了，因为`void`就是`void`，它不能是别的。

```cpp
template <typename T>
struct is_void : public false_type {};

template <>
struct is_void<void> : public true_type {};
```

当然了，我们可以通过元函数做简化，这也是libstdc++和libcxx的实现方法：

```cpp
template< class T >
struct is_void : std::is_same<void, typename std::remove_cv<T>::type> {};
```

这里还要注意到，元函数转发时还做了`remove_cv`，估计大多数同学都是懵逼的，都`void`类型了，咋还有CV限定呢？实际上C++语法上是支持对`void`做CV限定的，如果这里不做`remove_cv`、直接用我们上面给出的特化模板，那么实际上像是`is_void<const volatile void>`这种是无法匹配到特化模板的，这也不符合预期。

> 尽管`void`的CV限定没啥作用，但是语法上支持它主要是为了兼容，这样就可以一视同仁的做CV限定（比如应用到返回类型）。

#### `is_null_pointer`

C++14还引入了`is_null_pointer`来判断C++11中引入的一种特殊类型：`nullptr_t`。`std::nullptr_t`呢实际上是`typedef decltype(nullptr) nullptr_t;`，也就是一个特别的值：`nullptr`的类型定义（这玩意拯救了NULL与0傻傻分不清的窘境）。那么有了`nullptr_t`，`is_null_pointer`的判定也就很简单了：

```cpp
template< class T >
struct is_null_pointer : std::is_same<std::nullptr_t, std::remove_cv_t<T>> {};
```

---

到此，C++里各种类型的trait判断，我们都已了然。接下来，我们看看真正的properties trait。

#### `is_const`





