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
