---
title: 【Program】LambdaExpressions《Effective Modern C++》
author: DH Wang
date: 1000-01-01
category: Program
layout: post
---
 # 第6章 *lambda*表达式

**CHAPTER 6 Lambda Expressions**

*lambda*表达式是C++编程中的游戏规则改变者。这有点令人惊讶，因为它没有给语言带来新的表达能力。*lambda*可以做的所有事情都可以通过其他方式完成。但是*lambda*是创建函数对象相当便捷的一种方法，对于日常的C++开发影响是巨大的。没有*lambda*时，STL中的“`_if`”算法（比如，`std::find_if`，`std::remove_if`，`std::count_if`等）通常需要繁琐的谓词，但是当有*lambda*可用时，这些算法使用起来就变得相当方便。用比较函数（比如，`std::sort`，`std::nth_element`，`std::lower_bound`等）来自定义算法也是同样方便的。在STL外，*lambda*可以快速创建`std::unique_ptr`和`std::shared_ptr`的自定义删除器（见[Item18](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)和[19](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item19.md)），并且使线程API中条件变量的谓词指定变得同样简单（参见[Item39](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item39.md)）。除了标准库，*lambda*有利于即时的回调函数，接口适配函数和特定上下文中的一次性函数。*lambda*确实使C++成为更令人愉快的编程语言。

与*lambda*相关的词汇可能会令人疑惑，这里做一下简单的回顾：

- ***lambda*表达式**（*lambda expression*）就是一个表达式。下面是部分源代码。在

  ```cpp
  std::find_if(container.begin(), container.end(),
               [](int val){ return 0 < val && val < 10; });   //译者注：本行高亮
  ```

  中，代码的高亮部分就是*lambda*。

- **闭包**（*enclosure*）是*lambda*创建的运行时对象。依赖捕获模式，闭包持有被捕获数据的副本或者引用。在上面的`std::find_if`调用中，闭包是作为第三个实参在运行时传递给`std::find_if`的对象。

- **闭包类**（*closure class*）是从中实例化闭包的类。每个*lambda*都会使编译器生成唯一的闭包类。*lambda*中的语句成为其闭包类的成员函数中的可执行指令。

*lambda*通常被用来创建闭包，该闭包仅用作函数的实参。上面对`std::find_if`的调用就是这种情况。然而，闭包通常可以拷贝，所以可能有多个闭包对应于一个*lambda*。比如下面的代码：

```cpp
{
    int x;                                  //x是局部对象
    …

    auto c1 =                               //c1是lambda产生的闭包的副本
        [x](int y) { return x * y > 55; };

    auto c2 = c1;                           //c2是c1的拷贝

    auto c3 = c2;                           //c3是c2的拷贝
    …
}
```

`c1`，`c2`，`c3`都是*lambda*产生的闭包的副本。

非正式的讲，模糊*lambda*，闭包和闭包类之间的界限是可以接受的。但是，在随后的Item中，区分什么存在于编译期（*lambdas* 和闭包类），什么存在于运行时（闭包）以及它们之间的相互关系是重要的。

## 三十一：避免使用默认捕获模式

**Item 31: Avoid default capture modes**

C++11中有两种默认的捕获模式：按引用捕获和按值捕获。但默认按引用捕获模式可能会带来悬空引用的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。

这就是本的一个总结。如果你偏向技术，渴望了解更多内容，就让从按引用捕获的危害谈起吧。

按引用捕获会导致闭包中包含了对某个局部变量或者形参的引用，变量或形参只在定义*lambda*的作用域中可用。如果该*lambda*创建的闭包生命周期超过了局部变量或者形参的生命周期，那么闭包中的引用将会变成悬空引用。举个例子，假如有元素是过滤函数（filtering function）的一个容器，该函数接受一个`int`，并返回一个`bool`，该`bool`的结果表示传入的值是否满足过滤条件：

```c++
using FilterContainer =                     //“using”参见9，
	std::vector<std::function<bool(int)>>;  //std::function参见2

FilterContainer filters;                    //过滤函数
```

可以添加一个过滤器，用来过滤掉5的倍数：

```c++
filters.emplace_back(                       //emplace_back的信息见42
	[](int value) { return value % 5 == 0; }
);
```

然而可能需要的是能够在运行期计算除数（divisor），即不能将5硬编码到*lambda*中。因此添加的过滤器逻辑将会是如下这样：

```c++
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                               //危险！对divisor的引用
        [&](int value) { return value % divisor == 0; } //将会悬空！
    );
}
```

这个代码实现是一个定时炸弹。*lambda*对局部变量`divisor`进行了引用，但该变量的生命周期会在`addDivisorFilter`返回时结束，刚好就是在语句`filters.emplace_back`返回之后。因此添加到`filters`的函数添加完，该函数就死亡了。使用这个过滤器（译者注：就是那个添加进`filters`的函数）会导致未定义行为，这是由它被创建那一刻起就决定了的。

现在，同样的问题也会出现在`divisor`的显式按引用捕获。

```c++
filters.emplace_back(
    [&divisor](int value) 			    //危险！对divisor的引用将会悬空！
    { return value % divisor == 0; }
);
```

但通过显式的捕获，能更容易看到*lambda*的可行性依赖于变量`divisor`的生命周期。另外，写下“divisor”这个名字能够提醒要注意确保`divisor`的生命周期至少跟*lambda*闭包一样长。比起“`[&]`”传达的意思，显式捕获能让人更容易想起“确保没有悬空变量”。

如果你知道一个闭包将会被马上使用（例如被传入到一个STL算法中）并且不会被拷贝，那么在它的*lambda*被创建的环境中，将不会有持有的引用比局部变量和形参活得长的风险。在这种情况下，你可能会争论说，没有悬空引用的危险，就不需要避免使用默认的引用捕获模式。例如，的过滤*lambda*只会用做C++11中`std::all_of`的一个实参，返回满足条件的所有元素：

```c++
template<typename C>
void workWithContainer(const C& container)
{
    auto calc1 = computeSomeValue1();               //同上
    auto calc2 = computeSomeValue2();               //同上
    auto divisor = computeDivisor(calc1, calc2);    //同上

    using ContElemT = typename C::value_type;       //容器内元素的类型
    using std::begin;                               //为了泛型，见13
    using std::end;

    if (std::all_of(                                //如果容器内所有值都为
            begin(container), end(container),       //除数的倍数
            [&](const ContElemT& value)
            { return value % divisor == 0; })
        ) {
        …                                           //它们...
    } else {
        …                                           //至少有一个不是的话...
    }
}
```

的确如此，这是安全的做法，但这种安全是不确定的。如果发现*lambda*在其它上下文中很有用（例如作为一个函数被添加在`filters`容器中），然后拷贝粘贴到一个`divisor`变量已经死亡，但闭包生命周期还没结束的上下文中，你又回到了悬空的使用上了。同时，在该捕获语句中，也没有特别提醒了你注意分析`divisor`的生命周期。

从长期来看，显式列出*lambda*依赖的局部变量和形参，是更加符合软件工程规范的做法。

额外提一下，C++14支持了在*lambda*中使用`auto`来声明变量，上面的代码在C++14中可以进一步简化，`ContElemT`的别名可以去掉，`if`条件可以修改为：

```c++
if (std::all_of(begin(container), end(container),
			   [&](const auto& value)               // C++14
			   { return value % divisor == 0; }))			
```

一个解决问题的方法是，`divisor`默认按值捕获进去，也就是说可以按照以下方式来添加*lambda*到`filters`：

```c++
filters.emplace_back( 							    //现在divisor不会悬空了
    [=](int value) { return value % divisor == 0; }
);
```

这足以满足本实例的要求，但在通常情况下，按值捕获并不能完全解决悬空引用的问题。这里的问题是如果你按值捕获的是一个指针，你将该指针拷贝到*lambda*对应的闭包里，但这样并不能避免*lambda*外`delete`这个指针的行为，从而导致你的副本指针变成悬空指针。

也许你要抗议说：“这不可能发生。看过了[第4章](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)，我对智能指针的使用非常热衷。只有那些失败的C++98的程序员才会用裸指针和`delete`语句。”这也许是正确的，但却是不相关的，因为事实上你的确会使用裸指针，也的确存在被你`delete`的可能性。只不过在现代的C++编程风格中，不容易在源代码中显露出来而已。

假设在一个`Widget`类，可以实现向过滤器的容器添加条目：

```c++
class Widget {
public:
    …                       //构造函数等
    void addFilter() const; //向filters添加条目
private:
    int divisor;            //在Widget的过滤器使用
};
```

这是`Widget::addFilter`的定义：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}	
```

这个做法看起来是安全的代码。*lambda*依赖于`divisor`，但默认的按值捕获确保`divisor`被拷贝进了*lambda*对应的所有闭包中，对吗？

错误，完全错误。

捕获只能应用于*lambda*被创建时所在作用域里的non-`static`局部变量（包括形参）。在`Widget::addFilter`的视线里，`divisor`并不是一个局部变量，而是`Widget`类的一个成员变量。它不能被捕获。而如果默认捕获模式被删除，代码就不能编译了：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(                               //错误！
        [](int value) { return value % divisor == 0; }  //divisor不可用
    ); 
} 
```

另外，如果尝试去显式地捕获`divisor`变量（或者按引用或者按值——这不重要），也一样会编译失败，因为`divisor`不是一个局部变量或者形参。

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor](int value)                //错误！没有名为divisor局部变量可捕获
        { return value % divisor == 0; }
    );
}
```

所以如果默认按值捕获不能捕获`divisor`，而不用默认按值捕获代码就不能编译，这是怎么一回事呢？

解释就是这里隐式使用了一个原始指针：`this`。每一个non-`static`成员函数都有一个`this`指针，每次你使用一个类内的数据成员时都会使用到这个指针。例如，在任何`Widget`成员函数中，编译器会在内部将`divisor`替换成`this->divisor`。在默认按值捕获的`Widget::addFilter`版本中，

```c++
void Widget::addFilter() const
{
    filters.emplace_back(
        [=](int value) { return value % divisor == 0; }
    );
}
```

真正被捕获的是`Widget`的`this`指针，而不是`divisor`。编译器会将上面的代码看成以下的写法：

```c++
void Widget::addFilter() const
{
    auto currentObjectPtr = this;

    filters.emplace_back(
        [currentObjectPtr](int value)
        { return value % currentObjectPtr->divisor == 0; }
    );
}
```

明白了这个就相当于明白了*lambda*闭包的生命周期与`Widget`对象的关系，闭包内含有`Widget`的`this`指针的拷贝。特别是考虑以下的代码，参考[第4章](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)的内容，只使用智能指针：

```c++
using FilterContainer = 					//跟之前一样
    std::vector<std::function<bool(int)>>;

FilterContainer filters;                    //跟之前一样

void doSomeWork()
{
    auto pw =                               //创建Widget；std::make_unique
        std::make_unique<Widget>();         //见21

    pw->addFilter();                        //添加使用Widget::divisor的过滤器

    …
}                                           //销毁Widget；filters现在持有悬空指针！
```

当调用`doSomeWork`时，就会创建一个过滤器，其生命周期依赖于由`std::make_unique`产生的`Widget`对象，即一个含有指向`Widget`的指针——`Widget`的`this`指针——的过滤器。这个过滤器被添加到`filters`中，但当`doSomeWork`结束时，`Widget`会由管理它的`std::unique_ptr`来销毁（见[Item18](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)）。从这时起，`filter`会含有一个存着悬空指针的条目。

这个特定的问题可以通过给你想捕获的数据成员做一个局部副本，然后捕获这个副本去解决：

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员

    filters.emplace_back(
        [divisorCopy](int value)                //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}
```

事实上如果采用这种方法，默认的按值捕获也是可行的。

```c++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;                 //拷贝数据成员

    filters.emplace_back(
        [=](int value)                          //捕获副本
        { return value % divisorCopy == 0; }	//使用副本
    );
}
```

但为什么要冒险呢？当一开始你认为你捕获的是`divisor`的时候，默认捕获模式就是造成可能意外地捕获`this`的元凶。

在C++14中，一个更好的捕获成员变量的方式时使用通用的*lambda*捕获：

```c++
void Widget::addFilter() const
{
    filters.emplace_back(                   //C++14：
        [divisor = divisor](int value)      //拷贝divisor到闭包
        { return value % divisor == 0; }	//使用这个副本
    );
}
```

这种通用的*lambda*捕获并没有默认的捕获模式，因此在C++14中，本的建议——避免使用默认捕获模式——仍然是成立的。

使用默认的按值捕获还有另外的一个缺点，它们预示了相关的闭包是独立的并且不受外部数据变化的影响。一般来说，这是不对的。*lambda*可能会依赖局部变量和形参（它们可能被捕获），还有**静态存储生命周期**（static storage duration）的对象。这些对象定义在全局空间或者命名空间，或者在类、函数、文件中声明为`static`。这些对象也能在*lambda*里使用，但它们不能被捕获。但默认按值捕获可能会因此误导你，让你以为捕获了这些变量。参考下面版本的`addDivisorFilter`函数：

```c++
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();    //现在是static
    static auto calc2 = computeSomeValue2();    //现在是static
    static auto divisor =                       //现在是static
    computeDivisor(calc1, calc2);

    filters.emplace_back(
        [=](int value)                          //什么也没捕获到！
        { return value % divisor == 0; }        //引用上面的static
    );

    ++divisor;                                  //调整divisor
}
```

随意地看了这份代码的读者可能看到“`[=]`”，就会认为“好的，*lambda*拷贝了所有使用的对象，因此这是独立的”。但其实不独立。这个*lambda*没有使用任何的non-`static`局部变量，所以它没有捕获任何东西。然而*lambda*的代码引用了`static`变量`divisor`，在每次调用`addDivisorFilter`的结尾，`divisor`都会递增，通过这个函数添加到`filters`的所有*lambda*都展示新的行为（分别对应新的`divisor`值）。这个*lambda*是通过引用捕获`divisor`，这和默认的按值捕获表示的含义有着直接的矛盾。如果你一开始就避免使用默认的按值捕获模式，你就能解除代码的风险。

**请记住：**

+ 默认的按引用捕获可能会导致悬空引用。
+ 默认的按值捕获对于悬空指针很敏感（尤其是`this`指针），并且它会误导人产生*lambda*是独立的想法。

## 三十二：使用初始化捕获来移动对象到闭包中

**Item 32: Use init capture to move objects into closures**

在某些场景下，按值捕获和按引用捕获都不是你所想要的。如果你有一个只能被移动的对象（例如`std::unique_ptr`或`std::future`）要进入到闭包里，使用C++11是无法实现的。如果你要复制的对象复制开销非常高，但移动的成本却不高（例如标准库中的大多数容器），并且你希望的是宁愿移动该对象到闭包而不是复制它。然而C++11却无法实现这一目标。

但那是C++11的时候。到了C++14就另一回事了，它能支持将对象移动到闭包中。如果你的编译器兼容支持C++14，那么请愉快地阅读下去。如果你仍然在使用仅支持C++11的编译器，也请愉快阅读，因为在C++11中有很多方法可以实现近似的移动捕获。

缺少移动捕获被认为是C++11的一个缺点，直接的补救措施是将该特性添加到C++14中，但标准化委员会选择了另一种方法。他们引入了一种新的捕获机制，该机制非常灵活，移动捕获是它可以执行的技术之一。新功能被称作**初始化捕获**（*init capture*），C++11捕获形式能做的所有事它几乎可以做，甚至能完成更多功能。你不能用初始化捕获表达的东西是默认捕获模式，但[Item31](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item31.md)说明提醒了你无论如何都应该远离默认捕获模式。（在C++11捕获模式所能覆盖的场景里，初始化捕获的语法有点不大方便。因此在C++11的捕获模式能完成所需功能的情况下，使用它是完全合理的）。

使用初始化捕获可以让你指定：

1. 从lambda生成的闭包类中的**数据成员名称**；
2. 初始化该成员的**表达式**；

这是使用初始化捕获将`std::unique_ptr`移动到闭包中的方法：

```c++
class Widget {                          //一些有用的类型
public:
    …
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
private:
    …
};

auto pw = std::make_unique<Widget>();   //创建Widget；使用std::make_unique
                                        //的有关信息参见21

…                                       //设置*pw

auto func = [pw = std::move(pw)]        //使用std::move(pw)初始化闭包数据成员
            { return pw->isValidated()
                     && pw->isArchived(); };
```

高亮的文本包含了初始化捕获的使用（译者注：高亮了“`pw = std::move(pw)`”），“`=`”的左侧是指定的闭包类中数据成员的名称，右侧则是初始化表达式。有趣的是，“`=`”左侧的作用域不同于右侧的作用域。左侧的作用域是闭包类，右侧的作用域和*lambda*定义所在的作用域相同。在上面的示例中，“`=`”左侧的名称`pw`表示闭包类中的数据成员，而右侧的名称`pw`表示在*lambda*上方声明的对象，即由调用`std::make_unique`去初始化的变量。因此，“`pw = std::move(pw)`”的意思是“在闭包中创建一个数据成员`pw`，并使用将`std::move`应用于局部变量`pw`的结果来初始化该数据成员”。

一般来说，*lambda*主体中的代码在闭包类的作用域内，因此`pw`的使用指的是闭包类的数据成员。

在此示例中，注释“设置`*pw`”表示在由`std::make_unique`创建`Widget`之后，*lambda*捕获到指向`Widget`的`std::unique_ptr`之前，该`Widget`以某种方式进行了修改。如果不需要这样的设置，即如果`std::make_unique`创建的`Widget`处于适合被*lambda*捕获的状态，则不需要局部变量`pw`，因为闭包类的数据成员可以通过`std::make_unique`直接初始化：

```c++
auto func = [pw = std::make_unique<Widget>()]   //使用调用make_unique得到的结果
            { return pw->isValidated()          //初始化闭包数据成员
                     && pw->isArchived(); };
```

这清楚地表明了，这个C++14的捕获概念是从C++11发展出来的的，在C++11中，无法捕获表达式的结果。 因此，初始化捕获的另一个名称是**通用*lambda*捕获**（*generalized lambda capture*）。

但是，如果你使用的一个或多个编译器不支持C++14的初始捕获怎么办？ 如何使用不支持移动捕获的语言完成移动捕获？

请记住，*lambda*表达式只是生成一个类和创建该类型对象的一种简单方式而已。没什么事是你用*lambda*可以做而不能自己手动实现的。 那么刚刚看到的C++14的示例代码可以用C++11重新编写，如下所示：

```c++
class IsValAndArch {                            //“is validated and archived”
public:
    using DataType = std::unique_ptr<Widget>;
    
    explicit IsValAndArch(DataType&& ptr)       //25解释了std::move的使用
    : pw(std::move(ptr)) {}
    
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }
    
private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```

这个代码量比*lambda*表达式要多，但这并不难改变这样一个事实，即如果你希望使用一个C++11的类来支持其数据成员的移动初始化，那么你唯一要做的就是在键盘上多花点时间。

如果你坚持要使用*lambda*（并且考虑到它们的便利性，你可能会这样做），移动捕获可以在C++11中这样模拟：

1. **将要捕获的对象移动到由`std::bind`产生的函数对象中；**
2. **将“被捕获的”对象的引用赋予给*lambda*。**

如果你熟悉`std::bind`，那么代码其实非常简单。如果你不熟悉`std::bind`，那可能需要花费一些时间来习惯它，但这无疑是值得的。

假设你要创建一个本地的`std::vector`，在其中放入一组适当的值，然后将其移动到闭包中。在C++14中，这很容易实现：

```c++
std::vector<double> data;               //要移动进闭包的对象

…                                       //填充data

auto func = [data = std::move(data)]    //C++14初始化捕获
            { /*使用data*/ };
```

我已经对该代码的关键部分进行了高亮：要移动的对象的类型（`std::vector<double>`），该对象的名称（`data`）以及用于初始化捕获的初始化表达式（`std::move(data)`）。C++11的等效代码如下，其中我强调了相同的关键事项：

```c++
std::vector<double> data;               //同上

…                                       //同上

auto func =
    std::bind(                              //C++11模拟初始化捕获
        [](const std::vector<double>& data) //译者注：本行高亮
        { /*使用data*/ },
        std::move(data)                     //译者注：本行高亮
    );
```

如*lambda*表达式一样，`std::bind`产生函数对象。我将由`std::bind`返回的函数对象称为**bind对象**（*bind objects*）。`std::bind`的第一个实参是可调用对象，后续实参表示要传递给该对象的值。

一个bind对象包含了传递给`std::bind`的所有实参的副本。对于每个左值实参，bind对象中的对应对象都是复制构造的。对于每个右值，它都是移动构造的。在此示例中，第二个实参是一个右值（`std::move`的结果，请参见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)），因此将`data`移动构造到绑定对象中。这种移动构造是模仿移动捕获的关键，因为将右值移动到bind对象是解决无法将右值移动到C++11闭包中的方法。

当“调用”bind对象（即调用其函数调用运算符）时，其存储的实参将传递到最初传递给`std::bind`的可调用对象。在此示例中，这意味着当调用`func`（bind对象）时，`func`中所移动构造的`data`副本将作为实参传递给`std::bind`中的*lambda*。

该*lambda*与在C++14中使用的*lambda*相同，只是添加了一个形参`data`来对应的伪移动捕获对象。此形参是对bind对象中`data`副本的左值引用。（这不是右值引用，因为尽管用于初始化`data`副本的表达式（`std::move(data)`）为右值，但`data`副本本身为左值。）因此，*lambda*将对绑定在对象内部的移动构造的`data`副本进行操作。

默认情况下，从*lambda*生成的闭包类中的`operator()`成员函数为`const`的。这具有在*lambda*主体内把闭包中的所有数据成员渲染为`const`的效果。但是，bind对象内部的移动构造的`data`副本不是`const`的，因此，为了防止在*lambda*内修改该`data`副本，*lambda*的形参应声明为reference-to-`const`。 如果将*lambda*声明为`mutable`，则闭包类中的`operator()`将不会声明为`const`，并且在*lambda*的形参声明中省略`const`也是合适的：

```c++
auto func =
    std::bind(                                  //C++11对mutable lambda
        [](std::vector<double>& data) mutable	//初始化捕获的模拟
        { /*使用data*/ },
        std::move(data)
    );
```

因为bind对象存储着传递给`std::bind`的所有实参的副本，所以在的示例中，bind对象包含由*lambda*生成的闭包副本，这是它的第一个实参。 因此闭包的生命周期与bind对象的生命周期相同。 这很重要，因为这意味着只要存在闭包，包含伪移动捕获对象的bind对象也将存在。

如果这是你第一次接触`std::bind`，则可能需要先阅读你最喜欢的C++11参考资料，然后再讨论所有详细信息。 即使是这样，这些基本要点也应该清楚：

* 无法移动构造一个对象到C++11闭包，但是可以将对象移动构造进C++11的bind对象。
* 在C++11中模拟移动捕获包括将对象移动构造进bind对象，然后通过传引用将移动构造的对象传递给*lambda*。
* 由于bind对象的生命周期与闭包对象的生命周期相同，因此可以将bind对象中的对象视为闭包中的对象。

作为使用`std::bind`模仿移动捕获的第二个示例，这是之前看到的在闭包中创建`std::unique_ptr`的C++14代码：

```c++
auto func = [pw = std::make_unique<Widget>()]   //同之前一样
            { return pw->isValidated()          //在闭包中创建pw
                     && pw->isArchived(); };
```

这是C++11的模拟实现：

```c++
auto func = std::bind(
                [](const std::unique_ptr<Widget>& pw)
                { return pw->isValidated()
                         && pw->isArchived(); },
                std::make_unique<Widget>()
            );
```

具备讽刺意味的是，这里我展示了如何使用`std::bind`解决C++11 *lambda*中的限制，因为在[Item34](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item34.md)中，我主张使用*lambda*而不是`std::bind`。但是，该解释的是在C++11中有些情况下`std::bind`可能有用，这就是其中一种。 （在C++14中，初始化捕获和`auto`形参等特性使得这些情况不再存在。）

**请记住：**

* 使用C++14的初始化捕获将对象移动到闭包中。
* 在C++11中，通过手写类或`std::bind`的方式来模拟初始化捕获。
## 三十三：对`auto&&`形参使用`decltype`以`std::forward`它们

**Item 33: Use `decltype` on `auto&&` parameters to `std::forward` them**

**泛型*lambda***（*generic lambdas*）是C++14中最值得期待的特性之一——因为在*lambda*的形参中可以使用`auto`关键字。这个特性的实现是非常直截了当的：即在闭包类中的`operator()`函数是一个函数模版。例如存在这么一个*lambda*，

```c++
auto f = [](auto x){ return func(normalize(x)); };
```

对应的闭包类中的函数调用操作符看来就变成这样：

```c++
class SomeCompilerGeneratedClassName {
public:
    template<typename T>                //auto返回类型见3
    auto operator()(T x) const
    { return func(normalize(x)); }
    …                                   //其他闭包类功能
};
```

在这个样例中，*lambda*对变量`x`做的唯一一件事就是把它转发给函数`normalize`。如果函数`normalize`对待左值右值的方式不一样，这个*lambda*的实现方式就不大合适了，因为即使传递到*lambda*的实参是一个右值，*lambda*传递进`normalize`的总是一个左值（形参`x`）。

实现这个*lambda*的正确方式是把`x`完美转发给函数`normalize`。这样做需要对代码做两处修改。首先，`x`需要改成通用引用（见[Item24](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item24.md)），其次，需要使用`std::forward`将`x`转发到函数`normalize`（见[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)）。理论上，这都是小改动：

```c++
auto f = [](auto&& x)
         { return func(normalize(std::forward<???>(x))); };
```

在理论和实际之间存在一个问题：你应该传递给`std::forward`的什么类型，即确定我在上面写的`???`该是什么。

一般来说，当你在使用完美转发时，你是在一个接受类型参数为`T`的模版函数里，所以你可以写`std::forward<T>`。但在泛型*lambda*中，没有可用的类型参数`T`。在*lambda*生成的闭包里，模版化的`operator()`函数中的确有一个`T`，但在*lambda*里却无法直接使用它，所以也没什么用。

[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)解释过如果一个左值实参被传给通用引用的形参，那么形参类型会变成左值引用。传递的是右值，形参就会变成右值引用。这意味着在这个*lambda*中，可以通过检查形参`x`的类型来确定传递进来的实参是一个左值还是右值，`decltype`就可以实现这样的效果（见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）。传递给*lambda*的是一个左值，`decltype(x)`就能产生一个左值引用；如果传递的是一个右值，`decltype(x)`就会产生右值引用。

[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)也解释过在调用`std::forward`时，惯例决定了类型实参是左值引用时来表明要传进左值，类型实参是非引用就表明要传进右值。在前面的*lambda*中，如果`x`绑定的是一个左值，`decltype(x)`就能产生一个左值引用。这符合惯例。然而如果`x`绑定的是一个右值，`decltype(x)`就会产生右值引用，而不是常规的非引用。

再看一下[Item28](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item28.md)中关于`std::forward`的C++14实现：

```c++
template<typename T>                        //在std命名空间
T&& forward(remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```

如果用户想要完美转发一个`Widget`类型的右值时，它会使用`Widget`类型（即非引用类型）来实例化`std::forward`，然后产生以下的函数：

```c++
Widget&& forward(Widget& param)             //当T是Widget时的std::forward实例
{
    return static_cast<Widget&&>(param);
}
```

思考一下如果用户代码想要完美转发一个`Widget`类型的右值，但没有遵守规则将`T`指定为非引用类型，而是将`T`指定为右值引用，这会发生什么。也就是，思考将`T`换成`Widget&&`会如何。在`std::forward`实例化、应用了`std::remove_reference_t`后，引用折叠之前，`std::forward`看起来像这样：

```c++
Widget&& && forward(Widget& param)          //当T是Widget&&时的std::forward实例
{                                           //（引用折叠之前）
    return static_cast<Widget&& &&>(param);
}
```

应用了引用折叠之后（右值引用的右值引用变成单个右值引用），代码会变成：

```c++
Widget&& forward(Widget& param)             //当T是Widget&&时的std::forward实例
{                                           //（引用折叠之后）
    return static_cast<Widget&&>(param);
}
```

对比这个实例和用`Widget`设置`T`去实例化产生的结果，它们完全相同。表明用右值引用类型和用非引用类型去初始化`std::forward`产生的相同的结果。

那是一个很好的消息，因为当传递给*lambda*形参`x`的是一个右值实参时，`decltype(x)`可以产生一个右值引用。前面已经确认过，把一个左值传给*lambda*时，`decltype(x)`会产生一个可以传给`std::forward`的常规类型。而现在也验证了对于右值，把`decltype(x)`产生的类型传递给`std::forward`是非传统的，不过它产生的实例化结果与传统类型相同。所以无论是左值还是右值，把`decltype(x)`传递给`std::forward`都能得到想要的结果，因此*lambda*的完美转发可以写成：

```c++
auto f =
    [](auto&& param)
    {
        return
            func(normalize(std::forward<decltype(param)>(param)));
    };
```

再加上6个点，就可以让的*lambda*完美转发接受多个形参了，因为C++14中的*lambda*也可以是可变形参的：

```c++
auto f =
    [](auto&&... params)
    {
        return
            func(normalize(std::forward<decltype(params)>(params)...));
    };
```

**请记住：**

+ 对`auto&&`形参使用`decltype`以`std::forward`它们。
## 三十四：考虑*lambda*而非`std::bind`

**Item 34: Prefer lambdas to `std::bind`**

C++11中的`std::bind`是C++98的`std::bind1st`和`std::bind2nd`的后续，但在2005年已经非正式成为了标准库的一部分。那时标准化委员采用了TR1的文档，其中包含了`bind`的规范。（在TR1中，`bind`位于不同的命名空间，因此它是`std::tr1::bind`，而不是`std::bind`，接口细节也有所不同）。这段历史意味着一些程序员有十年及以上的`std::bind`使用经验。如果你是其中之一，可能会不愿意放弃一个对你有用的工具。这是可以理解的，但是在这种情况下，改变是更好的，因为在C++11中，*lambda*几乎总是比`std::bind`更好的选择。 从C++14开始，*lambda*的作用不仅强大，而且是完全值得使用的。

这个假设你熟悉`std::bind`。 如果不是这样，你将需要获得基本的了解，然后再继续。 无论如何，这样的理解都是值得的，因为你永远不知道何时会在阅读或维护的代码库中遇到`std::bind`。

与[Item32](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item32.md)中一样，将从`std::bind`返回的函数对象称为**bind对象**（*bind objects*）。

优先*lambda*而不是`std::bind`的最重要原因是*lambda*更易读。 例如，假设有一个设置警报器的函数：

```c++
//一个时间点的类型定义（语法见9）
using Time = std::chrono::steady_clock::time_point;

//“enum class”见10
enum class Sound { Beep, Siren, Whistle };

//时间段的类型定义
using Duration = std::chrono::steady_clock::duration;

//在时间t，使用s声音响铃时长d
void setAlarm(Time t, Sound s, Duration d);
```

进一步假设，在程序的某个时刻，已经确定需要设置一个小时后响30秒的警报器。 但是，具体声音仍未确定。可以编写一个*lambda*来修改`setAlarm`的界面，以便仅需要指定声音：

```c++
//setSoundL（“L”指代“lambda”）是个函数对象，允许指定一小时后响30秒的警报器的声音
auto setSoundL =
    [](Sound s) 
    {
        //使std::chrono部件在不指定限定的情况下可用
        using namespace std::chrono;

        setAlarm(steady_clock::now() + hours(1),    //一小时后响30秒的闹钟
                 s,                                 //译注：setAlarm三行高亮
                 seconds(30));
    };
```

在*lambda*中高亮了对`setAlarm`的调用。这看来起是一个很正常的函数调用，即使是几乎没有*lambda*经验的读者也可以看到：传递给*lambda*的形参`s`又作为实参被传递给了`setAlarm`。

通过使用标准后缀如秒（`s`），毫秒（`ms`）和小时（`h`）等简化在C++14中的代码，其中标准后缀基于C++11对用户自定义常量的支持。这些后缀在`std::literals`命名空间中实现，因此上述代码可以按照以下方式重写：

```c++
auto setSoundL =
    [](Sound s)
    {
        using namespace std::chrono;
        using namespace std::literals;      //对于C++14后缀

        setAlarm(steady_clock::now() + 1h,	//C++14写法，但是含义同上
                 s,
                 30s);
    };
```

下面是第一次编写对应的`std::bind`调用。这里存在一个后续会修复的错误，但正确的代码会更加复杂，即使是此简化版本也会凸显一些重要问题：

```c++
using namespace std::chrono;                //同上
using namespace std::literals;
using namespace std::placeholders;          //“_1”使用需要

auto setSoundB =                            //“B”代表“bind”
    std::bind(setAlarm,
              steady_clock::now() + 1h,     //不正确！见下
              _1,
              30s);
```

我想像在之前的*lambda*中一样高亮对`setAlarm`的调用，但是没这么个调用让我高亮。这段代码的读者只需知道，调用`setSoundB`会使用在对`std::bind`的调用中所指定的时间和持续时间来调用`setAlarm`。对于门外汉来说，占位符“`_1`”完全是一个魔法，但即使是知情的读者也必须从思维上将占位符中的数字映射到其在`std::bind`形参列表中的位置，以便明白调用`setSoundB`时的第一个实参会被传递进`setAlarm`，作为调用`setAlarm`的第二个实参。在对`std::bind`的调用中未标识此实参的类型，因此读者必须查阅`setAlarm`声明以确定将哪种实参传递给`setSoundB`。

但正如我所说，代码并不完全正确。在*lambda*中，表达式`steady_clock::now() + 1h`显然是`setAlarm`的实参。调用`setAlarm`时将对其进行计算。可以理解：希望在调用`setAlarm`后一小时响铃。但是，在`std::bind`调用中，将`steady_clock::now() + 1h`作为实参传递给了`std::bind`，而不是`setAlarm`。这意味着将在调用`std::bind`时对表达式进行求值，并且该表达式产生的时间将存储在产生的bind对象中。结果，警报器将被设置为在**调用`std::bind`后一小时**发出声音，而不是在调用`setAlarm`一小时后发出。

要解决此问题，需要告诉`std::bind`推迟对表达式的求值，直到调用`setAlarm`为止，而这样做的方法是将对`std::bind`的第二个调用嵌套在第一个调用中：

```c++
auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<>(), std::bind(steady_clock::now), 1h),
              _1,
              30s);
```

如果你熟悉C++98的`std::plus`模板，你可能会惊讶地发现在此代码中，尖括号之间未指定任何类型，即该代码包含“`std::plus<>`”，而不是“`std::plus<type>`”。 在C++14中，通常可以省略标准运算符模板的模板类型实参，因此无需在此处提供。 C++11没有提供此类功能，因此等效于*lambda*的C++11 `std::bind`为：

```c++
using namespace std::chrono;                //同上
using namespace std::placeholders;
auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<steady_clock::time_point>(),
                        std::bind(steady_clock::now),
                        hours(1)),
              _1,
              seconds(30));
```

如果此时*lambda*看起来还没有吸引力，那么应该检查一下视力了。

当`setAlarm`重载时，会出现一个新问题。 假设有一个重载函数，其中第四个形参指定了音量：

```c++
enum class Volume { Normal, Loud, LoudPlusPlus };

void setAlarm(Time t, Sound s, Duration d, Volume v);
```

*lambda*能继续像以前一样使用，因为根据重载规则选择了`setAlarm`的三实参版本：

```c++
auto setSoundL =                            //和之前一样
    [](Sound s)
    {
        using namespace std::chrono;
        setAlarm(steady_clock::now() + 1h,  //可以，调用三实参版本的setAlarm
                 s,
                 30s);
    };
```

然而，`std::bind`的调用将会编译失败：

```c++
auto setSoundB =                            //错误！哪个setAlarm？
    std::bind(setAlarm,
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h),
              _1,
              30s);
```

这里的问题是，编译器无法确定应将两个`setAlarm`函数中的哪一个传递给`std::bind`。 它们仅有的是一个函数名称，而这个单一个函数名称是有歧义的。

要使对`std::bind`的调用能编译，必须将`setAlarm`强制转换为适当的函数指针类型：

```c++
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);

auto setSoundB =                                            //现在可以了
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm),
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h), 
              _1,
              30s);
```

但这在*lambda*和`std::bind`的使用上带来了另一个区别。 在`setSoundL`的函数调用操作符（即*lambda*的闭包类对应的函数调用操作符）内部，对`setAlarm`的调用是正常的函数调用，编译器可以按常规方式进行内联：

```c++
setSoundL(Sound::Siren);    //setAlarm函数体在这可以很好地内联
```

但是，对`std::bind`的调用是将函数指针传递给`setAlarm`，这意味着在`setSoundB`的函数调用操作符（即绑定对象的函数调用操作符）内部，对`setAlarm`的调用是通过一个函数指针。 编译器不太可能通过函数指针内联函数，这意味着与通过`setSoundL`进行调用相比，通过`setSoundB`对`setAlarm的`调用，其函数不大可能被内联：

```c++
setSoundB(Sound::Siren); 	//setAlarm函数体在这不太可能内联
```

因此，使用*lambda*可能会比使用`std::bind`能生成更快的代码。

`setAlarm`示例仅涉及一个简单的函数调用。如果你想做更复杂的事情，使用*lambda*会更有利。 例如，考虑以下C++14的*lambda*使用，它返回其实参是否在最小值（`lowVal`）和最大值（`highVal`）之间的结果，其中`lowVal`和`highVal`是局部变量：

```c++
auto betweenL =
    [lowVal, highVal]
    (const auto& val)                           //C++14
    { return lowVal <= val && val <= highVal; };
```

使用`std::bind`可以表达相同的内容，但是该构造是一个通过晦涩难懂的代码来保证工作安全性的示例：

```c++
using namespace std::placeholders;              //同上
auto betweenB =
    std::bind(std::logical_and<>(),             //C++14
              std::bind(std::less_equal<>(), lowVal, _1),
              std::bind(std::less_equal<>(), _1, highVal));
```

在C++11中，必须指定要比较的类型，然后`std::bind`调用将如下所示：

```c++
auto betweenB =
    std::bind(std::logical_and<bool>(),         //C++11版本
              std::bind(std::less_equal<int>(), lowVal, _1),
              std::bind(std::less_equal<int>(), _1, highVal));
```

当然，在C++11中，*lambda*也不能采用`auto`形参，因此它也必须指定一个类型：

```c++
auto betweenL =                                 //C++11版本
    [lowVal, highVal]
    (int val)
    { return lowVal <= val && val <= highVal; };
```

无论哪种方式，我希望都能同意，*lambda*版本不仅更短，而且更易于理解和维护。

之前我就说过，对于那些没有`std::bind`使用经验的人，其占位符（例如`_1`，`_2`等）都是魔法。 但是这不仅仅在于占位符的行为是不透明的。 假设有一个函数可以创建`Widget`的压缩副本，

```c++
enum class CompLevel { Low, Normal, High }; //压缩等级

Widget compress(const Widget& w,            //制作w的压缩副本
                CompLevel lev);
```

并且想创建一个函数对象，该函数对象允许指定`Widget w`的压缩级别。这种使用`std::bind`的话将创建一个这样的对象：

```c++
Widget w;
using namespace std::placeholders;
auto compressRateB = std::bind(compress, w, _1);
```

现在，当将`w`传递给`std::bind`时，必须将其存储起来，以便以后进行压缩。它存储在对象`compressRateB`中，但是它是如何被存储的呢——是通过值还是引用？之所以会有所不同，是因为如果在对`std::bind`的调用与对`compressRateB`的调用之间修改了`w`，则按引用捕获的`w`将反映这个更改，而按值捕获则不会。

答案是它是按值捕获的（`std::bind`总是拷贝它的实参，但是调用者可以使用引用来存储实参，这要通过应用`std::ref`到实参上实现。`auto compressRateB = std::bind(compress, std::ref(w), _1);`的结果就是`compressRateB`行为像是持有`w`的引用而非副本。），但唯一知道的方法是记住`std::bind`的工作方式；在对`std::bind`的调用中没有任何迹象。然而在*lambda*方法中，其中`w`是通过值还是通过引用捕获是显式的：

```c++
auto compressRateL =                //w是按值捕获，lev是按值传递
    [w](CompLevel lev)
    { return compress(w, lev); };
```

同样明确的是形参是如何传递给*lambda*的。 在这里，很明显形参`lev`是通过值传递的。 因此：

```c++
compressRateL(CompLevel::High);     //实参按值传递
```

但是在对由`std::bind`生成的对象调用中，实参如何传递？

```c++
compressRateB(CompLevel::High);     //实参如何传递？
```

同样，唯一的方法是记住`std::bind`的工作方式。（答案是传递给bind对象的所有实参都是通过引用传递的，因为此类对象的函数调用运算符使用完美转发。）

与*lambda*相比，使用`std::bind`进行编码的代码可读性较低，表达能力较低，并且效率可能较低。 在C++14中，没有`std::bind`的合理用例。 但是，在C++11中，可以在两个受约束的情况下证明使用`std::bind`是合理的：

+ **移动捕获**。C++11的*lambda*不提供移动捕获，但是可以通过结合*lambda*和`std::bind`来模拟。 有关详细信息，请参阅[Item32](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/6.LambdaExpressions/item32.md)，该还解释了在C++14中，*lambda*对初始化捕获的支持消除了这个模拟的需求。
+ **多态函数对象**。因为bind对象上的函数调用运算符使用完美转发，所以它可以接受任何类型的实参（以[Item30](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item30.md)中描述的完美转发的限制为界限）。当你要绑定带有模板化函数调用运算符的对象时，此功能很有用。 例如这个类，

```c++
class PolyWidget {
public:
    template<typename T>
    void operator()(const T& param);
    …
};
```

`std::bind`可以如下绑定一个`PolyWidget`对象：

```c++
PolyWidget pw;
auto boundPW = std::bind(pw, _1);
```

`boundPW`可以接受任意类型的对象了：

```c++
boundPW(1930);              //传int给PolyWidget::operator()
boundPW(nullptr);           //传nullptr给PolyWidget::operator()
boundPW("Rosebud"); 		//传字面值给PolyWidget::operator()
```

这一点无法使用C++11的*lambda*做到。 但是，在C++14中，可以通过带有`auto`形参的*lambda*轻松实现：

```c++
auto boundPW = [pw](const auto& param)  //C++14 
               { pw(param); };
```

当然，这些是特殊情况，并且是暂时的特殊情况，因为支持C++14 *lambda*的编译器越来越普遍了。

当`bind`在2005年被非正式地添加到C++中时，与1998年的前身相比有了很大的改进。 在C++11中增加了*lambda*支持，这使得`std::bind`几乎已经过时了，从C++14开始，更是没有很好的用例了。

**请记住：**

+ 与使用`std::bind`相比，*lambda*更易读，更具表达力并且可能更高效。
+ 只有在C++11中，`std::bind`可能对实现移动捕获或绑定带有模板化函数调用运算符的对象时会很有用。

