# Part2 构造/析构/赋值函数

## 条款05：了解C++默默编写并调用哪些函数

当C++处理过empty class(空类)后，编译器会为它声明一个copy构造函数、一个copy assignment操作符和一个析构函数。此外，如果没有声明任何构造函数，编译器也会自动声明一个default构造函数。。所有这些函数都是public且inline。

```C++
    class Empty {};

    // 实际为
    class Empty {
    punlic:
        Empty() {}      // default构造函数
        Empty(const Empty& rhs) {}  // copy构造函数
        ~Empty() {}      // 析构函数，暂不确定是否为virtual
        Empty& operator=(const Empty& rhs) {} // copy assignment操作符
    };
```

惟有当这些函数被需要（被调用），它们才会被编译器创建出来。其中，copy构造函数和copy assignment操作符，编译器创建的版本只是单纯地将来源对象的每一个non-static成员变量拷贝到目标对象。考虑一个NarnedObjec template，它允许你将一个个名称和类型为T的对象产生关联：

```C++
    template<typename T>
    class NamedObject{
    public:
        NamedObject(const char* name, const T& value);
        NamedObject(const std::string& name, const T& value);
    private:
        std::string nameValue;
        T objectValue;
    };
```

由于其中声明了一个构造函数，编译器于是不再为它创建default构造函数。

NarnedObject既没有声明copy构造函数，也没有声明copy assignment操作符，所以编译器会为它创建那些函数（如果它们被调用的话）。现在，看看copy构造函数的用法：

```C++
    NamedObject<int> no1("Smallest Prime Number", 2);
    NamedObject<int> no1(no1);
```

编译器生成的copy构造函数必须以no1.nameValue和no1.objectValue为初值设定no2.nameValue和no2.objectValue。两者之中，nameValue的类型是string，而标准string有个copy构造函数，所以no2.nameValue的初始化方式是调用string的copy构造函数并以no1.nameValue为实参。另一个成员NamedObject< int >::objectValue的类型是int(因为对此template具现体而言T是int)，那是个内置类型，所以no2.objectValue会以“拷贝nol.objectValue内的每一个bits”来完成初始化。

编译器为NamedObject< int >所生的copy assignment操作符，其行为基本上与copy构造函数如出一辙，但一般而言只有当生出的代码合法且有适当机会证明它有意义，其表现才会如我先前所说。万一两个条件有一个不符合，编译器会拒绝为class生出operator=。

举例，改变NarnedObject种nameValue为reference to string，objectValue是const T：

```C++
    emplate<typename T>
    class NamedObject{
    public:
        // 以下构造函数不再接收一个const name，因为nameValue 是一个reference-to-non-const string。
        NamedObject(std::string& name, const T& value);
    private:
        std::string& nameValue;
        const T objectValue;
    };
```

有如下代码：

```C++
    std::string newDog("Persephone");
    std::string oldDog("Satch");
    NamedObject<int> p(newDog, 2);
    NamedObject<int> s(oldDog, 36);

    p = s;
```

赋值前，无论p.nameValue和s.nameValue都指向string对象。但赋值后呢，赋值后p.nameValue应该指向s.nameValue所指的那个string吗？如果是，则违背了“C++所不允许的让reference改指向不同对象”。因此，C++的响应是拒绝编译那一行赋值动作。

**如果打算在一个“内含reference成员”的class内支持赋值操作(assignment)，必须自己定义copy assignment操作符**。面对“内含const成员”的classes，编译器的反应也一样。

> 请记住

编译器可以暗自为class创建default构造函数、copy构造函数、copy assignment操作符，以及析构函数。

## 条款06：若不想使用编译器自动生成的函数，就该明确拒绝

Explicitly disallow the use of compiler-generated functions you do not want

为了阻止编译器自动生成copy构造函数或copy assignment操作符，可以将copy构造函数或copy assignment操作声明为private。

但这种做法还是不安全，member函数和friend函数仍可以调用private函数。因此，在声明private函数的基础上，不去实现它们，那么在不慎调用时则会出现连接错误(linkage error)。例如：

```C++
    class HomeForSale{
    public:
        ...
    private:
        HomeForSale(const HomeForSale&);
        HomeForSale& operator=(const HomeForSale&);
    }
```

将连接期错误移至编译期是可能的（而且那是好事，毕竟愈早侦测出错误愈好），只要将copy构造函数和copy assignment操作符声明为private就可以办到，但不是在HomeForSale自身，而是在一个专门为了阻止copying动作而设计的base class内。这个base class非常简单：

```C++
    class Uncopyable{
    protected:              // 允许derived对象构造和析构
        Uncopyable() {}
        ~Uncopyable() {}
    private:
        Uncopyable(const Uncopyable&);
        Uncopyable& operator=(const Uncopyable&);
    };

    // 为求阻止HomeForSale对象拷贝，唯一需要做的是继承Uncopyable

    class HomeForSale : private Uncopyable 
    {
        // class不再声明copy构造函数和copy assignment操作符
    };
```

Uncopyable class的实现和运用颇为微妙，包括不一定得以public继承它，以及Uncopyable的析构函数不一定得是virtual等等。

> 请记住

为驳回编译器自动（暗自）提供的机能，可将相应的成员函数声明为private并且不予实现。使用像Uncopyable这样的base class也是一种做法。

## 条款07：为多态积累声明virtual析构函数

Declare destrucotrs virtual in polymorphic base classes.

设计一个TimeKeeper base class和一些derived classes作为不同的计时方法：

```C++
    class TimeKeeper
    {
    public:
        TimeKeeper();
        ~TimeKeeper();
        ...
    };
    class AutomicClock : public TimeKeeper{};   // 原子钟
    class WaterClock : public TimeKeeper{};   // 水钟
    class WristWatch : public TimeKeeper{};   // 腕表
```

许多客户只想在程序中使用时间，不想操心时间如何计算等细节，这时候我们可以设计factory（工厂）函数，返回指针指向一个计时对象。Factory函数会“返回一个base class指针，指向新生成的derived class对象”：

```C++
    TimeKeeper* getTimeKeeper();    // 返回一个指针，指向一个TimeKeeper派生类的动态分配对象
```

为遵守factory函数的规矩，被getTimeKeeper()返回的对象必须位于heap。因此为了避免泄漏内存和其他资源，将factory函数返回的每一个对象适当地delete掉很重要：

```C++
    TimeKeeper* ptk = getTimeKeeper();
    ...
    delete ptk; // 释放，避免内存泄露
```

但上述代码存在：“纵使客户每一件事都做对了，仍然没有办法知道程序如何行动”。

问题出在getTimeKeeper返回的指针指向一个derived class对象（例如AtomicClock)，而那个对象却经由一个base class指针（例如一个TimeKeeper*指针）被删除，而目前的base class(TimeKeeper)有个non-virtual析构函数。

这是一个灾难的来源，因为C++明确指出，当derived class对象经由一个base class指针被删除，而该base class带着一个non-virtual析构函数，其**结果是未定义的**——实际执行时通常发生的是对象的derived成分没被销毁。

最终结果可能是：getTimeKeeper返回的对象（假如为AtomicClock），其内的AtomicClock成分（声明的成员变量）很可能没被销毁，其析构函数可能也未被执行。最终导致产生“局部销毁”的对象，形成资源泄露、损坏数据结构等。

消除这个问题的做法很简单：给base class一个virtual析构函数。此后删除derived class会销毁整个对象，包括所有derived class成分：

```C++
    class TimeKeeper
    {
    public:
        TimeKeeper();
        virtual ~TimeKeeper();
        ...
    }; 
    TimeKeeper* ptk = getTimeKeeper();
    ...
    delete ptk;
```

像TimeKeeper这样的base classes除了析构函数之外通常还有其他virtual函数，因为virtual函数的目的是允许derived class的实现得以客制化(见条款34)。例如TimeKeeper就可能拥有一个virtual getCurrentTime，它在不同的derived classes中有不同的实现码。任何class只要带有virtual函数都几乎确定应该也有一个virtual析构函数。

但是，如果class不含virtual函数，通常表示它并不意图被用做一个base class。当class不企图被当作base class，令其析构函数为virtual往往是个馈主意。考虑一个用来表示二维空间点坐标的class:

```C++
    class Point{
    public:
        Point(int xCoord, int yCoord);
        ~Point();
    private:
        int x, y;
    };
```

如果int占用32 bits，那么Point对象可塞入一个64-bit缓存器中。然而当Point的析构函数是virtual，形势起了变化。

欲实现出virtual函数，对象必须携带某些信息，主要用来在运行期决定哪一个virtual函数该被调用。这份信息通常是由一个所谓vptr(virtual table pointer)指针指出。vptr指向一个由函数指针构成的数组，称为vtbl(virtual table)；每一个带有virtual函数的class都有一个相应的vtbl。当对象调用某一virtual函数，实际被调用的函数取决于该对象的vptr所指的那个vtbl——编译器在其中寻找适当的函数指针。

因此，如果Point class内含virtual函数，其对象的体积会增加：添加一个vptr会增加其对象大小达50%~100%！而且，Point对不再能够塞入一个64-bit缓存器，而C++的Point对象也不再和其他语言(如C)内的相同声明有着一样的结构(因为其他语言的对应物并没有vptr)，因此也就不再可能把它传递至（或接受自）其他语言所写的函数，除非你明确补偿vptr——那属于实现细节，也因此不再具有移植性。

**只有当class内含至少一个virtual函数，才为它声明virtual析构函数。**

> 请记住

- polymorphic（带多态性质的）base classes应该声明一个virtual析构函数。如果class带有任何virtual函数，它就应该拥有一个virtual析构函数。

- classes的设计目的如果不是作为base classes使用，或不是为了具备多态性(polymorphically)，就不该声明virtual析构函数。

## 条款08: 别让异常逃离析构函数

Prevent exceptions from leaving destrucotrs

