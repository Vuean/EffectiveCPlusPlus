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

想象有个class用来表示网页浏览器。这样的class可能提供的众多函数中，有一些用来清除下载元素高速缓存区(cache of downloaded elements)、清除访问过的URLs的历史记录(history of visited URLs)、以及移除系统中的所有cookies：

```C++
    class WebBrowser{
    public:
        void clearCache();
        void clearHistory();
        void removeCookies();
    };
```

许多用户会想一整个执行所有这些动作，因此WebBrowser也提供这样一个函数：

```C++
    class WebBrowser{
    public:
        void clearEverything(); // 调用clearCache、clearHistory、removeCookies
    };
```

当然，这一机能也可由一个non-member函数调用适当的member函数而提供出来：

```C++
    void clearBrowser(WebBrowser& wb){
        wb.clearCache();
        wb.clearHistory();
        wb.removeCookies();
    }
```

可能很直观的根据面向对象守则要求认为，数据以及操作数据的那些函数应该被捆绑在一块，这意味它建议member函数是较好的选择。但其实不然，面向对象守则要求**数据应该尽可能被封装**，然而与直观相反地，member函数clearEverything带来的封装性比non-member函数clearBrowser低。

至于其中原因，需要从封装开始讨论。如果某些东西被封装，它就不再可见。越多东西被封装，越少人可以看到它。因此，越多东西被封装，我们改变那些东西的能力也就越大。这就是我们首先推崇封装的原因：**它使我们能够改变事物而只影响有限客户**。

对于对象内的数据，愈少代码可以看到数据（也就是访问它），愈多的数据可被封装，而我们也就愈能自由地改变对象数据。

成员变量设为private后，那么能够访问private成员变量的函数只有class的member函数加上friend函数而已。因此，在一个member函数和一个non-member、non-friend函数中做抉择，而且两者提供相同机能，那么，导致较大封装性的是non-member、non-friend函数，因为它并不增加“能够访问class内之private成分”的函数数量。即不降低类的封装性。

在这一点上有两件事情值得注意。第一，这个论述只适用于non-member、non-friend函数。friends函数对class private成员的访问权力和member函数相同，因此两者对封装的冲击力道也相同。从封装的角度看，这里的选择关键并不在member和non-member函数之间，而是在member和non-member、non-friend函数之间。

第二件值得注意的事情是，只因在意封装性而让函数“成为class的non-member”，并不意味它“不可以是另一个class的member”。

> 请记住

- 宁可拿non-member、non-friend函数替换member函数。这样做可以增加封装件、包裹弹性(packaging flexibility)和机能扩充性。

## 条款24：若所有参数皆需类型转换，请为此采用non-member函数

Declare non-member functions when type conversions should apply to all parameters.

令classes支持隐式类型转换通常是个糟糕的主意。当然这条规则有其例外，最常见的例外是在建立数值类型时。假设设计一个class用来表现有理数，允许整数“隐式转换”为有理数似乎颇为合理。

```C++
    class Rational{
    public:
        Ration(int numerator = 0, int denominator = 1);
        int numerator() const;
        int denominator() const;
        const Rational operator* (const Rational& rhs) const;
    private:
        ...
    };
```

声明operator*函数，实现将两个有理数相乘：

```C++
    Rational oneEight(1, 8);
    Rathonal oneGalf(1, 2);
    Rational result = oneHalf * oneEight;
    result = result * oneEight;
```

但如果想要实现Rational与int相乘呢？结果发现只有一半行得通：

```C++
    result = oneHalf * 2;   // 可行
    result = 2 * oneHalf;   // 不可行
```

以函数形式重写，将会更清楚的表明原因：

```C++
    result = oneHalf.operator*(2);   // 可行
    result = 2.operator*(oneHalf);   // 不可行
```

因为，oneHalf是一个内含`operator*`函数的class对象，所以可以调用改函数。然而整数2并没有响应的class，也就没有`operator*`成员函数。

但回头查看第一个成功的调用。其实函数内部参数为int型的2，但其实`Rational::operator*`的参数应该是一个Rational对象，其中能成功调用的关键在于发生了**隐式类型转换**(**implicit type conversion**)。

编译器在这里自动地将int变成了Rational：

```C++
    const Rational temp(2); // 根据2建立了一个临时的Rational对象
    result = oneHalf * temp;
```

**当然，只因为涉及non-explicit构造函数，编译器才会这样做。如果Rational构造函数是explicit，**以下语句没有一个可通过编译：

```C++
    result = oneHalf * 2;   // 错误，在explicit构造函数的情况下，无法将2转换为一个Rational
    result = 2 * oneHalf;   // 不可行
```

针对第二个无法成功调用的案例，发现只有**当参数被列于参数列(parameter list)内，这个参数才是隐式类型转换的合格参与者**。

为了实现混合式算术运算，可以将operator*成为一个non-member函数：

```C++
    class Rational{
        // 不包括operator*
    };

    const Rational operator*(const Rational& lhs, const Rational& rhs){
        return Rational(lhs.numerator() * rhs.numerator(), lsh.denominator() * rhs.denominator());
    }

    Rational oneFourth(1, 4);
    Rational result;
    result = oneFourth * 2;
    result = 2 * oneFourth;
```

operator*是否应该成为Rational class的一个friend函数呢？

答案是否定的。member函数的反面是non-member函数，不是friend函数。

> 请记住

- 如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member。

## 条款25: 考虑写出一个不抛异常的swap函数

Consider support for a non-throwing swap.

swap(置换)两对象值，意思是将两对象的值彼此赋予对方，其实现方式：

```C++
    namespace std{
        template<typename T>
        void swap(T& a, T& b){
            T temp(a);
            a = b;
            b = temp;
        }
    }
```

只要类型T支持copying(通过copy构造函数和copy assignment操作符完成)，缺省的swap实现代码就可完成置换类型为T的对象。这缺省的swap实现版本涉及三个对象的复制：a复制到temp，b复制到a，以及temp复制到b。但是对某些类型而言，这些复制动作无一必要：其中最主要的就是“**以指针指向一个对象，内含真正数据**”那种类型。这种设计的常见表现形式是所谓“pimpl手法”(pimpl：pointer to implementation)。如果以这种手法设计Widget class，看起来会像这样：

```C++
    // 针对Widget数据而设计的class;
    class WidgetImpl{
    public:
        ...
    private:
        // 可能有很多数据，意味着复制时间很长
        int a, b, c;
        std::vector<double> v;
    };
```

```C++
    class Widget{
    public:
        Widget(const Widget& rhs);
        // 复制Widget时，令它复制其WidgetImpl对象。
        Widget& operator=(const Widget& rhs){
            *pImpl = *(rhs.pImpl)
        }
    private:
        WidgetImpl* pImpl
    };    
```

一旦要置换两个Widget对象值，我们唯一需要做的就是置换其pImpl指针，但缺省的swap算法不知道这一点。

我们希望能够告诉std::swap：当Widgets被置换时真正该做的是置换其内部的pImpl指针。确切实践这个思路的一个做法是：将std::swap针对Widget特化。下面是基本构想，但目前这个形式无法通过编译：

```C++
    namespace std{
        template<> void swap<Widget>(Widget& a, Widget& b){
            swap(a.pImpl, b.pImpl);
        }
    }
```

但是一如稍早我说，这个函数无法通过编译。因为它企图访问a和b内的pImpl指针，而那却是private。我们可以将这个特化版本声明为friend，但和以往的规矩不太一样：我们令Widget声明一个名为swap的public成员函数做真正的置换工作，然后将std::swap特化，令它调用该成员函数：

```C++
    class Widget{
    public:
        void swap(Widget& other){
            using std::swap;
            swap(pImpl, other.pImpl);
        }
    };
    // 修订后的std::swap特化版本
    namespace std{
        template<>
        void swap<Widget>(Widget& a, Widget& b){
            a.swap(b);
        }
    }
```

这种做法不只能够通过编译，还与STL容器有一致性，因为所有STL容器也都提供有public swap成员函数和std::swap特化版本（用以调用前者）。

> 请记住

- 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。

- 如果你提供一个member swap，也该提供一个non-member swap用来调用前者。对于classes(而非templates)，也请特化std::swap。

- 调用swap时应针对std::swap使用using声明式，然后调用swap并且不带任何“命名空间资格修饰”。

- 为“用户定义类型”进行std templates全特化是好的，但千万不要尝试在std内加入某些对std而言全新的东西。
