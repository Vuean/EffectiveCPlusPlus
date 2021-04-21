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
