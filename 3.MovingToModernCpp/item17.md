## Item 17:理解特殊成员函数函数的生成
条款 17:理解特殊成员函数函数的生成

在C++术语中，特殊成员函数是指C++自己生成的函数。C++98有四个：默认构造函数函数，析构函数，拷贝构造函数，拷贝赋值运算符。这些函数仅在需要的时候才生成，比如某个代码使用它们但是它们没有在类中声明。默认构造函数仅在类完全没有构造函数的时候才生成。（防止编译器为某个类生成构造函数，但是你希望那个构造函数有参数）生成的特殊成员函数是隐式public且inline，除非该类是继承自某个具有虚函数的类，否则生成的析构函数是非虚的。

但是你早就知道这些了。好吧好吧，都说古老的历史：美索不达米亚，商朝，FORTRAN,C++98。但是时代改变了，C++生成特殊成员的规则也改变了。要留意这些新规则，因为用C++高效编程方面很少有像它们一样重要的东西需要知道。

C++11特殊成员函数俱乐部迎来了两位新会员：移动构造函数和移动赋值运算符。它们的签名是：
```cpp
class Widget {
public:
	...
    Widget(Widget&& rhs);
    Widget& operator=(Widget&& rhs);
	... 
};
```
掌控它们生成和行为的规则类似于拷贝系列。移动操作仅在需要的时候生成，如果生成了，就会对非static数据执行逐成员的移动。那意味着移动构造函数根据rhs参数里面对应的成员移动构造出新部分，移动赋值运算符根据参数里面对应的非static成员移动赋值。移动构造函数也移动构造基类部分（如果有的话），移动赋值运算符也是移动赋值基类部分。

现在，当我对一个数据成员或者基类使用移动构造或者移动赋值，没有任何保证移动一定会真的发生。逐成员移动，实际上，更像是逐成员移动**请求**，因为对不可移动类型使用移动操作实际上执行的是拷贝操作。逐成员移动的核心是对对象使用**std::move**，然后函数决议时会选择执行移动还是拷贝操作。**Item 23**包括了这个操作的细节。本章中，简单记住如果支持移动就会逐成员移动类成员和基类成员，如果不支持移动就执行拷贝操作就好了。

两个拷贝操作是独立的：声明一个不会限制编译器声明另一个。所以如果你声明一个拷贝构造哈说，但是没有声明拷贝赋值运算符，如果写的代码用到了拷贝赋值，编译器会帮助你生成拷贝赋值运算符重载。同样的，如果你声明拷贝赋值运算符但是没有拷贝构造，代码用到拷贝构造编译器就会生成它。上述规则在C++98和C++11中都成立。

再进一步，如果一个类显式声明了拷贝操作，编译器就不会生成移动操作。这种限制的解释是如果声明拷贝操作就暗示着默认逐成员拷贝操作不适用于该类，编译器会明白如果默认拷贝不适用于该类，移动操作也可能是不适用的。

这是另一个方向。声明移动操作使得编译器不会生成拷贝操作。（编译器通过给这些函数加上delete来保证，参见Item11）。比较，如果逐成员移动对该类来说不合适，也没有理由指望逐成员考吧操作是合适的。听起来会破坏C++98的某些代码，因为C++11中拷贝操作可用的条件比C++98更受限，但事实并非如此。C++98的代码没有移动操作，因为C++98中没有移动对象这种概念。只有一种方法能让老代码使用用户声明的移动操作，那就是使用C++11标准然后添加这些操作， 并在享受这些操作带来的好处同时接受C++11特殊成员函数生成规则的限制。

也许你早已听过_Rule of Three_规则。这个规则告诉我们如果你声明了拷贝构造函数，拷贝赋值运算符，或者析构函数三者之一，你应该也声明其余两个。它来源于长期的观察，即用户接管拷贝操作的需求几乎都是因为该类会做其他资源的管理，这也几乎意味着1）无论哪种资源管理如果能在一个拷贝操作内完成，也应该在另一个拷贝操作内完成2）类析构函数也需要参与资源的管理（通常是释放）。通常意义的资源管理指的是内存（如STL容器会动态管理内存），这也是为什么标准库里面那些管理内存的类都声明了“the big three”：拷贝构造，拷贝赋值和析构。

**Rule of Three**带来的后果就是只要出现用户定义的析构函数就意味着简单的逐成员拷贝操作不适用于该类。接着，如果一个类声明了析构也意味着拷贝操作可能不应该自定生成，因为它们做的事情可能是错误的。在C++98提出的时候，上述推理没有得倒足够的重视，所以C++98用户声明析构不会左右编译器生成拷贝操作的意愿。C++11中情况仍然如此，但仅仅是因为限制拷贝操作生成的条件会破坏老代码。