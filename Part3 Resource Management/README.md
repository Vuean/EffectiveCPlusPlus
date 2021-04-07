# Part3 资源管理

C++程序中最常使用的资源就是动态分配内存（如果你分配内存却从来不曾归还它，会导致内存泄漏），但内存只是你必须管理的众多资源之一。其他常见的资源还包括文件描述器(file descriptors)、互斥锁(mutex locks)、图形界面中的字型和笔刷、数据库连接、以及网络sockets。

## 条款13: 以对象管理资源

Use objects to manage resources

假设我们使用一个用来塑模投资行为（例如股票、债券等等）的程序库，其中各式各样的投资类型继承自一个root class Investment:

```C++
    class Investment{}; // 投资类型，继承体系中的root class
```

进一步假设，这个程序库系通过一个工厂函数(factory function)供应我们某特定的Investment对象：

```C++
    // 返回指针，指向Investment继承体系内的动态分配对象。
    // 调用者有责任删除它。
    Investment* createInvestment();
```

createinvestment的调用端使用了函数返回的对象后，有责任删除之。现在考虑有个f函数履行了这个责任：

```C++
    void f()
    {
        Inverstment* pInv = createInvestment();
        ...
        delete pInv;
    }
```

这看起来妥当，但若干情况下f可能尤法删除它得自create Investment的投资对象——或许因为“…”区域内的**一个过早的return语句**。如果这样一个return被执行起来，控制流就绝个会触及delete语句。

最后一种可能是“…”区域内的语句抛出异常，果真如此控制流将再次不会幸临delete。无论delete如何被略过去，我们泄漏的不只是内含投资对象的那块内存，还包括那些投资对象所保存的任何资源。

为确保createInvestment返回的资源总是被释放，我们需要将资源放进对象内，当控制流离开f，该对象的析构函数会自动释放那些资源。实际上这正是隐身于本条款背后的半边想法：**把资源放进对象内，我们使可倚赖C++的“析构函数自动调用机制”确保资源被释放**。

许多资源被动态分配于heap内而后被用于单一区块或函数内。它们应该在控制流离开那个区块或函数时被释放。标准程序库提供的auto_ptr正是针对这种形势而设计的特制产品。auto_ptr是个“类指针(pointer-like)对象”，也就是所谓“智能指针”，其析构函数自动对其所指对象调用delete。下面示范如何使用auto_ptr以避免f函数潜在的资源泄漏可能性：

```C++
    void f()
    {
        std::auto_ptr<Investment> pInv(createInvestment());
        // 调用factory函数一如既往得使用pInv
        // 经由auto_ptr的析构函数自动删除pInv
    }
```

- 获得资源后立刻放进管理对象(namaging object)内。

    实际上“以对象管理资源”的观念常被称为“资源取得时机便是初始化时机”(Resource Acquisition Is Initialization; RAII)，因为我们几乎总是在获得一笔资源后于同一语句内以它初始化某个管理对象。

- 管理对象(managing object)运用析构函数确保资源被释放

    不论控制流如何离开区块，一旦对象被销毁（例如当对象离开作用域）其析构函数自然会被自动调用，于是资源被释放。

auto_ptrs有一个不寻常的性质：若通过copy构造函数或copy assignment操作符复制它们，它们会变成null，而复制所得的指针将取得资源的唯一拥有权！

```C++
    std::auto_ptr<Investment> pInv1(createInverstment());

    std::auto_ptr<Investment> pInv2(pInv1); // 现在pInv2指向对象，pInv1被设为null

    pInv1 = pInv2;  // 现在pInv1指向对象，pInv2为null
```

这一诡异的复制行为，复加上其底层条件：“受auto_ptrs管理的资源必须绝对没有一个以上的auto_ptr同时指向它”，意味auto_ptrs并非管理动态分配资源的神兵利器。

auto_ptr的替代力案是“引用计数型智慧指针”(reference-counting smart pointer; RCSP)。所谓RCSP也是个智能指针，持续追踪共有多少对象指向某笔资源，并在无人指向它时自动删除该资源。RCSPs提供的行为类似垃圾回收(garbage collection)，不同的是RCSPs无法打破环状引用(cycles of references, 例如两个其实已经没被使用的对象彼此互指，因而好像还处在“被使用”状态)。

TR1的tr1::shared_ptr就是个RCSP，所以可这么写f():

```C++
    void f()
    {
        ...
        std::tr1::shared_ptr<Investment> pInv(createInvestment());  // 调用factory函数
        // 使用pInv经由shared_ptr析构函数自动删除pInv
    }
```

这段代码看起来几乎和使用auto_ptr的那个版本相同，但shared_ptrs的复制行为正常多了：

```C++
    void f()
    {
        ...
        std::tr1::shared_ptr<Investment> pInv1(createInvestment()); // pInv1指向createinvestrnent返回物
        std::tr1::shared_ptr<Investment> pInv2(pInv1);  // pInv1和pInv2指向同一个对象
        pInv1 = pInv2;
    }
    // pInv1和pInv2被销毁它们所指的对象也就被自动销毁
```

auto_ptr和tr1::shared_ptr两者都在其析构函数内做delete而不是delete[]动作。那意味在动态分配而得的array身上使用auto_ptr或tr1::shared_ptr是个馈主意。

> 请记住

- 为防止资源泄漏，请使用RAII对象，它们在构造函数中获得资源并在析构函数中释放资源。

- 两个常被使用的RAII classes分别是tr1::shared_ptr和auto_ptr。前者通常是较佳选择，因为其copy行为比较直观。若选择auto_ptr，复制动作会使它（被复制物）指向null。

## 条款14：在资源管理类中小心coping行为

Thing carefully about copying behavior in resource-managing classes.

当资源不是heap-based，对这种资源而言，像auto_ptr和tr1::shared_ptr这样的智能指针往往不适合作为资源掌管者(resource handlers)。既然如此，有可能偶而你会发现，你需要建立自己的资源管理类。

例如，假设我们使用CAPI函数处理类型为Mutex的互斥器对象(mutex objects)，共有lock和unlock两函数可用：

```C++
    void lock (Mutex* pm); // 锁定pm所指的互斥器
    void unlock (Mutex* pm); // 将互斥器解除锁定
```

为确保绝不会忘记将一个被锁住的Mutex解锁，你可能会希望建立一个class用来管理机锁。这样的class的基本结构由RAII守则支配，也就是“资源在构造期间获得，在析构期间释放”:

```C++
    class Lock{
    public:
        explicit Lock(Mutex* pm) : mutexPtr(pm)
        {Lock(mutexPtr);}   // 获得资源
        ~Lock() {unlock(mutexPtr);}
    private:
        Mutex* mutexPtr;
    };
```

客户对Lock的用法符合RAII方式：

```C++
    Mutex m;    // 定义所需的互斥器
    ...
    {   // 处立一个区块用来定义critical section.
        Lock m1(&m);    // 锁定互斥器
        ... // 执行critical section内的操作
    }   // ／在区块最末尾，自动解除互斥器锁定．
```

这很好，但如果Lock对象被复制，会发生什么事？

```C++
    Lock m11(&m);   // 锁定m
    Lock m12(m11);  // 将m11复制到m12身上，这会发生什么？
```

上述所描述的一般问题是：“当一个RAII对象被复制，会发牛什么事？”大多数时候你会选择以下两种可能：

- 禁止复制。

    许多时候允许RAII对象被复制并不合理。对一个像Lock这样的class这是有可能的，因为很少能够合理拥有“同步化基础器物”(synchronization primitives)的复件（副本）。因此，可将copying操作声明为private，来禁止复制。

- 对底层资源祭出“引用计数法”(reference-count)

    有时候我们希望保有资源，直到它的最后一个使用者（某对象）被销毁。这种情况下复制RAII对象时，应该将资源的”被引用数”递增。tr1::shared_ptr便是如此。

如果前述的Lock打算使用reference counting，它可以改变mutexPtr的类型，将它从Mutex*改为tr1::shared_ptr< Mutex >。然而很不幸tr1::shared_ptr的缺省行为是“当引用次数为0时删除其所指物”，那不是我们所要的行为。当我们用上一个Mutex，我们**想要做的释放动作是解除锁定而非删除**。

幸运的是tr1::shared_ptr允许指定所谓的“删除器”(deleter)，那是一个函数或函数对象(function object)，当引用次数为0时便被调用（此机能并不存在于auto_ptr——它总是将其指针删除）。删除器对tr1::shared_ptr构造函数而言是可有可无的第二参数，所以代码看起来像这样：

```C++
    class Locl{
    public:
        // 以某个Mutex初始化shared_ptr并以unlock函数为删除器
        explicit Lock(Mutex* pm) : mutexPtr(pm, unlock)
        {
            Lock(mutexPtr.get());
        }
    private:
        std::tr1::shared_ptr<Mutex> mutexPtr;   // 使用shared_ptr替换raw pointer
    };
```

请注意，本例的Lock class不再声明析构函数。因为没有必要。

- 复制底部资源。

    有时候，可以针对一份资源拥有其任意数量的复件（副本）。而你需要“资源管理类”的唯一理由是，当你不再需要某个复件时确保它被释放。在此情况下复制资源管理对象，应该同时也复制其所包覆的资源。也就是说，复制资源管理对象时，进行的是“深度拷贝”。

    某些标准字符串类型是由“指向heap内存”的指针构成（那内存被用来存放字符串的组成字符）。这种字符串对象内含一个指针指向一块heap内存。当这样一个字符串对象被复制，不论指针或其所指内存都会被制作出一个复件。这样的字符串展现深度复制(deep copying)行为。

- 转移底部资源的拥有权。

    某些罕见场合下你可能希望确保永远只有一个RAII对象指向一个未加工资源(raw resource)，即使RAII对象被复制依然如此。此时资源的拥有权会从被复制物转移到目标物。一如条款13所述，这是auto_ptr奉行的复制意义。

> 请记住

- 复制RAII对象必须并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。

- 普遍而常见的RAII class copying行为是：抑制copying、施行引用计数法(reference counting)。不过其他行为也都可能被实现。

## 条款15：在资源管理类中提供对原始资源的访间

Provide access to raw resources in resource-managing classes.

资源管理类(resource-managing classes)很棒，排除此等泄漏是良好设计系统的根本性质，避免直接处理原始资源(raw resources)。但这个世界并不完美。许多APIs直接指涉资源，所以只得绕过资源管理对象(resource-managing objects)直接访问原始资源(raw resources)。

举个例子，条款13导入一个观念：使用智能指针如auto_ptr或tr1:: shared_ptr保存factory函数如create Investment的调用结果：

std::tr1::shared_ptr<Investment> pInv(createlnvestment());

假设你希望以某个函数处理Investment对象，像这样：

```C++
    int daysHeld(const Investment* pi); // 返回投资天数
```

然后调用：

```C++
    int days = daysHeld(pInv);  // 错误
```

但结果是：无法通过编译，因为daysHeld需要的是Investment*指针，但传入的是tr1::shared_ptr<Investment>的对象。

这个时候需要一个函数可将RAII class对象(本例为tr1:: shared_ptr)转换为其所内含的原始资源(本例为底部之Investment*)。有两个做法可以达成目标：显式转换和隐式转换。

tr1::shared_ptr和auto_ptr都提供一个get成员函数，用来执行显式转换，也就是它会返回智能指针内部的原始指针(的副本)：

```C++
    int days = daysHeld(pInv.get());// 将pInv内的原始指针传给daysHeld
```

同样，和所有的智能指针一样，tr1::shared_ptr和auto_ptr也重载了指针取值(pointer dereferencing)操作符(operator->和operator*)，它们允许隐式转换至底部原始指针：

```C++
    class Investment{   // investment继承体系的根类
    public:
        bool isTaxFree() const;
    };
    Investment* createInvestment(); // factory函数
    std::tr1::shared_ptr<Investment> pi1(createInvestment());   // 令tr1::shared_ptr管理一笔资源
    bool taxable1 = !(pi1.isTaxFree());// 经由operator->访问资源
    ...
    std::auto_ptr<Investment> pi2(createInvestment());  // 令auto_ptr管理一笔资源
    bool taxable2 = !((*pi2).isTaxFree());  // 经由operator*访问资源
```

由于有时候还是必须取得RAII对象内的原始资源，某些RAII class设计者想法是提供**一个隐式转换函数**。考虑下面这个用于字体的RAII class（对CAPI而言字体是一种原生数据结构）:

```C++
    FontHandle getFont();   // 这是个CAPI。
    void releaseFont(FontHandle fh);    // 来自同组CAPI
    class Font{
    public:
        // 获得资源，采用pass-by-value，因为CAPI这么做
        explicit Font(FontHandle fh) : f(fh)
        {}
        // 释放资源
        ~Font() {releaseFont(f);}
    private:
        FontHandle f;
    };
```

假设有大量与字体相关的CAPI，它们处理的是FontHandles，那么“将Font对象转换为FontHandle”会是一种很频繁的需求。Font class可为此提供一个显式转换函数，像get那样：

```C++
    class Font{
    public:
        FontHandle get() const {return f;}  // 显示转换函数
    };
```

不幸的是这使得客户每当想要使用API时就必须调用get：

```C++
    void changeFontSize(FontHandle f, int newSize);
    Font f(getFont());
    int newFontSize;
    changeFontSize(f.get(), newFontSize);
```

某些程序员可能会认为，如此这般地到处要求显式转换，足以使人们倒尽胃口，不再愿意使用这个class，从而增加了泄漏字体的可能性，而Font class的主要设计目的就是为了防止资源（字体）泄漏。

另一个办法是令Font提供隐式转换函数，转型为FontHandle：

```C++
    class Font{
    public:
        operator FontHandle() const {return f;}  // 隐式转换函数
    };
```

这使得客户调用CAPI时比较轻松且自然：

```C++
    Font f(getFont());
    int newFontSize;
    changeFontSize(f, newFontSize);
```

但是这个隐式转换会增加错误发生机会。例如客户可能会在需要Font时意外创建一个FontHandle：

```C++
    Font f1(getFont());
    FontHandle f2 = f1;
```

提供显示转化函数还是隐式转换函数，取决于RAII class被设计执行的特定工作，以及被使用的情况。

> 请记住

- APIs往往要求访问原始资源(raw resources)，所以每一个RAII class应该提供一个“取得其所管理之资源”的办法。

- 对原始资源的访问可能经由显式转换或隐式转换。一般而言显式转换比较安全，但隐式转换对客户比较力使。

## 条款16: 成对使用new和delete时要采取相同形式

Use the same form in corresponding uses of new and delete
