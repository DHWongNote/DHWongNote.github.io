---
title: 【Program】MovingToModernCpp《Effective Modern C++》
author: DH Wang
date: 1000-01-01
category: Program
layout: post
---
 

  


# 第3章 移步现代C++

**CHAPTER 3 Moving to Modern C++**

说起知名的特性，C++11/14有一大堆可以吹的东西，`auto`，智能指针（*smart pointer*），移动语义（*move semantics*），*lambda*，并发（*concurrency*）——每个都是如此的重要，这章将覆盖这些内容。精通这些特性是必要的，但是成为高效率的现代C++程序员也要求一系列小步骤。从C++98移步现代C++遇到的每个细节问题都会在本章得到答复。你什么时候应该用{}而不是()创建对象？为什么别名（*alias*）声明比`typedef`好？`constexpr`和`const`有什么不同？常量（`const`）成员函数和线程安全有什么关系？这个列表越列越多。这章将会逐个回答这些问题。

## 七：区别使用`()`和`{}`创建对象

**Item 7: Distinguish between `()` and `{}` when creating objects**

从你的角度看，C++11初始化对象的语法选择既丰富得让人尴尬又混乱得让人糊涂。一般来说，初始化值要用()或者{}括起来或者放到等号"="的右边：
````cpp
int x(0);               //使用小括号初始化

int y = 0;              //使用"="初始化

int z{ 0 };             //使用花括号初始化
````
在很多情况下，你可以使用"="和花括号的组合：
````cpp
int z = { 0 };          //使用"="和花括号
````
在这个的剩下部分，我通常会忽略"="和花括号组合初始化的语法，因为C++通常把它视作和只有花括号一样。

“混乱得令人糊涂”指出在初始化中使用"="可能会误导C++新手，使他们以为这里发生了赋值运算。对于像`int`这样的内置类型，研究两者区别是个学术问题，但是对于用户定义的类型而言，区别赋值运算符和初始化就非常重要了，因为这可能包含不同的函数调用：

````cpp
Widget w1;              //调用默认构造函数

Widget w2 = w1;         //不是赋值运算，调用拷贝构造函数

w1 = w2;                //是赋值运算，调用拷贝赋值运算符（copy operator=）
````
甚至对于一些初始化语法，在一些情况下C++98没有办法去表达初始化。举个例子，要想直接表示一个存放一些特殊值的STL容器是不可能的（比如1,3,5）。

C++11使用统一初始化（*uniform initialization*）来整合这些混乱且不适于所有情景的初始化语法，所谓统一初始化是指使用单一初始化语法在任何地方（译注：结合上下文得知这里的“任何地方”指的是初始化表达式存在的地方而不是广义上源代码的各处）表达任何东西。
它基于花括号，出于这个原因我更喜欢称之为括号初始化。（译注：注意，这里的括号初始化指的是花括号初始化，在没有歧义的情况下下文的括号初始化指的都是用花括号进行初始化；当与小括号初始化同时存在并可能产生歧义时我会直接指出。）统一初始化是一个概念上的东西，而括号初始化是一个具体语法构型。

括号初始化让你可以表达以前表达不出的东西。使用花括号，指定一个容器的元素变得很容易：

````cpp
std::vector<int> v{ 1, 3, 5 };  //v初始内容为1,3,5
````
括号初始化也能被用于为非静态数据成员指定默认初始值。C++11允许"="初始化不加花括号也拥有这种能力：
````cpp
class Widget{
    …

private:
    int x{ 0 };                 //没问题，x初始值为0
    int y = 0;                  //也可以
    int z(0);                   //错误！
}
````
另一方面，不可拷贝的对象（例如`std::atomic`——见[Item40](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item40.md)）可以使用花括号初始化或者小括号初始化，但是不能使用"="初始化：
````cpp
std::atomic<int> ai1{ 0 };      //没问题
std::atomic<int> ai2(0);        //没问题
std::atomic<int> ai3 = 0;       //错误！
````
因此很容易理解为什么括号初始化又叫统一初始化，在C++中这三种方式都被指派为初始化表达式，但是只有括号任何地方都能被使用。

括号表达式有一个异常的特性，它不允许内置类型间隐式的变窄转换（*narrowing conversion*）。如果一个使用了括号初始化的表达式的值，不能保证由被初始化的对象的类型来表示，代码就不会通过编译：
````cpp
double x, y, z;

int sum1{ x + y + z };          //错误！double的和可能不能表示为int
````
使用小括号和"="的初始化不检查是否转换为变窄转换，因为由于历史遗留问题它们必须要兼容老旧代码：
````cpp
int sum2(x + y +z);             //可以（表达式的值被截为int）

int sum3 = x + y + z;           //同上
````
另一个值得注意的特性是括号表达式对于C++最令人头疼的解析问题有天生的免疫性。（译注：所谓最令人头疼的解析即*most vexing parse*，更多信息请参见[https://en.wikipedia.org/wiki/Most_vexing_parse](https://en.wikipedia.org/wiki/Most_vexing_parse)。）C++规定任何能被决议为一个声明的东西必须被决议为声明。这个规则的副作用是让很多程序员备受折磨：当他们想创建一个使用默认构造函数构造的对象，却不小心变成了函数声明。问题的根源是如果你想使用一个实参调用一个构造函数，你可以这样做：

````cpp
Widget w1(10);                  //使用实参10调用Widget的一个构造函数
````
但是如果你尝试使用相似的语法调用没有参数的`Widget`构造函数，它就会变成函数声明：
````cpp
Widget w2();                    //最令人头疼的解析！声明一个函数w2，返回Widget
````
由于函数声明中形参列表不能使用花括号，所以使用花括号初始化表明你想调用默认构造函数构造对象就没有问题：
````cpp
Widget w3{};                    //调用没有参数的构造函数构造对象
````
关于括号初始化还有很多要说的。它的语法能用于各种不同的上下文，它防止了隐式的变窄转换，而且对于C++最令人头疼的解析也天生免疫。既然好到这个程度那为什么这个不叫“Prefer braced initialization syntax”呢？

括号初始化的缺点是有时它有一些令人惊讶的行为。这些行为使得括号初始化、`std::initializer_list`和构造函数重载决议本来就不清不楚的暧昧关系进一步混乱。把它们放到一起会让看起来应该左转的代码右转。举个例子，[Item2](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item2.md)解释了当`auto`声明的变量使用花括号初始化，变量类型就会被推导为`std::initializer_list`，尽管使用相同内容的其他初始化方式会产生正常的结果。所以，你越喜欢用`auto`，你就越不能用括号初始化。

在构造函数调用中，只要不包含`std::initializer_list`形参，那么花括号初始化和小括号初始化都会产生一样的结果：
````cpp
class Widget { 
public:  
    Widget(int i, bool b);      //构造函数未声明
    Widget(int i, double d);    //std::initializer_list形参 
    …
};
Widget w1(10, true);            //调用第一个构造函数
Widget w2{10, true};            //也调用第一个构造函数
Widget w3(10, 5.0);             //调用第二个构造函数
Widget w4{10, 5.0};             //也调用第二个构造函数
````
然而，如果有一个或者多个构造函数的声明一个`std::initializer_list`形参，使用括号初始化语法的调用更倾向于适用`std::initializer_list`重载函数。而且只要某个使用括号表达式的调用能适用接受`std::initializer_list`的构造函数，编译器就会使用它。如果上面的`Widget`类有一个`std::initializer_list<long double>`构造函数并被传入实参，就像这样：
````cpp
class Widget { 
public:  
    Widget(int i, bool b);      //同上
    Widget(int i, double d);    //同上
    Widget(std::initializer_list<long double> il);      //新添加的
    …
}; 
````
`w2`和`w4`将会使用新添加的构造函数构造，即使另一个非`std::initializer_list`构造函数对于实参是更好的选择：

````cpp
Widget w1(10, true);    //使用小括号初始化，同之前一样
                        //调用第一个构造函数

Widget w2{10, true};    //使用花括号初始化，但是现在
                        //调用std::initializer_list版本构造函数
                        //(10 和 true 转化为long double)

Widget w3(10, 5.0);     //使用小括号初始化，同之前一样
                        //调用第二个构造函数 

Widget w4{10, 5.0};     //使用花括号初始化，但是现在
                        //调用std::initializer_list版本构造函数
                        //(10 和 5.0 转化为long double)
````
甚至普通的构造函数和移动构造函数都会被`std::initializer_list`构造函数劫持：
````cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    Widget(std::initializer_list<long double> il);      //同之前一样
    operator float() const;                             //转换为float
    …
};

Widget w5(w4);                  //使用小括号，调用拷贝构造函数

Widget w6{w4};                  //使用花括号，调用std::initializer_list构造
                                //函数（w4转换为float，float转换为double）

Widget w7(std::move(w4));       //使用小括号，调用移动构造函数

Widget w8{std::move(w4)};       //使用花括号，调用std::initializer_list构造
                                //函数（与w6相同原因）
````
编译器热衷于把括号初始化与使`std::initializer_list`构造函数匹配了，尽管最佳匹配`std::initializer_list`构造函数不能被调用也会凑上去。比如：
````cpp
class Widget { 
public: 
    Widget(int i, bool b);                      //同之前一样
    Widget(int i, double d);                    //同之前一样
    Widget(std::initializer_list<bool> il);     //现在元素类型为bool
    …                                           //没有隐式转换函数
};

Widget w{10, 5.0};              //错误！要求变窄转换
````
这里，编译器会直接忽略前面两个构造函数（其中第二个提供了所有实参类型的最佳匹配），然后尝试调用`std::initializer_list<bool>`构造函数。调用这个函数将会把`int(10)`和`double(5.0)`转换为`bool`，由于会产生变窄转换（`bool`不能准确表示其中任何一个值），括号初始化拒绝变窄转换，所以这个调用无效，代码无法通过编译。

只有当没办法把括号初始化中实参的类型转化为`std::initializer_list`时，编译器才会回到正常的函数决议流程中。比如在构造函数中用`std::initializer_list<std::string>`代替`std::initializer_list<bool>`，这时非`std::initializer_list`构造函数将再次成为函数决议的候选者，因为没有办法把`int`和`bool`转换为`std::string`:

````cpp
class Widget { 
public:  
    Widget(int i, bool b);                              //同之前一样
    Widget(int i, double d);                            //同之前一样
    //现在std::initializer_list元素类型为std::string
    Widget(std::initializer_list<std::string> il);
    …                                                   //没有隐式转换函数
};

Widget w1(10, true);     // 使用小括号初始化，调用第一个构造函数
Widget w2{10, true};     // 使用花括号初始化，现在调用第一个构造函数
Widget w3(10, 5.0);      // 使用小括号初始化，调用第二个构造函数
Widget w4{10, 5.0};      // 使用花括号初始化，现在调用第二个构造函数
````
代码的行为和刚刚的论述如出一辙。这里还有一个有趣的[edge case](https://en.wikipedia.org/wiki/Edge_case)。假如你使用的花括号初始化是空集，并且你欲构建的对象有默认构造函数，也有`std::initializer_list`构造函数。你的空的花括号意味着什么？如果它们意味着没有实参，就该使用默认构造函数，但如果它意味着一个空的`std::initializer_list`，就该调用`std::initializer_list`构造函数。

最终会调用默认构造函数。空的花括号意味着没有实参，不是一个空的`std::initializer_list`：
````cpp
class Widget { 
public:  
    Widget();                                   //默认构造函数
    Widget(std::initializer_list<int> il);      //std::initializer_list构造函数

    …                                           //没有隐式转换函数
};

Widget w1;                      //调用默认构造函数
Widget w2{};                    //也调用默认构造函数
Widget w3();                    //最令人头疼的解析！声明一个函数
````
如果你**想**用空`std::initializer`来调用`std::initializer_list`构造函数，你就得创建一个空花括号作为函数实参——通过把空花括号放在小括号或者另一花括号内来界定你想传递的东西。
````cpp
Widget w4({});                  //使用空花括号列表调用std::initializer_list构造函数
Widget w5{{}};                  //同上
````
此时，括号初始化，`std::initializer_list`和构造函数重载的晦涩规则就会一下子涌进你的脑袋，你可能会想研究了半天这些东西在你的日常编程中到底占多大比例。可能比你想象的要多。因为`std::vector`作为其中之一会直接受到影响。`std::vector`有一个非`std::initializer_list`构造函数允许你去指定容器的初始大小，以及使用一个值填满你的容器。但它也有一个`std::initializer_list`构造函数允许你使用花括号里面的值初始化容器。如果你创建一个数值类型的`std::vector`（比如`std::vector<int>`），然后你传递两个实参，把这两个实参放到小括号和放到花括号中大不相同：

````cpp
std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                //创建一个包含10个元素的std::vector，
                                //所有的元素的值都是20
std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                //创建包含两个元素的std::vector，
                                //元素的值为10和20
````
让退回之前的讨论。从这个讨论中我有两个重要结论。第一，作为一个类库作者，你需要意识到如果你的一堆重载的构造函数中有一个或者多个含有`std::initializer_list`形参，用户代码如果使用了括号初始化，可能只会看到你`std::initializer_list`版本的重载的构造函数。因此，你最好把你的构造函数设计为不管用户是小括号还是使用花括号进行初始化都不会有什么影响。换句话说，现在看到`std::vector`设计的缺点以后你设计的时候避免它。

这里的暗语是如果一个类没有`std::initializer_list`构造函数，然后你添加一个，用户代码中如果使用括号初始化，可能会发现过去被决议为非`std::initializer_list`构造函数而现在被决议为新的函数。当然，这种事情也可能发生在你添加一个函数到那堆重载函数的时候：过去被决议为旧的重载函数而现在调用了新的函数。`std::initializer_list`重载不会和其他重载函数比较，它直接盖过了其它重载函数，其它重载函数几乎不会被考虑。所以如果你要加入`std::initializer_list`构造函数，请三思而后行。

第二个，作为一个类库使用者，你必须认真的在花括号和小括号之间选择一个来创建对象。大多数开发者都使用其中一种作为默认情况，只有当他们不能使用这种的时候才会考虑另一种。如果使用默认使用花括号初始化，会得到大范围适用面的好处，它禁止变窄转换，免疫C++最令人头疼的解析。他们知道在一些情况下（比如给一个容器大小和一个值创建`std::vector`）要使用小括号。如果默认使用小括号初始化，它们能和C++98语法保持一致，它避开了`auto`自动推导`std::initializer_list`的问题，也不会不经意间就调用了`std::initializer_list`构造函数。他们承认有时候只能使用花括号（比如创建一个包含着特定值的容器）。关于花括号和小括号的使用没有一个一致的观点，所以我的建议是用一个，并坚持使用。

如果你是一个模板的作者，花括号和小括号创建对象就更麻烦了。通常不能知晓哪个会被使用。举个例子，假如你想创建一个接受任意数量的参数，然后用它们创建一个对象。使用可变参数模板（*variadic template*）可以非常简单的解决：

````cpp
template<typename T,            //要创建的对象类型
         typename... Ts>        //要使用的实参的类型
void doSomeWork(Ts&&... params)
{
    create local T object from params...
    …
} 
````
在现实中有两种方式实现这个伪代码（关于`std::forward`请参见[Item25](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item25.md)）：
````cpp
T localObject(std::forward<Ts>(params)...);             //使用小括号
T localObject{std::forward<Ts>(params)...};             //使用花括号
````
考虑这样的调用代码：
````cpp
std::vector<int> v; 
…
doSomeWork<std::vector<int>>(10, 20);
````
如果`doSomeWork`创建`localObject`时使用的是小括号，`std::vector`就会包含10个元素。如果`doSomeWork`创建`localObject`时使用的是花括号，`std::vector`就会包含2个元素。哪个是正确的？`doSomeWork`的作者不知道，只有调用者知道。

这正是标准库函数`std::make_unique`和`std::make_shared`（参见[Item21](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item21.md)）面对的问题。它们的解决方案是使用小括号，并被记录在文档中作为接口的一部分。（注：更灵活的设计——允许调用者决定从模板来的函数应该使用小括号还是花括号——是有可能的。详情参见[Andrzej’s C++ blog](http://akrzemi1.wordpress.com/)在2013年6月5日的文章，“[Intuitive interface — Part I.](http://akrzemi1.wordpress.com/2013/06/05/intuitive-interface-part-i/)”）

**请记住：**

+ 括号初始化是最广泛使用的初始化语法，它防止变窄转换，并且对于C++最令人头疼的解析有天生的免疫性
+ 在构造函数重载决议中，括号初始化尽最大可能与`std::initializer_list`参数匹配，即便其他构造函数看起来是更好的选择
+ 对于数值类型的`std::vector`来说使用花括号初始化和小括号初始化会造成巨大的不同
+ 在模板类选择使用小括号初始化或使用花括号初始化创建对象是一个挑战。

## 八：优先考虑`nullptr`而非`0`和`NULL`

**Item 8: Prefer `nullptr` to `0` and `NULL`**

你看这样对不对：字面值`0`是一个`int`不是指针。如果C++发现在当前上下文只能使用指针，它会很不情愿的把`0`解释为指针，但是那是最后的退路。一般来说C++的解析策略是把`0`看做`int`而不是指针。

实际上，`NULL`也是这样的。但在`NULL`的实现细节有些不确定因素，因为实现被允许给`NULL`一个除了`int`之外的整型类型（比如`long`）。这不常见，但也算不上问题所在。这里的问题不是`NULL`没有一个确定的类型，而是`0`和`NULL`都不是指针类型。

在C++98中，对指针类型和整型进行重载意味着可能导致奇怪的事情。如果给下面的重载函数传递`0`或`NULL`，它们绝不会调用指针版本的重载函数：
````cpp
void f(int);        //三个f的重载函数
void f(bool);
void f(void*);

f(0);               //调用f(int)而不是f(void*)

f(NULL);            //可能不会被编译，一般来说调用f(int)，
                    //绝对不会调用f(void*)
````
而`f(NULL)`的不确定行为是由`NULL`的实现不同造成的。如果`NULL`被定义为`0L`（指的是`0`为`long`类型），这个调用就具有二义性，因为从`long`到`int`的转换或从`long`到`bool`的转换或`0L`到`void*`的转换都同样好。有趣的是源代码**表现出**的意思（“我使用空指针`NULL`调用`f`”）和**实际表达出**的意思（“我是用整型数据而不是空指针调用`f`”）是相矛盾的。这种违反直觉的行为导致C++98程序员都将避开同时重载指针和整型作为编程准则（译注：请务必注意结合上下文使用这条规则）。在C++11中这个编程准则也有效，因为尽管我这个建议使用`nullptr`，可能很多程序员还是会继续使用`0`或`NULL`，哪怕`nullptr`是更好的选择。

`nullptr`的优点是它不是整型。老实说它也不是一个指针类型，但是你可以把它认为是**所有**类型的指针。`nullptr`的真正类型是`std::nullptr_t`，在一个完美的循环定义以后，`std::nullptr_t`又被定义为`nullptr`。`std::nullptr_t`可以隐式转换为指向任何内置类型的指针，这也是为什么`nullptr`表现得像所有类型的指针。

使用`nullptr`调用`f`将会调用`void*`版本的重载函数，因为`nullptr`不能被视作任何整型：
````cpp
f(nullptr);         //调用重载函数f的f(void*)版本
````
使用`nullptr`代替`0`和`NULL`可以避开了那些令人奇怪的函数重载决议，这不是它的唯一优势。它也可以使代码表意明确，尤其是当涉及到与`auto`声明的变量一起使用时。举个例子，假如你在一个代码库中遇到了这样的代码：
````cpp
auto result = findRecord( /* arguments */ );
if (result == 0) {
    …
} 
````
如果你不知道`findRecord`返回了什么（或者不能轻易的找出），那么你就不太清楚到底`result`是一个指针类型还是一个整型。毕竟，`0`（用来测试`result`的值的那个）也可以像之前讨论的那样被解析。但是换一种假设如果你看到这样的代码：
````cpp
auto result = findRecord( /* arguments */ );

if (result == nullptr) {  
    …
}
````
这就没有任何歧义：`result`的结果一定是指针类型。

当模板出现时`nullptr`就更有用了。假如你有一些函数只能被合适的已锁互斥量调用。每个函数都有一个不同类型的指针：
````cpp
int    f1(std::shared_ptr<Widget> spw);     //只能被合适的
double f2(std::unique_ptr<Widget> upw);     //已锁互斥量
bool   f3(Widget* pw);                      //调用
````
如果这样传递空指针：
````cpp
std::mutex f1m, f2m, f3m;       //用于f1，f2，f3函数的互斥量

using MuxGuard =                //C++11的typedef，参见Item9
    std::lock_guard<std::mutex>;
…

{  
    MuxGuard g(f1m);            //为f1m上锁
    auto result = f1(0);        //向f1传递0作为空指针
}                               //解锁 
…
{  
    MuxGuard g(f2m);            //为f2m上锁
    auto result = f2(NULL);     //向f2传递NULL作为空指针
}                               //解锁 
…
{
    MuxGuard g(f3m);            //为f3m上锁
    auto result = f3(nullptr);  //向f3传递nullptr作为空指针
}                               //解锁 
````
令人遗憾前两个调用没有使用`nullptr`，但是代码可以正常运行，这也许对一些东西有用。但是重复的调用代码——为互斥量上锁，调用函数，解锁互斥量——更令人遗憾。它让人很烦。模板就是被设计于减少重复代码，所以让模板化这个调用流程：
````cpp
template<typename FuncType,
         typename MuxType,
         typename PtrType>
auto lockAndCall(FuncType func,                 
                 MuxType& mutex,                 
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);  
    return func(ptr); 
}
````
如果你对函数返回类型（`auto ... -> decltype(func(ptr))`）感到困惑不解，[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)可以帮助你。在C++14中代码的返回类型还可以被简化为`decltype(auto)`：

````cpp
template<typename FuncType,
         typename MuxType,
         typename PtrType>
decltype(auto) lockAndCall(FuncType func,       //C++14
                           MuxType& mutex,
                           PtrType ptr)
{ 
    MuxGuard g(mutex);  
    return func(ptr); 
}
````
可以写这样的代码调用`lockAndCall`模板（两个版本都可）：
````cpp
auto result1 = lockAndCall(f1, f1m, 0);         //错误！
...
auto result2 = lockAndCall(f2, f2m, NULL);      //错误！
...
auto result3 = lockAndCall(f3, f3m, nullptr);   //没问题
````
代码虽然可以这样写，但是就像注释中说的，前两个情况不能通过编译。在第一个调用中存在的问题是当`0`被传递给`lockAndCall`模板，模板类型推导会尝试去推导实参类型，`0`的类型总是`int`，所以这就是这次调用`lockAndCall`实例化出的`ptr`的类型。不幸的是，这意味着`lockAndCall`中`func`会被`int`类型的实参调用，这与`f1`期待的`std::shared_ptr<Widget>`形参不符。传递`0`给`lockAndCall`本来想表示空指针，结果`f1`得到的是和它相差十万八千里的`int`。把`int`类型看做`std::shared_ptr<Widget>`类型给`f1`自然是一个类型错误。在模板`lockAndCall`中使用`0`之所以失败是因为在模板中，传给的是`int`但实际上函数期待的是一个`std::shared_ptr<Widget>`。

第二个使用`NULL`调用的分析也是一样的。当`NULL`被传递给`lockAndCall`，形参`ptr`被推导为整型（译注：由于依赖于具体实现所以不一定是整数类型，所以用整型泛指`int`，`long`等类型），然后当`ptr`——一个`int`或者类似`int`的类型——传递给`f2`的时候就会出现类型错误，`f2`期待的是`std::unique_ptr<Widget>`。

然而，使用`nullptr`是调用没什么问题。当`nullptr`传给`lockAndCall`时，`ptr`被推导为`std::nullptr_t`。当`ptr`被传递给`f3`的时候，隐式转换使`std::nullptr_t`转换为`Widget`，因为`std::nullptr_t`可以隐式转换为任何指针类型。


模板类型推导将`0`和`NULL`推导为一个错误的类型（即它们的实际类型，而不是作为空指针的隐含意义），这就导致在当你想要一个空指针时，它们的替代品`nullptr`很吸引人。使用`nullptr`，模板不会有什么特殊的转换。另外，使用`nullptr`不会让你受到同重载决议特殊对待`0`和`NULL`一样的待遇。当你想用一个空指针，使用`nullptr`，不用`0`或者`NULL`。

**记住**
+ 优先考虑`nullptr`而非`0`和`NULL`
+ 避免重载指针和整型

## 九：优先考虑别名声明而非`typedef`s

**Item 9: Prefer alias declarations to `typedef`s**

我相信每个人都同意使用STL容器是个好主意，并且我希望[Item18](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)能说服你让你觉得使用`std:unique_ptr`也是个好主意，但我猜没有人喜欢写上几次 `std::unique_ptr<std::unordered_map<std::string, std::string>>`这样的类型，它可能会让你患上腕管综合征的风险大大增加。

避免上述医疗悲剧也很简单，引入`typedef`即可：
````cpp
typedef
    std::unique_ptr<std::unordered_map<std::string, std::string>>
    UPtrMapSS; 
````
但`typedef`是C++98的东西。虽然它可以在C++11中工作，但是C++11也提供了一个别名声明（*alias declaration*）：
````cpp
using UPtrMapSS =
    std::unique_ptr<std::unordered_map<std::string, std::string>>;
````
由于这里给出的`typedef`和别名声明做的都是完全一样的事情，有理由想知道会不会出于一些技术上的原因两者有一个更好。

这里，在说它们之前我想提醒一下很多人都发现当声明一个函数指针时别名声明更容易理解：
````cpp
//FP是一个指向函数的指针的同义词，它指向的函数带有
//int和const std::string&形参，不返回任何东西
typedef void (*FP)(int, const std::string&);    //typedef

//含义同上
using FP = void (*)(int, const std::string&);   //别名声明
````

当然，两个结构都不是非常让人满意，没有人喜欢花大量的时间处理函数指针类型的别名（译注：指`FP`），所以至少在这里，没有一个吸引人的理由让你觉得别名声明比`typedef`好。

不过有一个地方使用别名声明吸引人的理由是存在的：模板。特别地，别名声明可以被模板化（这种情况下称为别名模板*alias template*s）但是`typedef`不能。这使得C++11程序员可以很直接的表达一些C++98中只能把`typedef`嵌套进模板化的`struct`才能表达的东西。考虑一个链表的别名，链表使用自定义的内存分配器，`MyAlloc`。使用别名模板，这真是太容易了：

````cpp
template<typename T>                            //MyAllocList<T>是
using MyAllocList = std::list<T, MyAlloc<T>>;   //std::list<T, MyAlloc<T>>
                                                //的同义词

MyAllocList<Widget> lw;                         //用户代码
````
使用`typedef`，你就只能从头开始：
````cpp
template<typename T>                            //MyAllocList<T>是
struct MyAllocList {                            //std::list<T, MyAlloc<T>>
    typedef std::list<T, MyAlloc<T>> type;      //的同义词  
};

MyAllocList<Widget>::type lw;                   //用户代码
````
更糟糕的是，如果你想使用在一个模板内使用`typedef`声明一个链表对象，而这个对象又使用了模板形参，你就不得不在`typedef`前面加上`typename`：
````cpp
template<typename T>
class Widget {                              //Widget<T>含有一个
private:                                    //MyAllocLIst<T>对象
    typename MyAllocList<T>::type list;     //作为数据成员
    …
}; 
````
这里`MyAllocList<T>::type`使用了一个类型，这个类型依赖于模板参数`T`。因此`MyAllocList<T>::type`是一个依赖类型（*dependent type*），在C++很多讨人喜欢的规则中的一个提到必须要在依赖类型名前加上`typename`。

如果使用别名声明定义一个`MyAllocList`，就不需要使用`typename`（同时省略麻烦的“`::type`”后缀）：

````cpp
template<typename T> 
using MyAllocList = std::list<T, MyAlloc<T>>;   //同之前一样

template<typename T> 
class Widget {
private:
    MyAllocList<T> list;                        //没有“typename”
    …                                           //没有“::type”
};
````
对你来说，`MyAllocList<T>`（使用了模板别名声明的版本）可能看起来和`MyAllocList<T>::type`（使用`typedef`的版本）一样都应该依赖模板参数`T`，但是你不是编译器。当编译器处理`Widget`模板时遇到`MyAllocList<T>`（使用模板别名声明的版本），它们知道`MyAllocList<T>`是一个类型名，因为`MyAllocList`是一个别名模板：它**一定**是一个类型名。因此`MyAllocList<T>`就是一个**非依赖类型**（*non-dependent type*），就不需要也不允许使用`typename`修饰符。

当编译器在`Widget`的模板中看到`MyAllocList<T>::type`（使用`typedef`的版本），它不能确定那是一个类型的名称。因为可能存在一个`MyAllocList`的它们没见到的特化版本，那个版本的`MyAllocList<T>::type`指代了一种不是类型的东西。那听起来很不可思议，但不要责备编译器穷尽考虑所有可能。因为人确实能写出这样的代码。

举个例子，一个误入歧途的人可能写出这样的代码：

````cpp
class Wine { … };

template<>                          //当T是Wine
class MyAllocList<Wine> {           //特化MyAllocList
private:  
    enum class WineType             //参见Item10了解  
    { White, Red, Rose };           //"enum class"

    WineType type;                  //在这个类中，type是
    …                               //一个数据成员！
};
````
就像你看到的，`MyAllocList<Wine>::type`不是一个类型。如果`Widget`使用`Wine`实例化，在`Widget`模板中的`MyAllocList<Wine>::type`将会是一个数据成员，不是一个类型。在`Widget`模板内，`MyAllocList<T>::type`是否表示一个类型取决于`T`是什么，这就是为什么编译器会坚持要求你在前面加上`typename`。

如果你尝试过模板元编程（*template metaprogramming*，TMP）， 你一定会碰到取模板类型参数然后基于它创建另一种类型的情况。举个例子，给一个类型`T`，如果你想去掉`T`的常量修饰和引用修饰（`const`- or reference qualifiers），比如你想把`const std::string&`变成`std::string`。又或者你想给一个类型加上`const`或变为左值引用，比如把`Widget`变成`const Widget`或`Widget&`。（如果你没有用过模板元编程，太遗憾了，因为如果你真的想成为一个高效C++程序员，你需要至少熟悉C++在这方面的基本知识。你可以看看在[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)，[27](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item27.md)里的TMP的应用实例，包括我提到的类型转换）。

C++11在*type traits*（类型特性）中给了你一系列工具去实现类型转换，如果要使用这些模板请包含头文件`<type_traits>`。里面有许许多多*type traits*，也不全是类型转换的工具，也包含一些可预测接口的工具。给一个你想施加转换的类型`T`，结果类型就是`std::`transformation`<T>::type`，比如：

````cpp
std::remove_const<T>::type          //从const T中产出T
std::remove_reference<T>::type      //从T&和T&&中产出T
std::add_lvalue_reference<T>::type  //从T中产出T&
````
注释仅仅简单的总结了类型转换做了什么，所以不要太随便的使用。在你的项目使用它们之前，你最好看看它们的详细说明书。

尽管写了一些，但我这里不是想给你一个关于*type traits*使用的教程。注意类型转换尾部的`::type`。如果你在一个模板内部将他们施加到类型形参上（实际代码中你也总是这么用），你也需要在它们前面加上`typename`。至于为什么要这么做是因为这些C++11的*type traits*是通过在`struct`内嵌套`typedef`来实现的。是的，它们使用类型同义词（译注：根据上下文指的是使用`typedef`的做法）技术实现，而正如我之前所说这比别名声明要差。

关于为什么这么实现是有历史原因的，但是跳过它（我认为太无聊了），因为标准委员会没有及时认识到别名声明是更好的选择，所以直到C++14它们才提供了使用别名声明的版本。这些别名声明有一个通用形式：对于C++11的类型转换`std::`transformation`<T>::type`在C++14中变成了`std::`transformation`_t`。举个例子或许更容易理解：

````cpp
std::remove_const<T>::type          //C++11: const T → T 
std::remove_const_t<T>              //C++14 等价形式

std::remove_reference<T>::type      //C++11: T&/T&& → T 
std::remove_reference_t<T>          //C++14 等价形式

std::add_lvalue_reference<T>::type  //C++11: T → T& 
std::add_lvalue_reference_t<T>      //C++14 等价形式
````
C++11的的形式在C++14中也有效，但是我不能理解为什么你要去用它们。就算你没办法使用C++14，使用别名模板也是小儿科。只需要C++11的语言特性，甚至每个小孩都能仿写，对吧？如果你有一份C++14标准，就更简单了，只需要复制粘贴：
````cpp
template <class T> 
using remove_const_t = typename remove_const<T>::type;

template <class T> 
using remove_reference_t = typename remove_reference<T>::type;

template <class T> 
using add_lvalue_reference_t =
  typename add_lvalue_reference<T>::type; 
````
看见了吧？不能再简单了。

**请记住：**

+ `typedef`不支持模板化，但是别名声明支持。
+ 别名模板避免了使用“`::type`”后缀，而且在模板中使用`typedef`还需要在前面加上`typename`
+ C++14提供了C++11所有*type traits*转换的别名声明版本

## 十：优先考虑限域`enum`而非未限域`enum`

**Item 10: Prefer scoped `enum`s to unscoped `enum`s**

通常来说，在花括号中声明一个名字会限制它的作用域在花括号之内。但这对于C++98风格的`enum`中声明的枚举名（译注：*enumerator*，连同下文“枚举名”都指*enumerator*）是不成立的。这些枚举名的名字（译注：*enumerator* names，连同下文“名字”都指names）属于包含这个`enum`的作用域，这意味着作用域内不能含有相同名字的其他东西：
```cpp
enum Color { black, white, red };   //black, white, red在
                                    //Color所在的作用域
auto white = false;                 //错误! white早已在这个作用
                                    //域中声明
```
这些枚举名的名字泄漏进它们所被定义的`enum`在的那个作用域，这个事实有一个官方的术语：未限域枚举(*unscoped `enum`*)。在C++11中它们有一个相似物，限域枚举(*scoped `enum`*)，它不会导致枚举名泄漏：
```cpp
enum class Color { black, white, red }; //black, white, red
                                        //限制在Color域内
auto white = false;                     //没问题，域内没有其他“white”

Color c = white;                        //错误，域中没有枚举名叫white

Color c = Color::white;                 //没问题
auto c = Color::white;                  //也没问题（也符合Item5的建议）
```
因为限域`enum`是通过“`enum class`”声明，所以它们有时候也被称为枚举类(*`enum` classes*)。

使用限域`enum`来减少命名空间污染，这是一个足够合理使用它而不是它的同胞未限域`enum`的理由，其实限域`enum`还有第二个吸引人的优点：在它的作用域中，枚举名是强类型。未限域`enum`中的枚举名会隐式转换为整型（现在，也可以转换为浮点类型）。因此下面这种歪曲语义的做法也是完全有效的：
```cpp
enum Color { black, white, red };       //未限域enum

std::vector<std::size_t>                //func返回x的质因子
  primeFactors(std::size_t x);

Color c = red;
…

if (c < 14.5) {                         // Color与double比较 (!)
    auto factors =                      // 计算一个Color的质因子(!)
      primeFactors(c);
    …
}
```
在`enum`后面写一个`class`就可以将非限域`enum`转换为限域`enum`，接下来就是完全不同的故事展开了。现在不存在任何隐式转换可以将限域`enum`中的枚举名转化为任何其他类型：
```cpp
enum class Color { black, white, red }; //Color现在是限域enum

Color c = Color::red;                   //和之前一样，只是
...                                     //多了一个域修饰符

if (c < 14.5) {                         //错误！不能比较
                                        //Color和double
    auto factors =                      //错误！不能向参数为std::size_t
      primeFactors(c);                  //的函数传递Color参数
    …
}
```
如果你真的很想执行`Color`到其他类型的转换，和平常一样，使用正确的类型转换运算符扭曲类型系统：
```cpp
if (static_cast<double>(c) < 14.5) {    //奇怪的代码，
                                        //但是有效
    auto factors =                                  //有问题，但是
      primeFactors(static_cast<std::size_t>(c));    //能通过编译
    …
}
```
似乎比起非限域`enum`而言，限域`enum`有第三个好处，因为限域`enum`可以被前置声明。也就是说，它们可以不指定枚举名直接声明：
```cpp
enum Color;         //错误！
enum class Color;   //没问题
```
其实这是一个误导。在C++11中，非限域`enum`也可以被前置声明，但是只有在做一些其他工作后才能实现。这些工作来源于一个事实：在C++中所有的`enum`都有一个由编译器决定的整型的底层类型。对于非限域`enum`比如`Color`，
```cpp
enum Color { black, white, red };
```
编译器可能选择`char`作为底层类型，因为这里只需要表示三个值。然而，有些`enum`中的枚举值范围可能会大些，比如：
```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              indeterminate = 0xFFFFFFFF
            };
```
这里值的范围从`0`到`0xFFFFFFFF`。除了在不寻常的机器上（比如一个`char`至少有32bits的那种），编译器都会选择一个比`char`大的整型类型来表示`Status`。

为了高效使用内存，编译器通常在确保能包含所有枚举值的前提下为`enum`选择一个最小的底层类型。在一些情况下，编译器将会优化速度，舍弃大小，这种情况下它可能不会选择最小的底层类型，而是选择对优化大小有帮助的类型。为此，C++98只支持`enum`定义（所有枚举名全部列出来）；`enum`声明是不被允许的。这使得编译器能在使用之前为每一个`enum`选择一个底层类型。

但是不能前置声明`enum`也是有缺点的。最大的缺点莫过于它可能增加编译依赖。再次考虑`Status` `enum`：
```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              indeterminate = 0xFFFFFFFF
            };
```
这种`enum`很有可能用于整个系统，因此系统中每个包含这个头文件的组件都会依赖它。如果引入一个新状态值，
```cpp
enum Status { good = 0,
              failed = 1,
              incomplete = 100,
              corrupt = 200,
              audited = 500,
              indeterminate = 0xFFFFFFFF
            };
```
那么可能整个系统都得重新编译，即使只有一个子系统——或者只有一个函数——使用了新添加的枚举名。这是大家都**不希望**看到的。C++11中的前置声明`enum`s可以解决这个问题。比如这里有一个完全有效的限域`enum`声明和一个以该限域`enum`作为形参的函数声明：

```cpp
enum class Status;                  //前置声明
void continueProcessing(Status s);  //使用前置声明enum
```
即使`Status`的定义发生改变，包含这些声明的头文件也不需要重新编译。而且如果`Status`有改动（比如添加一个`audited`枚举名），`continueProcessing`的行为不受影响（比如因为`continueProcessing`没有使用这个新添加的`audited`），`continueProcessing`也不需要重新编译。

但是如果编译器在使用它之前需要知晓该`enum`的大小，该怎么声明才能让C++11做到C++98不能做到的事情呢？答案很简单：限域`enum`的底层类型总是已知的，而对于非限域`enum`，你可以指定它。

默认情况下，限域枚举的底层类型是`int`：

```cpp
enum class Status;                  //底层类型是int
```
如果默认的`int`不适用，你可以重写它：
```cpp
enum class Status: std::uint32_t;   //Status的底层类型
                                    //是std::uint32_t
                                    //（需要包含 <cstdint>）
```
不管怎样，编译器都知道限域`enum`中的枚举名占用多少字节。

要为非限域`enum`指定底层类型，你可以同上，结果就可以前向声明：

```cpp
enum Color: std::uint8_t;   //非限域enum前向声明
                            //底层类型为
                            //std::uint8_t
```
底层类型说明也可以放到`enum`定义处：
```cpp
enum class Status: std::uint32_t { good = 0,
                                   failed = 1,
                                   incomplete = 100,
                                   corrupt = 200,
                                   audited = 500,
                                   indeterminate = 0xFFFFFFFF
                                 };
```
限域`enum`避免命名空间污染而且不接受荒谬的隐式类型转换，但它并非万事皆宜，你可能会很惊讶听到至少有一种情况下非限域`enum`是很有用的。那就是牵扯到C++11的`std::tuple`的时候。比如在社交网站中，假设有一个*tuple*保存了用户的名字，email地址，声望值：
```cpp
using UserInfo =                //类型别名，参见Item9
    std::tuple<std::string,     //名字
               std::string,     //email地址
               std::size_t> ;   //声望
```
虽然注释说明了tuple各个字段对应的意思，但当你在另一文件遇到下面的代码那之前的注释就不是那么有用了：
```cpp
UserInfo uInfo;                 //tuple对象
…
auto val = std::get<1>(uInfo);	//获取第一个字段
```
作为一个程序员，你有很多工作要持续跟进。你应该记住第一个字段代表用户的email地址吗？我认为不。可以使用非限域`enum`将名字和字段编号关联起来以避免上述需求：
```cpp
enum UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;                         //同之前一样
…
auto val = std::get<uiEmail>(uInfo);    //啊，获取用户email字段的值
```
之所以它能正常工作是因为`UserInfoFields`中的枚举名隐式转换成`std::size_t`了，其中`std::size_t`是`std::get`模板实参所需的。

对应的限域`enum`版本就很啰嗦了：

```cpp
enum class UserInfoFields { uiName, uiEmail, uiReputation };

UserInfo uInfo;                         //同之前一样
…
auto val =
    std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>
        (uInfo);
```
为避免这种冗长的表示，可以写一个函数传入枚举名并返回对应的`std::size_t`值，但这有一点技巧性。`std::get`是一个模板（函数），需要你给出一个`std::size_t`值的模板实参（注意使用`<>`而不是`()`），因此将枚举名变换为`std::size_t`值的函数必须**在编译期**产生这个结果。如[Item15](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item15.md)提到的，那必须是一个`constexpr`函数。

事实上，它也的确该是一个`constexpr`函数模板，因为它应该能用于任何`enum`。如果想让它更一般化，还要泛化它的返回类型。较之于返回`std::size_t`，更应该返回枚举的底层类型。这可以通过`std::underlying_type`这个*type trait*获得。（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)关于*type trait*的内容）。最终还要再加上`noexcept`修饰（参见[Item14](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item14.md)），因为知道它肯定不会产生异常。根据上述分析最终得到的`toUType`函数模板在编译期接受任意枚举名并返回它的值：

```cpp
template<typename E>
constexpr typename std::underlying_type<E>::type
    toUType(E enumerator) noexcept
{
    return
        static_cast<typename
                    std::underlying_type<E>::type>(enumerator);
}
```
在C++14中，`toUType`还可以进一步用`std::underlying_type_t`（参见[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)）代替`typename std::underlying_type<E>::type`打磨：

```cpp
template<typename E>                //C++14
constexpr std::underlying_type_t<E>
    toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
还可以再用C++14 `auto`（参见[Item3](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)）打磨一下代码：
```cpp
template<typename E>                //C++14
constexpr auto
    toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
不管它怎么写，`toUType`现在允许这样访问tuple的字段了：
```cpp
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```
这仍然比使用非限域`enum`要写更多的代码，但同时它也避免命名空间污染，防止不经意间使用隐式转换。大多数情况下，你应该会觉得多敲几个（几行）字符作为避免使用未限域枚举这种老得和2400波特率猫同时代技术的代价是值得的。

**记住**
+ C++98的`enum`即非限域`enum`。
+ 限域`enum`的枚举名仅在`enum`内可见。要转换为其它类型只能使用*cast*。
+ 非限域/限域`enum`都支持底层类型说明语法，限域`enum`底层类型默认是`int`。非限域`enum`没有默认底层类型。
+ 限域`enum`总是可以前置声明。非限域`enum`仅当指定它们的底层类型时才能前置。
## 十一：优先考虑使用*deleted*函数而非使用未定义的私有声明

**Item 11: Prefer deleted functions to private undefined ones.**

如果你写的代码要被其他人使用，你不想让他们调用某个特殊的函数，你通常不会声明这个函数。无声明，不函数。简简单单！但有时C++会给你自动声明一些函数，如果你想防止客户调用这些函数，事情就不那么简单了。

上述场景见于特殊的成员函数，即当有必要时C++自动生成的那些函数。[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)详细讨论了这些函数，但是现在，只关心拷贝构造函数和拷贝赋值运算符重载。本节主要致力于讨论C++98中那些被C++11所取代的最佳实践，而且在C++98中，你想要禁止使用的成员函数，几乎总是拷贝构造函数或者赋值运算符，或者两者都是。

在C++98中防止调用这些函数的方法是将它们声明为私有（`private`）成员函数并且不定义。举个例子，在C++ 标准库*iostream*继承链的顶部是模板类`basic_ios`。所有*istream*和*ostream*类都继承此类（直接或者间接）。拷贝*istream*和*ostream*是不合适的，因为这些操作应该怎么做是模棱两可的。比如一个`istream`对象，代表一个输入值的流，流中有一些已经被读取，有一些可能马上要被读取。如果一个*istream*被拷贝，需要拷贝将要被读取的值和已经被读取的值吗？解决这个问题最好的方法是不定义这个操作。直接禁止拷贝流。

要使这些*istream*和*ostream*类不可拷贝，`basic_ios`在C++98中是这样声明的（包括注释）：
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …

private:
    basic_ios(const basic_ios& );           // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};
```
将它们声明为私有成员可以防止客户端调用这些函数。故意不定义它们意味着假如还是有代码用它们（比如成员函数或者类的友元`friend`），就会在链接时引发缺少函数定义（*missing function definitions*）错误。

在C++11中有一种更好的方式达到相同目的：用“`= delete`”将拷贝构造函数和拷贝赋值运算符标记为***deleted*函数**（译注：一些文献翻译为“删除的函数”）。上面相同的代码在C++11中是这样声明的：
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …

    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
};
```
删除这些函数（译注：添加"`= delete`"）和声明为私有成员可能看起来只是方式不同，别无其他区别。其实还有一些实质性意义。*deleted*函数不能以任何方式被调用，即使你在成员函数或者友元函数里面调用*deleted*函数也不能通过编译。这是较之C++98行为的一个改进，C++98中不正确的使用这些函数在链接时才被诊断出来。

通常，*deleted*函数被声明为`public`而不是`private`。这也是有原因的。当客户端代码试图调用成员函数，C++会在检查*deleted*状态前检查它的访问性。当客户端代码调用一个私有的*deleted*函数，一些编译器只会给出该函数是`private`的错误（译注：而没有诸如该函数被*deleted*修饰的错误），即使函数的访问性不影响它是否能被使用。所以值得牢记，如果要将老代码的“私有且未定义”函数替换为*deleted*函数时请一并修改它的访问性为`public`，这样可以让编译器产生更好的错误信息。

*deleted*函数还有一个重要的优势是**任何**函数都可以标记为*deleted*，而只有成员函数可被标记为`private`。（译注：从下文可知“任何”是包含普通函数和成员函数等所有可声明函数的地方，而`private`方法只适用于成员函数）假如有一个非成员函数，它接受一个整型参数，检查它是否为幸运数：

```cpp
bool isLucky(int number);
```
C++有沉重的C包袱，使得含糊的、能被视作数值的任何类型都能隐式转换为`int`，但是有一些调用可能是没有意义的：
```cpp
if (isLucky('a')) …         //字符'a'是幸运数？
if (isLucky(true)) …        //"true"是?
if (isLucky(3.5)) …         //难道判断它的幸运之前还要先截尾成3？
```
如果幸运数必须真的是整型，该禁止这些调用通过编译。

其中一种方法就是创建*deleted*重载函数，其参数就是想要过滤的类型：

```cpp
bool isLucky(int number);       //原始版本
bool isLucky(char) = delete;    //拒绝char
bool isLucky(bool) = delete;    //拒绝bool
bool isLucky(double) = delete;  //拒绝float和double
```
(上面`double`重载版本的注释说拒绝`float`和`double`可能会让你惊讶，但是请回想一下：将`float`转换为`int`和`double`，C++更喜欢转换为`double`。使用`float`调用`isLucky`因此会调用`double`重载版本，而不是`int`版本。好吧，它也会那么去尝试。事实是调用被删除的`double`重载版本不能通过编译。不再惊讶了吧。)

虽然*deleted*函数不能被使用，但它们还是存在于你的程序中。也即是说，重载决议会考虑它们。这也是为什么上面的函数声明导致编译器拒绝一些不合适的函数调用。
```cpp
if (isLucky('a')) …     //错误！调用deleted函数
if (isLucky(true)) …    //错误！
if (isLucky(3.5f)) …    //错误！
```
另一个*deleted*函数用武之地（`private`成员函数做不到的地方）是禁止一些模板的实例化。假如你要求一个模板仅支持原生指针（尽管[第四章](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/4.SmartPointers/item18.md)建议使用智能指针代替原生指针）：
```cpp
template<typename T>
void processPointer(T* ptr);
```
在指针的世界里有两种特殊情况。一是`void*`指针，因为没办法对它们进行解引用，或者加加减减等。另一种指针是`char*`，因为它们通常代表C风格的字符串，而不是正常意义下指向单个字符的指针。这两种情况要特殊处理，在`processPointer`模板里面，假设正确的函数应该拒绝这些类型。也即是说，`processPointer`不能被`void*`和`char*`调用。

要想确保这个很容易，使用`delete`标注模板实例：

```cpp
template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;
```
现在如果使用`void*`和`char*`调用`processPointer`就是无效的，按常理说`const void*`和`const void*`也应该无效，所以这些实例也应该标注`delete`:
```cpp
template<>
void processPointer<const void>(const void*) = delete;

template<>
void processPointer<const char>(const char*) = delete;
```
如果你想做得更彻底一些，你还要删除`const volatile void*`和`const volatile char*`重载版本，另外还需要一并删除其他标准字符类型的重载版本：`std::wchar_t`，`std::char16_t`和`std::char32_t`。

有趣的是，如果的类里面有一个函数模板，你可能想用`private`（经典的C++98惯例）来禁止这些函数模板实例化，但是不能这样做，因为不能给特化的成员模板函数指定一个不同于主函数模板的访问级别。如果`processPointer`是类`Widget`里面的模板函数， 你想禁止它接受`void*`参数，那么通过下面这样C++98的方法就不能通过编译：

```cpp
class Widget {
public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }

private:
    template<>                          //错误！
    void processPointer<void>(void*);
    
};
```
问题是模板特例化必须位于一个命名空间作用域，而不是类作用域。*deleted*函数不会出现这个问题，因为它不需要一个不同的访问级别，且他们可以在类外被删除（因此位于命名空间作用域）：
```cpp
class Widget {
public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }
    …

};

template<>                                          //还是public，
void Widget::processPointer<void>(void*) = delete;  //但是已经被删除了
```
事实上C++98的最佳实践即声明函数为`private`但不定义是在做C++11 *deleted*函数要做的事情。作为模仿者，C++98的方法不是十全十美。它不能在类外正常工作，不能总是在类中正常工作，它的罢工可能直到链接时才会表现出来。所以请坚定不移的使用*deleted*函数。

**请记住：**

+ 比起声明函数为`private`但不定义，使用*deleted*函数更好
+ 任何函数都能被删除（be deleted），包括非成员函数和模板实例（译注：实例化的函数）

## 十二：使用`override`声明重写函数

**Item 12: Declare overriding functions `override`**

在C++面向对象的世界里，涉及的概念有类，继承，虚函数。这个世界最基本的概念是派生类的虚函数**重写**基类同名函数。令人遗憾的是虚函数重写可能一不小心就错了。似乎这部分语言的设计理念是不仅仅要遵守墨菲定律，还应该尊重它。

虽然“重写（*overriding*）”听起来像“重载（*overloading*）”，然而两者完全不相关，所以让我澄清一下，正是虚函数重写机制的存在，才使可以通过基类的接口调用派生类的成员函数：

```cpp
class Base {
public:
    virtual void doWork();          //基类虚函数
    …
};

class Derived: public Base {
public:
    virtual void doWork();          //重写Base::doWork
    …                               //（这里“virtual”是可以省略的）
}; 

std::unique_ptr<Base> upb =         //创建基类指针指向派生类对象
    std::make_unique<Derived>();    //关于std：：make_unique
…                                   //请参见Item21

    
upb->doWork();                      //通过基类指针调用doWork，
                                    //实际上是派生类的doWork
                                    //函数被调用
```
要想重写一个函数，必须满足下列要求：

+ 基类函数必须是`virtual`
+ 基类和派生类函数名必须完全一样（除非是析构函数)
+ 基类和派生类函数形参类型必须完全一样
+ 基类和派生类函数常量性`const`ness必须完全一样
+ 基类和派生类函数的返回值和异常说明（*exception specifications*）必须兼容

除了这些C++98就存在的约束外，C++11又添加了一个：
+ 函数的引用限定符（*reference qualifiers*）必须完全一样。成员函数的引用限定符是C++11很少抛头露脸的特性，所以如果你从没听过它无需惊讶。它可以限定成员函数只能用于左值或者右值。成员函数不需要`virtual`也能使用它们：
```cpp
class Widget {
public:
    …
    void doWork() &;    //只有*this为左值的时候才能被调用
    void doWork() &&;   //只有*this为右值的时候才能被调用
}; 
…
Widget makeWidget();    //工厂函数（返回右值）
Widget w;               //普通对象（左值）
…
w.doWork();             //调用被左值引用限定修饰的Widget::doWork版本
                        //（即Widget::doWork &）
makeWidget().doWork();  //调用被右值引用限定修饰的Widget::doWork版本
                        //（即Widget::doWork &&）
```
后面我还会提到引用限定符修饰成员函数，但是现在，只需要记住如果基类的虚函数有引用限定符，派生类的重写就必须具有相同的引用限定符。如果没有，那么新声明的函数还是属于派生类，但是不会重写父类的任何函数。

这么多的重写需求意味着哪怕一个小小的错误也会造成巨大的不同。代码中包含重写错误通常是有效的，但它的意图不是你想要的。因此你不能指望当你犯错时编译器能通知你。比如，下面的代码是完全合法的，咋一看，还很有道理，但是它没有任何虚函数重写——没有一个派生类函数联系到基类函数。你能识别每种情况的错误吗，换句话说，为什么派生类函数没有重写同名基类函数？
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived: public Base {
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
```
需要一点帮助吗？
+ `mf1`在`Base`基类声明为`const`，但是`Derived`派生类没有这个常量限定符

+ `mf2`在`Base`基类声明为接受一个`int`参数，但是在`Derived`派生类声明为接受`unsigned int`参数

+ `mf3`在`Base`基类声明为左值引用限定，但是在`Derived`派生类声明为右值引用限定

+ `mf4`在`Base`基类没有声明为`virtual`虚函数

你可能会想，“哎呀，实际操作的时候，这些warnings都能被编译器探测到，所以我不需要担心。”你说的可能对，也可能不对。就我目前检查的两款编译器来说，这些代码编译时没有任何warnings，即使我开启了输出所有warnings。（其他编译器可能会为这些问题的部分输出warnings，但不是全部。）

由于正确声明派生类的重写函数很重要，但很容易出错，C++11提供一个方法让你可以显式地指定一个派生类函数是基类版本的重写：将它声明为`override`。还是上面那个例子，可以这样做：
```cpp
class Derived: public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```
代码不能编译，当然了，因为这样写的时候，编译器会抱怨所有与重写有关的问题。这也是你想要的，以及为什么要在所有重写函数后面加上`override`。

使用`override`的代码编译时看起来就像这样（假设的目的是`Derived`派生类中的所有函数重写`Base`基类的相应虚函数）:

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};

class Derived: public Base {
public:
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override;          //可以添加virtual，但不是必要
}; 
```
注意在这个例子中`mf4`有别于之前，它在`Base`中的声明有`virtual`修饰，所以能正常工作。大多数和重写有关的错误都是在派生类引发的，但也可能是基类的不正确导致。

比起让编译器（译注：通过warnings）告诉你想重写的而实际没有重写，不如给你的派生类重写函数全都加上`override`。如果你考虑修改修改基类虚函数的函数签名，`override`还可以帮你评估后果。如果派生类全都用上`override`，你可以只改变基类函数签名，重编译系统，再看看你造成了多大的问题（即，多少派生类不能通过编译），然后决定是否值得如此麻烦更改函数签名。没有`override`，你只能寄希望于完善的单元测试，因为，正如所见，派生类虚函数本想重写基类，但是没有，编译器也没有探测并发出诊断信息。

C++既有很多关键字，C++11引入了两个上下文关键字（*contextual keywords*），`override`和`final`（向虚函数添加`final`可以防止派生类重写。`final`也能用于类，这时这个类不能用作基类）。这两个关键字的特点是它们是保留的，它们只是位于特定上下文才被视为关键字。对于`override`，它只在成员函数声明结尾处才被视为关键字。这意味着如果你以前写的代码里面已经用过**override**这个名字，那么换到C++11标准你也无需修改代码：
```cpp
class Warning {         //C++98潜在的传统类代码
public:
    …
    void override();    //C++98和C++11都合法（且含义相同）
    …
};
```
关于`override`想说的就这么多，但对于成员函数引用限定（*reference qualifiers*）还有一些内容。我之前承诺我会在后面提供更多的关于它们的资料，现在就是"后面"了。

如果想写一个函数只接受左值实参，声明一个non-`const`左值引用形参：

```cpp
void doSomething(Widget& w);    //只接受左值Widget对象
```
如果想写一个函数只接受右值实参，声明一个右值引用形参：
```cpp
void doSomething(Widget&& w);   //只接受右值Widget对象
```
成员函数的引用限定可以很容易的区分一个成员函数被哪个对象（即`*this`）调用。它和在成员函数声明尾部添加一个`const`很相似，暗示了调用这个成员函数的对象（即`*this`）是`const`的。

对成员函数添加引用限定不常见，但是可以见。举个例子，假设的`Widget`类有一个`std::vector`数据成员，提供一个访问函数让客户端可以直接访问它：

```cpp
class Widget {
public:
    using DataType = std::vector<double>;   //“using”的信息参见Item9
    …
    DataType& data() { return values; }
    …
private:
    DataType values;
};
```
这是最具封装性的设计，只给外界保留一线光。但先把这个放一边，思考一下下面的客户端代码：
```cpp
Widget w;
…
auto vals1 = w.data();  //拷贝w.values到vals1
```
`Widget::data`函数的返回值是一个左值引用（准确的说是`std::vector<double>&`）,
因为左值引用是左值，所以`vals1`是从左值初始化的。因此`vals1`由`w.values`拷贝构造而得，就像注释说的那样。

现在假设有一个创建`Widget`s的工厂函数，

```cpp
Widget makeWidget();
```
想用`makeWidget`返回的`Widget`里的`std::vector`初始化一个变量：
```cpp
auto vals2 = makeWidget().data();   //拷贝Widget里面的值到vals2
```
再说一次，`Widgets::data`返回的是左值引用，还有，左值引用是左值。所以，的对象（`vals2`）得从`Widget`里的`values`拷贝构造。这一次，`Widget`是`makeWidget`返回的临时对象（即右值），所以将其中的`std::vector`进行拷贝纯属浪费。最好是移动，但是因为`data`返回左值引用，C++的规则要求编译器不得不生成一个拷贝。（这其中有一些优化空间，被称作“as if rule”，但是你依赖编译器使用这个优化规则就有点傻。）（译注：“as if rule”简单来说就是在不影响程序的“外在表现”情况下做一些改变）

需要的是指明当`data`被右值`Widget`对象调用的时候结果也应该是一个右值。现在就可以使用引用限定，为左值`Widget`和右值`Widget`写一个`data`的重载函数来达成这一目的：

```cpp
class Widget {
public:
    using DataType = std::vector<double>;
    …
    DataType& data() &              //对于左值Widgets,
    { return values; }              //返回左值
    
    DataType data() &&              //对于右值Widgets,
    { return std::move(values); }   //返回右值
    …

private:
    DataType values;
};
```
注意`data`重载的返回类型是不同的，左值引用重载版本返回一个左值引用（即一个左值），右值引用重载返回一个临时对象（即一个右值）。这意味着现在客户端的行为和的期望相符了：
```cpp
auto vals1 = w.data();              //调用左值重载版本的Widget::data，
                                    //拷贝构造vals1
auto vals2 = makeWidget().data();   //调用右值重载版本的Widget::data, 
                                    //移动构造vals2
```
这真的很棒，但别被这结尾的暖光照耀分心以致忘记了该的中心。这个的中心是只要你在派生类声明想要重写基类虚函数的函数，就加上`override`。

**请记住：**

+ 为重写函数加上`override`
+ 成员函数引用限定让可以区别对待左值对象和右值对象（即`*this`)
## 十三：优先考虑`const_iterator`而非`iterator`

**Item 13: Prefer `const_iterators` to `iterators`**

STL `const_iterator`等价于指向常量的指针（pointer-to-`const`）。它们都指向不能被修改的值。标准实践是能加上`const`就加上，这也指示需要一个迭代器时只要没必要修改迭代器指向的值，就应当使用`const_iterator`。

上面的说法对C++11和C++98都是正确的，但是在C++98中，标准库对`const_iterator`的支持不是很完整。首先不容易创建它们，其次就算你有了它，它的使用也是受限的。假如你想在`std::vector<int>`中查找第一次出现1983（C++代替C with classes的那一年）的位置，然后插入1998（第一个ISO C++标准被接纳的那一年）。如果*vector*中没有1983，那么就在*vector*尾部插入。在C++98中使用`iterator`可以很容易做到：

```cpp
std::vector<int> values;
…
std::vector<int>::iterator it =
    std::find(values.begin(), values.end(), 1983);
values.insert(it, 1998);
```

但是这里`iterator`真的不是一个好的选择，因为这段代码不修改`iterator`指向的内容。用`const_iterator`重写这段代码是很平常的，但是在C++98中就不是了。下面是一种概念上可行但是不正确的方法：

```cpp
typedef std::vector<int>::iterator IterT;               //typedef
typedef std::vector<int>::const_iterator ConstIterT;

std::vector<int> values;
…
ConstIterT ci =
    std::find(static_cast<ConstIterT>(values.begin()),  //cast
              static_cast<ConstIterT>(values.end()),    //cast
              1983);

values.insert(static_cast<IterT>(ci), 1998);    //可能无法通过编译，
                                                //原因见下
```

`typedef`不是强制的，但是可以让代码中的*cast*更好写。（你可能想知道为什么我使用`typedef`而不是[Item9](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item9.md)提到的别名声明，因为这段代码在演示C++98做法，别名声明是C++11加入的特性）

之所以`std::find`的调用会出现类型转换是因为在C++98中`values`是non-`const`容器，没办法简简单单的从non-`const`容器中获取`const_iterator`。严格来说类型转换不是必须的，因为用其他方法获取`const_iterator`也是可以的（比如你可以把`values`绑定到reference-to-`const`变量上，然后再用这个变量代替`values`），但不管怎么说，从non-`const`容器中获取`const_iterator`的做法都有点别扭。

当你费劲地获得了`const_iterator`，事情可能会变得更糟，因为C++98中，插入操作（以及删除操作）的位置只能由`iterator`指定，`const_iterator`是不被接受的。这也是我在上面的代码中，将`const_iterator`（我那么小心地从`std::find`搞出来的东西）转换为`iterator`的原因，因为向`insert`传入`const_iterator`不能通过编译。

老实说，上面的代码也可能无法编译，因为没有一个可移植的从`const_iterator`到`iterator`的方法，即使使用`static_cast`也不行。甚至传说中的牛刀`reinterpret_cast`也杀不了这条鸡。（它不是C++98的限制，也不是C++11的限制，只是`const_iterator`就是不能转换为`iterator`，不管看起来对它们施以转换是有多么合理。）不过有办法生成一个`iterator`，使其指向和`const_iterator`指向相同，但是看起来不明显，也没有广泛应用，在这本书也不值得讨论。除此之外，我希望目前我陈述的观点是清晰的：`const_iterator`在C++98中会有很多问题，不如它的兄弟（译注：指`iterator`）有用。最终，开发者们不再相信能加`const`就加它的教条，而是只在实用的地方加它，C++98的`const_iterator`不是那么实用。

所有的这些都在C++11中改变了，现在`const_iterator`既容易获取又容易使用。容器的成员函数`cbegin`和`cend`产出`const_iterator`，甚至对于non-`const`容器也可用，那些之前使用*iterator*指示位置（如`insert`和`erase`）的STL成员函数也可以使用`const_iterator`了。使用C++11 `const_iterator`重写C++98使用`iterator`的代码也稀松平常：
```cpp
std::vector<int> values;                                //和之前一样
…
auto it =                                               //使用cbegin
	std::find(values.cbegin(), values.cend(), 1983);//和cend
values.insert(it, 1998);
```
现在使用`const_iterator`的代码就很实用了！

唯一一个C++11对于`const_iterator`支持不足（译注：C++14支持但是C++11的时候还没）的情况是：当你想写最大程度通用的库，并且这些库代码为一些容器和类似容器的数据结构提供`begin`、`end`（以及`cbegin`，`cend`，`rbegin`，`rend`等）作为**非成员函数**而不是成员函数时。其中一种情况就是原生数组，还有一种情况是一些只由自由函数组成接口的第三方库。（译注：自由函数*free function*，指的是非成员函数，即一个函数，只要不是成员函数就可被称作*free function*）最大程度通用的库会考虑使用非成员函数而不是假设成员函数版本存在。

举个例子，可以泛化下面的`findAndInsert`：

```cpp
template<typename C, typename V>
void findAndInsert(C& container,            //在容器中查找第一次
                   const V& targetVal,      //出现targetVal的位置，
                   const V& insertVal)      //然后在那插入insertVal
{
    using std::cbegin;
    using std::cend;

    auto it = std::find(cbegin(container),  //非成员函数cbegin
                        cend(container),    //非成员函数cend
                        targetVal);
    container.insert(it, insertVal);
}
```

它可以在C++14工作良好，但是很遗憾，C++11不在良好之列。由于标准化的疏漏，C++11只添加了非成员函数`begin`和`end`，但是没有添加`cbegin`，`cend`，`rbegin`，`rend`，`crbegin`，`crend`。C++14修订了这个疏漏。

如果你使用C++11，并且想写一个最大程度通用的代码，而你使用的STL没有提供缺失的非成员函数`cbegin`和它的朋友们，你可以简单的写下你自己的实现。比如，下面就是非成员函数`cbegin`的实现：

```cpp
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    return std::begin(container);   //解释见下
}
```

你可能很惊讶非成员函数`cbegin`没有调用成员函数`cbegin`吧？我也是。但是请跟逻辑走。这个`cbegin`模板接受任何代表类似容器的数据结构的实参类型`C`，并且通过reference-to-`const`形参`container`访问这个实参。如果`C`是一个普通的容器类型（如`std::vector<int>`），`container`将会引用一个`const`版本的容器（如`const std::vector<int>&`）。对`const`容器调用非成员函数`begin`（由C++11提供）将产出`const_iterator`，这个迭代器也是模板要返回的。用这种方法实现的好处是就算容器只提供`begin`成员函数（对于容器来说，C++11的非成员函数`begin`调用这些成员函数）不提供`cbegin`成员函数也没问题。那么现在你可以将这个非成员函数`cbegin`施于只直接支持`begin`的容器。

如果`C`是原生数组，这个模板也能工作。这时，`container`成为一个`const`数组的引用。C++11为数组提供特化版本的非成员函数`begin`，它返回指向数组第一个元素的指针。一个`const`数组的元素也是`const`，所以对于`const`数组，非成员函数`begin`返回指向`const`的指针（pointer-to-`const`）。在数组的上下文中，所谓指向`const`的指针（pointer-to-`const`），也就是`const_iterator`了。

回到最开始，本的中心是鼓励你只要能就使用`const_iterator`。最原始的动机——只要它有意义就加上`const`——是C++98就有的思想。但是在C++98，它（译注：`const_iterator`）只是一般有用，到了C++11，它就是极其有用了，C++14在其基础上做了些修补工作。

**请记住：**

+ 优先考虑`const_iterator`而非`iterator`
+ 在最大程度通用的代码中，优先考虑非成员函数版本的`begin`，`end`，`rbegin`等，而非同名成员函数
## 十四：如果函数不抛出异常请使用`noexcept`

**Item 14: Declare functions `noexcept` if they won’t emit exceptions**

在C++98中，异常说明（*exception specifications*）是喜怒无常的野兽。你不得不写出函数可能抛出的异常类型，如果函数实现有所改变，异常说明也可能需要修改。改变异常说明会影响客户端代码，因为调用者可能依赖原版本的异常说明。编译器不会在函数实现，异常说明和客户端代码之间提供一致性保障。大多数程序员最终都认为不值得为C++98的异常说明做得如此麻烦。

在C++11标准化过程中，大家一致认为异常说明真正有用的信息是一个函数是否会抛出异常。非黑即白，一个函数可能抛异常，或者不会。这种"可能-绝不"的二元论构成了C++11异常说的基础，从根本上改变了C++98的异常说明。（C++98风格的异常说明也有效，但是已经标记为deprecated（废弃））。在C++11中，无条件的`noexcept`保证函数不会抛出任何异常。

关于一个函数是否已经声明为`noexcept`是接口设计的事。函数的异常抛出行为是客户端代码最关心的。调用者可以查看函数是否声明为`noexcept`，这个可以影响到调用代码的异常安全性（*exception safety*）和效率。就其本身而言，函数是否为`noexcept`和成员函数是否`const`一样重要。当你知道这个函数不会抛异常而没加上`noexcept`，那这个接口说明就有点差劲了。

不过这里还有给不抛异常的函数加上`noexcept`的动机：它允许编译器生成更好的目标代码。要想知道为什么，了解C++98和C++11指明一个函数不抛异常的方式是很有用了。考虑一个函数`f`，它保证调用者永远不会收到一个异常。两种表达方式如下：
```cpp
int f(int x) throw();   //C++98风格，没有来自f的异常
int f(int x) noexcept;  //C++11风格，没有来自f的异常
```
如果在运行时，`f`出现一个异常，那么就和`f`的异常说明冲突了。在C++98的异常说明中，调用栈（the *call stack*）会展开至`f`的调用者，在一些与这地方不相关的动作后，程序被终止。C++11异常说明的运行时行为有些不同：调用栈只是**可能**在程序终止前展开。

展开调用栈和**可能**展开调用栈两者对于代码生成（code generation）有非常大的影响。在一个`noexcept`函数中，当异常可能传播到函数外时，优化器不需要保证运行时栈（the runtime stack）处于可展开状态；也不需要保证当异常离开`noexcept`函数时，`noexcept`函数中的对象按照构造的反序析构。而标注“`throw()`”异常声明的函数缺少这样的优化灵活性，没加异常声明的函数也一样。可以总结一下：

```cpp
RetType function(params) noexcept;  //极尽所能优化
RetType function(params) throw();   //较少优化
RetType function(params);           //较少优化
```
这是一个充分的理由使得你当知道它不抛异常时加上`noexcept`。

还有一些函数更符合这个情况。移动操作是绝佳的例子。假如你有一份C++98代码，里面用到了`std::vector<Widget>`。`Widget`通过`push_back`一次又一次的添加进`std::vector`：
```cpp
std::vector<Widget> vw;
…
Widget w;
…                   //用w做点事
vw.push_back(w);    //把w添加进vw
…
```
假设这个代码能正常工作，你也无意修改为C++11风格。但是你确实想要C++11移动语义带来的性能优势，毕竟这里的类型是可以移动的（move-enabled types）。因此你需要确保`Widget`有移动操作，可以手写代码也可以让编译器自动生成，当然前提是能满足自动生成的条件（参见[Item17](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item17.md)）。

当新元素添加到`std::vector`，`std::vector`可能没地方放它，换句话说，`std::vector`的大小（size）等于它的容量（capacity）。这时候，`std::vector`会分配一个新的更大块的内存用于存放其中元素，然后将元素从老内存区移动到新内存区，然后析构老内存区里的对象。在C++98中，移动是通过复制老内存区的每一个元素到新内存区完成的，然后老内存区的每个元素发生析构。这种方法使得`push_back`可以提供很强的异常安全保证：如果在复制元素期间抛出异常，`std::vector`状态保持不变，因为老内存元素析构必须建立在它们已经成功复制到新内存的前提下。

在C++11中，一个很自然的优化就是将上述复制操作替换为移动操作。但是很不幸运，这会破坏`push_back`的异常安全保证。如果**n**个元素已经从老内存移动到了新内存区，但异常在移动第**n+1**个元素时抛出，那么`push_back`操作就不能完成。但是原始的`std::vector`已经被修改：有**n**个元素已经移动走了。恢复`std::vector`至原始状态也不太可能，因为从新内存移动到老内存本身又可能引发异常。

这是个很严重的问题，因为老代码可能依赖于`push_back`提供的强烈的异常安全保证。因此，C++11版本的实现不能简单的将`push_back`里面的复制操作替换为移动操作，除非知晓移动操作绝不抛异常，这时复制替换为移动就是安全的，唯一的副作用就是性能得到提升。

`std::vector::push_back`受益于“如果可以就移动，如果必要则复制”策略，并且它不是标准库中唯一采取该策略的函数。C++98中还有一些函数（如`std::vector::reverse`，`std::deque::insert`等）也受益于这种强异常保证。对于这个函数只有在知晓移动不抛异常的情况下用C++11的移动操作替换C++98的复制操作才是安全的。但是如何知道一个函数中的移动操作是否产生异常？答案很明显：它检查这个操作是否被声明为`noexcept`。（这个检查非常弯弯绕。像是`std::vector::push_back`之类的函数调用`std::move_if_noexcept`，这是个`std::move`的变体，根据其中类型的移动构造函数是否为`noexcept`的，视情况转换为右值或保持左值（参见[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)）。反过来，`std::move_if_noexcept`查阅`std::is_nothrow_move_constructible`这个*type trait*，基于移动构造函数是否有`noexcept`（或者`throw()`）的设计，编译器设置这个*type trait*的值。）

`swap`函数是`noexcept`的另一个绝佳用地。`swap`是STL算法实现的一个关键组件，它也常用于拷贝运算符重载中。它的广泛使用意味着对其施加不抛异常的优化是非常有价值的。有趣的是，标准库的`swap`是否`noexcept`有时依赖于用户定义的`swap`是否`noexcept`。比如，数组和`std::pair`的`swap`声明如下：

```cpp
template <class T, size_t N>
void swap(T (&a)[N],
          T (&b)[N]) noexcept(noexcept(swap(*a, *b)));  //见下文

template <class T1, class T2>
struct pair {
    …
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                                noexcept(swap(second, p.second)));
    …
};
```
这些函数**视情况**`noexcept`：它们是否`noexcept`依赖于`noexcept`声明中的表达式是否`noexcept`。假设有两个`Widget`数组，交换数组操作为`noexcept`的前提是数组中的元素交换是`noexcept`的，即`Widget`的`swap`是`noexcept`。因此`Widget`的`swap`的作者决定了交换`widget`的数组是否`noexcept`。对于`Widget`的交换是否`noexcept`决定了对于`Widget`数组的交换是否`noexcept`，以及其他交换，比如`Widget`的数组的数组的交换是否`noexcept`。类似地，交换两个含有`Widget`的`std::pair`是否`noexcept`依赖于`Widget`的`swap`是否`noexcept`。事实上交换高层次数据结构是否`noexcept`取决于它的构成部分的那些低层次数据结构是否`noexcept`，这激励你只要可以就提供`noexcept` `swap`函数（译注：因为如果你的函数不提供`noexcept`保证，其它依赖你的高层次`swap`就不能保证`noexcept`）。

现在，我希望你能为`noexcept`提供的优化机会感到高兴，同时我还得让你缓一缓别太高兴了。优化很重要，但是正确性更重要。我在这个的开头提到`noexcept`是函数接口的一部分，所以仅当你保证一个函数实现在长时间内不会抛出异常时才声明`noexcept`。如果你声明一个函数为`noexcept`，但随即又后悔了，你没有选择。你可以从函数声明中移除`noexcept`（即改变它的接口），这理所当然会影响客户端代码。你可以改变实现使得这个异常可以避免，再保留原版本（现在来看不正确的）异常说明。如果你这么做，在异常试图离开这个函数时程序将会终止。或者你就顺从了既有实现，舍弃了激起你兴趣的东西，从一开始就改变实现。这些选择都不尽人意。


这个问题的本质是实际上大多数函数都是异常中立（*exception-neutral*）的。这些函数自己不抛异常，但是它们内部的调用可能抛出。此时，异常中立函数允许那些抛出异常的函数在调用链上更进一步直到遇到异常处理程序，而不是就地终止。异常中立函数决不应该声明为`noexcept`，因为它们可能抛出那种“让它们过吧”的异常（译注：也就是说在当前这个函数内不处理异常，但是又不立即终止程序，而是让调用这个函数的函数处理异常。）因此大多数函数缺少`noexcept`设计。

然而，一些函数很自然的不应该抛异常，更进一步——尤其是移动操作和`swap`——使其`noexcept`有重大意义，只要可能就应该将它们实现为`noexcept`。（STL对容器的移动操作的接口规范里缺少`noexcept`。然而实现者可以增强STL函数的异常说明，实际上，至少有些容器的移动操作已被声明为`noexcept`，这些做法就是本条例所给建议的好示例。发现了容器移动操作可以写成不抛异常的之后，实现者经常将这些操作声明为`noexcept`的，尽管标准并没有要求他们这么做。）老实说，当你确保函数决不抛异常的时候，一定要将它们声明为`noexcept`。

请注意我说有些函数有**自然的**`noexcept`实现法。为了`noexcept`而扭曲函数实现来达成目的是本末倒置。是把马车放到马前，是一叶障目不见泰山。是...选择你喜欢的比喻吧。（译注：几个英语熟语，都是想说明“本末倒置”。）如果一个简单的函数实现可能引发异常（比如调用一个可能抛异常的函数），而你为了讨好调用者隐藏了这个（比如捕获所有异常，然后替换为状态码或者特殊返回值），这不仅会使你的函数实现变得复杂，还会让调用点的代码变得复杂。调用者可能不得不检查状态码或特殊返回值。而这些复杂的运行时开销（比如额外的分支，大的函数给指令缓存带来的压力等）可能超出`noexcept`带来的性能提升，再加上你会悲哀的发现这些代码又难读又难维护。那是糟糕的软件工程化。

对于一些函数，使其成为`noexcept`是很重要的，它们应当默认如是。在C++98，允许内存释放（memory deallocation）函数（即`operator delete`和`operator delete[]`）和析构函数抛出异常是糟糕的代码设计，C++11将这种作风升级为语言规则。默认情况下，内存释放函数和析构函数——不管是用户定义的还是编译器生成的——都是隐式`noexcept`。因此它们不需要声明`noexcept`。（这么做也不会有问题，只是不合常规）。析构函数非隐式`noexcept`的情况仅当类的数据成员（包括继承的成员还有继承成员内的数据成员）明确声明它的析构函数可能抛出异常（如声明“`noexcept(false)`”）。这种析构函数不常见，标准库里面没有。如果一个对象的析构函数可能被标准库使用（比如在容器内或者被传给一个算法），析构函数又可能抛异常，那么程序的行为是未定义的。

值得注意的是一些库接口设计者会区分有宽泛契约（**wild contracts**）和严格契约（**narrow contracts**）的函数。有宽泛契约的函数没有前置条件。这种函数不管程序状态如何都能调用，它对调用者传来的实参不设约束。（“不管程序状态如何”和“不设约束”对已经行为未定义的程序无效。比如`std::vector::size`有宽泛契约，但是并不保证如果你把一块随机内存转换为一个`std::vector`，在这块内存上调用它会有合理的表现。转换的结果是未定义的，所以包含这个转换的程序也无法保证表现合理）宽泛契约的函数决不表现出未定义行为。

反之，没有宽泛契约的函数就有严格契约。对于这些函数，如果违反前置条件，结果将会是未定义的。

如果你写了一个有宽泛契约的函数并且你知道它不会抛异常，那么遵循这个给它声明一个`noexcept`是很容易的。对于严格契约的函数，情况就有点微妙了。举个例子，假如你在写一个形参为`std::string`的函数`f`，并且假定这个函数`f`很自然的决不引发异常。这就表明`f`应该被声明为`noexcept`。

现在假如`f`有一个前置条件：类型为`std::string`的参数的长度不能超过32个字符。如果现在调用`f`并传给它一个大于32字符的`std::string`，函数行为将是未定义的，因为**根据定义**违反了前置条件，导致了未定义行为。`f`没有义务去检查前置条件，它假设这些前置条件都是满足的。（调用者有责任确保参数字符不超过32字符等这些假设有效。）即使有前置条件，将`f`声明为`noexcept`似乎也是合适的：
```cpp
void f(const std::string& s) noexcept;  //前置条件：
                                        //s.length() <= 32 
```
假定`f`的实现者在函数里面检查前置条件冲突。虽然检查是没有必要的，但是也没禁止这么做，检查前置条件可能也是有用的，比如在系统测试时。debug一个抛出的异常一般都比跟踪未定义行为起因更容易。那么怎么报告前置条件冲突使得测试工具或客户端错误处理程序能检测到它呢？简单直接的做法是抛出“precondition was violated”异常，但是如果`f`声明了`noexcept`，这就行不通了；抛出一个异常会导致程序终止。因为这个原因，区分严格/宽泛契约库设计者一般会将`noexcept`留给宽泛契约函数。

作为结束语，让我详细说明一下之前的观察，即编译器不会为函数实现和异常规范提供一致性保障。考虑下面的代码，它是完全正确的：
```cpp
void setup();           //函数定义另在一处
void cleanup();

void doWork() noexcept
{
 setup();               //设置要做的工作
 …                      //真实工作
 cleanup();             //执行清理动作
}
```
这里，`doWork`声明为`noexcept`，即使它调用了non-`noexcept`函数`setup`和`cleanup`。看起来有点矛盾，其实可以猜想`setup`和`cleanup`在文档上写明了它们决不抛出异常，即使它们没有写上`noexcept`。至于为什么明明不抛异常却不写`noexcept`也是有合理原因的。比如，它们可能是用C写的库函数的一部分。（即使一些函数从C标准库移动到了`std`命名空间，也可能缺少异常规范，`std::strlen`就是一个例子，它没有声明`noexcept`。）或者它们可能是C++98库的一部分，它们不使用C++98异常规范，到了C++11还没有修订。

因为有很多合理原因解释为什么`noexcept`依赖于缺少`noexcept`保证的函数，所以C++允许这些代码，编译器一般也不会给出warnings。

**请记住：**

+ `noexcept`是函数接口的一部分，这意味着调用者可能会依赖它
+ `noexcept`函数较之于non-`noexcept`函数更容易优化
+ `noexcept`对于移动语义，`swap`，内存释放函数和析构函数非常有用
+ 大多数函数是异常中立的（译注：可能抛也可能不抛异常）而不是`noexcept`
## 十五：尽可能的使用`constexpr`
**Item 15: Use `constexpr` whenever possible**

如果要给C++11颁一个“最令人困惑新词”奖，`constexpr`十有八九会折桂。当用于对象上面，它本质上就是`const`的加强形式，但是当它用于函数上，意思就大不相同了。有必要消除困惑，因为你绝对会用它的，特别是当你发现`constexpr` “正合吾意”的时候。

从概念上来说，`constexpr`表明一个值不仅仅是常量，还是编译期可知的。这个表述并不全面，因为当`constexpr`被用于函数的时候，事情就有一些细微差别了。为了避免我毁了结局带来的surprise，我现在只想说，你不能假设`constexpr`函数的结果是`const`，也不能保证它们的（译注：返回）值是在编译期可知的。最有意思的是，这些是**特性**。关于`constexpr`函数返回的结果不需要是`const`，也不需要编译期可知这一点是**良好的**行为！

不过还是先从`constexpr`对象开始说起。这些对象，实际上，和`const`一样，它们是编译期可知的。（技术上来讲，它们的值在翻译期（translation）决议，所谓翻译不仅仅包含是编译（compilation）也包含链接（linking），除非你准备写C++的编译器和链接器，否则这些对你不会造成影响，所以你编程时无需担心，把这些`constexpr`对象值看做编译期决议也无妨的。）

编译期可知的值“享有特权”，它们可能被存放到只读存储空间中。对于那些嵌入式系统的开发者，这个特性是相当重要的。更广泛的应用是“其值编译期可知”的常量整数会出现在需要“整型常量表达式（**integral constant expression**）的上下文中，这类上下文包括数组大小，整数模板参数（包括`std::array`对象的长度），枚举名的值，对齐修饰符（译注：[`alignas(val)`](https://en.cppreference.com/w/cpp/language/alignas)），等等。如果你想在这些上下文中使用变量，你一定会希望将它们声明为`constexpr`，因为编译器会确保它们是编译期可知的：

```cpp
int sz;                             //non-constexpr变量
…
constexpr auto arraySize1 = sz;     //错误！sz的值在
                                    //编译期不可知
std::array<int, sz> data1;          //错误！一样的问题
constexpr auto arraySize2 = 10;     //没问题，10是
                                    //编译期可知常量
std::array<int, arraySize2> data2;  //没问题, arraySize2是constexpr
```
注意`const`不提供`constexpr`所能保证之事，因为`const`对象不需要在编译期初始化它的值。
 ```cpp
int sz;                            //和之前一样
…
const auto arraySize = sz;         //没问题，arraySize是sz的const复制
std::array<int, arraySize> data;   //错误，arraySize值在编译期不可知
 ```

 简而言之，所有`constexpr`对象都是`const`，但不是所有`const`对象都是`constexpr`。如果你想编译器保证一个变量有一个值，这个值可以放到那些需要编译期常量（compile-time constants）的上下文的地方，你需要的工具是`constexpr`而不是`const`。

涉及到`constexpr`函数时，`constexpr`对象的使用情况就更有趣了。如果实参是编译期常量，这些函数将产出编译期常量；如果实参是运行时才能知道的值，它们就将产出运行时值。这听起来就像你不知道它们要做什么一样，那么想是错误的，请这么看：
+ `constexpr`函数可以用于需求编译期常量的上下文。如果你传给`constexpr`函数的实参在编译期可知，那么结果将在编译期计算。如果实参的值在编译期不知道，你的代码就会被拒绝。
+ 当一个`constexpr`函数被一个或者多个编译期不可知值调用时，它就像普通函数一样，运行时计算它的结果。这意味着你不需要两个函数，一个用于编译期计算，一个用于运行时计算。`constexpr`全做了。

假设需要一个数据结构来存储一个实验的结果，而这个实验可能以各种方式进行。实验期间风扇转速，温度等等都可能导致亮度值改变，亮度值可以是高，低，或者无。如果有**n**个实验相关的环境条件，它们每一个都有三个状态，最终可以得到的组合有3<sup>n</sup>个。储存所有实验结果的所有组合需要足够存放3<sup>n</sup>个值的数据结构。假设每个结果都是`int`并且**n**是编译期已知的（或者可以被计算出的），一个`std::array`是一个合理的选择。需要一个方法在编译期计算3<sup>n</sup>。C++标准库提供了`std::pow`，它的数学功能正是所需要的，但是，对来说，这里还有两个问题。第一，`std::pow`是为浮点类型设计的，需要整型结果。第二，`std::pow`不是`constexpr`（即，不保证使用编译期可知值调用而得到编译期可知的结果），所以不能用它作为`std::array`的大小。

幸运的是，可以应需写个`pow`。我将展示怎么快速完成它，不过现在让先看看它应该怎么被声明和使用：
```cpp
constexpr                                   //pow是绝不抛异常的
int pow(int base, int exp) noexcept         //constexpr函数
{
 …                                          //实现在下面
}
constexpr auto numConds = 5;                //（上面例子中）条件的个数
std::array<int, pow(3, numConds)> results;  //结果有3^numConds个元素
```
回忆下`pow`前面的`constexpr`不表明`pow`返回一个`const`值，它只说了如果`base`和`exp`是编译期常量，`pow`的值可以被当成编译期常量使用。如果`base`和/或`exp`不是编译期常量，`pow`结果将会在运行时计算。这意味着`pow`不止可以用于像`std::array`的大小这种需要编译期常量的地方，它也可以用于运行时环境：
```cpp
auto base = readFromDB("base");     //运行时获取这些值
auto exp = readFromDB("exponent"); 
auto baseToExp = pow(base, exp);    //运行时调用pow函数
```
因为`constexpr`函数必须能在编译期值调用的时候返回编译期结果，就必须对它的实现施加一些限制。这些限制在C++11和C++14标准间有所出入。

C++11中，`constexpr`函数的代码不超过一行语句：一个`return`。听起来很受限，但实际上有两个技巧可以扩展`constexpr`函数的表达能力。第一，使用三元运算符“`?:`”来代替`if`-`else`语句，第二，使用递归代替循环。因此`pow`可以像这样实现：
```cpp
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```
这样没问题，但是很难想象除了使用函数式语言的程序员外会觉得这样硬核的编程方式更好。在C++14中，`constexpr`函数的限制变得非常宽松了，所以下面的函数实现成为了可能：
```cpp
constexpr int pow(int base, int exp) noexcept   //C++14
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    
    return result;
}
```
`constexpr`函数限制为只能获取和返回**字面值类型**，这基本上意味着那些有了值的类型能在编译期决定。在C++11中，除了`void`外的所有内置类型，以及一些用户定义类型都可以是字面值类型，因为构造函数和其他成员函数可能是`constexpr`：

```cpp
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal)
    {}

    constexpr double xValue() const noexcept { return x; } 
    constexpr double yValue() const noexcept { return y; }

    void setX(double newX) noexcept { x = newX; }
    void setY(double newY) noexcept { y = newY; }

private:
    double x, y;
};
```
`Point`的构造函数可被声明为`constexpr`，因为如果传入的参数在编译期可知，`Point`的数据成员也能在编译器可知。因此这样初始化的`Point`就能为`constexpr`：

```cpp
constexpr Point p1(9.4, 27.7);  //没问题，constexpr构造函数
                                //会在编译期“运行”
constexpr Point p2(28.8, 5.3);  //也没问题
```
类似的，`xValue`和`yValue`的*getter*（取值器）函数也能是`constexpr`，因为如果对一个编译期已知的`Point`对象（如一个`constexpr` `Point`对象）调用*getter*，数据成员`x`和`y`的值也能在编译期知道。这使得可以写一个`constexpr`函数，里面调用`Point`的*getter*并初始化`constexpr`的对象：
```cpp
constexpr
Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { (p1.xValue() + p2.xValue()) / 2,   //调用constexpr
             (p1.yValue() + p2.yValue()) / 2 }; //成员函数
}
constexpr auto mid = midpoint(p1, p2);      //使用constexpr函数的结果
                                            //初始化constexpr对象
```
这太令人激动了。它意味着`mid`对象通过调用构造函数，*getter*和非成员函数来进行初始化过程就能在只读内存中被创建出来！它也意味着你可以在模板实参或者需要枚举名的值的表达式里面使用像`mid.xValue() * 10`的表达式！（因为`Point::xValue`返回`double`，`mid.xValue() * 10`也是个`double`。浮点数类型不可被用于实例化模板或者说明枚举名的值，但是它们可以被用来作为产生整数值的大表达式的一部分。比如，`static_cast<int>(mid.xValue() * 10)`可以被用来实例化模板或者说明枚举名的值。）它也意味着以前相对严格的编译期完成的工作和运行时完成的工作的界限变得模糊，一些传统上在运行时的计算过程能并入编译时。越多这样的代码并入，你的程序就越快。（然而，编译会花费更长时间）

在C++11中，有两个限制使得`Point`的成员函数`setX`和`setY`不能声明为`constexpr`。第一，它们修改它们操作的对象的状态， 并且在C++11中，`constexpr`成员函数是隐式的`const`。第二，它们有`void`返回类型，`void`类型不是C++11中的字面值类型。这两个限制在C++14中放开了，所以C++14中`Point`的*setter*（赋值器）也能声明为`constexpr`：
```cpp
class Point {
public:
    …
    constexpr void setX(double newX) noexcept { x = newX; } //C++14
    constexpr void setY(double newY) noexcept { y = newY; } //C++14
    …
};
```
现在也能写这样的函数：
```cpp
//返回p相对于原点的镜像
constexpr Point reflection(const Point& p) noexcept
{
    Point result;                   //创建non-const Point
    result.setX(-p.xValue());       //设定它的x和y值
    result.setY(-p.yValue());
    return result;                  //返回它的副本
}
```
客户端代码可以这样写：
```cpp
constexpr Point p1(9.4, 27.7);          //和之前一样
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);

constexpr auto reflectedMid =         //reflectedMid的值
    reflection(mid);                  //(-19.1, -16.5)在编译期可知
```
本的建议是尽可能的使用`constexpr`，现在我希望大家已经明白缘由：`constexpr`对象和`constexpr`函数可以使用的范围比non-`constexpr`对象和函数大得多。使用`constexpr`关键字可以最大化你的对象和函数可以使用的场景。

还有个重要的需要注意的是`constexpr`是对象和函数接口的一部分。加上`constexpr`相当于宣称“我能被用在C++要求常量表达式的地方”。如果你声明一个对象或者函数是`constexpr`，客户端程序员就可能会在那些场景中使用它。如果你后面认为使用`constexpr`是一个错误并想移除它，你可能造成大量客户端代码不能编译。（为了debug或者性能优化而添加I/O到一个函数中这样简单的动作可能就导致这样的问题，因为I/O语句一般不被允许出现在`constexpr`函数里）“尽可能”的使用`constexpr`表示你需要长期坚持对某个对象或者函数施加这种限制。

**请记住：**

+ `constexpr`对象是`const`，它被在编译期可知的值初始化
+ 当传递编译期可知的值时，`constexpr`函数可以产出编译期可知的结果
+ `constexpr`对象和函数可以使用的范围比non-`constexpr`对象和函数要大
+ `constexpr`是对象和函数接口的一部分
## 十六：让`const`成员函数线程安全
**Item 16: Make `const` member functions thread safe**

如果在数学领域中工作，就会发现用一个类表示多项式是很方便的。在这个类中，使用一个函数来计算多项式的根是很有用的，也就是多项式的值为零的时候。这样的一个函数它不会更改多项式。所以，它自然被声明为`const`函数。

```c++
class Polynomial {
public:
    using RootsType =           //数据结构保存多项式为零的值
          std::vector<double>;  //（“using” 的信息查看9）
    …
    RootsType roots() const;
    …
};
```

计算多项式的根是很复杂的，因此如果不需要的话，就不做。如果必须做，肯定不想再做第二次。所以，如果必须计算它们，就缓存多项式的根，然后实现`roots`来返回缓存的值。下面是最基本的实现：

```c++
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        if (!rootsAreValid) {               //如果缓存不可用
            …                               //计算根
                                            //用rootVals存储它们
            rootsAreValid = true;
        }
        
        return rootVals;
    }
    
private:
    mutable bool rootsAreValid{ false };    //初始化器（initializer）的
    mutable RootsType rootVals{};           //更多信息请查看7
};
```

从概念上讲，`roots`并不改变它所操作的`Polynomial`对象。但是作为缓存的一部分，它也许会改变`rootVals`和`rootsAreValid`的值。这就是`mutable`的经典使用样例，这也是为什么它是数据成员声明的一部分。

假设现在有两个线程同时调用`Polynomial`对象的`roots`方法:

```c++
Polynomial p;
…

/*------ Thread 1 ------*/      /*-------- Thread 2 --------*/
auto rootsOfp = p.roots();      auto valsGivingZero = p.roots();
```

这些用户代码是非常合理的。`roots`是`const`成员函数，那就表示着它是一个读操作。在没有同步的情况下，让多个线程执行读操作是安全的。它最起码应该做到这点。在本例中却没有做到线程安全。因为在`roots`中，这些线程中的一个或两个可能尝试修改成员变量`rootsAreValid`和`rootVals`。这就意味着在没有同步的情况下，这些代码会有不同的线程读写相同的内存，这就是数据竞争（*data race*）的定义。这段代码的行为是未定义的。

问题就是`roots`被声明为`const`，但不是线程安全的。`const`声明在C++11中与在C++98中一样正确（检索多项式的根并不会更改多项式的值），因此需要纠正的是线程安全的缺乏。

解决这个问题最普遍简单的方法就是——使用`mutex`（互斥量）：
```c++
class Polynomial {
public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);       //锁定互斥量
        
        if (!rootsAreValid) {                   //如果缓存无效
            …                                   //计算/存储根值
            rootsAreValid = true;
        }
        
        return rootsVals;
    }                                           //解锁互斥量
    
private:
    mutable std::mutex m;
    mutable bool rootsAreValid { false };
    mutable RootsType rootsVals {};
};
```

`std::mutex m`被声明为`mutable`，因为锁定和解锁它的都是non-`const`成员函数。在`roots`（`const`成员函数）中，`m`却被视为`const`对象。

值得注意的是，因为`std::mutex`是一种只可移动类型（*move-only type*，一种可以移动但不能复制的类型），所以将`m`添加进`Polynomial`中的副作用是使`Polynomial`失去了被复制的能力。不过，它仍然可以移动。

在某些情况下，互斥量的副作用显会得过大。例如，如果你所做的只是计算成员函数被调用了多少次，使用`std::atomic` 修饰的计数器（保证其他线程视它的操作为不可分割的整体，参见[item40](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/item40.md)）通常会是一个开销更小的方法。（然而它是否轻量取决于你使用的硬件和标准库中互斥量的实现。）以下是如何使用`std::atomic`来统计调用次数。

```c++
class Point {                                   //2D点
public:
    …
    double distanceFromOrigin() const noexcept  //noexcept的使用
    {                                           //参考14
        ++callCount;                            //atomic的递增
        
        return std::sqrt((x * x) + (y * y));
    }

private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
};
```

与`std::mutex`一样，`std::atomic`是只可移动类型，所以在`Point`中存在`callCount`就意味着`Point`也是只可移动的。

因为对`std::atomic`变量的操作通常比互斥量的获取和释放的消耗更小，所以你可能会过度倾向与依赖`std::atomic`。例如，在一个类中，缓存一个开销昂贵的`int`，你就会尝试使用一对`std::atomic`变量而不是互斥量。

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;              //第一步
            cacheValid = true;                      //第二步
            return cachedValid;
        }
    }
    
private:
    mutable std::atomic<bool> cacheValid{ false };
    mutable std::atomic<int> cachedValue;
};
```

这是可行的，但难以避免有时出现重复计算的情况。考虑：

+ 一个线程调用`Widget::magicValue`，将`cacheValid`视为`false`，执行这两个昂贵的计算，并将它们的和分配给`cachedValue`。
+ 此时，第二个线程调用`Widget::magicValue`，也将`cacheValid`视为`false`，因此执行刚才完成的第一个线程相同的计算。（这里的“第二个线程”实际上可能是其他**几个**线程。）

这种行为与使用缓存的目的背道而驰。将`cachedValue`和`CacheValid`的赋值顺序交换可以解决这个问题，但结果会更糟：

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cacheValid = true;                      //第一步
            return cachedValue = val1 + val2;       //第二步
        }
    }
    …
}
```

假设`cacheValid`是false，那么：

+ 一个线程调用`Widget::magicValue`，刚执行完将`cacheValid`设置`true`的语句。
+ 在这时，第二个线程调用`Widget::magicValue`，检查`cacheValid`。看到它是`true`，就返回`cacheValue`，即使第一个线程还没有给它赋值。因此返回的值是不正确的。

这里有一个坑。对于需要同步的是单个的变量或者内存位置，使用`std::atomic`就足够了。不过，一旦你需要对两个以上的变量或内存位置作为一个单元来操作的话，就应该使用互斥量。对于`Widget::magicValue`是这样的。

```c++
class Widget {
public:
    …
    int magicValue() const
    {
        std::lock_guard<std::mutex> guard(m);   //锁定m
        
        if (cacheValid) return cachedValue;
        else {
            auto val1 = expensiveComputation1();
            auto val2 = expensiveComputation2();
            cachedValue = val1 + val2;
            cacheValid = true;
            return cachedValue;
        }
    }                                           //解锁m
    …

private:
    mutable std::mutex m;
    mutable int cachedValue;                    //不再用atomic
    mutable bool cacheValid{ false };           //不再用atomic
};

```

现在，这个是基于，多个线程可以同时在一个对象上执行一个`const`成员函数这个假设的。如果你不是在这种情况下编写一个`const`成员函数——你可以**保证**在一个对象上永远不会有多个线程执行该成员函数——该函数的线程安全是无关紧要的。比如，为独占单线程使用而设计的类的成员函数是否线程安全并不重要。在这种情况下，你可以避免因使用互斥量和`std::atomics`所消耗的资源，以及包含它们的类只能使用移动语义带来的副作用。然而，这种线程无关的情况越来越少见，而且很可能会越来越少。可以肯定的是，`const`成员函数应支持并发执行，这就是为什么你应该确保`const`成员函数是线程安全的。

**请记住：**

+ 确保`const`成员函数线程安全，除非你**确定**它们永远不会在并发上下文（*concurrent context*）中使用。
+ 使用`std::atomic`变量可能比互斥量提供更好的性能，但是它只适合操作单个变量或内存位置。
## 十七：理解特殊成员函数的生成
**Item 17: Understand special member function generation**

在C++术语中，**特殊成员函数**是指C++自己生成的函数。C++98有四个：默认构造函数，析构函数，拷贝构造函数，拷贝赋值运算符。当然在这里有些细则要注意。这些函数仅在需要的时候才生成，比如某个代码使用它们但是它们没有在类中明确声明。默认构造函数仅在类完全没有构造函数的时候才生成。（防止编译器为某个类生成构造函数，但是你希望那个构造函数有参数）生成的特殊成员函数是隐式public且`inline`，它们是非虚的，除非相关函数是在派生类中的析构函数，派生类继承了有虚析构函数的基类。在这种情况下，编译器为派生类生成的析构函数是虚的。

但是你早就知道这些了。好吧好吧，都说古老的历史：美索不达米亚，商朝，FORTRAN，C++98。但是时代改变了，C++生成特殊成员的规则也改变了。要留意这些新规则，知道什么时候编译器会悄悄地向你的类中添加成员函数，因为没有什么比这件事对C++高效编程更重要。

C++11特殊成员函数俱乐部迎来了两位新会员：移动构造函数和移动赋值运算符。它们的签名是：
```cpp
class Widget {
public:
    …
    Widget(Widget&& rhs);               //移动构造函数
    Widget& operator=(Widget&& rhs);    //移动赋值运算符
    …
};
```
掌控它们生成和行为的规则类似于拷贝系列。移动操作仅在需要的时候生成，如果生成了，就会对类的non-static数据成员执行逐成员的移动。那意味着移动构造函数根据`rhs`参数里面对应的成员移动构造出新non-static部分，移动赋值运算符根据参数里面对应的non-static成员移动赋值。移动构造函数也移动构造基类部分（如果有的话），移动赋值运算符也是移动赋值基类部分。

现在，当我对一个数据成员或者基类使用移动构造或者移动赋值时，没有任何保证移动一定会真的发生。逐成员移动，实际上，更像是逐成员移动**请求**，因为对**不可移动类型**（即对移动操作没有特殊支持的类型，比如大部分C++98传统类）使用“移动”操作实际上执行的是拷贝操作。逐成员移动的核心是对对象使用`std::move`，然后函数决议时会选择执行移动还是拷贝操作。[Item23](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item23.md)包括了这个操作的细节。本中，简单记住如果支持移动就会逐成员移动类成员和基类成员，如果不支持移动就执行拷贝操作就好了。

像拷贝操作情况一样，如果你自己声明了移动操作，编译器就不会生成。然而它们生成的精确条件与拷贝操作的条件有点不同。

两个拷贝操作是独立的：声明一个不会限制编译器生成另一个。所以如果你声明一个拷贝构造函数，但是没有声明拷贝赋值运算符，如果写的代码用到了拷贝赋值，编译器会帮助你生成拷贝赋值运算符。同样的，如果你声明拷贝赋值运算符但是没有拷贝构造函数，代码用到拷贝构造函数时编译器就会生成它。上述规则在C++98和C++11中都成立。

两个移动操作不是相互独立的。如果你声明了其中一个，编译器就不再生成另一个。如果你给类声明了，比如，一个移动构造函数，就表明对于移动操作应怎样实现，与编译器应生成的默认逐成员移动有些区别。如果逐成员移动构造有些问题，那么逐成员移动赋值同样也可能有问题。所以声明移动构造函数阻止移动赋值运算符的生成，声明移动赋值运算符同样阻止编译器生成移动构造函数。

再进一步，如果一个类显式声明了拷贝操作，编译器就不会生成移动操作。这种限制的解释是如果声明拷贝操作（构造或者赋值）就暗示着平常拷贝对象的方法（逐成员拷贝）不适用于该类，编译器会明白如果逐成员拷贝对拷贝操作来说不合适，逐成员移动也可能对移动操作来说不合适。

这是另一个方向。声明移动操作（构造或赋值）使得编译器禁用拷贝操作。（编译器通过给拷贝操作加上*delete*来保证，参见[Item11](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item11.md)。）（译注：禁用的是自动生成的拷贝操作，对于用户声明的拷贝操作不受影响）毕竟，如果逐成员移动对该类来说不合适，也没有理由指望逐成员拷贝操作是合适的。听起来会破坏C++98的某些代码，因为C++11中拷贝操作可用的条件比C++98更受限，但事实并非如此。C++98的代码没有移动操作，因为C++98中没有移动对象这种概念。只有一种方法能让老代码使用用户声明的移动操作，那就是使用C++11标准然后添加这些操作，使用了移动语义的类必须接受C++11特殊成员函数生成规则的限制。

也许你早已听过_Rule of Three_规则。这个规则告诉如果你声明了拷贝构造函数，拷贝赋值运算符，或者析构函数三者之一，你应该也声明其余两个。它来源于长期的观察，即用户接管拷贝操作的需求几乎都是因为该类会做其他资源的管理，这也几乎意味着（1）无论哪种资源管理如果在一个拷贝操作内完成，也应该在另一个拷贝操作内完成（2）类的析构函数也需要参与资源的管理（通常是释放）。通常要管理的资源是内存，这也是为什么标准库里面那些管理内存的类（如会动态内存管理的STL容器）都声明了“*the big three*”：拷贝构造，拷贝赋值和析构。

*Rule of Three*带来的后果就是只要出现用户定义的析构函数就意味着简单的逐成员拷贝操作不适用于该类。那意味着如果一个类声明了析构，拷贝操作可能不应该自动生成，因为它们做的事情可能是错误的。在C++98提出的时候，上述推理没有得倒足够的重视，所以C++98用户声明析构函数不会左右编译器生成拷贝操作的意愿。C++11中情况仍然如此，但仅仅是因为限制拷贝操作生成的条件会破坏老代码。

*Rule of Three*规则背后的解释依然有效，再加上对声明拷贝操作阻止移动操作隐式生成的观察，使得C++11不会为那些有用户定义的析构函数的类生成移动操作。

所以仅当下面条件成立时才会生成移动操作（当需要时）：

+ 类中没有拷贝操作
+ 类中没有移动操作
+ 类中没有用户定义的析构

有时，类似的规则也会扩展至拷贝操作上面，C++11抛弃了已声明拷贝操作或析构函数的类的拷贝操作的自动生成。这意味着如果你的某个声明了析构或者拷贝的类依赖自动生成的拷贝操作，你应该考虑升级这些类，消除依赖。假设编译器生成的函数行为是正确的（即逐成员拷贝类non-static数据是你期望的行为），你的工作很简单，C++11的`= default`就可以表达你想做的：
```cpp
class Widget {
    public:
    … 
    ~Widget();                              //用户声明的析构函数
    …                                       //默认拷贝构造函数
    Widget(const Widget&) = default;        //的行为还可以

    Widget&                                 //默认拷贝赋值运算符
        operator=(const Widget&) = default; //的行为还可以
    … 
};
```
这种方法通常在多态基类中很有用，即通过操作的是哪个派生类对象来定义接口。多态基类通常有一个虚析构函数，因为如果它们非虚，一些操作（比如通过一个基类指针或者引用对派生类对象使用`delete`或者`typeid`）会产生未定义或错误结果。除非类继承了一个已经是*virtual*的析构函数，否则要想析构函数为虚函数的唯一方法就是加上`virtual`关键字。通常，默认实现是对的，`= default`是一个不错的方式表达默认实现。然而用户声明的析构函数会抑制编译器生成移动操作，所以如果该类需要具有移动性，就为移动操作加上`= default`。声明移动会抑制拷贝生成，所以如果拷贝性也需要支持，再为拷贝操作加上`= default`：
```cpp
class Base {
public:
    virtual ~Base() = default;              //使析构函数virtual
    
    Base(Base&&) = default;                 //支持移动
    Base& operator=(Base&&) = default;
    
    Base(const Base&) = default;            //支持拷贝
    Base& operator=(const Base&) = default;
    … 
};
```
实际上，就算编译器乐于为你的类生成拷贝和移动操作，生成的函数也如你所愿，你也应该手动声明它们然后加上`= default`。这看起来比较多余，但是它让你的意图更明确，也能帮助你避免一些微妙的bug。比如，你有一个类来表示字符串表，即一种支持使用整数ID快速查找字符串值的数据结构：
```cpp
class StringTable {
public:
    StringTable() {}
    …                   //插入、删除、查找等函数，但是没有拷贝/移动/析构功能
private:
    std::map<int, std::string> values;
};
```
假设这个类没有声明拷贝操作，没有移动操作，也没有析构，如果它们被用到编译器会自动生成。没错，很方便。

后来需要在对象构造和析构中打日志，增加这种功能很简单：
```cpp
class StringTable {
public:
    StringTable()
    { makeLogEntry("Creating StringTable object"); }    //增加的

    ~StringTable()                                      //也是增加的
    { makeLogEntry("Destroying StringTable object"); }
    …                                               //其他函数同之前一样
private:
    std::map<int, std::string> values;              //同之前一样
};
```
看起来合情合理，但是声明析构有潜在的副作用：它阻止了移动操作的生成。然而，拷贝操作的生成是不受影响的。因此代码能通过编译，运行，也能通过功能（译注：即打日志的功能）测试。功能测试也包括移动功能，因为即使该类不支持移动操作，对该类的移动请求也能通过编译和运行。这个请求正如之前提到的，会转而由拷贝操作完成。它意味着对`StringTable`对象的移动实际上是对对象的拷贝，即拷贝里面的`std::map<int, std::string>`对象。拷贝`std::map<int, std::string>`对象很可能比移动慢**几个数量级**。简单的加个析构就引入了极大的性能问题！对拷贝和移动操作显式加个`= default`，问题将不再出现。

受够了我喋喋不休的讲述C++11拷贝移动规则了吧，你可能想知道什么时候我才会把注意力转入到剩下两个特殊成员函数，默认构造函数和析构函数。现在就是时候了，但是只有一句话，因为它们几乎没有改变：它们在C++98中是什么样，在C++11中就是什么样。

C++11对于特殊成员函数处理的规则如下：

+ **默认构造函数**：和C++98规则相同。仅当类不存在用户声明的构造函数时才自动生成。
+ **析构函数**：基本上和C++98相同；稍微不同的是现在析构默认`noexcept`（参见[Item14](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/3.MovingToModernCpp/item14.md)）。和C++98一样，仅当基类析构为虚函数时该类析构才为虚函数。
+ **拷贝构造函数**：和C++98运行时行为一样：逐成员拷贝non-static数据。仅当类没有用户定义的拷贝构造时才生成。如果类声明了移动操作它就是*delete*的。当用户声明了拷贝赋值或者析构，该函数自动生成已被废弃。
+ **拷贝赋值运算符**：和C++98运行时行为一样：逐成员拷贝赋值non-static数据。仅当类没有用户定义的拷贝赋值时才生成。如果类声明了移动操作它就是*delete*的。当用户声明了拷贝构造或者析构，该函数自动生成已被废弃。
+ **移动构造函数**和**移动赋值运算符**：都对非static数据执行逐成员移动。仅当类没有用户定义的拷贝操作，移动操作或析构时才自动生成。

注意没有“成员函数**模版**阻止编译器生成特殊成员函数”的规则。这意味着如果`Widget`是这样：
```cpp
class Widget {
    …
    template<typename T>                //从任何东西构造Widget
    Widget(const T& rhs);

    template<typename T>                //从任何东西赋值给Widget
    Widget& operator=(const T& rhs);
    …
};
```
编译器仍会生成移动和拷贝操作（假设正常生成它们的条件满足），即使可以模板实例化产出拷贝构造和拷贝赋值运算符的函数签名。（当`T`为`Widget`时。）很可能你会觉得这是一个不值得承认的边缘情况，但是我提到它是有道理的，[Item26](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/5.RRefMovSemPerfForw/item26.md)将会详细讨论它可能带来的后果。

**请记住：**

+ 特殊成员函数是编译器可能自动生成的函数：默认构造函数，析构函数，拷贝操作，移动操作。
+ 移动操作仅当类没有显式声明移动操作，拷贝操作，析构函数时才自动生成。
+ 拷贝构造函数仅当类没有显式声明拷贝构造函数时才自动生成，并且如果用户声明了移动操作，拷贝构造就是*delete*。拷贝赋值运算符仅当类没有显式声明拷贝赋值运算符时才自动生成，并且如果用户声明了移动操作，拷贝赋值运算符就是*delete*。当用户声明了析构函数，拷贝操作的自动生成已被废弃。
+ 成员函数模板不抑制特殊成员函数的生成。
