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

关键字`const`可以用在classes外部修饰`global`或`namespace`作用域中的常量，或修饰文件、函数、或区块作用域(block scope)中被声明为`static`的对象。也可以用来修饰classes内部的`static`和`non-static`的成员变量。面对指针，也可以指出指针自身，指针所指物、或者两者都（或都不）是`const`：

```C++
    char greeting[] = "Hello";
    char* p = greeting;          // non-const pointer, non-const data 
    const char* p = greeting; // non-const pointer, const data
    char* const p = greeting; // const pointer, non-const data
    const char* const p = greeting; // const pointer, const data
```

**如果关键字`const`出现在星号左边，表示被指物是常量；如果出现在星号右边，表示指针自身是常量；如果出现在星号两边，表示被指物和指针两者都是常量。**

> 区分：

1. 指向const的指针

首先是一个指针，但是这个指针所指向的是一个const类型的指针。`const int *p`或`int const *p`

第一种：首先p是一个指针，p所指向的内容(*p)是const int类型的；

第二种：首先p也是一个指针，指向的内容是const int类型的，也就是所指向的内容是绝对不能被修改的。

2. const指针

 首先是一个指针，只不过这个指针是const类型的，也就是这个指针变量的地址，只能在初始化之后，就一直指向这个地址，地址所被保存的内容是可变的。 

 `int * const p = 地址; // 因为p所指向的地址不能被修改，所以必须初始化`

首先，这个p是一个指针，而这个指针是指向了int类型的const指针。只不过地址是被固定，不能改变，但是地址所保存的数值确实可变的。

如果被指物是常量，则存在两种写法：

```C++
    void f1(const Widget* pw);      // f1获得一个指针，指向一个常量的Widget对象
    void f2(Widget cosnt * pw);     // f2也是
```

STL迭代器系以指针为根据塑模出来，所以迭代器的作用就像个`T*`指针。声明迭代器为`const`就像声明指针为`const`一样（即声明一个`T* const`指针），表示这个迭代器**不得指向不同的东西**，但它所指的东西的值是可以改动的。如果你希望迭代器所指的东西不可被改动（即希望STL模拟一个`const T*`指针），你需要的是`const_iterator`。

const最具威力的用法是面对函数声明时的应用。在一个函数声明式内，const可以和函数返回值、各参数、函数自身（如果是成员函数）产生关联。

### const成员函数

将const实施于成员函数的目的，**是为了确认该成员函数可作用于const对象身上**。这一类成员函数重要的原因有，第一，可以使class接口容易被理解，可以知道哪个函数可以改动对象内容而哪个函数不行。第二，使得“操作const对象”成为可能。

在类中，两个成员函数如果只是常量性(constness)不同，可以被重载。

```C++
    class TextBlock{
    public:
        const char& operator[](std::size_t position) const
        {return text[position];}    // operator[] for const对象
        char& operator[](std::size_t position)
        {return text[position];}    // operator[] for non-const对象
    private:
        std::string text;
    };
```

TextBlock得operator[]可被这么使用：

```C++
    TextBlock tb("Hello");
    std::cout << tb[0]; // 调用non-const
    const TextBlock ctb("World");
    std::cout << ctb[0];    // 调用const
```

只要重载operator[]并对不同的版本给予不同的返回类型，就可以令const和non-const TextBlock获得不同的处理。

成员函数如果是const意味什么？这有两个流行概念：bitwise constness(又称physical constness)和logical constness。

bitwise const阵营的人相信，成员函数只有在不更改对象的任何成员变量(static 除外）时才可以说是const。

而logical constness主张一个const成员函数可以修改它所处理的对象内的某些bits，但只有在客户侦测不出的情况下才如此。

```C++
    class CTextBlock{
    public:
        std::size_t length() const;
    private:
        char* pText;
        std::size_t textLength;
        bool lengthIsValid;
    };
    std::size_t CTextBlock::length() const
    {
        if(!lengthIsValid)
        {
            textLength = std::strlen(pText);    // 错误！在const成员函数内不能赋值给textLength和lengthIsValid
            lengthIsValid = true;
        }
        return textLength;
    }
```

编译器不允许，在const成员函数中实现上述功能，但可通过`mutable`释放掉non-static成员变量的bitwise constness约束：

```C++
    class CTextBlock{
    public:
        std::size_t length() const;
    private:
        char* pText;
        mutable std::size_t textLength; // 这些成员变量可能总会被更改，即使在cosnt成员函数内
        mutable bool lengthIsValid;
    };
    std::size_t CTextBlock::length() const
    {
        if(!lengthIsValid)
        {
            textLength = std::strlen(pText);    // 现在可以
            lengthIsValid = true;
        }
        return textLength;
    }
```

### 在const和non-const成员函数中避免重复

```C++
    class TextBlock{
    public:
        const char& operator[](std::size_t position) const
        {
            ... // 边界检查(bounds checking)
            ... // 日志数据访问(log access data)
            ... // 检验数据完整性(verify data integrity)
            return text[position];
        }
        char& operator[](std::size_t position)
        {
            ... // 边界检查(bounds checking)
            ... // 日志数据访问(log access data)
            ... // 检验数据完整性(verify data integrity)
            return text[position];
        }
    private:
        std::string text;
    };
```

为了避免在const和non-const成员函数中使用重复代码，需要实现operator[]的技能一次并使用它两次，也就是说，必须令其中一个调用另一个，促使我们将**常量性转移**(**casting away constness**)。

```C++
    class TextBlock{
    public:
        const char& operator[](std::size_t position)   const    // 一如既往
        {
            ... // 边界检查(bounds checking)
            ... // 日志数据访问(log access data)
            ... // 检验数据完整性(verify data integrity)
            return text[position];
        }
        char& operator[](std::size_t position)  // 现在只调用const op[]
        {
            return const_cast<char&>(   // 将op[]返回值的const移除
            static_cast<const TextBlock&>(*this)    // 为*this加上const
            [position]      // 调用const op[]
            );
        }
    private:
        std::string text;
    };
```

上述代码中，有两个转型动作。在`non-const operator[]`内部为了避免递归调用自己，需要明确指出调用的是`const operator[]`，因此这里将`*this`从其原始类型`TextBlock&`转型为`const TextBlock&`。第二次转型则是从`const operator[]`的返回值中移除const。

如果反向调用，即在const版本内调用non-const版本，这就会出现曾经承诺不改动的那个对象被改动了，存在const相关危险。

### 请记住

1. 将某些东西声明为`const`可帮助编译器侦测出错误用法。`const`可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。

2. 编译器强制实施bitwise constness，但你编写程序时应该使用“概念上的常量性”(conceptual constness)。

3. 当`const`和`non-const`成员函数有着实质等价的实现时，令`non-const`版本调用`const`版本可避免代码重复。

## 条款04：确定对象被使用前已先被初始化

Make sure that objects are initialized before they're used.

永远在使用对象之前先将它初始化。对于无任何成员的内置类型，你必须手工完成此事。

```C++
    int x = 0;      // 对int进行手动初始化
    const char* text = "A C-style string";  // 对指针进行手动初始化

    double d;
    std::cin >> d;      // 以读取input stream的方式完成初始化
```

至于内置类型以外的任何其他东西，初始化责任落在构造函数(constructors)身上。规则很简单：**确保每一个构造函数都将对象的每一个成员初始化**。但重要的是别混淆了**赋值**(**assignment**)和**初始化**(**initialization**)。

考虑一个通讯簿class，其构造函数如下：

```C++
    class PhoneNumber {};
    class ABEntry{  // Address book Entry
    public:
        ABEntey(const std::string& name, const std:: string& address, 
        cosnt std::list<PhoneNumber>& phones);
    private:
        std::string theName;
        std::string theAddress;
        std::list<PhoneNumber> thePhones;
        int numTimesConsulted;
    };
    ABEntry::ABEntey(const std::string& name, const std:: string& address, 
        cosnt std::list<PhoneNumber>& phones)
    {
        theName = name;     // 这些都是赋值，不是初始化
        theAddress = address;
        thePhones = phones;
        numTimesConsulted = 0;
    }
```

这会导致ABEntry对象带有你期望（你指定）的值，但不是最佳做法。C++规定，**对象的成员变量的初始化动作发生在进入构造函数本体之前**。

ABEntry构造函数的一个较佳写法是，使用所谓的member initialization list(成员初值列)替换赋值动作：

```C++
    ABEntry::ABEntey(const std::string& name, const std:: string& address, 
        cosnt std::list<PhoneNumber>& phones)
    :theName(name), thePhones(phones), theAddress(address), numTimesConsulted(0)
    {}
```

这个构造函数和上一个的最终结果相同，但通常效率较高。对大多数类型而言，比起先调用default构造函数然后再调用copy assignment操作符，单只调用一次copy构造函数是比较高效的，有时甚至高效得多。对于内置型对象如nurnTimesConsulted，其初始化和赋值的成本相同。

有些情况下即使面对的成员变量属于内置类型（那么其初始化与赋值的成本相同），也一定得使用初值列。是的，如果成员变量是const或references，它们就一定需要初值，不能被赋值。

C++有着十分固定的“成员初始化次序”，次序总是相同：base classes更早于其derived classes被初始化，而class的成员变量总是以其声明次序被初始化，即使它们在成员初值列中以不同的次序出现。

但还需要注意的是**不同编译单元内定义的non-local static对象的初始化顺序**。

所谓static对象，其寿命从被构造出来直到程序结束为止，这种对象包括global对象、定义于namespace作用域内的对象、在classes内、在函数内、以及在file作用域内被声明为static的对象。函数内的static对象称为local static对象，其他static对象成为non-local static对象。

所谓编译单元(translation unit)是指产出单一目标文件(single object file)的那些源码。基本上它是单源码文件加上其所含入的头文件(#include files)。

```C++
    class FileSystem
    {
    public:
        std::size_t numDisks() const;
    };
    extern FileSystem tfs;
```

```C++
    class Directory
    {
    public:
        Directory(params);
    };
    Directory::Directory(params)
    {
        std::size_t disks = tfs.numDisks(); // 调用tfs对象
    }
```

进一步假设，这些客户决定创建一个Directory对象，用来放置临时文件：`Directory tempDir(params);`。

现在，初始化次序的重要性显现出来了：除非tfs在tempDir之前先被初始化，否则tempDir的构造函数会用到尚未初始化的tfs。但tfs和tempDir是不同的人在不同的时间于不同的源码文件建立起来的，它们是定义千不同编译单元内的non-local static对象。

C++对“定义于不同的编译单元内的non-local static对象”的初始化相对次序并无明确定义。但可通过一个小小的设计消除这个问题：将每个non-local static对象搬到自己的专属函数内，这些函数返回一个reference指向它所含的对象。然后用户调用这些函数，而不直接指向这些对象。

也就是说，non-local static对象被local static对象替换掉了。这一点的基础在于**C++保证，函数内的local static对象会在“该函数被调用期间”、“首次遇上该对象之定义式”时被初始化**。

```C++
    class FileSystem {};    // 声明
    FileSystem& tfs()   // 这个函数用来替换tfs对象，它在FileSystem class中可能是个static。
    {
        static FileSystem fs;   // 定义并初始化一个local static对象，返回一个reference指向上述对象。
        return fs;
    };

    class Directory {};     // 同前
    Directory::Directory(params)
    {
        std::size_t disks = tfs().numDisks(); // 改为tfs()
    }

    Directory& tempDir()    // 这个函数用来替换tempDir对象;它在Directory class中可能是个static
    {
        static Directory td;
        return td;
    }
```

这么修改之后，这个系统程序的客户完全像以前一样地用它，唯一不同的是他们现在使用tfs()和tempDir()而不再是tfs和tempDir。也就是说他们使用函数返回的“指向static对象”的references, 而不再使用static对象自身。

### 请记住

1. 为内置型对象进行手动初始化，因为C++不保证初始化它们。

2. 构造函数最好使用成员初值列(member initialization list)，而不要在构造函数本体内使用赋值操作(assignment)。初值列列出的成员变量，其排列次序应该和它们在class中的声明次序相同。

3. 为免除“跨编译单元之初始化次序”问题，请以local static对象替换non-local static对象。
