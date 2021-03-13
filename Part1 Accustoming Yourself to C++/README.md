# Part1 让自己习惯C++

## 条款01：视C++为一个语言联邦

View C++ as a federation of languages

C++已经是个**多重范型编程语言**(multiparadigm programming
language)，一个同时支持**过程形式**(**procedural**) 、**面向对象形式**(**object-oriented**)、**函数形式**(**functional**)、**泛型形式**(**generic**)、**元编程形式**(**metaprogramming**)的语言。

最简单的方法是将C++视为一个由相关语言组成的联邦而非单一语言。

1. C。说到底C++仍是以C为基础。
2. Object-Oriented C++。面向对象设计：classes（包括构造函数和析构函数），封装(encapsulation)、继承(inheritance)、多态(polymorphism)、virtual函数（动态绑定）等等。
3. Template C++。模板：属于C++**泛型编程**(**generic programming**)。
4. STL。STL是个template程序库。

C++高效编程守则视状况而变化，取决于你使用C++的哪一部分。

## 条款02：尽量以const, enum, inline替换#define

Perfer consts, enums, inlines to #defines.

当定义一个宏时：`#define ASPECT_RATIO 1.653`，记号名称`ASPECT_RATIO`也许从未被编译器看见；也许在编译器开始处理源码之前它就被预处理器移走了。于是记号名称`ASPECT_RATIO`有可能没进入**记号表**(**symbol table**)内。于是当你运用此常量但获得一个编译错误信息时，可能会带来困惑，因为这个错误信息也许会提到1.653而不是`ASPECT_RATIO`。

这个问题也可能出现在**记号式调试器**(**symbolic debugger**)中，原因相同：你所使用的名称可能并未进入**记号表**(**symbol table**)。

解决之道是以一个常量替换上述的宏(#define):

```C++
   const double AspectRatio = 1.653; // 大写名称通常用于宏
```

当我们以常量替换`#defines`，有两种特殊情况值得说说：

1. 第一是定义**常量指针**(**constant pointers**)。
2. 第二个值得注意的是class专属常量。**为了将常量的作用域(scope)限制于class内，你必须让它成为class的一个成员(member)；而为确保此常量至多只有一份实体，你必须让它成为一个static成员**：

```C++
    class GamePlayer{
    private:
        static const int NumTurns = 5;  // 常量声明式
        int scores[NumTurns];           // 使用该常量
    };
```

然而你所看到的是`NumTurns`的声明式曲非定义式。

通常C++要求你对你所使用的任何东西提供个定义式，但**如果它是个class专属常量又是static且为整数类型**(integral type, 例如ints, chars, bools)，则需特殊处理。

只要不取它们的地址，你可以声明并使用它们而无须提供定义式。但如果你取某个class专属常量的地址，或纵使你不取其地址而你的编译器却（不正确地）坚持要看到一个定义式，你就必须另外提供定义式如下：

```C++
    const int GamePlayer::NumTurns; // NumTurns的定义式
                                    // 下面阐述为什么没有给予数值
```

请把这个式子放进一个实现文件而非头文件。由于class常量已在声明时获得初值（例如先前声明NumTurns时为它设初值5)，因此定义时不可以再设初值。

偶尔旧式编译器不允许上述语法，它们不允许`static`成员在其声明式上获得初值。此外所谓的“in-class初值设定”也只允许对整数常量进行。如果你的编译器不支持上述语法，你可以将初值放在定义式：

```C++
    class CostEstimate {
    private:
        static const double FudgeFactor;    // static class 常量声明
    }; 
    const double CostEstimate::FudgeFactor = 1.35;  // static class常量定义位于实现文件内
```

当然也存在例外，比如在class编译期间需要一个class常量值，例如在上述`GamePlayer::scores`的数组声明式中（编译器在编译期间必须要知道数组大小），则常量值不能先声明再定义。

此时，编译器不允许“static整数型class常量”完成“in-class初值设定”，可改用所谓的“the enum hack”补偿做法。其理论基础是：**一个属于枚举类型(enumerated type)的数值可权充ints被使用**，于是GamePlayer可以定义如下：

```C++
    class GamePlayer {
    private:
        enum { NumTurns = 5};
        int scores[NumTurns];
    };
```

由此对enum hack进一步认识：

1. 第一，enum hack的行为比较像#define而不像const；例如，取一个const的地址是合法的，**但是取一个enum的地址就不合法**，取一个#define的地址通常也不合法。
2. 第二，纯粹为了实用主义。

另一个常见的#define误用情况是以它实现宏(macros)。宏看起来像函数，但不会招致函数调用(function call)带来的额外开销。

下面这个宏夹带着宏实参，调用函数f：

```C++
    // 以a和b的较大值调用f
    #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

无论何时当你写出这种宏，你必须记住为宏中的所有实参加上小括号，否则某些人在表达式中调用这个宏时可能会遭遇麻烦。但纵使你为所有实参加上小括号，看看卜面不可思议的事情：

```C++
    int a = 5, b = 0;
    CALL_WITH_MAX(++a, b);  // a被累加两次
    CALL_WITH_MAX(++a, b+10);   // a被累加一次
```

但同各国template inline函数，可以获得宏带来的效率以及一般函数的所有可预料行为和类型安全性。

```C++
    template<typename T>
    inline void callWithMax(const T& a, const T& b)
    {
        f(a > b ? a : b);
    }
```

此外由于callWithMax是个真正的函数，它遵守作用域(scope)
和访问规则。

> 请记住

1. 对于单纯常量，最好以const对象或enums替换#defines。
2. 对于形似函数的宏(macros)最好改用inline函数替换#defines。

## 条款03：尽可能使用const

Use const whenever possible

