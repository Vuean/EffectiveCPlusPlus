# 第5章 实现

## 条款26: 尽可能延后变量定义式的出现时间

Postpone variable definitions as long as possible.

只要定义了一个变量而其类型带有一个构造函数或析构函数，那么当程序的控制流(control flow)到达这个变量定义式时，就需要承受构造成本；当这个变量离开其作用域时，需要承受析构成本。即使这个变量最终并未被使用，仍需耗费这些成本，所以应该尽可能避免这种情形。

考虑如下函数，计算同行米达的加密版本而后返回，前提是密码足够长。如果密码太短，函数会丢出一个异常，类型为logic_error。

```C++
    std::string encryptPassword(const std::string& password){
        using namespace std;
        string encrypted;
        if(password.lenght() < MinimumPasswordLength){
            throw logic_error("Password is too short");
        }
        ...
        return encryted;
    }
```

对象encrypted在此函数中并非完全未被使用，但如果有个异常被丢出，它就真的没被使用。也就是说如果函数encryptPassword丢出异常，你仍得付出encrypted的构造成本和析构成本。所以最好延后encrypted的定义式，直到确实需要它：

```C++
    std::string encryptPassword(const std::string& password){
        using namespace std;
        if(password.lenght() < MinimumPasswordLength){
            throw logic_error("Password is too short");
        }
        string encrypted;
        ...
        return encryted;
    }
```

通常，“通过default构造函数构造出一个对象然后对它赋值”比“直接在构造时指定初值”效率差，因此，上述encrypted对象，应该在初始化的时候赋初值。

因此“尽可能延后”的真正意义在于：不止应该延后变量的定义，直到非得使用该变量的前一刻为止，甚至应该尝试**延后这份定义直到能够给它初值实参为止**。

同样，如果变量在循环内，那么定义于循环外并在每次循环迭代时赋值给它比较好，还是该把它定义于循坏内比较好呢？

```C++
    // 方法A：定义于循环外
    Widget w;
    for(int i = 0; i < n; ++i){
        w = 取决于i的某个值;
    }

    // 方法B：定义于循环内
    for(int i = 0; i < n; ++i){
        Widget w(取决于i的某个值);
    }
```

在Widget函数内部，上述两种写法的成本如下：

- 做法A: 1个构造函数 + 1个析构函数 + n个赋值操作

- 做法B: n个构造函数 + n个析构函数

如果classes的一个赋值成本低于一组构造+析构成本，做法A大体而言比较高效，尤其当n值很大的时候。否则做法B或许较好。此外，做法A会造成名称w的作用域比做法B大。

因此，除非明确个赋值成本比“构造+析构”成本低，且正在处理代码中效率高度敏感的部分，否则应该使用方法B。

> 请记住

- 尽可能延后变量定义式的出现。这样做可增加程序的清晰度并改善程序效率。

## 条款27: 尽量少做转型动作

Minimize casting.

类型转换的常见三种语法形式：

```C++
    // 旧式类型转换（old-style class）
    // C风格的类型转换动作：
    (T)expression   // 将expression转型为T
    // 函数风格的类型转换动作：
    T(expression)   // 将expression转型为T
```

C++还提供四种新式类型转换：

```C++
    const_cast<T>(expression)
    dynamic_cast<T>(expression)
    reinterpret_cast<T>(expression)
    static_cast<T>(expression)
```

- `const_cast`通常被用来将对象的常量性转除(cast away the constness)。它也是唯一有此能力的C++-style转型操作符。

- `dynamic_cast`主要用来执行“安全向下转型”(safe downcasting)，也就是用来决定某对象是否归属继承体系中的某个类型。它是唯一无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作。

- `reinterpret_cast`意图执行低级转型，实际动作（及结果）可能取决于编译器，这也就表示它不可移植。

- `static_cast`用来强迫隐式转换(implicit conversions)。

新式转型动作优点在于：第一，它们很容易在代码中被辨识出来；第二，各转型动作的目标愈窄化，编译器愈可能诊断出错误的运用。

常用的旧式类型转换的地方在，调用一个explicit构造函数将一个对象传递给一个函数时：

```C++
    class Widget{
    public:
        explicit Widget(int size);
    };
    void doSomeWork(const Widget& w);
    doSomeWork(Widget(15)); // 以一个int加上函数风格的类型转换动作创建一个Widget
    doSomeWork(static_cast<Widget>(15));    // 以一个int加上C++风格的类型转换动作创建一个Widget
```

任何一个类型转换（不论是通过转型操作而进行的显式转换，或通过编译器完成的隐式转换）往往真的令编译器编译出运行期间执行的码。如：

```C++
    int x, y;
    double d = static_cast<double>(x) / y;  // 使用浮点数除法
```

```C++
    class Base{...};
    class Derived : public Base{...};
    Derived d;
    Base* pb = &d;  // 隐式的将Derived*转换为Base*
```

上述建立一个base class指针指向－个derived class对象，但有时候上述的两个指针值并不相同。这种情况下会有个偏移量(offset)在运行期被施行于`Derived*`指针身上，用以取得正确的`Base*`指针值。

另一件关于转型的有趣事情是：我们很容易写出某些似是而非的代码（在其他语言中也许真是对的）。例如许多应用框架(application frameworks)都要求derived classes内的virtual函数代码的第一个动作就先调用base class的对应函数。假设我们有个Window base class和一个SpecialWindow derived class，两者都定义了virtual函数onResize。进一步假设SpecialWindow的onResize函数被要求首先调用Window的onResize。下面是实现方式之一，它看起来对，但实际上错：

```C++
    class Window{
    public:
        virtual void onResize(){...}
    };

    class SpecialWindow : public Window{
    public:
        virtual void onResize(){
            // 实现将*this转换为Window，再调用其onResize，这可不行。
            static_cast<Window>(*this).onResize();
            ...// 这里执行SpecialWindow专属行为
        }
    };
```

上述函数中预期是：将*this转换为Window，对函数onResize调用，实现调用Window::onResize。但是实际上，调用的并不是当前对象上的函数，而实稍早类型转换动作所建立的一个`*this`对象的base class部分的展示副本身上的onResize。

上述代码并非在当前对象身上调用Window::onResize之后又在该对象身上执行SpecialWindow专属动作。不，**它是在“当前对象之base class成分”的副本上调用Window::onResize，然后在当前对象身上执行SpecialWindow专属动作**。

如果Window::onResize修改了对象内容，当前对象其实没被改动，改动的是副本。然而SpecialWindow::onResize内如果也修改对象，当前对象真的会被改动。这使当前对象进入一种“伤残”状态：其base class成分的更改没有落实，而derived class成分的更改倒是落实了。

解决办法是：拿掉类型转换动作，真实的告诉编译器真正的述求。

```C++
    class SpecialWindow : public Window
    {
    public:
        virtual void onResize(){
            Window::onResize(); // 调用Window::onResize作用于*this身上
        }
    };
```

这个例子也说明，如果你发现你自己打算转型，那活脱是个警告信号：你可能正将局面发展至错误的方向上。如果你用的是dynamic_cast更是如此。

dynamic_cast的许多实现版本执行速度相当慢。之所以需要dynamic_cast，通常是因为你想在一个你认定为derived class对象身上执行derived class操作函数，但你的手上却只有一个”指向base”的pointer或reference，你只能靠它们来处理对象。通常有两种一般性方法可避免上述问题：

第一，使用容器并在其中存储直接指向derived class对象的指针(通常是智能指针)，如此便消除了“通过base class接口处理对象”的需要。假设先前的Window/SpecialWindow继承体系中只有SpecialWindows才支持闪烁效果，试着不要这样做：

```C++
    class Window{...};
    class SpecialWindow : public Window{
    public:
        void blink();
    };

    typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
    VPW winPtrs;

    for(VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++){
        if(SpecialWindow* psw = dynamic_cast<SpecialWindow*>(iter->get()))
            psw->blink();
    }
```

应该改为：

```C++
    typedef std::vector<std::tr1::shared_ptr<SpecialWindow>> VPSW;
    VPSW winPtrs;
    for(VPSW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++)
        (*iter)->blink();
```

另一种做法可让你通过base class接口处理“所有可能的各种Window派生类”，那就是在base class内提供virtual函数做你想对各个Window派生类做的事。举个例子，虽然只有SpecialWindows可以闪烁，但或许将闪烁函数声明于base class内并提供一份“什么也没做＂的缺省实现码是有意义的：

```C++
    class Window{
    public:
        virtual void blink() {} // 缺省实现代码;
    };

    class SpecialWindow : public Window{
    public:
        virtual void blink(){...}   // 在此class内blink做某些事
    };
    typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
    VPW winPtrs;
    for(VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++)
        (*iter)->blink();
```

不论哪一种写法——“使用类型安全容器”或“将virtual函数往继承体系上方移动”——都并非放之四海皆准，但在许多情况下它们都提供一个可行的dynamic_cast替代方案。

但是需要注意的是，要绝对避免所谓的“连串(cascading) dynamic_cast”，即：

```C++
    class Window{...};
    ...
    typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
    VPW winPtrs;
    ...
    for(VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++){
        if(SpecialWindow1* psw1) = dynamic_cast<SpecialWindow1*>(iter->get()){...}
        else if(SpecialWindow2* psw2) = dynamic_cast<SpecialWindow2*>(iter->get()){...}
        else if(SpecialWindow3* psw3) = dynamic_cast<SpecialWindow3*>(iter->get()){...}
        ...
    }
```

这样产生出来的代码又大又慢，而且基础不稳，因为每次Window class继承体系一有改变，所有这一类代码都必须再次检阅看看是否需要修改。例如一旦加入新的derived class，或许上述连串判断中需要加入新的条件分支。这样的代码应该总是以某些“基于virtual 函数调用”的东西取而代之。

> 请记住

- 如果可以，尽量避免转型，特别是在注重效率的代码中避免dynamic_casts。如果有个设计需要转型动作，试着发展无需转型的替代设计。

- 如果转型是必要的，试着将它隐藏于某个函数背后。客户随后可以调用该函数，而不需将转型放进他们自己的代码内。

- 宁可使用C++-style(新式)转型，不要使用旧式转型。前者很容易辨识出来，而且也比较有着分门别类的职掌。

## 条款28: 避免返回handles指向对象内部成分

Avoid returing "handles" to object internals.

假设你的程序涉及矩形。每个矩形由其左上角和右下角表示。为了让一个Rectangle对象尽可能小，你可能会决定不把定义矩形的这些点存放在Rectangle对象内，而是放在一个辅助的struct内再让Rectangle去指它：

```C++
    class Point{
    public:
        Point(int x, int y);
        void SetX(int newValue);
        void SetY(int newValue);
    };
    struct RectData{
        Point ulhc; // 左上角
        Point lrhc; // 右下角
    };
    class Rectangle{
    private:
        std::tr1::shared_ptr<RecData> pData;
    };
```

Rectangle的客户必须能够计算Rectangle的范围，所以这个class提供upperLeft函数和lowerRight函数。Point是个用户自定义类型，函数返回references，代表底层的Point对象：

```C++
    class Rectangle{
    public:
        Point& upperLeft() const {return pData->ulhc;}
        Point& lowerRight() const {return pData->lrhc;}
    };
```

这样的设计可通过编译，但却是错误的。实际上它是自我矛盾的。一方面upperLeft和lowerRight被声明为const成员函数，因为它们的目的只是为了提供客户一个得知Rectangle相关坐标点的方法，而不是让客户修改Rectangle。另一方面两个函数却都返回references指向private内部数据，调用者于是可通过这些references更改内部数据！例如：

```C++
    Point coord1(0, 0);
    Point coord2(100, 100);
    const Rectangle rec(coord1, coord2);    // rec是个const矩形，
    rec.upperLeft().setX(50);   // rec值被修改
```

这里请注意，upperLeft的调用者能够使用被返回的reference(指向rec内部的Point成员变量)来更改成员。但rec其实应该是不可变的(const)!

这给我们的启示是：第一，成员变量的封装性最多只等于“返回其reference”的函数的访问级别。本例之中虽然ulhc和lrhc都被声明为private，它们实际上却是public，因为public函数upperLeft和lowerRight传出了它们的references。第二，如果const成员函数传出一个reference，后者所指数据与对象自身有关联，而它又被存储于对象之外，那么这个函数的调用者可以修改那笔数据。

专注于Rectangle class和它的upperLeft以及lowerRight成员函数。我们在这些函数身上遭遇的两个问题可以轻松去除，只要对它们的返回类型加上const即可：

```C++
    class Rectangle{
    public:
        const Point& upperLeft() const { return pData->ulhc;}
        const Point& upperRight() const { return pData->lrhc;}
    };
```

但即使如此，upperLeft和lowerRight还是返回了“代表对象内部”的handles，有可能在其他场合带来问题。更明确地说，它可能导致dangling handles（空悬的号码牌） ：这种handles所指东西（的所属对象）不复存在。这种“不复存在的对象”最常见的来源就是函数返回值。例如某个函数返回GUI对象的外框(bounding box)，这个外框采用矩形形式：

```C++
    class GUIObject{...};
    const Rectangle boundingBox(const GUIObject& obj);  // 以by value方式返回一个矩形
```

现在，客户有可能这么使用这个函数：

```C++
    GUIObject* pgo;
    ...
    const Point* pUpperLeft = &(boundingBox(*pgo).upperLeft());
```

对boundingBox的调用获得个新的、暂时的Rectangle对象。这个对象没有名称，所以我们权且称它为temp。随后upperLeft作用于temp身上，返回一个reference指向temp的一个内部成分，更具体地说是指向一个用以标示temp的Points。于是pUpperLeft指向那个Point对象。目前为止一切还好，但故事尚未结束，因为在那个语旬结束之后，boundingBox的返回值，也就是我们所说的temp，将被销毁，而那间接导致temp内的Points析构。最终导致pUpperLeft指向一个不再存在的对象；也就是说一旦产出pUpperLeft的那个语句结束，pUpperLeft也就变成空悬、虚吊(dangling)!

> 请记住

- 避免返回handles(包括references、指针、迭代器)指向对象内部。遵守这个条款可增加封装性，帮助const成员函数的行为像个const，并将发生“虚吊号码牌”(dangling handles)的可能性降至最低。

## 条款29: 为“异常安全”而努力是值得的

Strive for exception-safe code.
