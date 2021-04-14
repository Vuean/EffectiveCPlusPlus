# 第4章 设计与声明

## 条款18: 让接口容易被正确使用，不易被误用

Make inference easy to use correctly and hard to incorrectly.

欲开发一个“容易被正确使用，不容易被误用”的接口，**首先必须考虑客户可能做出什么样的错误**。假设你为一个用来表现日期的class设计构造函数：

```C++
    class Date{
    public:
        Date(int month, int day, int year);
    };
```

初看这个接口合理，但是但它的客户很容易犯下至少两个错误。第一，他们也许会以错误的次序传递参数：`Date d(30, 3, 1995);`；第二，他们可能传递一个无效的月份或天数：`Date d(2, 30, 1995);`。

许多客户端错误可以因为导入新类型而获得预防。在防范“不值得拥有的代码”上，类型系统(type system)是主要方法。通过导入简单
的**外覆类型**(**wrapper types**)来区别天数、月份和年份，然后于Date构造函数中使用这些类型：

```C++
    struct Day{
    explicit Day(int d) : val(d){}
        int val;
    };

    struct Month{
    explicit Month(int m) : val(m){}
        int val;
    };

    struct Year{
    explicit Year(int y) : val(y){}
        int val;
    };

    class Date{
    public:
        Date(const Month& m, const Day& d, const Year& y);
    };

    Date d(30, 3, 1995);    // 错误！不正确的类型
    Date d(Day(30), Month(3), Year(1995));  // 错误！不正确的类型
    Date d(Month(3), Day(3), Year(1995)); // OK
```

令Day，Month和Year成为成熟且经充分锻炼的classes并封装其内数据，比简单使用上述的structs好。

为限制月份的取值，可利用enum表现月份，但enums不具备我们希望拥有的类型安全性，例如enums可被拿来当一个ints使用。比较安全的解法是预先定义所有有效的Months：

```C++
    class Month{
    public:
        static Month Jan(){return Month(1);}    // 函数返回有效月份。
        static Month Feb(){return Month(2);}
        ...
        static Month Dec(){return Month(12);}
        ...
    private:
        // 阻止生成新的月份。这是月份专属数据。
        explicit Month(int m);
    };
    Date d(Month::Mar(), Day(30), Year(1995))l
```

另一个一般性准则“让types容易被正确使用，小容易被误用”的表现形式：“除非有好理由，否则应该尽量令你的types的行为与内置types一致”。

任何接口如果要求客户必须记得做某些事情，就是有着“不正确使用”的倾向，因为客户可能会忘记做那件事。

> 请记住

- 好的接口很容易被正确使用，不容易被误用。你应该在你的所有接口中努力达成这些性质。

- “促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。

- “阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。

- tr1::shared_ptr支持定制型删除器(custom deleter)。这可防范DLL问题，可被用来自动解除互斥锁(mutexes; 见条款14)等等。

## 条款19: 设计class犹如设计type

Treat class design as type design.

C++就像在其他OOP(面向对象编程)语言一样，当定义一个新class，也就定义了一个新type。

如何设计高效的classes呢？首先必须了解面对的问题。几乎每一个class都要求面对以下提问：

- 新type的对象应该如何被创建和销毁？

    这会影响到class的构造函数和析构函数以及内存分配函数和释放函数(operator new, operator new []; operator delete和operator delete [])的设计，当然前提是如果你打算撰写它们。

- 对象的初始化和对象的赋值该有什么样的差别？

    这个答案决定你的构造函数和赋值(assignment)操作符的行为，以及其间的差异。很重要的是别混淆了“初始化”和“赋值”，因为它们对应于不同的函数调用。

- 新type的对象如果被passed by value(以值传递)，意味着什么？

    记住，copy构造函数用来定义一个type的pass-by-value该如何实现。

- 什么是新type的“合法值”？

    对class的成员变量而言，通常只有某些数值集是有效的。那些数值集决定了class必须维护的约束条件(invariants)，也就决定了成员函数（特别是构造函数、赋值操作符和所谓“setter”函数）必须进行的错误检查工作。它也影响函数抛出的异常、以及（极少被使用的）函数异常明细列(exception specifications)。

- 新type需要配合某个继承图系(inheritance graph)吗？

    如果新type继承自某些既有的classes，那就会受到那些classes的设计的束缚，特别是受到“它们的函数是virtual或non-virtual”的影响。如果你允许其他classes继承你的class，那会影响你所声明的函数——尤其是析构函数——是否为virtual。

- 你的新type需要什么样的转换？

    如果你希望允许类型T1之物被隐式转换为类型T2之物，就必须在class T1内写一个**类型转换函数**(operator T2)或在class T2内写一个non-explicit-one-argument（可被单一实参调用）的构造函数。如果你只允许explicit构造函数存在，就得写出专门负责执行转换的函数，且不得为类型转换操作符(type conversion operators)或non-explicit-one-argument构造函数。

- 什么样的操作符和函数对此新type而言是合理的？

    这个问题的答案决定将为新class声明哪些函数。其中某些该是member函数，某些则不是。

- 什么样的标准函数应该驳回？

    那些正是必须声明为private的函数。

- 谁该取用新type的成员？

    这个提问可以帮助你决定哪个成员为public，哪个为protected，哪个为private。它也帮助你决定哪一个classes和/或functions应该是friends，以及将它们嵌套于另一个之内是否合理。

- 什么是新type的“未声明接口”(undeclared interface)？

    它对效率、异常安全性以及资源运用（例如多任务锁定和动态内存）提供何种保证？你在这些方面提供的保证将为你的class实现代码加上相应的约束条件。

- 你的新type有多么一般化？

    或许你其实并非定义一个新type，而是定义一整个types家族。果真如此你就不该定义一个新class，而是应该定义一个新的class template。

- 你真的需要一个新type吗？

    如果只是定义新的derived class以便为既有的class添加机能，那么说不定单纯定义一或多个non-member函数或templates，更能够达到目标。

> 请记住

- Class的设计就是type的设计。在定义一个新type之前，请确定你已经考虑过本条款覆盖的所有讨论主题。

## 条款20：宁以pass-by-reference-to-const替换pass-by-value

Prefer pass-by-reference-to-const to pass-by-value.

通常则函数参数都是以实际实参的复件（副本）为初值，这些副本都是由对象的copy构造函数产出，这可能使得pass-by-value操作代价过高。考虑以下class继承体系：

```C++
    class Person{
    public:
        Person();
        virtual ~Person();
    private:
        std::string name;
        std::string address;
    };
    class Student : public Person{
    public:
        Student();
        ~Student();
    private:
        std::string schoolName;
        std::string schoolAddress;
    }
```

现考虑，调用函数validateStudent，后者需要一个Student实参(by value)并返回它是否有效：

```C++
    bool validateStudent(Student s);    // 函数已by value方式接收参数
    Student plato;
    bool platoIsOK = validdateStudent(plato);
```

当上述函数被调用时，Student的copy构造函数会被调用，以plato为蓝本将s初始化。同样明显地，当validateStudent返回时s会被销毁。因此，对此函数而言，参数的传递成本是“一次Student copy构造函数调用，加上一次Student析构函数调用”。

通过pass by reference-to-const可回避所有那些构造和析构动作：`bool validateStudent(const Student& s);`。

这种传递方式比传值的效率高得多：没有任何构造函数或析构函数被调用，因为没有任何新对象被创建。

以by reference方式传递参数也可以避免**slicing**(对象切割)问题，当一个derived class对象以by value方式传递并被视为一个base class对象，base class的copy构造函数会被调用，而“造成此对象的行为像个derived class对象”的那些特化性质全被切割掉了，仅仅留下一个base class对象。假设你在一组classes上工作，用来实现个图形窗口系统：

```C++
    class Window{
    public:
        std::string name() const;   // 返回窗口名称
        virtual void display() const;   // 显示窗口和其内容
    };
    class WindowWithScrollBars : public Window{
    public:
        virtual void display() const;
    };
```

所有Window对象都带有一个名称，你可以通过name函数取得它。所有窗口都可显示，你可以通过display函数完成它。display是个virtual函数，这意味简易朴素的base class Window对象的显示方式和华丽高贵的`WindowWithScrollBars`对象的显示方式不同。

现在假设你希望写个函数打印窗口名称，然后显示该窗口。下面是错误示范：

```C++
    // 不正确！参数可能被切割。
    void printNameAndDisplay(Window w){
        std::cout << w.name();
        w.display();
    }
```

当调用上述函数并交给它个`WindowWithScrollBars`对象：

```C++
    WindowWithScrollBars wwsb;
    printNameAndDisplay(wwsb);
```

因为是pass by value，所以参数w会被构造成为一个Window对象，将会造成wwsb是个`WindowWithScrollBars`对象的所有特化信息都会被切除。

解决切割(slicing)问题的办法，就是以by reference-to-const的方式传递w：

```C++
    // 很好，参数不会被切割
    void printNameAndDisplay(const Window& w){
        std::cout << w.name();
        w.display();
    }
```

references往往以指针实现出来，因此pass by reference通常意味真正传递的是指针。对于内置类型而言，pass by value往往比pass by reference的效率高些。

> 请记住

- 尽量以pass-by-reference-to-const替换pass-by-value。前者通常比较高效，并可避免切割问题(slicing problem)。

- 以上规则并不适用于内置类型，以及STL的迭代器和函数对象。对它们而言，pass-by-value往往比较适当。

## 条款21：必须返回对象时，别妄想返回其reference

Don't try to return a reference when you must return an object.

根据条款20，在过度追求pass-by-reference的过程中，往往会范另外的致命错误：**开始传递一些references指向其实并不存在的对象**。

考虑一个用以表现有理数(rational numbers)的class，内含一个函数用来计算两个有理数的乘积：

```C++
    class Rational{
    public:
        Rational(int numerator = 0, int denominator = 1);
    private:
        int n, d;
        friend const Rational operator* (const Rational& lhs, const Rational& rhs);
    };
```

应该清楚的是：所谓reference只是个名称，代表某个既有对象。它一定是某物的另一个名称。以上述operator*为例，如果它返回－个reference，后者定指向某个既有的Rational对象，内含两个Rational对象的乘积。

我们当然不可能期望这样一个（内含乘积的）Rational对象在调用operator*之前就存在。也就是说，如果你有：

```C++
    Rational a(1, 2);   // a= 1/2
    Rational b(3, 5);  // b = 3/5
    Rational c = a * b; // c应该是3/10
```

期望“原本就存在一个其值为3/10的Rational对象”并不合理。如果operator*要返回一个reference指向如此数值，它必须自己创建那个Rational对象。

函数创建新对象的途径有二：在stack空间或在heap空间创建之。如果定义个local变量，就是在stack空间创建对象。根据这个策略试写operator*如下：

```C++
    // 糟糕的代码
    const Rational& operator* (const Rational& lhs, const Rational& rhs){
        Rational result(lhs.n * rhs.n, lhs.d * lhs.d);
        return result;
    }
```

其中，存在的问题包括，使用了构造函数，而本来目的确实避免调用构造函数，更严重的问题是，这个函数返回一个reference指向result，但result是个local对象，而local对象在函数退出前被销毁了。

于是，让我们考虑在heap内构造一个对象，并返回reference指向它。Heap-based对象由new创建，所以你得写一个heap-based operator*如下：

```C++
    // 更糟糕的代码
    const Rational& operator* (const Rational& lhs, const Rational& rhs){
        Rational* result = new Rational(lhs.n * rhs.n, lhs.d * lhs.d);
        return *result;
    }
```

其中，不仅同样的使用了构造函数，而且new出来的对象无法实施删除，会导致资源泄露。

为了避免使用构造函数，可能会想到“让operator*返回的reference指向一个被定义于函数内部的static Rational对象来实现”：

```C++
    // 警告，烂代码
    const Rational& operator* (const Rational& lhs, const Rational& rhs){
        static Rational result;
        result = ...;
        return result;
    }
```

就像所有用上static对象的设计一样，这一个也立刻造成我们对**多线程安全性**的疑虑。不过那还只是它显而易见的弱点。如果想看看更深层的瑕疵，考虑以下面这些完全合理的客户代码：

```C++
    bool operator==(const Rational& lhs, const Rational& rhs);
    Rational a, b, c, d;
    if((a * b) == (c * d)){
        ...
    } else{
        ...
    }
```

实际运行结果是，表达式`((a * b) == (c * d))`总是返回true，无论a, b, c, d值是什么。

一旦将代码重新写为等价的函数形式，很容易就可以了解出了什么意外：`if(operator==(opeartor*(a, b), operator*(c, d)))`

其中，在`operator==`被调用前，已有两个`operator*`调用式起作用，每一个都返回reference指向`operator*`内部定义的static Rational对象。因此，`operator==`被要求将“`operator*`内的static Rational对象值”拿来和“`operator*`内的static Rational对象值”比较，如果比较结果不相等，那才奇怪呢。(两次`operator*`调用的确各自改变了static Rational对象值，但由于它们返回的都是reference，因此调用端到的永远是static Rational对象的“现值”。)

一个“必须返回新对象”的函数的正确写法是：**就让那个函数返回一个新对象**。对Rational的operator*而言意味以下写法：

```C++
    inline const Rational operator* (const Rational& lhs, const Rational& rhs){
        return Rational(lhs.n * rhs.n, lhs.d * lhs.d);
    }
```

当然，还是需要承担operator*返回值的构造成本和析构成本。

> 请记住

绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象而有可能同时需要多个这样的对象。条款4已经为“在单线程环境中合理返回reference指向一个local static对象”提供了一份设计实例。

## 条款22: 将成员变量声明为private

Declare data members private.

本条款主要阐述为什么成员变量不该是public，所有反对public成员变量的论点同样适用于protected成员变量，成员变量应该是private。

如果成员变量不是public，客户唯一能够访问对象的办法就是通过成员函数。如果令成员变量为public，每个人都可以读写它，但如果以函数取得或设定其值，就可以实现出“不允许访问”、“只读访问”以及“读写访问”：

```C++
    class AccessLevels{
    public:
        int getReadOnly() const { return readOnly;}
        void getReadWrite(int value){readWrite = value;}
        int getReadWrite(){return readWrite;}
        void setWriteOnly(int value){writeOnly = value;}
    private:
        int noAccess;       // 对此int无任何访问动作
        int readOnly;       // 对此int做只读访问
        int readWrite;      // 对此int做读写访问
        int writeOnly;      // 对此int做惟写访问
    }；
```

另外，通过函数访问成员变量，日后可改变某个计算替换这个成员变量，而class客户一点也不会知道class的内部实现已经起了变化。

举个例子，假设你正在写一个自动测速程序，当汽车通过，其速度便被计算并填入一个速度收集器内：

```C++
    class SpeedDataCollection{
    public:
        void addValue(int speed);   // 添加一笔新数据
        double averageSoFar() const;    // 返回平均速度
    };
```

考虑成员函数averageSoFar。做法之一是在class内设计一个成员变量，记录至今以来所有速度的平均值。当averageSoFar被调用，只需返回那个成员变置就好。另一个做法是令averageSoFar每次被调用时重新计算平均值，此函数有权力调取收集器内的每一笔速度值。

第一种做法（随时保存平均值）会使得每一个SpeedDataCollection对象变大，因为你必须为用来存放目前平均值、累积总量、数据点数的每一个成员变量分配空间。然而averageSoFar却可因此而十分高效；它可以只是一个返回目前平均值的inline函数。相反地，“被询问才计算平均值”会使得averageSoFar执行较慢，但每一个SpeedDataCollection对象比较小。

> 请记住

- 切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。

- protected并不比public更具封装性。

## 条款23：宁以non-member、non-friend替换member函数

Perfer non-member non-friend functions to member functions
