# EffectiveCPlusPlus

[1 让自己习惯C++](https://github.com/Vuean/EffectiveCPlusPlus/blob/main/Part1%20Accustoming%20Yourself%20to%20C%2B%2B/README.md)

01. 视C++为一个语言联邦
02. const, enum, inline
03. 尽可能使用const
04. 确定对象初始化

[2 构造/析构、赋值运算]()

05. 了解C++默默编写并调用哪些函数
06. 若不想使用编译器自动生成的函数，就该明确拒绝
07. 多态基类声明virtual析构函数
08. 别让异常逃离析构函数
09. 不在构造析构过程中调用virtual函数
10. operator=返回reference to *this
11. 在operator=中处理自我赋值
12. 复制对象时勿忘其每一个成分

[3 资源管理]()

13. 以对象管理资源
14. 在资源管理类中小心coping行为
15. 在资源管理类中提供对原始资源的访问
16. 成对使用new和delete时要采取相同形式
17. 以独立语句将newed对象植入智能指针

[4 设计与声明]()

18. 让接口容易被正确使用，不易被误用
19. 设计class犹如设计type
20. 宁以pass-by-reference-to-canst替换pass-by-value
21. 必须返回对象时，别妄想返回其reference
22. 将成员变量声明为private
23. 宁以non-member、non-friend替换member函数
24. 若所有参数皆需类型转换，请为此采用non-member函数
25. 考虑写出一个不抛异常的swap函数

[5 实现]()

26. 尽可能延后变量定义式的出现时间
27. 尽量少做转型动作
28. 避免返回handles指向对象内部成分
29. 为“异常安全”而努力是值得的
30. 透彻了解inlining的里里外外
31. 将文件间的编译依存关系降至最低

[6 继承与面向对象设计]()

32. 确定你的public继承塑模出is-a关系
33. 避免遮掩继承而来的名称
34. 区分接口继承和实现继承
35. 考虑virtual函数以外的其他选择
36. 绝不重新定义继承而来的non-virtual函数
37. 绝不重新定义继承而来的缺省参数值
38. 通过复合塑模出has-a或“根据某物实现出”
39. 明智而审慎地使用private继承
40. 明智而审慎地使用多重继承

[7 模板与泛型编程]()

41. 了解隐式接口和编译期多态
42. 了解typename的双重意义
43. 学习处理模板化基类内的名称
44. 将与参数无关的代码抽离templates
45. 运用成员函数模板接受所有兼容类型
46. 需要类型转换时请为模板定义非成员函数
47. 请使用traits classes表现类型信息
48. 认识template元编程

[8 定制new和delete]()

49. 了解new-handler的行为
50. 了解new和delete的合理替换时机
51. 编写new和delete时需固守常规
52. 写了placement new也要写placement delete

[9 杂项讨论]()

53. 不要轻忽编译器的警告
54. 让自己熟悉包括TR1在内的标准程序库
55. 让自己熟悉Boost