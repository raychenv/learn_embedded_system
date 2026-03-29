# 31 面向对象编程之多态

这一课，Miro Samek 继续讲解 Object-Oriented Programming，进入第三个核心概念：polymorphism。和前两课的 encapsulation、inheritance 不同，多态是更“纯正”的面向对象特性，在传统过程式语言 C 里没有直接对应物，因此这一课的讲法也反过来了：先看 C++ 中多态到底是什么、解决什么问题、底层如何实现；下一课再去看如何在 C 中模拟它。

课程的切入点很自然。上一课里，`Rectangle` 继承了 `Shape`，并增加了自己的 `draw()` 和 `area()` 操作。但仔细想会发现：任何 `Shape` 都应该“能被画出来”，也应该“有面积”这个接口。问题不在于要不要有这两个操作，而在于高层的 `Shape` 并不知道自己到底是 Rectangle、Circle、Triangle 还是其他具体图形，因此它只能给出接口，不能给出真正的具体实现。

这就引出了 polymorphism：同一个高层接口，在不同具体对象上表现出不同实现。

lesson 31 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-31/tm4c123-qxk-keil

## 什么是 polymorphism

polymorphism 来自希腊词根：

1. `poly`：多
2. `morphe`：形态

因此 polymorphism 的字面意思就是“多种形态”。在 OOP 里，它表示：**同一个操作接口，会根据对象的实际类型表现为不同的方法实现。**

比如：

1. `Shape::draw()` 是高层接口。
2. `Rectangle::draw()` 是矩形版本。
3. `Circle::draw()` 是圆形版本。

调用者只需要知道“我要 draw”，而不必在调用点手工判断当前对象到底是什么具体类型。

## 为什么仅有继承还不够

课程先做了一个很重要的对比：如果只是把 `draw()` 和 `area()` 声明加到 `Shape` 基类里，但不使用 `virtual`，那么通过 `Shape *` 指针调用这些函数时，编译器只会根据**指针的静态类型**来决定调用哪个函数。

也就是说：

1. 即使 `ps` 实际上指向一个 `Rectangle` 对象。
2. 只要 `ps` 的类型是 `Shape *`。
3. 调用 `ps->draw()` 时，进入的仍然是 `Shape::draw()`。

这显然不符合我们对“图形对象应该自动表现出自身行为”的预期。inheritance 只能保证“Rectangle is a Shape”，但如果没有 polymorphism，它仍然无法让基类接口自动分派到正确的派生类实现。

## virtual 打开多态机制

当课程在 `Shape` 基类的 `draw()` 和 `area()` 前加上 `virtual` 关键字后，情况就变了。

这时：

1. `Shape *` 指针仍然可以指向 `Rectangle`。
2. 但 `ps->draw()` 会根据对象的实际类型，跳转到 `Rectangle::draw()`。
3. `ps->area()` 也会跳转到 `Rectangle::area()`。

这就是 polymorphism 的核心：**调用选择基于对象的真实类型，而不是指针或引用的静态类型。**

## method 与 override

课程顺带引入了一些 OOP 术语。

当基类已经声明了 `draw()` 和 `area()` 这类 virtual operation 后，派生类里的同名实现就不再只是“新增函数”，而是这个高层操作的不同 method。派生类提供自己的版本，称为 override，也就是“覆盖”基类继承下来的默认实现。

因此：

1. `Shape::draw()` 是一个抽象层级较高的方法接口。
2. `Rectangle::draw()`、`Circle::draw()` 是不同具体 method。
3. 它们都在 override 同一个基类 operation。

课程里编译器也因此发出提示：一旦基类把函数声明为 virtual，派生类对应签名的函数天然也参与多态。把 `virtual` 在派生类声明中也显式写出来，可以让代码表达更清晰，并消除警告。

## 多态最大的价值：写更高层、更通用的代码

Samek 很快给出一个非常典型的例子：绘制图形集合。

假设要在 LCD 上绘制一个 graph，而 graph 可以抽象成“一个 `Shape *` 指针数组”。数组中的每个元素都指向某种具体图形对象，比如：

1. `Rectangle`
2. `Circle`
3. 甚至某个只有基类接口的普通 `Shape`

现在可以在 `Shape` 层级写出一个通用函数 `drawGraph()`，其逻辑非常简单：

1. 遍历 `Shape *` 数组。
2. 遇到空指针结束。
3. 对每个元素调用 `graph[i]->draw()`。

关键点在于，`drawGraph()` 完全不需要知道数组中到底是哪种具体图形。多态会自动把每次 `draw()` 分派到正确的方法实现。

这正是 polymorphism 在工程中的核心价值：**让高层算法依赖抽象接口，而不依赖具体类型。**

## extensibility：高层代码不必为新类型修改

课程随后新增了一个 `Circle` 类，并让它同样继承 `Shape`，实现自己的：

1. `draw()`
2. `area()`

然后把 `Circle` 对象加入 graph 数组，再次调用 `drawGraph()`。

最值得注意的地方是：`drawGraph()` 本身根本没有修改，甚至 `shape.cpp` 里它的实现都没有重新编译，但程序运行时仍然能正确调用 `Circle::draw()`。

这说明多态带来的不是简单的“语法优雅”，而是真正的 extensibility：

1. 新增一个派生类时，高层算法往往不需要改。
2. 只要新类型遵守已有抽象接口，就能被无缝接入原系统。
3. 这使软件结构更容易扩展，也更符合开闭原则的方向。

## early binding 与 late binding

为了说明多态为什么能做到这一点，课程开始对比两种调用绑定方式。

### early binding

如果一个函数调用在编译期就已经确定要跳到哪一个函数体，这叫 early binding。过程式语言 C 只使用这种机制。

例如：

1. 直接对对象 `r1.draw()` 这种调用。
2. 编译器在编译时已经知道 `r1` 就是 `Rectangle`。
3. 因此生成的代码里会直接写死调用 `Rectangle::draw()`。

这类调用在汇编里通常就是：

1. 把 `this` 指针放到 `R0`
2. 用 `BL` 指令直接跳到目标函数地址

### late binding

如果调用目标要等到运行时根据对象真实类型才能确定，这叫 late binding，也常叫 dynamic binding 或 runtime binding。

例如：

1. `Shape * ps = &r1;`
2. `ps->draw();`

此时编译器只知道 `ps` 是 `Shape *`，但运行时它可能指向 `Shape`、`Rectangle`、`Circle` 或其他派生类。因此函数地址不能在编译时写死，必须到运行时查找。

多态依赖的正是这种 late binding 机制。

## C++ 多态的典型实现：VPTR 与 VTABLE

课程接下来从调试器和反汇编中，直接 reverse-engineer C++ 的 virtual call 实现方式。

当类引入 virtual function 后，编译器会做两件关键事情：

1. 在对象内增加一个隐藏指针，通常放在对象起始位置，叫 VPTR。
2. 为每个含有 virtual function 的类，在 ROM 中生成一张函数地址表，叫 VTABLE。

其中：

1. VPTR 存在于每个对象实例里。
2. VTABLE 是每个类一份。
3. VTABLE 内部存放该类所有 virtual function 的入口地址。

课程在 `Rectangle` 对象的内存中看到，原本只有 `x`、`y`、`width`、`height` 这些成员，现在最前面多出一个指针，这就是编译器自动加入的 VPTR。继续跟到这个指针指向的 ROM 地址，又能看到表项内容正好是 `Rectangle::draw()` 和 `Rectangle::area()` 的地址。

## virtual call 到底怎么执行

基于 VPTR/VTABLE，virtual call 的过程就变成了：

1. 从对象中取出 VPTR。
2. 通过 VPTR 找到类对应的 VTABLE。
3. 从 VTABLE 中按偏移取出目标 virtual function 的地址。
4. 再通过寄存器间接跳转到该地址。

所以和普通调用相比，late binding 多了两层关键成本：

1. 每个对象多一个 VPTR
2. 每个 virtual call 要多做一次表查找和一次间接跳转

在 ARM Cortex-M 上，课程看到普通调用使用 `BL` 直接分支，而 virtual call 最终使用 `BLX` 跳到寄存器中保存的地址，这正说明调用目标是在运行时动态决定的。

## 多态的代价与收益

Samek 对这部分的态度很典型：既不神化，也不贬低。

多态的收益很明显：

1. 高层算法可以面向抽象接口编写。
2. 设计更易扩展。
3. 新类型更容易接入旧系统。

但它也不是零成本的。课程总结了典型开销：

1. 每个对象在 RAM 中多一个 VPTR。
2. 每个类在 ROM 中多一张 VTABLE。
3. 每次 virtual call 比普通调用多若干指令。

这也是为什么 C++ 不会默认让所有函数都 virtual，而是要求程序员显式写出 `virtual`。只有真正需要 late binding 的地方，才值得付出这部分开销。

## 谁来初始化 VPTR

最后一个很自然的问题是：VTABLE 在编译期生成就行，但每个对象实例里的 VPTR 是谁设置的？

课程通过调试 constructor 发现：**VPTR 的初始化是编译器自动插入到构造函数中的。**

而且在继承链上，VPTR 还会随着构造过程被反复重设：

1. 先执行基类 constructor 时，VPTR 先指向基类的 VTABLE。
2. 再回到派生类 constructor 时，VPTR 被重新设为派生类的 VTABLE。

因为构造顺序总是“先基类，后派生类”，所以最终对象里的 VPTR 会落在最末端那个实际类上，也就是正确的最终类型。

这也解释了为什么虚函数机制能与继承体系正确结合：对象在构造完成后，其 VPTR 已经指向“自己真正所属类”的 VTABLE，之后再通过基类指针去调用 virtual function，就能获得正确的方法分派。

## 本课与下一课的衔接

这一课主要完成了两件事：

1. 建立对 polymorphism 概念的直觉理解。
2. 看懂 C++ virtual function 的底层实现。

下一课才会进一步进入“如何在 C 里模拟 late binding”。也就是说，这一课先把原理讲清楚，下一课再去做 C 版本的工程实现。

## 总结

31 课的核心，是理解 polymorphism 为什么是 OOP 真正强大的地方：

1. 多态允许同一高层接口在不同对象上表现出不同实现。
2. 只有继承还不够，必须配合 `virtual` 才能按对象真实类型进行方法分派。
3. 多态最重要的工程价值，是让高层代码依赖抽象接口，从而获得良好的 extensibility。
4. `drawGraph()` 这种面向 `Shape *` 的通用算法，正是多态威力的典型例子。
5. C++ 中 virtual function 依赖 late binding，典型实现是对象内的 VPTR 加上类级别的 VTABLE。
6. 这种机制有开销，但相对于其带来的抽象能力和扩展性，通常是值得的。
7. VPTR 的设置由编译器在构造函数中自动完成，并在继承链初始化过程中逐步更新到最终派生类。

如果说 encapsulation 让代码有了清晰边界，inheritance 让共性得以复用，那么 polymorphism 则让系统真正具备“面向抽象扩展”的能力。这也是面向对象设计能在复杂软件中持续发挥作用的关键原因之一。