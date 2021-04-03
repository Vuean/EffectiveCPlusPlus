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

    

- 管理对象(managing object)运用析构函数确保资源被释放