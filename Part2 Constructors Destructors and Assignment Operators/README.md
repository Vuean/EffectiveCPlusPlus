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

编译器生成的copy构造函数必须以no1.nameValue和no1.objectValue为初值设定no2.nameValue和no2.objectValue。两者之中，nameValue的类型是string，而标准string有个copy构造函数，所以no2.nameValue的初始化方式是调用string的copy构造函数并以no1.nameValue为实参。另一个成员NamedObject<int>::objectValue的类型是int(因为对此template具现体而言T是int)，那是个内置类型，所以no2.objectValue会以“拷贝nol.objectValue内的每一个bits”来完成初始化。

编译器为NamedObject<int>所生的copy assignment操作符，其行为基本上与copy构造函数如出一辙，但一般而言只有当生出的代码合法且有适当机会证明它有意义，其表现才会如我先前所说。万一两个条件有一个不符合，编译器会拒绝为class生出operator=。

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

### 请记住

编译器可以暗自为class创建default构造函数、copy构造函数、copy assignment操作符，以及析构函数。

## 条款06：若不想使用编译器自动生成的函数，就该明确拒绝

Explicitly disallow the use of compiler-generated functions you do not want

