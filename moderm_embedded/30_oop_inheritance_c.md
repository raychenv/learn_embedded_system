# 30 面向对象编程之继承

这一课，Miro Samek 继续讲解 Object-Oriented Programming，主题从上一课的 encapsulation 进入第二个核心概念：inheritance。课程的重点依旧不是停留在语法层面，而是说明“继承到底解决什么问题”，以及它在 C 和 C++ 中分别是如何落地实现的。

上一课已经用 `Shape` 类演示了封装：把坐标数据和相关操作打包到一起，通过 constructor、moveBy、distanceFrom 等操作来操纵对象，而不是直接暴露内部成员。但现实里的 LCD 图形系统并不会只需要一个抽象的 Shape，还会有 Rectangle、Circle、Triangle 等更具体的图形类型。问题就来了：这些具体图形都需要位置坐标、移动操作、距离计算，难道每个类都从头写一遍吗？

inheritance 给出的答案是：不用。可以基于已有类定义新类，复用共同的属性和操作，并在其上增加更具体的内容。

lesson 30 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-30/tm4c123-qxk-keil

## 继承要解决的问题

课程先从需求出发解释继承存在的意义。假设系统里有：

1. `Shape`：抽象图形，具备位置坐标。
2. `Rectangle`：矩形。
3. `Circle`：圆形。
4. `Triangle`：三角形。

这些具体图形虽然表现不同，但都共享一部分通用能力，比如：

1. 都有屏幕位置。
2. 都可以移动。
3. 都可以和其他图形计算距离。

如果每个类都重复定义这些内容，既浪费代码，也破坏结构的一致性。inheritance 的作用，就是把“共性”放在基类里，再让派生类在此基础上扩展“个性”。

## 在 C 里模拟 Rectangle 继承 Shape

课程先继续沿用上一课的方法，在 C 里手工模拟继承。

### 核心技巧：把基类对象放在派生类结构体的第一个成员

`Rectangle` 的定义不是完全从零开始，而是在结构体中把一个完整的 `Shape` 对象作为第一个成员放进去，并按约定把它命名为 `super`。然后再增加矩形自己独有的属性，比如：

1. `width`
2. `height`

因此，`Rectangle` 可以理解成：

1. 一个完整的 `Shape`
2. 再加上矩形特有的数据和操作

这就是在 C 中模拟 inheritance 的基础做法。

### Rectangle 自己增加的操作

`Rectangle` 在继承 `Shape` 通用能力的同时，也定义自己特有的 operation，例如：

1. `Rectangle_draw()`：在 LCD 上画出矩形。
2. `Rectangle_area()`：计算矩形面积。

构造函数 `Rectangle_ctor()` 则需要同时初始化两部分内容：

1. 继承自 `Shape` 的 `x`、`y`
2. `Rectangle` 自己新增的 `width`、`height`

### constructor 的顺序

课程特别强调，在 `Rectangle_ctor()` 中，必须先调用 `Shape_ctor(&me->super, x0, y0)`，先把“继承来的基类部分”初始化好，然后再初始化 `Rectangle` 自己增加的成员。

这个顺序非常重要，因为它反映了继承关系的本质：派生类对象内部先有一个完整的基类对象，然后才是自己附加的那部分内容。

## 为什么“第一个成员”这么关键

这课最核心的底层知识点，是 C 语言结构体内存布局规则。

课程在调试器中直接观察 `Rectangle` 对象的内存，发现：

1. `Rectangle` 的起始地址就是其第一个成员 `super` 的地址。
2. 当进入 `Shape_ctor()` 时，虽然 `me` 的类型从 `Rectangle *` 变成了 `Shape *`，但指针值保持不变。
3. `Shape` 的成员先写入对象内存前半部分。
4. `Rectangle` 自己新增的成员紧跟在 `Shape` 那部分之后。

这不是巧合，而是 C 标准保证的结果：**结构体对象的地址与其第一个成员的地址相同**。因此，只要基类对象被放在派生类结构体的第一个成员位置，就可以安全地把 `Rectangle *` 转成 `Shape *`。

## upcasting：为什么 Rectangle 可以当成 Shape 用

正因为 `Rectangle` 结构体的开头就是一个 `Shape`，所以任何需要 `Shape *` 的函数，其实都可以接受一个 `Rectangle *`，只不过在 C 里通常需要显式强制类型转换。

这就带来了 inheritance 的真正价值：

1. `Shape_moveBy()` 可以直接作用于 `Rectangle`。
2. `Shape_distanceFrom()` 也可以直接作用于 `Rectangle`。

换句话说，`Rectangle` 不只是“拥有一个 Shape”，而是**可以被当成一个 Shape 来使用**。这就是 inheritance 建立的 is-a 关系：

1. a Rectangle is a Shape

课程也顺带解释了 upcasting 这个术语为什么叫“向上转型”：在类图中，基类 `Shape` 在上面，派生类 `Rectangle` 在下面，因此从派生类转成基类，就是“往上走”。

## 继承关系的图示与术语

课程补充了类图和术语：

1. base class：基类
2. superclass / parent class：父类
3. derived class：派生类
4. subclass / child class：子类

类图中的继承箭头指向更一般化的类，也就是基类，因此箭头会指向 `Shape`。这体现的是 generalization，也就是“从更具体到更一般”的关系。

同时，继承也可以一层层继续往下扩展。比如 `Rectangle` 还可以继续派生出 `FilledRectangle`。于是整个系统会形成一棵不断细化的类层次结构。

## 如何正确理解继承：is-a，而不是家庭关系

Samek 在这里专门提醒不要用“家庭关系”来理解继承。虽然术语里会说 parent class、child class，但这其实是个容易误导人的比喻。

原因是：

1. 派生类对象内部真的包含一个完整基类对象。
2. 派生类实例可以被当成基类实例使用。

所以继承更准确的理解方式应该是 biological classification，也就是生物分类学式的层级：

1. House Cat is a Felis
2. House Cat is a Carnivore
3. House Cat is a Mammal

这类关系才符合 OOP inheritance 的逻辑，因为低层级类别天然拥有高层级类别的通用属性和行为。

## inheritance 与 composition 的区别

这一段非常重要，因为很多初学者会把继承和组合混在一起。

课程明确区分：

1. inheritance 对应 is-a 关系
2. composition 对应 has-a 关系

在 C 里，二者看起来都像“在一个结构体里嵌另一个结构体”，但关键区别在于嵌入的位置：

1. 如果把基类对象放在第一个成员位置，就可以安全 upcast，于是成立 is-a，也就是 inheritance。
2. 如果把它放在其他位置，那么就只是“Rectangle has a Shape”，也就是 composition。

这时虽然仍然是组合，但你已经不能把整个 `Rectangle *` 安全地当作 `Shape *` 使用了；若想访问内部 `Shape`，就必须显式写出成员路径，这会破坏封装性，也失去了继承带来的多态基础。

## 在 C++ 里实现真正的继承

课程下半部分把前面在 C 里的模拟，翻译成真正的 C++ 语法，以便直接对比两者。

### 类声明中的继承语法

在 `rectangle.h` 中，C++ 的写法是：

1. 用 `class Rectangle` 声明类。
2. 在类名后使用 `: public Shape` 表示公开继承自 `Shape`。

也就是说，C++ 不再要求你手工写一个 `super` 成员来表达继承关系，而是通过语言级语法直接声明“Rectangle 公开继承 Shape”。

这里的 `public` 意味着：基类对外可见的接口，在派生类中仍然保持对外可见。

### 派生类 constructor 如何调用基类 constructor

和 C 一样，派生类构造函数必须先初始化基类部分。但在 C++ 里，这通过 constructor initializer list 完成，而不是在函数体里手工调用 `Shape_ctor()`。

这再次体现出 C++ 的设计思路：把上一课在 C 中靠约定完成的模式，变成语言原生支持的语法。

### private 与 protected

这课还引入了一个新的访问控制关键字：`protected`。

上一课 `Shape` 中的成员如果是 `private`，那么只有 `Shape` 自己的成员函数能访问；即便 `Rectangle` 继承了 `Shape`，也不能直接访问这些成员。

如果希望派生类也能直接使用这些继承来的属性，就需要把它们改成 `protected`。这样：

1. 外部代码依然不能直接访问。
2. 派生类可以直接访问。

因此在 `Rectangle::draw()` 里，就能直接使用继承来的 `x` 和 `y`，而不需要像 C 版本那样写 `me->super.x` 之类的路径。

## C++ 中对象的使用方式

在 `main.cpp` 中，使用 `Rectangle` 的方式与上一课的 `Shape` 类似，但更自然：

1. 定义对象时直接调用构造函数初始化。
2. 调用 `draw()`、`area()` 这些本类定义的操作时，使用点运算符。
3. 调用从 `Shape` 继承来的 `moveBy()`、`distanceFrom()` 时，也直接像访问自身成员函数一样使用。

这里最关键的变化是：**C++ 会自动执行 upcasting**。因为语言本身知道 `Rectangle` 公开继承自 `Shape`，所以当某个接口需要 `Shape` 引用或指针时，编译器会自动把 `Rectangle` 视为 `Shape`，不再需要像 C 中那样手工强制转换。

## 调试时看到的 C++ 继承本质

Samek 继续通过调试器说明：即便 C++ 提供了更高级的继承语法，底层对象布局和 C 版本仍然高度一致。

在调试器中可以观察到：

1. 进入 `Rectangle` constructor 后，`this` 指针指向整个 `Rectangle` 对象。
2. 步入 `Shape` constructor 时，`this` 的地址值不变，只是类型视角从 `Rectangle` 变成了 `Shape`。
3. `Shape` 的成员位于对象起始位置。
4. `Rectangle` 自己新增的成员紧随其后。

这说明 C++ 的继承对象布局，在这个简单场景里与前面 C 里“第一个成员为 super”的做法本质一致。

## 继承的运行效果

当程序进入 `Rectangle::draw()`、`Rectangle::area()` 时，`this` 显示的是整个 `Rectangle` 对象；而当调用继承自 `Shape` 的 `moveBy()` 或 `distanceFrom()` 时，调试器里可以看到 `this` 被自动 upcast 成 `Shape` 视角，于是只看到对象中属于 `Shape` 的那一部分。

这很好地说明了 inheritance 的工作方式：

1. 派生类对象确实包含基类子对象。
2. 继承来的操作只需要处理基类那部分内容。
3. 对调用者来说，这个转换可以由语言自动完成。

## 总结

30 课的核心，是把 inheritance 从“语法糖”还原成一种结构复用机制来理解：

1. 继承用于复用共同属性和操作，并在其上扩展更具体的类。
2. 在 C 里可以通过“把基类对象放到派生类结构体的第一个成员”来模拟继承。
3. 这种布局允许把派生类安全地 upcast 为基类，从而复用基类操作。
4. inheritance 建立的是 is-a 关系，而 composition 建立的是 has-a 关系。
5. C++ 用 `: public Base`、initializer list、自动 upcasting、`protected` 等语言特性，把同样的思想表达得更直接、更安全。
6. 在简单场景下，C++ 的继承底层对象布局与 C 的手工模拟几乎一致。

这也为下一课的 polymorphism 铺平了道路。因为只有先建立“派生类对象可以当作基类对象来使用”的继承体系，后面的多态机制才有意义。