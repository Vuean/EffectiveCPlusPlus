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

设计一个class负责数据库连接：

```C++
    class DBConnection
    {
    public:
        static DBConnection create();   // 返回DBConnection 对象；
        void close();   // 关闭联机，失败则抛出异常
    };
```

为确保客户不忘记在DBConnection对象身上调用close()，一个合理的想法是创建一个用来管理DBConnection资源的class，并在其析构函数中调用close。

```C++
    class DBConn
    {
    public:
        ~DBConn()
        {
            db.close();
        }
    private:
        DBConnection db;
    };
```

只要调用close成功，一切都美好。但如果该调用导致异常，DBConn析构函数会传播该异常，也就是允许它离开这个析构函数。那会造成问题，因为那就是抛出了难以驾驭的麻烦。

两个方法可以避免这一问题。

- 如果close抛出异常就结束程序。通常通过调用abort完成：

```C++
    DBConn()::~DBConn()
    {
        try{ db.close(); }
        catch(...)
        {
            // 制作运转记录，记下对close的调用失败；
            std::abort();
        }
    }
```

也就是说调用abort可以抢先制“不明确行为”于死地。

- 吞下因调用close而发生的异常：

```C++
    DBConn()::~DBConn()
    {
        try{ db.close(); }
        catch(...)
        {
            // 制作运转记录，记下对close的调用失败；
        }
    }
```

一个较佳策略是重新设计DBConn接口，使其客户有机会对可能出现的问题作出反应。例如DBConn自己可以提供一个close函数，因而赋予客户一个机会得以处理“因该操作而发生的异常”。DBConn也可以追踪其所管理之DBConnection是否已被关闭，并在答案为否的情况下由其析构函数关闭之。这可防止遗失数据库连接。

```C++
    class DBConn
    {
    public:
        void close()
        {
            db.close();
            closed = true;
        }
        ~DBConn()
        {
            if(!closed)
            {
                try{
                    db.close();
                }
                catch(...)
                {
                     // 制作运转记录，记下对close的调用失败；
                }
            }
        }
    private:
        DBConnection db;
        bool closed;
    };
```

> 请记住

- 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们（不传播）或结束程序。

- 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作。

## 条款09: 绝不在构造和析构过程中调用virtual函数

Nerver call virtual functions during constructions or destructions

不该在构造函数和析构函数期间调用virtual函数，因为这样的调用不会带来你预想的结果。

假设有一个class继承体系，用来模拟股市交易如买进、卖出的订单等等。这样的交易一定要经过审计，所以每当创建一个交易对象，在审计日志(audit log)中也需要创建一笔适当记录。下面是一个看起来颇为合理的做法：

```C++
    class Transaction
    {
    public:
        Transction();
        virtual void logTransaction() cosnt = 0;
    };

    Transaction::Transaction()
    {
        logTransaction();
    }
    
    class BuyTransaction: public Transaction
    {
    public:
        virtual void logTransaction() const;
    };

    class SellTransaction : public Transaction
    {
    public:
        virtual void logTransaction() const;
    };
```

现在，执行`BuyTransaction b;`，会发生什么呢？

无疑地会有一个BuyTransaction构造函数被调用，但首先Transaction构造函数一定会更早被调用；是的，derived class对象内的base class成分会在derived class自身被构造之前先构造妥当。Transaction构造函数的最后一行调用virtua函数logTransaction，这正是引发惊奇的起点。这时候被调用的logTransaction是Transaction内的版本，不是BuyTransaction内的版本——即使目前即将建立的对象类型是BuyTransaction。是的，base class构造期间virtual函数绝不会下降到derived classes阶层。取而代之的是，对象的作为就像隶属base类型一样。非正式的说法或许比较传神：**在base class构造期间，virtual函数不是virtual函数**。

更根本的原因是：**在derived class对象的base class构造期间，对象的类型是base class而不是derived class**。

相同道理也适用于析构函数。一旦derived class析构函数开始执行，对象内的derived class成员变量便呈现未定义值。

但是侦测“构造函数或析构函数运行期间是否调用virtual函数”并不总是这般轻松。如果Transaction有多个构造函数，每个都需执行某些相同工作，那么避免代码重复的一个优秀做法是把共同的初始化代码（其中包括对logTransaction的调用）放进一个初始化函数如init内：

```C++
    class Transaction
    {
    public:
        Transction() {init();};
        virtual void logTransaction() cosnt = 0;
    private:
        void init()
        {
            ...
            logTransaction();
        }
    };
```

这段代码概念上和稍早版本相同，但它比较潜藏并且暗中为害，因为它通常不会引发任何编译器和连接器的抱怨。

但如何确保每次一有Transaction继承体系上的对象被创建，就会有适当版本的logTransaction被调用呢？

一种做法是在class Transaction内将logTransaction函数改为non-virtual，然后要求derived class构造函数传递必要信息给Transaction构造函数，而后那个构造函数便可安全地调用non-virtual logTransaction。像这样：

```C++
    class Transaction
    {
    public:
        explicit Transction(const std::string& logInfo);
        void logTransaction(const std::string& logInfo);
    };

    Transaction:: Transaction(const std::string& logInfo)
    {
        logTransaction(logInfo);
    }

    class BuyTransaction: public Transaction
    {
    public:
        BuyTransaction(parameters): Transaction(createLogString(parameters))
        {};
    private:
        static std::string createLogString(parameters);
    };
```

换句话说由于无法使用virtual函数从base classes向下调用，在构造期间，可以藉由“令derived classes将必要的构造信息向上传递至base class构造函数”替换之而加以弥补。

比起在成员初值列(member initialization list)内给予base class所需数据，利用辅助函数创建一个值传给base class构造函数往往比较方便（也比较可读）。令此函数为static，也就不可能意外指向“初期未成熟之BuyTransaction对象内尚未初始化的成员变量”。

> 请记住

构造和析构期间不要调用virtual函数，因为这类调用从不下降至derived class（比起当前执行构造函数和析构函数的那层）。

## 条款10: 令operator=返回一个reference to *this

Have assignment operators return a reference to *this.

关于赋值，有趣的是你可以把它们写成连锁形式：

```C++
    int x, y, z;
    x = y = z = 15; // 赋值连锁形式
```

同样有趣的是，赋值采用右结合律，所以上述连锁赋值被解析为：

```C++
    x = (y = (z = 15));
```

这里15先被赋值给z，然后其结果(更新后的z)再被赋值给y，然后其结果(更新后的y)再被赋值给x。

为了实现“连锁赋值”，赋值操作符必须返回一个reference指向操作符的左侧实参。这是你为classes实现赋值操作符时应该遵循的协议：

```C++
    class Widget
    {
    public:
        Widget& operator=(const Widget& rhs)
        {
            return *this;
        }
    };
```

这个协议不仅适用于以上的标准赋值形式，也适用于所有赋值相关运算，例如：

注意，这只是个协议，并无强制性。如果不遵循它，代码一样可通过编译。然而这份协议被所有内置类型和标准程序库提供的类型如string、vector、complex、tr1::shared_ptr或即将提供的类型共同遵守。

> 请记住

令赋值(assignment)操作符返回一个reference to *this。

## 条款11：在operator=中处理“自我赋值”

Handle assignment to self in operator=

“自我赋值”发生在对象被赋值给自己时：

```C++
    class Widget{...};
    Widget w;
    ...
    w = w;  // 自我赋值
```

在尝试自行管理资源（如打算写一个用于资源管理的class就得这么做），可能会掉进“在停止使用资源之前就释放了它”的陷阱。假设建立了一个class用来保存一个指针指向一块动态分配的位图(bitmap)：

```C++
    class Bitmap{...};
    class Widget{
        ...
    private:
        Bitmap* pb; // 指针，指向一个heap分配的对象
    };
```

下面是operator=实现代码，表面上看起来合理，但自我赋值出现时并不安全：

```C++
    Widget& Widget::operator=(const Widget& rhs)
    {
        delete pb;
        pb = new Bitmap(*rhs.pb);
        return *this;
    }
```

这里的自我赋值问题是，operator=函数内的*this（赋值的目的端）和rhs有可能是同一个对象。果真如此delete就不只是销毁当前对象的bitmap，它也销毁rhs的bitmap。在函数末尾，Widget——它原本不该被自我赋值动作改变的——发现自己待有一个指针指向一个已被删除的对象！

欲阻止这种错误，传统做法是藉由operator=最前面的一个“证同测试(identity test)”达到“自我赋值”的检验目的：

```C++
    Widget& Widget::operator=(const Widget& rhs)
    {
        if(this == rhs) return *this;
        delete pb;
        pb = new Bitmap(*rhs.pb);
        return *this;
    }
```

但这个新版本仍然存在异常方面的麻烦。更明确地说，如果“new Bitmap”，导致异常（不论是因为分配时内存不足或因为Bitmap的
copy构造函数抛出异常），Widget最终会持有一个指针指向一块被删除的Bitmap。

但本条款只要你注意“许多时候一群精心安排的语句就可以导出异常安全（以及自我赋值安全）的代码”，这就够了。例如以下代码，我们只需注意在复制pb所指东西之前别删除pb：

```C++
    Widget& Widget::operator=(const Widget& rhs)
    {
        Bitmap* pOrig = pb; // 记住原先的pb
        pb = new Bitmap(*rhs.pb);
        delete pOrig;
        return *this;
    }
```

现在，如果“newBitmap”抛出异常，pb(及其栖身的那个Widget)保持原状。即使没有证同测试(identity test)，这段代码还是能够处理自我赋值，因为我们对原bitmap做了一份复件、删除原bitmap、然后指向新制造的那个复件。它或许不是处理“自我赋值”的最高效办法，但它行得通。

在operator=函数内手工排列语句(确保代码不但“异常安全”而且“自我赋值安全”)的一个替代方案是，使用所谓的copy and swap技术：

```C++
    class Widget{
        ...
        void swap(Widget& rhs); // 交换*this和rhs数据
    }
    
    Widget& Widget::operator=(const Widget& rhs)
    {
        Widget temp(rhs);   // 备份rhs数据（副本）
        swap(temp);
        return *this;
    }
```

上述方法成立的前提和后果是：

1. class的copy assignment操作符可能被声明为“以by value方式接受实参”;

2. 以by value方式传递东西会造成一份复件／副本

> 请记住

- 确保当对象自我赋值时operator=有良好行为。其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序、以及copy-and-swap。

- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

## 条款12: 复制对象时勿忘其每一个成分

Copy all parts of an object

在声明自己的copying函数(copy构造函数和copy assignment操作符)时，如果实现代码几乎必然出错时，编译器并不会告诉你。

考虑一个class用来表现顾客，其中手工写出（而非由编译器创建）copying函数，使得外界对它们的调用会被记(logged)下来：

```C++
    void logCall(const std::string& funcName)   // 创造一个log entry
    class Customer{
    public:
        Customer(const Customer&);
        Customer& operator=(const Customer& rhs);
    private:
        std::string name;
    };

    Customer::Customer(const Customer& rhs) : name(rhs.name)
    {
        logCall("Customer copy constructor");
    }
    Customer& Customer::operator=(const Customer& rhs)
    {
        logCall("Customer copy assignment operator");name = rhs.name;
        return *this;
    }
```

这里的每一件事情看起来都很好，而实际上每件事情也的确都好，直到另一个成员变量加入战局：

```C++
    class Date {};
    class Customer{
    public:
        ...
    private:
        std::string name;
        Date lastTransaction;
    };
```

这时候既有的copying函数执行的是局部拷贝(partial copy)：它们的确复制了顾客的name，但没有复制新添加的lastTransaction 。大多数编译器对此不出任何怨言，针对自己写出copying函数，如果你的代码不完全，它们也不提醒。结论很明显：如果你为class添加一个成员变量，你必须同时修改copying函数。

一旦发生继承，可能会造成此一主题最暗中肆虐的一个潜藏危机：

```C++
    class PriorityCustomer : public Customer{
    public:
        PriorityCustomer(const PriorityCustomer& rhs);
        PriorityCustomer& operator=(const PriorityCustomer& rhs);
    private:
        int priority;
    };

    PriorityCustomer :: PriorityCustomer(const PriorityCustomer& rhs) : priority(rhs.priority)
    {
        logCall("PriorityCustomer copy constructor");
    }

    PriorityCustomer& operator=(const PriorityCustomer& rhs)
    {
        logCall("PriorityCustomer copy assignment operator");
        priority = rhs.priority;
        return *this;
    }
```

PriorityCustomer的copying函数看起来好像复制了PriorityCustomer内的每样东西，但是请再看一眼。是的，它们复制了PriorityCustomer声明的成员变量，但每个PriorityCustomer还内含它所继承的Customer成员变量复件（副本），而那些成员变量却未被复制。PriorityCustomer的copy构造函数并没有指定实参传给其base class构造函数（也就是说它在它的成员初值列(member initialization list)中没有提到Customer)，因此PriorityCustomer对象的Customer成分会被不带实参的Customer构造函数（即default构造函数必定有一个否则无法通过编译）初始化。default构造函数将针对name和lastTransacLlon执行缺省的初始化动作。

应该让derived class的copying函数调用相应的base class函数：

```C++
    PriorityCustomer :: PriorityCustomer(const PriorityCustomer& rhs) : Customer(rhs), priority(rhs.priority)
    {
        logCall("PriorityCustomer copy constructor");
    }

    PriorityCustomer& operator=(const PriorityCustomer& rhs)
    {
        logCall("PriorityCustomer copy assignment operator");
        Customer::operator=(rhs);   // 对base class成分进行赋值动作
        priority = rhs.priority;
        return *this;
    }
```

当你编写一个copying函数，**请确保(1):复制所有local成员变量，(2)调用所有base classes内的适当的copying函数**。

> 请记住

- Copying函数应该确保复制“对象内的所有成员变量”及“所有base class成分”。

- 不要尝试以某个copying函数实现另一个copying函数。应该将共同机能放进第三个函数中，并由两个coping函数共同调用。
