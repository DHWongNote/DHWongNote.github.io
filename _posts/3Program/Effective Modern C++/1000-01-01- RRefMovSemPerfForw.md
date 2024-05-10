---
title: 【Program】RRefMovSemPerfForw《Effective Modern C++》
author: DH Wang
date: 1000-01-01
category: Program
layout: post
---
 # 第5章 右值引用，移动语义，完美转发

**CHAPTER 5 RValue References, Move Semantics and Perfect Forwarding**

当你第一次了解到移动语义（*move semantics*）和完美转发（*perfect forwarding*）的时候，它们看起来非常直观：

- **移动语义**使编译器有可能用廉价的移动操作来代替昂贵的拷贝操作。正如拷贝构造函数和拷贝赋值操作符给了你控制拷贝语义的权力，移动构造函数和移动赋值操作符也给了你控制移动语义的权力。移动语义也允许创建只可移动（*move-only*）的类型，例如`std::unique_ptr`，`std::future`和`std::thread`。

- **完美转发**使接收任意数量实参的函数模板成为可能，它可以将实参转发到其他的函数，使目标函数接收到的实参与被传递给转发函数的实参保持一致。

**右值引用**是连接这两个截然不同的概念的胶合剂。它是使移动语义和完美转发变得可能的基础语言机制。

你对这些特点越熟悉，你就越会发现，你的初印象只不过是冰山一角。移动语义、完美转发和右值引用的世界比它所呈现的更加微妙。举个例子，`std::move`并不移动任何东西，完美转发也并不完美。移动操作并不永远比复制操作更廉价；即便如此，它也并不总是像你期望的那么廉价。而且，它也并不总是被调用，即使在当移动操作可用的时候。构造“`type&&`”也并非总是代表一个右值引用。

无论你挖掘这些特性有多深，它们看起来总是还有更多隐藏起来的部分。幸运的是，它们的深度总是有限的。本章将会带你到最基础的部分。一旦到达，C++11的这部分特性将会具有非常大的意义。比如，你会掌握`std::move`和`std::forward`的惯用法。你能够适应“`type&&`”的歧义性质。你会理解移动操作的令人惊奇的不同表现的背后真相。这些片段都会豁然开朗。在这一点上，你会重新回到一开始的状态，因为移动语义、完美转发和右值引用都会又一次显得直截了当。但是这一次，它们不再使人困惑。

在本章的这些小节中，非常重要的一点是要牢记形参永远是**左值**，即使它的类型是一个右值引用。比如，假设
```c++
void f(Widget&& w);
```
形参`w`是一个左值，即使它的类型是一个rvalue-reference-to-`Widget`。（如果这里震惊到你了，请重新回顾从本书[简介](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/Introduction.md)开始的关于左值和右值的总览。）

## 二十三：理解`std::move`和`std::forward`

**Item 23: Understand `std::move` and `std::forward`**

为了了解`std::move`和`std::forward`，一种有用的方式是从**它们不做什么**这个角度来了解它们。`std::move`不移动（move）任何东西，`std::forward`也不转发（forward）任何东西。在运行时，它们不做任何事情。它们不产生任何可执行代码，一字节也没有。

`std::move`和`std::forward`仅仅是执行转换（cast）的函数（事实上是函数模板）。`std::move`无条件的将它的实参转换为右值，而`std::forward`只在特定情况满足时下进行转换。它们就是如此。这样的解释带来了一些新的问题，但是从根本上而言，这就是全部内容。

为了使这个故事更加的具体，这里是一个C++11的`std::move`的示例实现。它并不完全满足标准细则，但是它已经非常接近了。

```cpp
template<typename T>                            //在std命名空间
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType =                          //别名声明，见9
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}
```

我为你们高亮了这段代码的两部分（译者注：高亮的部分为函数名`move`和`static_cast<ReturnType>(param)`）。一个是函数名字，因为函数的返回值非常具有干扰性，而且我不想你们被它搞得晕头转向。另外一个高亮的部分是包含这段函数的本质的转换。正如你所见，`std::move`接受一个对象的引用（准确的说，一个通用引用（universal reference），见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md))，返回一个指向同对象的引用。

该函数返回类型的`&&`部分表明`std::move`函数返回的是一个右值引用，但是，正如[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)所解释的那样，如果类型`T`恰好是一个左值引用，那么`T&&`将会成为一个左值引用。为了避免如此，*type trait*（见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)）`std::remove_reference`应用到了类型`T`上，因此确保了`&&`被正确的应用到了一个不是引用的类型上。这保证了`std::move`返回的真的是右值引用，这很重要，因为函数返回的右值引用是右值。因此，`std::move`将它的实参转换为一个右值，这就是它的全部作用。

此外，`std::move`在C++14中可以被更简单地实现。多亏了函数返回值类型推导（见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）和标准库的模板别名`std::remove_reference_t`（见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)），`std::move`可以这样写：

```cpp
template<typename T>
decltype(auto) move(T&& param)          //C++14，仍然在std命名空间
{
    using ReturnType = remove_referece_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

看起来更简单，不是吗？

因为`std::move`除了转换它的实参到右值以外什么也不做，有一些提议说它的名字叫`rvalue_cast`之类可能会更好。虽然可能确实是这样，但是它的名字已经是`std::move`，所以记住`std::move`做什么和不做什么很重要。它只进行转换，不移动任何东西。

当然，右值本来就是移动操作的候选者，所以对一个对象使用`std::move`就是告诉编译器，这个对象很适合被移动。所以这就是为什么`std::move`叫现在的名字：更容易指定可以被移动的对象。

事实上，右值只不过**经常**是移动操作的候选者。假设你有一个类，它用来表示一段注解。这个类的构造函数接受一个包含有注解的`std::string`作为形参，然后它复制该形参到数据成员。假设你了解[Item41](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/8.Tweaks/item41.md)，你声明一个值传递的形参：

```cpp
class Annotation {
public:
    explicit Annotation(std::string text);  //将会被复制的形参，
    …                                       //如同41所说，
};                                          //值传递
```

但是`Annotation`类的构造函数仅仅是需要读取`text`的值，它并不需要修改它。为了和历史悠久的传统：能使用`const`就使用`const`保持一致，你修订了你的声明以使`text`变成`const`：

```cpp
class Annotation {
public:
    explicit Annotation(const std::string text);
    …
};
```

当复制`text`到一个数据成员的时候，为了避免一次复制操作的代价，你仍然记得来自[Item41](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/8.Tweaks/item41.md)的建议，把`std::move`应用到`text`上，因此产生一个右值：

```cpp
class Annotation {
public:
    explicit Annotation(const std::string text)
    ：value(std::move(text))    //“移动”text到value里；这段代码执行起来
    { … }                       //并不是看起来那样
    
    …

private:
    std::string value;
};
```

这段代码可以编译，可以链接，可以运行。这段代码将数据成员`value`设置为`text`的值。这段代码与你期望中的完美实现的唯一区别，是`text`并不是被移动到`value`，而是被**拷贝**。诚然，`text`通过`std::move`被转换到右值，但是`text`被声明为`const std::string`，所以在转换之前，`text`是一个左值的`const std::string`，而转换的结果是一个右值的`const std::string`，但是纵观全程，`const`属性一直保留。

当编译器决定哪一个`std::string`的构造函数被调用时，考虑它的作用，将会有两种可能性：

```cpp
class string {                  //std::string事实上是
public:                         //std::basic_string<char>的类型别名
    …
    string(const string& rhs);  //拷贝构造函数
    string(string&& rhs);       //移动构造函数
    …
};
```

在类`Annotation`的构造函数的成员初始化列表中，`std::move(text)`的结果是一个`const std::string`的右值。这个右值不能被传递给`std::string`的移动构造函数，因为移动构造函数只接受一个指向**non-`const`**的`std::string`的右值引用。然而，该右值却可以被传递给`std::string`的拷贝构造函数，因为lvalue-reference-to-`const`允许被绑定到一个`const`右值上。因此，`std::string`在成员初始化的过程中调用了**拷贝**构造函数，即使`text`已经被转换成了右值。这样是为了确保维持`const`属性的正确性。从一个对象中移动出某个值通常代表着修改该对象，所以语言不允许`const`对象被传递给可以修改他们的函数（例如移动构造函数）。

从这个例子中，可以总结出两点。第一，不要在你希望能移动对象的时候，声明他们为`const`。对`const`对象的移动请求会悄无声息的被转化为拷贝操作。第二点，`std::move`不仅不移动任何东西，而且它也不保证它执行转换的对象可以被移动。关于`std::move`，你能确保的唯一一件事就是将它应用到一个对象上，你能够得到一个右值。

关于`std::forward`的故事与`std::move`是相似的，但是与`std::move`总是**无条件**的将它的实参为右值不同，`std::forward`只有在满足一定条件的情况下才执行转换。`std::forward`是**有条件**的转换。要明白什么时候它执行转换，什么时候不，想想`std::forward`的典型用法。最常见的情景是一个模板函数，接收一个通用引用形参，并将它传递给另外的函数：

```cpp
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发param到process的模板
void logAndProcess(T&& param)
{
    auto now =                              //获取现在时间
        std::chrono::system_clock::now();
    
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}
```

考虑两次对`logAndProcess`的调用，一次左值为实参，一次右值为实参：

```cpp
Widget w;

logAndProcess(w);               //用左值调用
logAndProcess(std::move(w));    //用右值调用
```

在`logAndProcess`函数的内部，形参`param`被传递给函数`process`。函数`process`分别对左值和右值做了重载。当使用左值来调用`logAndProcess`时，自然期望该左值被当作左值转发给`process`函数，而当使用右值来调用`logAndProcess`函数时，期望`process`函数的右值重载版本被调用。

但是`param`，正如所有的其他函数形参一样，是一个左值。每次在函数`logAndProcess`内部对函数`process`的调用，都会因此调用函数`process`的左值重载版本。为防如此，需要一种机制：当且仅当传递给函数`logAndProcess`的用以初始化`param`的实参是一个右值时，`param`会被转换为一个右值。这就是`std::forward`做的事情。这就是为什么`std::forward`是一个**有条件**的转换：它的实参用右值初始化时，转换为一个右值。

你也许会想知道`std::forward`是怎么知道它的实参是否是被一个右值初始化的。举个例子，在上述代码中，`std::forward`是怎么分辨`param`是被一个左值还是右值初始化的？ 简短的说，该信息藏在函数`logAndProcess`的模板参数`T`中。该参数被传递给了函数`std::forward`，它解开了含在其中的信息。该机制工作的细节可以查询[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)。

考虑到`std::move`和`std::forward`都可以归结于转换，它们唯一的区别就是`std::move`总是执行转换，而`std::forward`偶尔为之。你可能会问是否可以免于使用`std::move`而在任何地方只使用`std::forward`。 从纯技术的角度，答案是yes：`std::forward`是可以完全胜任，`std::move`并非必须。当然，其实两者中没有哪一个函数是**真的必须**的，因为可以到处直接写转换代码，但是我希望能同意：这将相当的，嗯，让人恶心。

`std::move`的吸引力在于它的便利性：减少了出错的可能性，增加了代码的清晰程度。考虑一个类，希望统计有多少次移动构造函数被调用了。只需要一个`static`的计数器，它会在移动构造的时候自增。假设在这个类中，唯一一个非静态的数据成员是`std::string`，一种经典的移动构造函数（即，使用`std::move`）可以被实现如下：

```cpp
class Widget {
public:
    Widget(Widget&& rhs)
    : s(std::move(rhs.s))
    { ++moveCtorCalls; }

    …

private:
    static std::size_t moveCtorCalls;
    std::string s;
};
```

如果要用`std::forward`来达成同样的效果，代码可能会看起来像：

```cpp
class Widget{
public:
    Widget(Widget&& rhs)                    //不自然，不合理的实现
    : s(std::forward<std::string>(rhs.s))
    { ++moveCtorCalls; }

    …

}
```

注意，第一，`std::move`只需要一个函数实参（`rhs.s`），而`std::forward`不但需要一个函数实参（`rhs.s`），还需要一个模板类型实参`std::string`。其次，传递给`std::forward`的类型应当是一个non-reference，因为惯例是传递的实参应该是一个右值（见[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)）。同样，这意味着`std::move`比起`std::forward`来说需要打更少的字，并且免去了传递一个表示正在传递一个右值的类型实参。同样，它根绝了传递错误类型的可能性（例如，`std::string&`可能导致数据成员`s`被复制而不是被移动构造）。

更重要的是，`std::move`的使用代表着无条件向右值的转换，而使用`std::forward`只对绑定了右值的引用进行到右值转换。这是两种完全不同的动作。前者是典型地为了移动操作，而后者只是传递（亦为转发）一个对象到另外一个函数，保留它原有的左值属性或右值属性。因为这些动作实在是差异太大，所以拥有两个不同的函数（以及函数名）来区分这些动作。

**请记住：**

+ `std::move`执行到右值的无条件的转换，但就自身而言，它不移动任何东西。
+ `std::forward`只有当它的参数被绑定到一个右值时，才将参数转换为右值。
+ `std::move`和`std::forward`在运行期什么也不做。
## 二十四：区分通用引用与右值引用

**Item 24: Distinguish universal references from rvalue references**

据说，真相使人自由，然而在特定的环境下，一个精心挑选的谎言也同样使人解放。这一就是这样一个谎言。因为在和软件打交道，然而，让避开“谎言（lie）”这个词，不妨说，本包含了一种“抽象（abstraction）”。

为了声明一个指向某个类型`T`的右值引用，你写下了`T&&`。由此，一个合理的假设是，当你看到一个“`T&&`”出现在源码中，你看到的是一个右值引用。唉，事情并不如此简单:

```cpp
void f(Widget&& param);             //右值引用
Widget&& var1 = Widget();           //右值引用
auto&& var2 = var1;                 //不是右值引用

template<typename T>
void f(std::vector<T>&& param);     //右值引用

template<typename T>
void f(T&& param);                  //不是右值引用
```

事实上，“`T&&`”有两种不同的意思。第一种，当然是右值引用。这种引用表现得正如你所期待的那样：它们只绑定到右值上，并且它们主要的存在原因就是为了识别可以移动操作的对象。

“`T&&`”的另一种意思是，它既可以是右值引用，也可以是左值引用。这种引用在源码里看起来像右值引用（即“`T&&`”），但是它们可以表现得像是左值引用（即“`T&`”）。它们的二重性使它们既可以绑定到右值上（就像右值引用），也可以绑定到左值上（就像左值引用）。 此外，它们还可以绑定到`const`或者non-`const`的对象上，也可以绑定到`volatile`或者non-`volatile`的对象上，甚至可以绑定到既`const`又`volatile`的对象上。它们可以绑定到几乎任何东西。这种空前灵活的引用值得拥有自己的名字。我把它叫做**通用引用**（*universal references*）。（[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)解释了`std::forward`几乎总是可以应用到通用引用上，并且在这本书即将出版之际，一些C++社区的成员已经开始将这种通用引用称之为**转发引用**（*forwarding references*））。

在两种情况下会出现通用引用。最常见的一种是函数模板形参，正如在之前的示例代码中所出现的例子：

```cpp
template<typename T>
void f(T&& param);                  //param是一个通用引用
```

第二种情况是`auto`声明符，它是从以上示例中拿出的：

```cpp
auto&& var2 = var1;                 //var2是一个通用引用
```

这两种情况的共同之处就是都存在**类型推导**（*type deduction*）。在模板`f`的内部，`param`的类型需要被推导，而在变量`var2`的声明中，`var2`的类型也需要被推导。同以下的例子相比较（同样来自于上面的示例代码），下面的例子不带有类型推导。如果你看见“`T&&`”不带有类型推导，那么你看到的就是一个右值引用：

```cpp
void f(Widget&& param);         //没有类型推导，
                                //param是一个右值引用
Widget&& var1 = Widget();       //没有类型推导，
                                //var1是一个右值引用
```

因为通用引用是引用，所以它们必须被初始化。一个通用引用的初始值决定了它是代表了右值引用还是左值引用。如果初始值是一个右值，那么通用引用就会是对应的右值引用，如果初始值是一个左值，那么通用引用就会是一个左值引用。对那些是函数形参的通用引用来说，初始值在调用函数的时候被提供：

```cpp
template<typename T>
void f(T&& param);              //param是一个通用引用

Widget w;
f(w);                           //传递给函数f一个左值；param的类型
                                //将会是Widget&，也即左值引用

f(std::move(w));                //传递给f一个右值；param的类型会是
                                //Widget&&，即右值引用
```

对一个通用引用而言，类型推导是必要的，但是它还不够。引用声明的**形式**必须正确，并且该形式是被限制的。它必须恰好为“`T&&`”。再看看之前已经看过的代码示例：

```cpp
template <typename T>
void f(std::vector<T>&& param);     //param是一个右值引用
```

当函数`f`被调用的时候，类型`T`会被推导（除非调用者显式地指定它，这种边缘情况不考虑）。但是`param`的类型声明并不是`T&&`，而是一个`std::vector<T>&&`。这排除了`param`是一个通用引用的可能性。`param`因此是一个右值引用——当你向函数`f`传递一个左值时，你的编译器将会乐于帮你确认这一点:

```cpp
std::vector<int> v;
f(v);                           //错误！不能将左值绑定到右值引用
```

即使一个简单的`const`修饰符的出现，也足以使一个引用失去成为通用引用的资格:

```cpp
template <typename T>
void f(const T&& param);        //param是一个右值引用
```
如果你在一个模板里面看见了一个函数形参类型为“`T&&`”，你也许觉得你可以假定它是一个通用引用。错！这是由于在模板内部并不保证一定会发生类型推导。考虑如下`push_back`成员函数，来自`std::vector`：

```cpp
template<class T, class Allocator = allocator<T>>   //来自C++标准
class vector
{
public:
    void push_back(T&& x);
    …
}
```

`push_back`函数的形参当然有一个通用引用的正确形式，然而，在这里并没有发生类型推导。因为`push_back`在有一个特定的`vector`实例之前不可能存在，而实例化`vector`时的类型已经决定了`push_back`的声明。也就是说，

```cpp
std::vector<Widget> v;
```

将会导致`std::vector`模板被实例化为以下代码：

```cpp
class vector<Widget, allocator<Widget>> {
public:
    void push_back(Widget&& x);             //右值引用
    …
};
```

现在你可以清楚地看到，函数`push_back`不包含任何类型推导。`push_back`对于`vector<T>`而言（有两个函数——它被重载了）总是声明了一个类型为rvalue-reference-to-`T`的形参。

作为对比，`std::vector`内的概念上相似的成员函数`emplace_back`，却确实包含类型推导:

```cpp
template<class T, class Allocator = allocator<T>>   //依旧来自C++标准
class vector {
public:
    template <class... Args>
    void emplace_back(Args&&... args);
    …
};
```

这儿，类型参数（*type parameter*）`Args`是独立于`vector`的类型参数`T`的，所以`Args`会在每次`emplace_back`被调用的时候被推导。（好吧，`Args`实际上是一个[*parameter pack*](https://en.cppreference.com/w/cpp/language/parameter_pack)，而不是一个类型参数，但是为了方便讨论，可以把它当作是一个类型参数。）

虽然函数`emplace_back`的类型参数被命名为`Args`，但是它仍然是一个通用引用，这补充了我之前所说的，通用引用的格式必须是“`T&&`”。你使用的名字`T`并不是必要的。举个例子，如下模板接受一个通用引用，因为形式（“`type&&`”）是正确的，并且`param`的类型将会被推导（重复一次，不考虑边缘情况，即当调用者明确给定类型的时候）。

```cpp
template<typename MyTemplateType>           //param是通用引用
void someFunc(MyTemplateType&& param);
```

我之前提到，类型为`auto`的变量可以是通用引用。更准确地说，类型声明为`auto&&`的变量是通用引用，因为会发生类型推导，并且它们具有正确形式(`T&&`)。`auto`类型的通用引用不如函数模板形参中的通用引用常见，但是它们在C++11中常常突然出现。而它们在C++14中出现得更多，因为C++14的*lambda*表达式可以声明`auto&&`类型的形参。举个例子，如果你想写一个C++14标准的*lambda*表达式，来记录任意函数调用的时间开销，你可以这样写：

```cpp
auto timeFuncInvocation =
    [](auto&& func, auto&&... params)           //C++14
    {
        start timer;
        std::forward<decltype(func)>(func)(     //对params调用func
            std::forward<delctype(params)>(params)...
        );
        stop timer and record elapsed time;
    };
```

如果你对*lambda*里的代码“`std::forward<decltype(blah blah blah)>`”反应是“这是什么鬼...?!”，只能说你可能还没有读[Item33](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item33.md)。别担心。在本，重要的事是*lambda*表达式中声明的`auto&&`类型的形参。`func`是一个通用引用，可以被绑定到任何可调用对象，无论左值还是右值。`args`是0个或者多个通用引用（即它是个通用引用*parameter pack*），它可以绑定到任意数目、任意类型的对象上。多亏了`auto`类型的通用引用，函数`timeFuncInvocation`可以对**近乎任意**（pretty much any）函数进行计时。(如果你想知道任意（any）和近乎任意（pretty much any）的区别，往后翻到[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md))。

牢记整个本——通用引用的基础——是一个谎言，噢不，是一个“抽象”。其底层真相被称为**引用折叠**（*reference collapsing*），[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)的专题将致力于讨论它。但是这个真相并不降低该抽象的有用程度。区分右值引用和通用引用将会帮助你更准确地阅读代码（“究竟我眼前的这个`T&&`是只绑定到右值还是可以绑定任意对象呢？”），并且，当你在和你的合作者交流时，它会帮助你避免歧义（“在这里我在用一个通用引用，而非右值引用”）。它也可以帮助你弄懂[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)和[26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)，它们依赖于右值引用和通用引用的区别。所以，拥抱这份抽象，陶醉于它吧。就像牛顿的力学定律（本质上不正确），比起爱因斯坦的广义相对论（这是真相）而言，往往更简单，更易用。所以通用引用的概念，相较于穷究引用折叠的细节而言，是更合意之选。

**请记住：**

- 如果一个函数模板形参的类型为`T&&`，并且`T`需要被推导得知，或者如果一个对象被声明为`auto&&`，这个形参或者对象就是一个通用引用。
- 如果类型声明的形式不是标准的`type&&`，或者如果类型推导没有发生，那么`type&&`代表一个右值引用。
- 通用引用，如果它被右值初始化，就会对应地成为右值引用；如果它被左值初始化，就会成为左值引用。
## 二十五：对右值引用使用`std::move`，对通用引用使用`std::forward`

**Item 25: Use `std::move` on rvalue references, `std::forward` on universal references**

右值引用仅绑定可以移动的对象。如果你有一个右值引用形参就知道这个对象可能会被移动：

```cpp
class Widget {
    Widget(Widget&& rhs);       //rhs定义上引用一个有资格移动的对象
    …
};
```

这是个例子，你将希望传递这样的对象给其他函数，允许那些函数利用对象的右值性（rvalueness）。这样做的方法是将绑定到此类对象的形参转换为右值。如[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)中所述，这不仅是`std::move`所做，而且它的创建就是为了这个目的：

```cpp
class Widget {
public:
    Widget(Widget&& rhs)        //rhs是右值引用
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
      { … }
    …
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```

另一方面（查看[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），通用引用**可能**绑定到有资格移动的对象上。通用引用使用右值初始化时，才将其强制转换为右值。[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)阐释了这正是`std::forward`所做的：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)           //newName是通用引用
    { name = std::forward<T>(newName); }

    …
};
```

总而言之，当把右值引用转发给其他函数时，右值引用应该被**无条件转换**为右值（通过`std::move`），因为它们**总是**绑定到右值；当转发通用引用时，通用引用应该**有条件地转换**为右值（通过`std::forward`），因为它们只是**有时**绑定到右值。

[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)解释说，可以在右值引用上使用`std::forward`表现出适当的行为，但是代码较长，容易出错，所以应该避免在右值引用上使用`std::forward`。更糟的是在通用引用上使用`std::move`，这可能会意外改变左值（比如局部变量）：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)       //通用引用可以编译，
    { name = std::move(newName); }  //但是代码太太太差了！
    …

private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName();        //工厂函数

Widget w;

auto n = getWidgetName();           //n是局部变量

w.setName(n);                       //把n移动进w！

…                                   //现在n的值未知
```

上面的例子，局部变量`n`被传递给`w.setName`，调用方可能认为这是对`n`的只读操作——这一点倒是可以被原谅。但是因为`setName`内部使用`std::move`无条件将传递的引用形参转换为右值，`n`的值被移动进`w.name`，调用`setName`返回时`n`最终变为未定义的值。这种行为使得调用者蒙圈了——还有可能变得狂躁。

你可能争辩说`setName`不应该将其形参声明为通用引用。此类引用不能使用`const`（见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），但是`setName`肯定不应该修改其形参。你可能会指出，如果为`const`左值和为右值分别重载`setName`可以避免整个问题，比如这样：

```cpp
class Widget {
public:
    void setName(const std::string& newName)    //用const左值设置
    { name = newName; }
    
    void setName(std::string&& newName)         //用右值设置
    { name = std::move(newName); }
    
    …
};
```

这样的话，当然可以工作，但是有缺点。首先编写和维护的代码更多（两个函数而不是单个模板）；其次，效率下降。比如，考虑如下场景：

```cpp
w.setName("Adela Novak");
```

使用通用引用的版本的`setName`，字面字符串“`Adela Novak`”可以被传递给`setName`，再传给`w`内部`std::string`的赋值运算符。`w`的`name`的数据成员通过字面字符串直接赋值，没有临时`std::string`对象被创建。但是，`setName`重载版本，会有一个临时`std::string`对象被创建，`setName`形参绑定到这个对象，然后这个临时`std::string`移动到`w`的数据成员中。一次`setName`的调用会包括`std::string`构造函数调用（创建中间对象），`std::string`赋值运算符调用（移动`newName`到`w.name`），`std::string`析构函数调用（析构中间对象）。这比调用接受`const char*`指针的`std::string`赋值运算符开销昂贵许多。增加的开销根据实现不同而不同，这些开销是否值得担心也跟应用和库的不同而有所不同，但是事实上，将通用引用模板替换成对左值引用和右值引用的一对函数重载在某些情况下会导致运行时的开销。如果把例子泛化，`Widget`数据成员是任意类型（而不是知道是个`std::string`），性能差距可能会变得更大，因为不是所有类型的移动操作都像`std::string`开销较小（参看[Item29](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item29.md)）。

但是，关于对左值和右值的重载函数最重要的问题不是源代码的数量，也不是代码的运行时性能。而是设计的可扩展性差。`Widget::setName`有一个形参，因此需要两种重载实现，但是对于有更多形参的函数，每个都可能是左值或右值，重载函数的数量几何式增长：n个参数的话，就要实现2<sup>n</sup>种重载。这还不是最坏的。有的函数——实际上是函数模板——接受**无限制**个数的参数，每个参数都可以是左值或者右值。此类函数的典型代表是`std::make_shared`，还有对于C++14的`std::make_unique`（见[Item21](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item21.md)）。查看他们的的重载声明：

```cpp
template<class T, class... Args>                //来自C++11标准
shared_ptr<T> make_shared(Args&&... args);

template<class T, class... Args>                //来自C++14标准
unique_ptr<T> make_unique(Args&&... args);
```

对于这种函数，对于左值和右值分别重载就不能考虑了：通用引用是仅有的实现方案。对这种函数，我向你保证，肯定使用`std::forward`传递通用引用形参给其他函数。这也是你应该做的。

好吧，通常，最终。但是不一定最开始就是如此。在某些情况，你可能需要在一个函数中多次使用绑定到右值引用或者通用引用的对象，并且确保在完成其他操作前，这个对象不会被移动。这时，你只想在最后一次使用时，使用`std::move`（对右值引用）或者`std::forward`（对通用引用）。比如：

```cpp
template<typename T>
void setSignText(T&& text)                  //text是通用引用
{
  sign.setText(text);                       //使用text但是不改变它
  
  auto now = 
      std::chrono::system_clock::now();     //获取现在的时间
  
  signHistory.add(now, 
                  std::forward<T>(text));   //有条件的转换为右值
}
```

这里，想要确保`text`的值不会被`sign.setText`改变，因为想要在`signHistory.add`中继续使用。因此`std::forward`只在最后使用。

对于`std::move`，同样的思路（即最后一次用右值引用的时候再调用`std::move`），但是需要注意，在有些稀少的情况下，你需要调用`std::move_if_noexcept`代替`std::move`。要了解何时以及为什么，参考[Item14](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item14.md)。

如果你在**按值**返回的函数中，返回值绑定到右值引用或者通用引用上，需要对返回的引用使用`std::move`或者`std::forward`。要了解原因，考虑两个矩阵相加的`operator+`函数，左侧的矩阵为右值（可以被用来保存求值之后的和）：

```cpp
Matrix                              //按值返回
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs);	        //移动lhs到返回值中
}
```

通过在`return`语句中将`lhs`转换为右值（通过`std::move`），`lhs`可以移动到返回值的内存位置。如果省略了`std::move`调用，

```cpp
Matrix                              //同之前一样
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return lhs;                     //拷贝lhs到返回值中
}
```

`lhs`是个左值的事实，会强制编译器拷贝它到返回值的内存空间。假定`Matrix`支持移动操作，并且比拷贝操作效率更高，在`return`语句中使用`std::move`的代码效率更高。

如果`Matrix`不支持移动操作，将其转换为右值不会变差，因为右值可以直接被`Matrix`的拷贝构造函数拷贝（见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)）。如果`Matrix`随后支持了移动操作，`operator+`将在下一次编译时受益。就是这种情况，通过将`std::move`应用到按值返回的函数中要返回的右值引用上，不会损失什么（还可能获得收益）。

使用通用引用和`std::forward`的情况类似。考虑函数模板`reduceAndCopy`收到一个未规约（unreduced）对象`Fraction`，将其规约，并返回一个规约后的副本。如果原始对象是右值，可以将其移动到返回值中（避免拷贝开销），但是如果原始对象是左值，必须创建副本，因此如下代码：

```cpp
template<typename T>
Fraction                            //按值返回
reduceAndCopy(T&& frac)             //通用引用的形参
{
    frac.reduce();
    return std::forward<T>(frac);		//移动右值，或拷贝左值到返回值中
}
```

如果`std::forward`被忽略，`frac`就被无条件复制到`reduceAndCopy`的返回值内存空间。

有些开发者获取到上面的知识后，并尝试将其扩展到不适用的情况。“如果对要被拷贝到返回值的右值引用形参使用`std::move`，会把拷贝构造变为移动构造，”他们想，“我也可以对我要返回的局部对象应用同样的优化。”换句话说，他们认为有个按值返回局部对象的函数，像这样，

```cpp
Widget makeWidget()                 //makeWidget的“拷贝”版本
{
    Widget w;                       //局部对象
    …                               //配置w
    return w;                       //“拷贝”w到返回值中
}
```

他们想要“优化”代码，把“拷贝”变为移动：

```cpp
Widget makeWidget()                 //makeWidget的移动版本
{
    Widget w;
    …
    return std::move(w);            //移动w到返回值中（不要这样做！）
}
```

我的注释告诉你这种想法是有问题的，但是问题在哪？

这是错的，因为对于这种优化，标准化委员会远领先于开发者。早就为人认识到的是，`makeWidget`的“拷贝”版本可以避免复制局部变量`w`的需要，通过在分配给函数返回值的内存中构造`w`来实现。这就是所谓的**返回值优化**（*return value optimization*，RVO），这在C++标准中已经实现了。

对这种好事遣词表达是个讲究活，因为你想只在那些不影响软件外在行为的地方允许这样的**拷贝消除**（copy elision）。对标准中教条的（也可以说是有毒的）絮叨做些解释，这个特定的好事就是说，编译器可能会在按值返回的函数中消除对局部对象的拷贝（或者移动），如果满足（1）局部对象与函数返回值的类型相同；（2）局部对象就是要返回的东西。（适合的局部对象包括大多数局部变量（比如`makeWidget`里的`w`），还有作为`return`语句的一部分而创建的临时对象。函数形参不满足要求。一些人将RVO的应用区分为命名的和未命名的（即临时的）局部对象，限制了RVO术语应用到未命名对象上，并把对命名对象的应用称为**命名返回值优化**（*named return value optimization*，NRVO）。）把这些记在脑子里，再看看`makeWidget`的“拷贝”版本：

```cpp
Widget makeWidget()                 //makeWidget的“拷贝”版本
{
    Widget w;
    …
    return w;                       //“拷贝”w到返回值中
}
```

这里两个条件都满足，你一定要相信我，对于这些代码，每个合适的C++编译器都会应用RVO来避免拷贝`w`。那意味着`makeWidget`的“拷贝”版本实际上不拷贝任何东西。

移动版本的`makeWidget`行为与其名称一样（假设`Widget`有移动构造函数），将`w`的内容移动到`makeWidget`的返回值位置。但是为什么编译器不使用RVO消除这种移动，而是在分配给函数返回值的内存中再次构造`w`呢？答案很简单：它们不能。条件（2）中规定，仅当返回值为局部对象时，才进行RVO，但是`makeWidget`的移动版本不满足这条件，再次看一下返回语句：

```cpp
return std::move(w);
```

返回的已经不是局部对象`w`，而是**`w`的引用**——`std::move(w)`的结果。返回局部对象的引用不满足RVO的第二个条件，所以编译器必须移动`w`到函数返回值的位置。开发者试图对要返回的局部变量用`std::move`帮助编译器优化，反而限制了编译器的优化选项。

但是RVO就是个优化。编译器不被**要求**消除拷贝和移动操作，即使他们被允许这样做。或许你会疑惑，并担心编译器用拷贝操作惩罚你，因为它们确实可以这样。或者你可能有足够的了解，意识到有些情况很难让编译器实现RVO，比如当函数不同控制路径返回不同局部变量时。（编译器必须产生一些代码在分配的函数返回值的内存中构造适当的局部变量，但是编译器如何确定哪个变量是合适的呢？）如果这样，你可能会愿意以移动的代价来保证不会产生拷贝。那就是，极可能仍然认为应用`std::move`到一个要返回的局部对象上是合理的，只因为可以不再担心拷贝的代价。

那种情况下，应用`std::move`到一个局部对象上**仍然**是一个坏主意。C++标准关于RVO的部分表明，如果满足RVO的条件，但是编译器选择不执行拷贝消除，则返回的对象**必须被视为右值**。实际上，标准要求当RVO被允许时，或者实行拷贝消除，或者将`std::move`隐式应用于返回的局部对象。因此，在`makeWidget`的“拷贝”版本中，

```cpp
Widget makeWidget()                 //同之前一样
{
    Widget w;
    …
    return w;
}
```

编译器要不消除`w`的拷贝，要不把函数看成像下面写的一样：

```cpp
Widget makeWidget()
{
    Widget w;
    …
    return std::move(w);            //把w看成右值，因为不执行拷贝消除
}
```

这种情况与按值返回函数形参的情况很像。形参们没资格参与函数返回值的拷贝消除，但是如果作为返回值的话编译器会将其视作右值。结果就是，如果代码如下：

```cpp
Widget makeWidget(Widget w)         //传值形参，与函数返回的类型相同
{
    …
    return w;
}
```

编译器必须看成像下面这样写的代码：

```cpp
Widget makeWidget(Widget w)
{
    …
    return std::move(w);
}
```

这意味着，如果对从按值返回的函数返回来的局部对象使用`std::move`，你并不能帮助编译器（如果不能实行拷贝消除的话，他们必须把局部对象看做右值），而是阻碍其执行优化选项（通过阻止RVO）。在某些情况下，将`std::move`应用于局部变量可能是一件合理的事（即，你把一个变量传给函数，并且知道不会再用这个变量），但是满足RVO的`return`语句或者返回一个传值形参并不在此列。

**请记住：**

- 最后一次使用时，在右值引用上使用`std::move`，在通用引用上使用`std::forward`。
- 对按值返回的函数要返回的右值引用和通用引用，执行相同的操作。
- 如果局部对象可以被返回值优化消除，就绝不使用`std::move`或者`std::forward`。
## 二十六：避免在通用引用上重载

**Item 26: Avoid overloading on universal references**

假定你需要写一个函数，它使用名字作为形参，打印当前日期和时间到日志中，然后将名字加入到一个全局数据结构中。你可能写出来这样的代码：

```cpp
std::multiset<std::string> names;           //全局数据结构
void logAndAdd(const std::string& name)
{
    auto now =                              //获取当前时间
        std::chrono::system_clock::now();
    log(now, "logAndAdd");                  //志记信息
    names.emplace(name);                    //把name加到全局数据结构中；
}                                           //emplace的信息见42
```

这份代码没有问题，但是同样的也没有效率。考虑这三个调用：

```cpp
std::string petName("Darla");
logAndAdd(petName);                     //传递左值std::string
logAndAdd(std::string("Persephone"));	//传递右值std::string
logAndAdd("Patty Dog");                 //传递字符串字面值
```

在第一个调用中，`logAndAdd`的形参`name`绑定到变量`petName`。在`logAndAdd`中`name`最终传给`names.emplace`。因为`name`是左值，会拷贝到`names`中。没有方法避免拷贝，因为是左值（`petName`）传递给`logAndAdd`的。

在第二个调用中，形参`name`绑定到右值（显式从“`Persephone`”创建的临时`std::string`）。`name`本身是个左值，所以它被拷贝到`names`中，但是意识到，原则上，它的值可以被移动到`names`中。本次调用中，有个拷贝代价，但是应该能用移动勉强应付。

在第三个调用中，形参`name`也绑定一个右值，但是这次是通过“`Patty Dog`”隐式创建的临时`std::string`变量。就像第二个调用中，`name`被拷贝到`names`，但是这里，传递给`logAndAdd`的实参是一个字符串字面量。如果直接将字符串字面量传递给`emplace`，就不会创建`std::string`的临时变量，而是直接在`std::multiset`中通过字面量构建`std::string`。在第三个调用中，有个`std::string`拷贝开销，但是连移动开销都不想要，更别说拷贝的。

可以通过使用通用引用（参见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)）重写`logAndAdd`来使第二个和第三个调用效率提升，按照[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)的说法，`std::forward`转发这个引用到`emplace`。代码如下：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");           //跟之前一样
logAndAdd(petName);                     //跟之前一样，拷贝左值到multiset
logAndAdd(std::string("Persephone"));	//移动右值而不是拷贝它
logAndAdd("Patty Dog");                 //在multiset直接创建std::string
                                        //而不是拷贝一个临时std::string
```

非常好，效率优化了！

在故事的最后，可以骄傲的交付这个代码，但是我还没有告诉你客户不总是有直接访问`logAndAdd`要求的名字的权限。有些客户只有索引，`logAndAdd`拿着索引在表中查找相应的名字。为了支持这些客户，`logAndAdd`需要重载为：

```cpp
std::string nameFromIdx(int idx);   //返回idx对应的名字

void logAndAdd(int idx)             //新的重载
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
```

之后的两个调用按照预期工作：

```cpp
std::string petName("Darla");           //跟之前一样

logAndAdd(petName);                     //跟之前一样，
logAndAdd(std::string("Persephone")); 	//这些调用都去调用
logAndAdd("Patty Dog");                 //T&&重载版本

logAndAdd(22);                          //调用int重载版本
```

事实上，这只能基本按照预期工作，假定一个客户将`short`类型索引传递给`logAndAdd`：

```cpp
short nameIdx;
…                                       //给nameIdx一个值
logAndAdd(nameIdx);                     //错误！
```

最后一行的注释并不清楚明白，下面让我来说明发生了什么。

有两个重载的`logAndAdd`。使用通用引用的那个推导出`T`的类型是`short`，因此可以精确匹配。对于`int`类型参数的重载也可以在`short`类型提升后匹配成功。根据正常的重载解决规则，精确匹配优先于类型提升的匹配，所以被调用的是通用引用的重载。

在通用引用那个重载中，`name`形参绑定到要传入的`short`上，然后`name`被`std::forward`给`names`（一个`std::multiset<std::string>`）的`emplace`成员函数，然后又被转发给`std::string`构造函数。`std::string`没有接受`short`的构造函数，所以`logAndAdd`调用里的`multiset::emplace`调用里的`std::string`构造函数调用失败。（译者注：这句话比较绕，实际上就是调用链。）所有这一切的原因就是对于`short`类型通用引用重载优先于`int`类型的重载。

使用通用引用的函数在C++中是最贪婪的函数。它们几乎可以精确匹配任何类型的实参（极少不适用的实参在[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)中介绍）。这也是把重载和通用引用组合在一块是糟糕主意的原因：通用引用的实现会匹配比开发者预期要多得多的实参类型。

一个更容易掉入这种陷阱的例子是写一个完美转发构造函数。简单对`logAndAdd`例子进行改造就可以说明这个问题。不用写接受`std::string`或者用索引查找`std::string`的自由函数，只是想一个构造函数有着相同操作的`Person`类：

```cpp
class Person {
public:
    template<typename T>
    explicit Person(T&& n)              //完美转发的构造函数，初始化数据成员
    : name(std::forward<T>(n)) {}

    explicit Person(int idx)            //int的构造函数
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

就像在`logAndAdd`的例子中，传递一个不是`int`的整型变量（比如`std::size_t`，`short`，`long`等）会调用通用引用的构造函数而不是`int`的构造函数，这会导致编译错误。这里这个问题甚至更糟糕，因为`Person`中存在的重载比肉眼看到的更多。在[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)中说明，在适当的条件下，C++会生成拷贝和移动构造函数，即使类包含了模板化的构造函数，模板函数能实例化产生与拷贝和移动构造函数一样的签名，也在合适的条件范围内。如果拷贝和移动构造被生成，`Person`类看起来就像这样：

```cpp
class Person {
public:
    template<typename T>            //完美转发的构造函数
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx);       //int的构造函数

    Person(const Person& rhs);      //拷贝构造函数（编译器生成）
    Person(Person&& rhs);           //移动构造函数（编译器生成）
    …
};
```

只有你在花了很多时间在编译器领域时，下面的行为才变得直观（译者注：这里意思就是这种实现会导致不符合人类直觉的结果，下面就解释了这种现象的原因）：

```cpp
Person p("Nancy"); 
auto cloneOfP(p);                   //从p创建新Person；这通不过编译！
```

这里试图通过一个`Person`实例创建另一个`Person`，显然应该调用拷贝构造即可。（`p`是左值，可以把通过移动操作来完成“拷贝”的想法请出去了。）但是这份代码不是调用拷贝构造函数，而是调用完美转发构造函数。然后，完美转发的函数将尝试使用`Person`对象`p`初始化`Person`的`std::string`数据成员，编译器就会报错。

“为什么？”你可能会疑问，“为什么拷贝构造会被完美转发构造替代？显然想拷贝`Person`到另一个`Person`”。确实是这样想的，但是编译器严格遵循C++的规则，这里的相关规则就是控制对重载函数调用的解析规则。

编译器的理由如下：`cloneOfP`被non-`const`左值`p`初始化，这意味着模板化构造函数可被实例化为采用`Person`类型的non-`const`左值。实例化之后，`Person`类看起来是这样的：

```cpp
class Person {
public:
    explicit Person(Person& n)          //由完美转发模板初始化
    : name(std::forward<Person&>(n)) {}

    explicit Person(int idx);           //同之前一样

    Person(const Person& rhs);          //拷贝构造函数（编译器生成的）
    …
};
```

在这个语句中，

```cpp
auto cloneOfP(p);
```

其中`p`被传递给拷贝构造函数或者完美转发构造函数。调用拷贝构造函数要求在`p`前加上`const`的约束来满足函数形参的类型，而调用完美转发构造不需要加这些东西。从模板产生的重载函数是更好的匹配，所以编译器按照规则：调用最佳匹配的函数。“拷贝”non-`const`左值类型的`Person`交由完美转发构造函数处理，而不是拷贝构造函数。

如果将本例中的传递的对象改为`const`的，会得到完全不同的结果：

```cpp
const Person cp("Nancy");   //现在对象是const的
auto cloneOfP(cp);          //调用拷贝构造函数！
```

因为被拷贝的对象是`const`，是拷贝构造函数的精确匹配。虽然模板化的构造函数可以被实例化为有完全一样的函数签名，

```cpp
class Person {
public:
    explicit Person(const Person& n);   //从模板实例化而来
  
    Person(const Person& rhs);          //拷贝构造函数（编译器生成的）
    …
};
```

但是没啥影响，因为重载规则规定当模板实例化函数和非模板函数（或者称为“正常”函数）匹配优先级相当时，优先使用“正常”函数。拷贝构造函数（正常函数）因此胜过具有相同签名的模板实例化函数。

（如果你想知道为什么编译器在生成一个拷贝构造函数时还会模板实例化一个相同签名的函数，参考[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)。）

当继承纳入考虑范围时，完美转发的构造函数与编译器生成的拷贝、移动操作之间的交互会更加复杂。尤其是，派生类的拷贝和移动操作的传统实现会表现得非常奇怪。来看一下：

```cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }

    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
};
```

如同注释表示的，派生类的拷贝和移动构造函数没有调用基类的拷贝和移动构造函数，而是调用了基类的完美转发构造函数！为了理解原因，要知道派生类将`SpecialPerson`类型的实参传递给其基类，然后通过模板实例化和重载解析规则作用于基类`Person`。最终，代码无法编译，因为`std::string`没有接受一个`SpecialPerson`的构造函数。

我希望到目前为止，已经说服了你，如果可能的话，避免对通用引用形参的函数进行重载。但是，如果在通用引用上重载是糟糕的主意，那么如果需要可转发大多数实参类型的函数，但是对于某些实参类型又要特殊处理应该怎么办？存在多种办法。实际上，下一个，[Item27](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item27.md)专门来讨论这个问题，敬请阅读。

**请记住：**

- 对通用引用形参的函数进行重载，通用引用函数的调用机会几乎总会比你期望的多得多。
- 完美转发构造函数是糟糕的实现，因为对于non-`const`左值，它们比拷贝构造函数而更匹配，而且会劫持派生类对于基类的拷贝和移动构造函数的调用。
## 二十七：熟悉通用引用重载的替代方法

**Item 27: Familiarize yourself with alternatives to overloading on universal references**

[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中说明了对使用通用引用形参的函数，无论是独立函数还是成员函数（尤其是构造函数），进行重载都会导致一系列问题。但是也提供了一些示例，如果能够按照期望的方式运行，重载可能也是有用的。这个探讨了几种，通过避免在通用引用上重载的设计，或者通过限制通用引用可以匹配的参数类型，来实现所期望行为的方法。

讨论基于[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中的示例，如果你还没有阅读那个，请先阅读那个再继续。

### 放弃重载

在[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中的第一个例子中，`logAndAdd`是许多函数的代表，这些函数可以使用不同的名字来避免在通用引用上的重载的弊端。例如两个重载的`logAndAdd`函数，可以分别改名为`logAndAddName`和`logAndAddNameIdx`。但是，这种方式不能用在第二个例子，`Person`构造函数中，因为构造函数的名字被语言固定了（译者注：即构造函数名与类名相同）。此外谁愿意放弃重载呢？

### 传递const T&

一种替代方案是退回到C++98，然后将传递通用引用替换为传递lvalue-refrence-to-`const`。事实上，这是[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中首先考虑的方法。缺点是效率不高。现在知道了通用引用和重载的相互关系，所以放弃一些效率来确保行为正确简单可能也是一种不错的折中。

### 传值

通常在不增加复杂性的情况下提高性能的一种方法是，将按传引用形参替换为按值传递，这是违反直觉的。该设计遵循[Item41](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/8.Tweaks/item41.md)中给出的建议，即在你知道要拷贝时就按值传递，因此会参考那个来详细讨论如何设计与工作，效率如何。这里，在`Person`的例子中展示：

```cpp
class Person {
public:
    explicit Person(std::string n)  //代替T&&构造函数，
    : name(std::move(n)) {}         //std::move的使用见41
  
    explicit Person(int idx)        //同之前一样
    : name(nameFromIdx(idx)) {}
    …

private:
    std::string name;
};
```

因为没有`std::string`构造函数可以接受整型参数，所有`int`或者其他整型变量（比如`std::size_t`、`short`、`long`等）都会使用`int`类型重载的构造函数。相似的，所有`std::string`类似的实参（还有可以用来创建`std::string`的东西，比如字面量“`Ruth`”等）都会使用`std::string`类型的重载构造函数。没有意外情况。我想你可能会说有些人使用`0`或者`NULL`指代空指针会调用`int`重载的构造函数让他们很吃惊，但是这些人应该参考[Item8](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item8.md)反复阅读直到使用`0`或者`NULL`作为空指针让他们恶心。

### 使用*tag dispatch*

传递lvalue-reference-to-`const`以及按值传递都不支持完美转发。如果使用通用引用的动机是完美转发，就只能使用通用引用了，没有其他选择。但是又不想放弃重载。所以如果不放弃重载又不放弃通用引用，如何避免在通用引用上重载呢？

实际上并不难。通过查看所有重载的所有形参以及调用点的所有传入实参，然后选择最优匹配的函数——考虑所有形参/实参的组合。通用引用通常提供了最优匹配，但是如果通用引用是包含其他**非**通用引用的形参列表的一部分，则非通用引用形参的较差匹配会使有一个通用引用的重载版本不被运行。这就是*tag dispatch*方法的基础，下面的示例会使这段话更容易理解。

将标签分派应用于`logAndAdd`例子，下面是原来的代码，以免你再分心回去查看：

```cpp
std::multiset<std::string> names;       //全局数据结构

template<typename T>                    //志记信息，将name添加到数据结构
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clokc::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

就其本身而言，功能执行没有问题，但是如果引入一个`int`类型的重载来用索引查找对象，就会重新陷入[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中描述的麻烦。这个的目标是避免它。不通过重载，重新实现`logAndAdd`函数分拆为两个函数，一个针对整型值，一个针对其他。`logAndAdd`本身接受所有实参类型，包括整型和非整型。

这两个真正执行逻辑的函数命名为`logAndAddImpl`，即使用重载。其中一个函数接受通用引用。所以同时使用了重载和通用引用。但是每个函数接受第二个形参，表征传入的实参是否为整型。这第二个形参可以帮助避免陷入到[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中提到的麻烦中，因为将其安排为第二个实参决定选择哪个重载函数。

是的，我知道，“不要在啰嗦了，赶紧亮出代码”。没有问题，代码如下，这是最接近正确版本的：

```cpp
template<typename T>
void logAndAdd(T&& name) 
{
    logAndAddImpl(std::forward<T>(name),
                  std::is_integral<T>());   //不那么正确
}
```

这个函数转发它的形参给`logAndAddImpl`函数，但是多传递了一个表示形参`T`是否为整型的实参。至少，这就是应该做的。对于右值的整型实参来说，这也是正确的。但是如同[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)中说明，如果左值实参传递给通用引用`name`，对`T`类型推断会得到左值引用。所以如果左值`int`被传入`logAndAdd`，`T`将被推断为`int&`。这不是一个整型类型，因为引用不是整型类型。这意味着`std::is_integral<T>`对于任何左值实参返回false，即使确实传入了整型值。

意识到这个问题基本相当于解决了它，因为C++标准库有一个*type trait*（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)），`std::remove_reference`，函数名字就说明做了希望的：移除类型的引用说明符。所以正确实现的代码应该是这样：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(
        std::forward<T>(name),
        std::is_integral<typename std::remove_reference<T>::type>()
    );
}
```

这个代码很巧妙。（在C++14中，你可以通过`std::remove_reference_t<T>`来简化写法，参看[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)）

处理完之后，可以将注意力转移到名为`logAndAddImpl`的函数上了。有两个重载函数，第一个仅用于非整型类型（即`std::is_integral<typename std::remove_reference<T>::type>`是false）：

```cpp
template<typename T>                            //非整型实参：添加到全局数据结构中
void logAndAddImpl(T&& name, std::false_type)	//译者注：高亮std::false_type
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

一旦你理解了高亮参数的含义，代码就很直观。概念上，`logAndAdd`传递一个布尔值给`logAndAddImpl`表明是否传入了一个整型类型，但是`true`和`false`是**运行时**值，需要使用重载决议——**编译时**决策——来选择正确的`logAndAddImpl`重载。这意味着需要一个**类型**对应`true`，另一个不同的类型对应`false`。这个需要是经常出现的，所以标准库提供了这样两个命名`std::true_type`和`std::false_type`。`logAndAdd`传递给`logAndAddImpl`的实参是个对象，如果`T`是整型，对象的类型就继承自`std::true_type`，反之继承自`std::false_type`。最终的结果就是，当`T`不是整型类型时，这个`logAndAddImpl`重载是个可供调用的候选者。

第二个重载覆盖了相反的场景：当`T`是整型类型。在这个场景中，`logAndAddImpl`简单找到对应传入索引的名字，然后传递给`logAndAdd`：

```cpp
std::string nameFromIdx(int idx);           //与26一样，整型实参：查找名字并用它调用logAndAdd
void logAndAddImpl(int idx, std::true_type) //译者注：高亮std::true_type
{
  logAndAdd(nameFromIdx(idx)); 
}
```

通过索引找到对应的`name`，然后让`logAndAddImpl`传递给`logAndAdd`（名字会被再`std::forward`给另一个`logAndAddImpl`重载），避免了将日志代码放入这个`logAndAddImpl`重载中。

在这个设计中，类型`std::true_type`和`std::false_type`是“标签”（tag），其唯一目的就是强制重载解析按照的想法来执行。注意到甚至没有对这些参数进行命名。他们在运行时毫无用处，事实上希望编译器可以意识到这些标签形参没被使用，然后在程序执行时优化掉它们。（至少某些时候有些编译器会这样做。）通过创建标签对象，在`logAndAdd`内部将重载实现函数的调用“分发”（*dispatch*）给正确的重载。因此这个设计名称为：*tag dispatch*。这是模板元编程的标准构建模块，你对现代C++库中的代码了解越多，你就会越多遇到这种设计。

就的目的而言，*tag dispatch*的重要之处在于它可以允许组合重载和通用引用使用，而没有[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中提到的问题。分发函数——`logAndAdd`——接受一个没有约束的通用引用参数，但是这个函数没有重载。实现函数——`logAndAddImpl`——是重载的，一个接受通用引用参数，但是重载规则不仅依赖通用引用形参，还依赖新引入的标签形参，标签值设计来保证有不超过一个的重载是合适的匹配。结果是标签来决定采用哪个重载函数。通用引用参数可以生成精确匹配的事实在这里并不重要。（译者注：这里确实比较啰嗦，如果理解了上面的内容，这段完全可以没有。）

### 约束使用通用引用的模板

*tag dispatch*的关键是存在单独一个函数（没有重载）给客户端API。这个单独的函数分发给具体的实现函数。创建一个没有重载的分发函数通常是容易的，但是[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中所述第二个问题案例是`Person`类的完美转发构造函数，是个例外。编译器可能会自行生成拷贝和移动构造函数，所以即使你只写了一个构造函数并在其中使用*tag dispatch*，有一些对构造函数的调用也被编译器生成的函数处理，绕过了分发机制。

实际上，真正的问题不是编译器生成的函数会绕过*tag dispatch*设计，而是不**总**会绕过去。你希望类的拷贝构造函数总是处理该类型的左值拷贝请求，但是如同[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中所述，提供具有通用引用的构造函数，会使通用引用构造函数在拷贝non-`const`左值时被调用（而不是拷贝构造函数）。那个还说明了当一个基类声明了完美转发构造函数，派生类实现自己的拷贝和移动构造函数时会调用那个完美转发构造函数，尽管正确的行为是调用基类的拷贝或者移动构造。

这种情况，采用通用引用的重载函数通常比期望的更加贪心，虽然不像单个分派函数一样那么贪心，而又不满足使用*tag dispatch*的条件。你需要另外的技术，可以让你确定允许使用通用引用模板的条件。朋友，你需要的就是`std::enable_if`。

`std::enable_if`可以给你提供一种强制编译器执行行为的方法，像是特定模板不存在一样。这种模板被称为被**禁止**（disabled）。默认情况下，所有模板是**启用**的（enabled），但是使用`std::enable_if`可以使得仅在`std::enable_if`指定的条件满足时模板才启用。在这个例子中，只在传递的类型不是`Person`时使用`Person`的完美转发构造函数。如果传递的类型是`Person`，要禁止完美转发构造函数（即让编译器忽略它），因为这会让拷贝或者移动构造函数处理调用，这是想要使用`Person`初始化另一个`Person`的初衷。

这个主意听起来并不难，但是语法比较繁杂，尤其是之前没有接触过的话，让我慢慢引导你。有一些`std::enbale_if`的contidion（条件）部分的样板，让从这里开始。下面的代码是`Person`完美转发构造函数的声明，多展示`std::enable_if`的部分来简化使用难度。我仅展示构造函数的声明，因为`std::enable_if`的使用对函数实现没影响。实现部分跟[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中没有区别。

```cpp
class Person {
public:
    template<typename T,
             typename = typename std::enable_if<condition>::type>   //译者注：本行高亮，condition为某其他特定条件
    explicit Person(T&& n);
    …
};
```

为了理解高亮部分发生了什么，我很遗憾的表示你要自行参考其他代码，因为详细解释需要花费一定空间和时间，而本书并没有足够的空间（在你自行学习过程中，请研究“SFINAE”以及`std::enable_if`，因为“SFINAE”就是使`std::enable_if`起作用的技术）。这里我想要集中讨论条件的表示，该条件表示此构造函数是否启用。

这里想表示的条件是确认`T`不是`Person`类型，即模板构造函数应该在`T`不是`Person`类型的时候启用。多亏了*type trait*可以确定两个对象类型是否相同（`std::is_same`），看起来需要的就是`!std::is_same<Person, T>::value`（注意语句开始的`!`，想要的是**不**相同）。这很接近想要的了，但是不完全正确，因为如同[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)中所述，使用左值来初始化通用引用的话会推导成左值引用，比如这个代码:

```cpp
Person p("Nancy");
auto cloneOfP(p);       //用左值初始化
```

`T`的类型在通用引用的构造函数中被推导为`Person&`。`Person`和`Person&`类型是不同的，`std::is_same`的结果也反映了：`std::is_same<Person, Person&>::value`是false。

如果更精细考虑仅当`T`不是`Person`类型才启用模板构造函数，会意识到当查看`T`时，应该忽略：

- **是否是个引用**。对于决定是否通用引用构造函数启用的目的来说，`Person`，`Person&`，`Person&&`都是跟`Person`一样的。
- **是不是`const`或者`volatile`**。如上所述，`const Person`，`volatile Person` ，`const volatile Person`也是跟`Person`一样的。

这意味着需要一种方法消除对于`T`的引用，`const`，`volatile`修饰。再次，标准库提供了这样功能的*type trait*，就是`std::decay`。`std::decay<T>::value`与`T`是相同的，只不过会移除引用和cv限定符（*cv-qualifiers*，即`const`或`volatile`标识符）的修饰。（这里我没有说出另外的真相，`std::decay`如同其名一样，可以将数组或者函数退化成指针，参考[Item1](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item1.md)，但是在这里讨论的问题中，它刚好合适）。想要控制构造函数是否启用的条件可以写成：

```cpp
!std::is_same<Person, typename std::decay<T>::type>::value
```

即`Person`和`T`的类型不同，忽略了所有引用和cv限定符。（如[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)所述，`std::decay`前的“`typename`”是必需的，因为`std::decay<T>::type`的类型取决于模板形参`T`。）

将其带回上面`std::enable_if`样板的代码中，加上调整一下格式，让各部分如何组合在一起看起来更容易，`Person`的完美转发构造函数的声明如下：

```cpp
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_same<Person, 
                                     typename std::decay<T>::type
                                    >::value
                   >::type
    >
    explicit Person(T&& n);
    …
};
```

如果你之前从没有看到过这种类型的代码，那你可太幸福了。最后才放出这种设计是有原因的。当你有其他机制来避免同时使用重载和通用引用时（你总会这样做），确实应该那样做。不过，一旦你习惯了使用函数语法和尖括号的使用，也不坏。此外，这可以提供你一直想要的行为表现。在上面的声明中，使用`Person`初始化一个`Person`——无论是左值还是右值，`const`还是non-`const`，`volatile`还是non-`volatile`——都不会调用到通用引用构造函数。

成功了，对吗？确实！

啊，不对。等会再庆祝。[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)还有一个情景需要解决，需要继续探讨下去。

假定从`Person`派生的类以常规方式实现拷贝和移动操作：

```cpp
class SpecialPerson: public Person {
public:
    SpecialPerson(const SpecialPerson& rhs) //拷贝构造函数，调用基类的
    : Person(rhs)                           //完美转发构造函数！
    { … }
    
    SpecialPerson(SpecialPerson&& rhs)      //移动构造函数，调用基类的
    : Person(std::move(rhs))                //完美转发构造函数！
    { … }
    
    …
};
```

这和[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)中的代码是一样的，包括注释也是一样。当拷贝或者移动一个`SpecialPerson`对象时，希望调用基类对应的拷贝和移动构造函数，来拷贝或者移动基类部分，但是这里，将`SpecialPerson`传递给基类的构造函数，因为`SpecialPerson`和`Person`类型不同（在应用`std::decay`后也不同），所以完美转发构造函数是启用的，会实例化为精确匹配`SpecialPerson`实参的构造函数。相比于派生类到基类的转化——这个转化对于在`Person`拷贝和移动构造函数中把`SpecialPerson`对象绑定到`Person`形参非常重要，生成的精确匹配是更优的，所以这里的代码，拷贝或者移动`SpecialPerson`对象就会调用`Person`类的完美转发构造函数来执行基类的部分。跟[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)的困境一样。

派生类仅仅是按照常规的规则生成了自己的移动和拷贝构造函数，所以这个问题的解决还要落实在基类，尤其是控制是否使用`Person`通用引用构造函数启用的条件。现在意识到不只是禁止`Person`类型启用模板构造函数，而是禁止`Person`**以及任何派生自`Person`**的类型启用模板构造函数。讨厌的继承！

你应该不意外在这里看到标准库中也有*type trait*判断一个类型是否继承自另一个类型，就是`std::is_base_of`。如果`std::is_base_of<T1, T2>`是true就表示`T2`派生自`T1`。类型也可被认为是从他们自己派生，所以`std::is_base_of<T, T>::value`总是true。这就很方便了，想要修正控制`Person`完美转发构造函数的启用条件，只有当`T`在消除引用和cv限定符之后，并且既不是`Person`又不是`Person`的派生类时，才满足条件。所以使用`std::is_base_of`代替`std::is_same`就可以了：

```cpp
class Person {
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_base_of<Person, 
                                        typename std::decay<T>::type
                                       >::value
                   >::type
    >
    explicit Person(T&& n);
    …
};
```

现在终于完成了最终版本。这是C++11版本的代码，如果使用C++14，这份代码也可以工作，但是可以使用`std::enable_if`和`std::decay`的别名模板来少写“`typename`”和“`::type`”这样的麻烦东西，产生了下面这样看起来舒爽的代码：

```cpp
class Person  {                                         //C++14
public:
    template<
        typename T,
        typename = std::enable_if_t<                    //这儿更少的代码
                       !std::is_base_of<Person,
                                        std::decay_t<T> //还有这儿
                                       >::value
                   >                                    //还有这儿
    >
    explicit Person(T&& n);
    …
};
```

好了，我承认，我又撒谎了。还没有完成，但是越发接近最终版本了。非常接近，我保证。

已经知道如何使用`std::enable_if`来选择性禁止`Person`通用引用构造函数，来使得一些实参类型确保使用到拷贝或者移动构造函数，但是还没将其应用于区分整型参数和非整型参数。毕竟，的原始目标是解决构造函数模糊性问题。

需要的所有东西——我确实意思是所有——是（1）加入一个`Person`构造函数重载来处理整型参数；（2）约束模板构造函数使其对于某些实参禁用。使用这些讨论过的技术组合起来，就能解决这个问题了：

```cpp
class Person {
public:
    template<
        typename T,
        typename = std::enable_if_t<
            !std::is_base_of<Person, std::decay_t<T>>::value
            &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)          //对于std::strings和可转化为
    : name(std::forward<T>(n))      //std::strings的实参的构造函数
    { … }

    explicit Person(int idx)        //对于整型实参的构造函数
    : name(nameFromIdx(idx))
    { … }

    …                               //拷贝、移动构造函数等

private:
    std::string name;
};
```

看！多么优美！好吧，优美之处只是对于那些迷信模板元编程之人，但是确实提出了不仅能工作的方法，而且极具技巧。因为使用了完美转发，所以具有最大效率，因为控制了通用引用与重载的结合而不是禁止它，这种技术可以被用于不可避免要用重载的情况（比如构造函数）。

### 折中

本提到的前三个技术——放弃重载、传递const T&、传值——在函数调用中指定每个形参的类型。后两个技术——*tag dispatch*和限制模板适用范围——使用完美转发，因此不需要指定形参类型。这一基本决定（是否指定类型）有一定后果。

通常，完美转发更有效率，因为它避免了仅仅去为了符合形参声明的类型而创建临时对象。在`Person`构造函数的例子中，完美转发允许将“`Nancy`”这种字符串字面量转发到`Person`内部的`std::string`的构造函数，不使用完美转发的技术则会从字符串字面值创建一个临时`std::string`对象，来满足`Person`构造函数指定的形参要求。

但是完美转发也有缺点。即使某些类型的实参可以传递给接受特定类型的函数，也无法完美转发。[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)中探索了完美转发失败的例子。

第二个问题是当客户传递无效参数时错误消息的可理解性。例如假如客户传递了一个由`char16_t`（一种C++11引入的类型表示16位字符）而不是`char`（`std::string`包含的）组成的字符串字面值来创建一个`Person`对象：

```cpp
Person p(u"Konrad Zuse");   //“Konrad Zuse”由const char16_t类型字符组成
```

使用本中讨论的前三种方法，编译器将看到可用的采用`int`或者`std::string`的构造函数，它们或多或少会产生错误消息，表示没有可以从`const char16_t[12]`转换为`int`或者`std::string`的方法。

但是，基于完美转发的方法，`const char16_t`不受约束地绑定到构造函数的形参。从那里将转发到`Person`的`std::string`数据成员的构造函数，在这里，调用者传入的内容（`const char16_t`数组）与所需内容（`std::string`构造函数可接受的类型）发生的不匹配会被发现。由此产生的错误消息会让人更印象深刻，在我使用的编译器上，会产生超过160行错误信息。

在这个例子中，通用引用仅被转发一次（从`Person`构造函数到`std::string`构造函数），但是更复杂的系统中，在最终到达判断实参类型是否可接受的地方之前，通用引用会被多层函数调用转发。通用引用被转发的次数越多，产生的错误消息偏差就越大。许多开发者发现，这种特殊问题是发生在留有通用引用形参的接口上，这些接口以性能作为首要考虑点。

在`Person`这个例子中，知道完美转发函数的通用引用形参要作为`std::string`的初始化器，所以可以用`static_assert`来确认它可以起这个作用。`std::is_constructible`这个*type trait*执行编译时测试，确定一个类型的对象是否可以用另一个不同类型（或多个类型）的对象（或多个对象）来构造，所以代码可以这样：

```cpp
class Person {
public:
    template<                       //同之前一样
        typename T,
        typename = std::enable_if_t<
            !std::is_base_of<Person, std::decay_t<T>>::value
            &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)
    : name(std::forward<T>(n))
    {
        //断言可以用T对象创建std::string
        static_assert(
        std::is_constructible<std::string, T>::value,
        "Parameter n can't be used to construct a std::string"
        );

        …               //通常的构造函数的工作写在这

    }
    
    …                   //Person类的其他东西（同之前一样）
};
```

如果客户代码尝试使用无法构造`std::string`的类型创建`Person`，会导致指定的错误消息。不幸的是，在这个例子中，`static_assert`在构造函数体中，但是转发的代码作为成员初始化列表的部分在检查之前。所以我使用的编译器，结果是由`static_assert`产生的清晰的错误消息在常规错误消息（多达160行以上那个）后出现。

**请记住：**

- 通用引用和重载的组合替代方案包括使用不同的函数名，通过lvalue-reference-to-`const`传递形参，按值传递形参，使用*tag dispatch*。
- 通过`std::enable_if`约束模板，允许组合通用引用和重载使用，但它也控制了编译器在哪种条件下才使用通用引用重载。
- 通用引用参数通常具有高效率的优势，但是可用性就值得斟酌。
## 二十八：理解引用折叠

**Item 28: Understand reference collapsing**

[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)中指出，当实参传递给模板函数时，被推导的模板形参`T`根据实参是左值还是右值来编码。但是那并没有提到只有当实参被用来实例化通用引用形参时，上述推导才会发生，但是有充分的理由忽略这一点：因为通用引用是[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)中才提到。回过头来看，对通用引用和左值/右值编码的观察意味着对于这个模板，

```cpp
template<typename T>
void func(T&& param);
```

不管传给param的实参是左值还是右值，模板形参`T`都会编码。

编码机制是简单的。当左值实参被传入时，`T`被推导为左值引用。当右值被传入时，`T`被推导为非引用。（请注意不对称性：左值被编码为左值引用，右值被编码为**非引用**。）因此：

```cpp
Widget widgetFactory();     //返回右值的函数
Widget w;                   //一个变量（左值）
func(w);                    //用左值调用func；T被推导为Widget&
func(widgetFactory());      //用右值调用func；T被推导为Widget
```

上面的两种`func`调用中，`Widget`被传入，因为一个是左值，一个是右值，模板形参`T`被推导为不同的类型。正如很快看到的，这决定了通用引用成为左值还是右值，也是`std::forward`的工作基础。

在更加深入`std::forward`和通用引用之前，必须明确在C++中引用的引用是非法的。不知道你是否尝试过下面的写法，编译器会报错：

```cpp
int x;
…
auto& & rx = x;             //错误！不能声明引用的引用
```

考虑下，如果一个左值传给接受通用引用的模板函数会发生什么：

```cpp
template<typename T>
void func(T&& param);       //同之前一样

func(w);                    //用左值调用func；T被推导为Widget&
```

如果用`T`推导出来的类型（即`Widget&`）初始化模板，会得到：

```cpp
void func(Widget& && param);
```

引用的引用！但是编译器没有报错。从[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)中了解到因为通用引用`param`被传入一个左值，所以`param`的类型应该为左值引用，但是编译器如何把`T`推导的类型带入模板变成如下的结果，也就是最终的函数签名？

```cpp
void func(Widget& param);
```

答案是**引用折叠**（*reference collapsing*）。是的，禁止**你**声明引用的引用，但是**编译器**会在特定的上下文中产生这些，模板实例化就是其中一种情况。当编译器生成引用的引用时，引用折叠指导下一步发生什么。

存在两种类型的引用（左值和右值），所以有四种可能的引用组合（左值的左值，左值的右值，右值的右值，右值的左值）。如果一个上下文中允许引用的引用存在（比如，模板的实例化），引用根据规则**折叠**为单个引用：

> 如果任一引用为左值引用，则结果为左值引用。否则（即，如果引用都是右值引用），结果为右值引用。

在上面的例子中，将推导类型`Widget&`替换进模板`func`会产生对左值引用的右值引用，然后引用折叠规则告诉结果就是左值引用。

引用折叠是`std::forward`工作的一种关键机制。就像[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)中解释的一样，`std::forward`应用在通用引用参数上，所以经常能看到这样使用：

```cpp
template<typename T>
void f(T&& fParam)
{
    …                                   //做些工作
    someFunc(std::forward<T>(fParam));  //转发fParam到someFunc
}
```

因为`fParam`是通用引用，知道类型参数`T`的类型根据`f`被传入实参（即用来实例化`fParam`的表达式）是左值还是右值来编码。`std::forward`的作用是当且仅当传给`f`的实参为右值时，即`T`为非引用类型，才将`fParam`（左值）转化为一个右值。

`std::forward`可以这样实现：

```cpp
template<typename T>                                //在std命名空间
T&& forward(typename
                remove_reference<T>::type& param)
{
    return static_cast<T&&>(param);
}
```

这不是标准库版本的实现（忽略了一些接口描述），但是为了理解`std::forward`的行为，这些差异无关紧要。

假设传入到`f`的实参是`Widget`的左值类型。`T`被推导为`Widget&`，然后调用`std::forward`将实例化为`std::forward<Widget&>`。`Widget&`带入到上面的`std::forward`的实现中：

```cpp
Widget& && forward(typename 
                       remove_reference<Widget&>::type& param)
{ return static_cast<Widget& &&>(param); }
```

`std::remove_reference<Widget&>::type`这个*type trait*产生`Widget`（查看[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)），所以`std::forward`成为：

```cpp
Widget& && forward(Widget& param)
{ return static_cast<Widget& &&>(param); }
```

根据引用折叠规则，返回值和强制转换可以化简，最终版本的`std::forward`调用就是：

```cpp
Widget& forward(Widget& param)
{ return static_cast<Widget&>(param); }
```

正如你所看到的，当左值实参被传入到函数模板`f`时，`std::forward`被实例化为接受和返回左值引用。内部的转换不做任何事，因为`param`的类型已经是`Widget&`，所以转换没有影响。左值实参传入`std::forward`会返回左值引用。通过定义，左值引用就是左值，因此将左值传递给`std::forward`会返回左值，就像期待的那样。

现在假设一下，传递给`f`的实参是一个`Widget`的右值。在这个例子中，`f`的类型参数`T`的推导类型就是`Widget`。`f`内部的`std::forward`调用因此为`std::forward<Widget>`，`std::forward`实现中把`T`换为`Widget`得到：

```cpp
Widget&& forward(typename
                     remove_reference<Widget>::type& param)
{ return static_cast<Widget&&>(param); }
```

将`std::remove_reference`引用到非引用类型`Widget`上还是相同的类型（`Widget`），所以`std::forward`变成：

```cpp
Widget&& forward(Widget& param)
{ return static_cast<Widget&&>(param); }
```

这里没有引用的引用，所以不需要引用折叠，这就是`std::forward`的最终实例化版本。

从函数返回的右值引用被定义为右值，因此在这种情况下，`std::forward`会将`f`的形参`fParam`（左值）转换为右值。最终结果是，传递给`f`的右值参数将作为右值转发给`someFunc`，正是想要的结果。

在C++14中，`std::remove_reference_t`的存在使得实现变得更简洁：

```cpp
template<typename T>                        //C++14；仍然在std命名空间
T&& forward(remove_reference_t<T>& param)
{
  return static_cast<T&&>(param);
}
```

引用折叠发生在四种情况下。第一，也是最常见的就是模板实例化。第二，是`auto`变量的类型生成，具体细节类似于模板，因为`auto`变量的类型推导基本与模板类型推导雷同（参见[Item2](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)）。考虑本前面的例子：

```cpp
Widget widgetFactory();     //返回右值的函数
Widget w;                   //一个变量（左值）
func(w);                    //用左值调用func；T被推导为Widget&
func(widgetFactory());      //用又值调用func；T被推导为Widget
```

在auto的写法中，规则是类似的。声明

```cpp
auto&& w1 = w;
```

用一个左值初始化`w1`，因此为`auto`推导出类型`Widget&`。把`Widget&`代回`w1`声明中的`auto`里，产生了引用的引用，
```cpp
Widget& && w1 = w;
```
应用引用折叠规则，就是
```cpp
Widget& w1 = w
```
结果就是`w1`是一个左值引用。

另一方面，这个声明，
```cpp
auto&& w2 = widgetFactory();
```
使用右值初始化`w2`，为`auto`推导出非引用类型`Widget`。把`Widget`代入`auto`得到：
```cpp
Widget&& w2 = widgetFactory()
```
没有引用的引用，这就是最终结果，`w2`是个右值引用。

现在真正理解了[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)中引入的通用引用。通用引用不是一种新的引用，它实际上是满足以下两个条件下的右值引用：

- **类型推导区分左值和右值**。`T`类型的左值被推导为`T&`类型，`T`类型的右值被推导为`T`。
- **发生引用折叠**。

通用引用的概念是有用的，因为它使你不必一定意识到引用折叠的存在，从直觉上推导左值和右值的不同类型，在凭直觉把推导的类型代入到它们出现的上下文中之后应用引用折叠规则。

我说了有四种情况会发生引用折叠，但是只讨论了两种：模板实例化和`auto`的类型生成。第三种情况是`typedef`和别名声明的产生和使用中（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)）。如果，在创建或者评估`typedef`过程中出现了引用的引用，则引用折叠就会起作用。举例子来说，假设有一个`Widget`的类模板，该模板具有右值引用类型的嵌入式`typedef`：

```cpp
template<typename T>
class Widget {
public:
    typedef T&& RvalueRefToT;
    …
};
```

假设使用左值引用实例化`Widget`：

```cpp
Widget<int&> w;
```

`Widget`模板中把`T`替换为`int&`得到：

```cpp
typedef int& && RvalueRefToT;
```

引用折叠就会发挥作用：

```cpp
typedef int& RvalueRefToT;
```

这清楚表明为`typedef`选择的名字可能不是希望的那样：当使用左值引用类型实例化`Widget`时，`RvalueRefToT`是**左值引用**的`typedef`。

最后一种引用折叠发生的情况是，`decltype`使用的情况。如果在分析`decltype`期间，出现了引用的引用，引用折叠规则就会起作用（关于`decltype`，参见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）

**请记住：**

- 引用折叠发生在四种情况下：模板实例化，`auto`类型推导，`typedef`与别名声明的创建和使用，`decltype`。
- 当编译器在引用折叠环境中生成了引用的引用时，结果就是单个引用。有左值引用折叠结果就是左值引用，否则就是右值引用。
- 通用引用就是在特定上下文的右值引用，上下文是通过类型推导区分左值还是右值，并且发生引用折叠的那些地方。
## 二十九：假定移动操作不存在，成本高，未被使用

**Item 29: Assume that move operations are not present, not cheap, and not used**

移动语义可以说是C++11最主要的特性。你可能会见过这些类似的描述“移动容器和拷贝指针一样开销小”， “拷贝临时对象现在如此高效，写代码避免这种情况简直就是过早优化”。这种情绪很容易理解。移动语义确实是这样重要的特性。它不仅允许编译器使用开销小的移动操作代替大开销的复制操作，而且默认这么做（当特定条件满足的时候）。以C++98的代码为基础，使用C++11重新编译你的代码，然后，哇，你的软件运行的更快了。

移动语义确实可以做这些事，这把这个特性封为一代传说。但是传说总有些夸大成分。这个的目的就是给你泼一瓢冷水，保持理智看待移动语义。

让从已知很多类型不支持移动操作开始这个过程。为了升级到C++11，C++98的很多标准库做了大修改，为很多类型提供了移动的能力，这些类型的移动实现比复制操作更快，并且对库的组件实现修改以利用移动操作。但是很有可能你工作中的代码没有完整地利用C++11。对于你的应用中（或者代码库中）的类型，没有适配C++11的部分，编译器即使支持移动语义也是无能为力的。的确，C++11倾向于为缺少移动操作的类生成它们，但是只有在没有声明复制操作，移动操作，或析构函数的类中才会生成移动操作（参考[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)）。数据成员或者某类型的基类禁止移动操作（比如通过delete移动操作，参考[Item11](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item11.md)），编译器不生成移动操作的支持。对于没有明确支持移动操作的类型，并且不符合编译器默认生成的条件的类，没有理由期望C++11会比C++98进行任何性能上的提升。

即使显式支持了移动操作，结果可能也没有你希望的那么好。比如，所有C++11的标准库容器都支持了移动操作，但是认为移动所有容器的开销都非常小是个错误。对于某些容器来说，压根就不存在开销小的方式来移动它所包含的内容。对另一些容器来说，容器的开销真正小的移动操作会有些容器元素不能满足的注意条件。

考虑一下`std::array`，这是C++11中的新容器。`std::array`本质上是具有STL接口的内置数组。这与其他标准容器将内容存储在堆内存不同。存储具体数据在堆内存的容器，本身只保存了指向堆内存中容器内容的指针（真正实现当然更复杂一些，但是基本逻辑就是这样）。这个指针的存在使得在常数时间移动整个容器成为可能，只需要从源容器拷贝保存指向容器内容的指针到目标容器，然后将源指针置为空指针就可以了：

```cpp
std::vector<Widget> vm1;

//把数据存进vw1
…

//把vw1移动到vw2。以常数时间运行。只有vw1和vw2中的指针被改变
auto vm2 = std::move(vm1);
```

![item29_fig1](media/item29_fig1.png)

`std::array`没有这种指针实现，数据就保存在`std::array`对象中：

```cpp
std::array<Widget, 10000> aw1;

//把数据存进aw1
…

//把aw1移动到aw2。以线性时间运行。aw1中所有元素被移动到aw2
auto aw2 = std::move(aw1);
```

![item29_fig2](media/item29_fig2.png)

注意`aw1`中的元素被**移动**到了`aw2`中。假定`Widget`类的移动操作比复制操作快，移动`Widget`的`std::array`就比复制要快。所以`std::array`确实支持移动操作。但是使用`std::array`的移动操作还是复制操作都将花费线性时间的开销，因为每个容器中的元素终归需要拷贝或移动一次，这与“移动一个容器就像操作几个指针一样方便”的含义相去甚远。

另一方面，`std::string`提供了常数时间的移动操作和线性时间的复制操作。这听起来移动比复制快多了，但是可能不一定。许多字符串的实现采用了小字符串优化（*small string optimization*，SSO）。“小”字符串（比如长度小于15个字符的）存储在了`std::string`的缓冲区中，并没有存储在堆内存，移动这种存储的字符串并不必复制操作更快。

SSO的动机是大量证据表明，短字符串是大量应用使用的习惯。使用内存缓冲区存储而不分配堆内存空间，是为了更好的效率。然而这种内存管理的效率导致移动的效率并不必复制操作高，即使一个半吊子程序员也能看出来对于这样的字符串，拷贝并不比移动慢。

即使对于支持快速移动操作的类型，某些看似可靠的移动操作最终也会导致复制。[Item14](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item14.md)解释了原因，标准库中的某些容器操作提供了强大的异常安全保证，确保依赖那些保证的C++98的代码在升级到C++11且仅当移动操作不会抛出异常，从而可能替换操作时，不会不可运行。结果就是，即使类提供了更具效率的移动操作，而且即使移动操作更合适（比如源对象是右值），编译器仍可能被迫使用复制操作，因为移动操作没有声明`noexcept`。

因此，存在几种情况，C++11的移动语义并无优势：

- **没有移动操作**：要移动的对象没有提供移动操作，所以移动的写法也会变成复制操作。
- **移动不会更快**：要移动的对象提供的移动操作并不比复制速度更快。
- **移动不可用**：进行移动的上下文要求移动操作不会抛出异常，但是该操作没有被声明为`noexcept`。

值得一提的是，还有另一个场景，会使得移动并没有那么有效率：

- **源对象是左值**：除了极少数的情况外（例如[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)），只有右值可以作为移动操作的来源。

但是该的标题是假定移动操作不存在，成本高，未被使用。这就是通用代码中的典型情况，比如编写模板代码，因为你不清楚你处理的具体类型是什么。在这种情况下，你必须像出现移动语义之前那样，像在C++98里一样保守地去复制对象。“不稳定的”代码也是如此，即那些由于经常被修改导致类型特性变化的源代码。

但是，通常，你了解你代码里使用的类型，依赖他们的特性不变性（比如是否支持快速移动操作）。这种情况，你无需这个的假设，只需要查找所用类型的移动操作详细信息。如果类型提供了快速移动操作，并且在调用移动操作的上下文中使用对象，可以安全的使用快速移动操作替换复制操作。

**请记住：**

- 假定移动操作不存在，成本高，未被使用。
- 在已知的类型或者支持移动语义的代码中，就不需要上面的假设。
## 三十：熟悉完美转发失败的情况

**Item 30: Familiarize yourself with perfect forwarding failure cases**

C++11最显眼的功能之一就是完美转发功能。**完美**转发，太**完美**了！哎，开始使用，你就发现“完美”，理想与现实还是有差距。C++11的完美转发是非常好用，但是只有当你愿意忽略一些误差情况（译者注：就是完美转发失败的情况），这个就是使你熟悉这些情形。

在开始误差探索之前，有必要回顾一下“完美转发”的含义。“转发”仅表示将一个函数的形参传递——就是**转发**——给另一个函数。对于第二个函数（被传递的那个）目标是收到与第一个函数（执行传递的那个）完全相同的对象。这规则排除了按值传递的形参，因为它们是原始调用者传入内容的**拷贝**。希望被转发的函数能够使用最开始传进来的那些对象。指针形参也被排除在外，因为不想强迫调用者传入指针。关于通常目的的转发，将处理引用形参。

**完美转发**（*perfect forwarding*）意味着不仅转发对象，还转发显著的特征：它们的类型，是左值还是右值，是`const`还是`volatile`。结合到会处理引用形参，这意味着将使用通用引用（参见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），因为通用引用形参被传入实参时才确定是左值还是右值。

假定有一些函数`f`，然后想编写一个转发给它的函数（事实上是一个函数模板）。需要的核心看起来像是这样：

```cpp
template<typename T>
void fwd(T&& param)             //接受任何实参
{
    f(std::forward<T>(param));  //转发给f
}
```

从本质上说，转发函数是通用的。例如`fwd`模板，接受任何类型的实参，并转发得到的任何东西。这种通用性的逻辑扩展是，转发函数不仅是模板，而且是可变模板，因此可以接受任何数量的实参。`fwd`的可变形式如下：

```cpp
template<typename... Ts>
void fwd(Ts&&... params)            //接受任何实参
{
    f(std::forward<Ts>(params)...); //转发给f
}
```

这种形式你会在标准化容器置入函数（emplace functions）中（参见[Item42](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/8.Tweaks/item42.md)）和智能指针的工厂函数`std::make_unique`和`std::make_shared`中（参见[Item21](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item21.md)）看到，当然还有其他一些地方。

给定的目标函数`f`和转发函数`fwd`，如果`f`使用某特定实参会执行某个操作，但是`fwd`使用相同的实参会执行不同的操作，完美转发就会失败

```cpp
f( expression );        //调用f执行某个操作
fwd( expression );		//但调用fwd执行另一个操作，则fwd不能完美转发expression给f
```

导致这种失败的实参种类有很多。知道它们是什么以及如何解决它们很重要，因此让来看看无法做到完美转发的实参类型。

### 花括号初始化器

假定`f`这样声明：

```cpp
void f(const std::vector<int>& v);
```

在这个例子中，用花括号初始化调用`f`通过编译，

```cpp
f({ 1, 2, 3 });         //可以，“{1, 2, 3}”隐式转换为std::vector<int>
```

但是传递相同的列表初始化给fwd不能编译

```cpp
fwd({ 1, 2, 3 });       //错误！不能编译
```

这是因为这是完美转发失效的一种情况。

所有这种错误有相同的原因。在对`f`的直接调用（例如`f({ 1, 2, 3 })`），编译器看看调用地传入的实参，看看`f`声明的形参类型。它们把调用地的实参和声明的实参进行比较，看看是否匹配，并且必要时执行隐式转换操作使得调用成功。在上面的例子中，从`{ 1, 2, 3 }`生成了临时`std::vector<int>`对象，因此`f`的形参`v`会绑定到`std::vector<int>`对象上。

当通过调用函数模板`fwd`间接调用`f`时，编译器不再把调用地传入给`fwd`的实参和`f`的声明中形参类型进行比较。而是**推导**传入给`fwd`的实参类型，然后比较推导后的实参类型和`f`的形参声明类型。当下面情况任何一个发生时，完美转发就会失败：

- **编译器不能推导出`fwd`的一个或者多个形参类型。**这种情况下代码无法编译。
- **编译器推导“错”了`fwd`的一个或者多个形参类型。**在这里，“错误”可能意味着`fwd`的实例将无法使用推导出的类型进行编译，但是也可能意味着使用`fwd`的推导类型调用`f`，与用传给`fwd`的实参直接调用`f`表现出不一致的行为。这种不同行为的原因可能是因为`f`是个重载函数的名字，并且由于是“不正确的”类型推导，在`fwd`内部调用的`f`重载和直接调用的`f`重载不一样。

在上面的`fwd({ 1, 2, 3 })`例子中，问题在于，将花括号初始化传递给未声明为`std::initializer_list`的函数模板形参，被判定为——就像标准说的——“非推导上下文”。简单来讲，这意味着编译器不准在对`fwd`的调用中推导表达式`{ 1, 2, 3 }`的类型，因为`fwd`的形参没有声明为`std::initializer_list`。对于`fwd`形参的推导类型被阻止，编译器只能拒绝该调用。

有趣的是，[Item2](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)说明了使用花括号初始化的`auto`的变量的类型推导是成功的。这种变量被视为`std::initializer_list`对象，在转发函数应推导出类型为`std::initializer_list`的情况，这提供了一种简单的解决方法——使用`auto`声明一个局部变量，然后将局部变量传进转发函数：

```cpp
auto il = { 1, 2, 3 };  //il的类型被推导为std::initializer_list<int>
fwd(il);                //可以，完美转发il给f
```

### `0`或者`NULL`作为空指针

[Item8](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item8.md)说明当你试图传递`0`或者`NULL`作为空指针给模板时，类型推导会出错，会把传来的实参推导为一个整型类型（典型情况为`int`）而不是指针类型。结果就是不管是`0`还是`NULL`都不能作为空指针被完美转发。解决方法非常简单，传一个`nullptr`而不是`0`或者`NULL`。具体的细节，参考[Item8](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item8.md)。

### 仅有声明的整型`static const`数据成员

通常，无需在类中定义整型`static const`数据成员；声明就可以了。这是因为编译器会对此类成员实行**常量传播**（*const propagation*），因此消除了保留内存的需要。比如，考虑下面的代码：

```cpp
class Widget {
public:
    static const std::size_t MinVals = 28;  //MinVal的声明
    …
};
…                                           //没有MinVals定义

std::vector<int> widgetData;
widgetData.reserve(Widget::MinVals);        //使用MinVals
```

这里，使用`Widget::MinVals`（或者简单点`MinVals`）来确定`widgetData`的初始容量，即使`MinVals`缺少定义。编译器通过将值28放入所有提到`MinVals`的位置来补充缺少的定义（就像它们被要求的那样）。没有为`MinVals`的值留存储空间是没有问题的。如果要使用`MinVals`的地址（例如，有人创建了指向`MinVals`的指针），则`MinVals`需要存储（这样指针才有可指向的东西），尽管上面的代码仍然可以编译，但是链接时就会报错，直到为`MinVals`提供定义。

按照这个思路，想象下`f`（`fwd`要转发实参给它的那个函数）这样声明：

```cpp
void f(std::size_t val);
```

使用`MinVals`调用`f`是可以的，因为编译器直接将值28代替`MinVals`：

```cpp
f(Widget::MinVals);         //可以，视为“f(28)”
```

不过如果尝试通过`fwd`调用`f`，事情不会进展那么顺利：

```cpp
fwd(Widget::MinVals);       //错误！不应该链接
```

代码可以编译，但是不应该链接。如果这让你想到使用`MinVals`地址会发生的事，确实，底层的问题是一样的。

尽管代码中没有使用`MinVals`的地址，但是`fwd`的形参是通用引用，而引用，在编译器生成的代码中，通常被视作指针。在程序的二进制底层代码中（以及硬件中）指针和引用是一样的。在这个水平上，引用只是可以自动解引用的指针。在这种情况下，通过引用传递`MinVals`实际上与通过指针传递`MinVals`是一样的，因此，必须有内存使得指针可以指向。通过引用传递的整型`static const`数据成员，通常需要定义它们，这个要求可能会造成在不使用完美转发的代码成功的地方，使用等效的完美转发失败。（译者注：这里意思应该是没有定义，完美转发就会失败）

可能你也注意到了在上述讨论中我使用了一些模棱两可的词。代码“不应该”链接。引用“通常”被看做指针。传递整型`static const`数据成员“通常”要求定义。看起来就像有些事情我没有告诉你......

确实，根据标准，通过引用传递`MinVals`要求有定义。但不是所有的实现都强制要求这一点。所以，取决于你的编译器和链接器，你可能发现你可以在未定义的情况使用完美转发，恭喜你，但是这不是那样做的理由。为了具有可移植性，只要给整型`static const`提供一个定义，比如这样：

```cpp
const std::size_t Widget::MinVals;  //在Widget的.cpp文件
```

注意定义中不要重复初始化（这个例子中就是赋值28）。但是不要忽略这个细节。如果你忘了，并且在两个地方都提供了初始化，编译器就会报错，提醒你只能初始化一次。

### 重载函数的名称和模板名称

假定的函数`f`（想通过`fwd`完美转发实参给的那个函数）可以通过向其传递执行某些功能的函数来自定义其行为。假设这个函数接受和返回值都是`int`，`f`声明就像这样：

```cpp
void f(int (*pf)(int));             //pf = “process function”
```

值得注意的是，也可以使用更简单的非指针语法声明。这种声明就像这样，含义与上面是一样的：

```cpp
void f(int pf(int));                //与上面定义相同的f
```

无论哪种写法都可，现在假设有了一个重载函数，`processVal`：

```cpp
int processVal(int value);
int processVal(int value, int priority);
```

可以传递`processVal`给`f`，

```cpp
f(processVal);                      //可以
```

但是会发现一些吃惊的事情。`f`要求一个函数指针作为实参，但是`processVal`不是一个函数指针或者一个函数，它是同名的两个不同函数。但是，编译器可以知道它需要哪个：匹配上`f`的形参类型的那个。因此选择了仅带有一个`int`的`processVal`地址传递给`f`。

工作的基本机制是`f`的声明让编译器识别出哪个是需要的`processVal`。但是，`fwd`是一个函数模板，没有它可接受的类型的信息，使得编译器不可能决定出哪个函数应被传递：

```cpp
fwd(processVal);                    //错误！那个processVal？
```

单用`processVal`是没有类型信息的，所以就不能类型推导，完美转发失败。

如果试图使用函数模板而不是（或者也加上）重载函数的名字，同样的问题也会发生。一个函数模板不代表单独一个函数，它表示一个函数族：

```cpp
template<typename T>
T workOnVal(T param)                //处理值的模板
{ … }

fwd(workOnVal);                     //错误！哪个workOnVal实例？
```

要让像`fwd`的完美转发函数接受一个重载函数名或者模板名，方法是指定要转发的那个重载或者实例。比如，你可以创造与`f`相同形参类型的函数指针，通过`processVal`或者`workOnVal`实例化这个函数指针（这可以引导选择正确版本的`processVal`或者产生正确的`workOnVal`实例），然后传递指针给`fwd`：

```cpp
using ProcessFuncType =                         //写个类型定义；见9
    int (*)(int);

ProcessFuncType processValPtr = processVal;     //指定所需的processVal签名

fwd(processValPtr);                             //可以
fwd(static_cast<ProcessFuncType>(workOnVal));   //也可以
```

当然，这要求你知道`fwd`转发的函数指针的类型。没有理由去假定完美转发函数会记录着这些东西。毕竟，完美转发被设计为接受任何内容，所以如果没有文档告诉你要传递什么，你又从何而知这些东西呢？

### 位域

完美转发最后一种失败的情况是函数实参使用位域这种类型。为了更直观的解释，IPv4的头部有如下模型：（这假定的是位域是按从最低有效位（*least significant bit*，lsb）到最高有效位（*most significant bit*，msb）布局的。C++不保证这一点，但是编译器经常提供一种机制，允许程序员控制位域布局。）

```cpp
struct IPv4Header {
    std::uint32_t version:4,
                  IHL:4,
                  DSCP:6,
                  ECN:2,
                  totalLength:16;
    …
};
```

如果声明的函数`f`（转发函数`fwd`的目标）为接收一个`std::size_t`的形参，则使用`IPv4Header`对象的`totalLength`字段进行调用没有问题：

```cpp
void f(std::size_t sz);         //要调用的函数

IPv4Header h;
…
f(h.totalLength);               //可以
```

如果通过`fwd`转发`h.totalLength`给`f`呢，那就是一个不同的情况了：

```cpp
fwd(h.totalLength);             //错误！
```

问题在于`fwd`的形参是引用，而`h.totalLength`是non-`const`位域。听起来并不是那么糟糕，但是C++标准非常清楚地谴责了这种组合：non-`const`引用不应该绑定到位域。禁止的理由很充分。位域可能包含了机器字的任意部分（比如32位`int`的3-5位），但是这些东西无法直接寻址。我之前提到了在硬件层面引用和指针是一样的，所以没有办法创建一个指向任意*bit*的指针（C++规定你可以指向的最小单位是`char`），同样没有办法绑定引用到任意*bit*上。

一旦意识到接收位域实参的函数都将接收位域的**副本**，就可以轻松解决位域不能完美转发的问题。毕竟，没有函数可以绑定引用到位域，也没有函数可以接受指向位域的指针，因为不存在这种指针。位域可以传给的形参种类只有按值传递的形参，有趣的是，还有reference-to-`const`。在传值形参的情况中，被调用的函数接受了一个位域的副本；在传reference-to-`const`形参的情况中，标准要求这个引用实际上绑定到存放位域值的副本对象，这个对象是某种整型（比如`int`）。reference-to-`const`不直接绑定到位域，而是绑定位域值拷贝到的一个普通对象。

传递位域给完美转发的关键就是利用传给的函数接受的是一个副本的事实。你可以自己创建副本然后利用副本调用完美转发。在`IPv4Header`的例子中，可以如下写法：

```cpp
//拷贝位域值；参看6了解关于初始化形式的信息
auto length = static_cast<std::uint16_t>(h.totalLength);

fwd(length);                    //转发这个副本
```

### 总结

在大多数情况下，完美转发工作的很好。你基本不用考虑其他问题。但是当其不工作时——当看起来合理的代码无法编译，或者更糟的是，虽能编译但无法按照预期运行时——了解完美转发的缺陷就很重要了。同样重要的是如何解决它们。在大多数情况下，都很简单。

**请记住：**

- 当模板类型推导失败或者推导出错误类型，完美转发会失败。
- 导致完美转发失败的实参种类有花括号初始化，作为空指针的`0`或者`NULL`，仅有声明的整型`static const`数据成员，模板和重载函数的名字，位域。
