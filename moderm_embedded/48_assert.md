# 48 软件断言与 Design by Contract（下）

这一课承接第 47 课，把 assertions 和 Design by Contract 从观念层推进到 embedded systems 的具体实现。上一课已经讲清楚：断言不是容错代码，而是用于暴露 programming error 的契约检查机制。这一课的重点就是回答一个更实际的问题：在没有 `stderr`、没有进程退出模型、而且系统可能仍要承担安全责任的 embedded 环境里，断言应该怎么落地。

Samek 的结论很鲜明：嵌入式断言不能只是桌面版 `assert()` 的简化移植，而应该成为系统最后一道 damage control 防线的一部分。

lesson 48 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-48

## 面向 embedded 的断言接口应该长什么样

课程先回顾了 QP/C 里的 `qassert.h`。Samek 强调这个文件本身是自包含的，即使不使用整个 QP/C framework，也可以把这套断言机制拿去放进自己的项目里。

其中最基础的是 `Q_ASSERT_ID()`：

1. 参数一是 assertion ID。
2. 参数二是要检查的 Boolean expression。
3. 条件成立时什么都不做。
4. 条件失败时调用 `Q_onAssert()`。

它被实现为一个 expression，而不是普通语句，这一点和标准 `assert()` 一样重要。这样做的好处是：

1. 可以安全放进 `if-else` 控制流里。
2. 不会引出 dangling-else 这类语法陷阱。

也就是说，断言接口本身的定义方式，也要照顾实际代码中的可用性和可组合性。

## 为什么不用 `__LINE__` 和 `__FILE__`

标准断言常常用 `__LINE__` 和 `__FILE__` 标识错误位置，但 Samek 在嵌入式版本里故意不采用这套方式，而是改成：

1. 模块名字符串 `Q_this_module_`。
2. 稳定的 assertion ID。

这样做的原因很实际：

1. `__LINE__` 很不稳定，前面多删一行或多加一行，后面所有编号都会变。
2. 如果要做 fault injection 或系统化测试，必须有稳定可重复的断言标识。
3. `__FILE__` 展开结果可能依赖编译路径，甚至在某些编译器里造成多份重复字符串，浪费 ROM。

因此 QP/C 提供 `Q_DEFINE_THIS_MODULE()`，让每个模块显式定义自己的模块名，再配合局部唯一的 ID，就能稳定且节省地标识断言来源。

## `Q_onAssert()` 是系统最后一道防线

所有断言失败最终都会落到 `Q_onAssert()`。课程里特别强调，这个函数必须被视为 never-return。

1. 从语义上，它不应该返回到已被破坏的上下文。
2. 从实现上，`Q_NORETURN` 还能帮助编译器和静态分析工具更好理解控制流。

如果只在调试期使用断言，一个简单的 endless loop 可能已经够用，因为你可以接上 debugger 观察调用栈。但如果断言在最终产品中仍然启用，那么 `Q_onAssert()` 的责任就完全不同了：

1. 它需要执行 damage control。
2. 它需要尽量让系统进入已知、安全的状态。
3. 最终通常应当 reset 系统，比如调用 `NVIC_SystemReset()`。

Samek 的重点并不是“所有产品都必须简单复位”，而是 assertion handler 必须是你明确设计和验证过的系统保护策略，而不是一个临时凑出来的死循环。

## assertion handler 也必须被认真测试

这一课有个很关键的提醒：assertion handler 运行时，系统往往已经处于 compromised state。

这意味着：

1. 不能假定栈一定没坏。
2. 不能假定中断上下文一定正常。
3. 不能假定当前所有基础设施都可安全调用。

所以 assertion handler 不是“普通业务代码”，反而更应该做 fault-injection testing。Samek 还特别提到，可以故意破坏 SP 来模拟 stack overflow，然后再触发断言，验证 handler 是否仍能完成必要的保护动作。

这跟前面课程里反复出现的思路一致：真正可靠的 embedded 机制，不只是“写出来”，而是要在坏情况下也专门验证过。

## 为什么不建议在断言里抛异常

针对某些支持 exception 的语言或风格，这一课专门解释了为什么 assertion handler 不适合通过 throw exception 来处理。

理由有两个：

1. 只有在异常能够被合理捕获、并且系统准备继续运行时，throw 才有意义。
2. throw 通常需要 stack unwinding，但断言失败时你根本不能确信 stack 没坏。

换句话说，assertion handler 的力量来自于简单、直接、保守。它的任务不是恢复复杂控制流，而是在系统可能已经受损时，尽可能可靠地执行最后的保护动作。

## 硬件 fault handler 本质上也是一种 assertion

课程后半段把讨论从 software assertion 扩展到 hardware faults，例如：

1. HardFault
2. MemManage
3. BusFault
4. UsageFault

Samek 的观点是：这些 fault handler 从作用上看，和软件断言处理器本质一致，都是在检测到系统已经进入非法状态后执行 damage control。

既然硬件 fault 在最终产品里不能被关闭，那么工程上最合理的方式往往就是：

1. 把硬件 fault 和软件 assertion 统一到一个共同的处理框架。
2. 在启动代码里先做最必要的现场修复，例如重置栈指针。
3. 然后统一转入 `Q_onAssert()`。

这也是课程里一直使用的 startup code 的组织方式。

## assertions 的真正价值：让 bug 统一暴露成 assertion failure

Samek 在这一课里花了不少篇幅强调 assertions 的实践收益，而且引用了 NASA JPL 和 Microsoft Research 的资料作为支撑。

核心观点可以概括为：

1. 断言密度足够高时，系统不会再随机表现为“奇怪行为”。
2. 很多 bug 会统一表现成 assertion failure。
3. 这会极大提高问题的可见性和可定位性。

也就是说，assertions 带来的不只是“检测多了一点”，而是改变整个调试体验：

1. 错误不再漫游到很远的地方才暴露。
2. 不再轻易被解释成偶发 glitch。
3. 开发者可以从稳定的 assertion ID 开始调查，而不是从随机症状倒推。

## 为什么发布版里不该轻易关闭断言

这一课最强烈的论点之一，是反对“开发阶段开 assertions，发布版全部关闭”这种行业常规。

Samek 认为这种做法在逻辑上站不住脚，因为如果你已经承认断言像 fuse、guardrail、insurance policy，那么到了真正运行环境却把它们拆掉，本身就非常荒谬。

他的类比很鲜明：

1. 保险丝不能因为平时不用，就在上车前都换成铁丝。
2. 山路护栏不能因为你本来不想撞上去，就全部拆掉。
3. 火灾保险不能因为你希望永远用不上，就在火季来前取消。

当然，这并不意味着“发布版保留全部开发期断言且不区分策略”。真正合理的含义是：一旦你决定用 assertions 作为系统完整性保护机制，就不应在产品阶段把这层防线整个拿掉。

## 本课小结

第 48 课把 assertions 从概念提升到了嵌入式工程实践层面，核心结论包括：

1. embedded assertions 需要稳定的标识方式和专门的 handler，而不是照搬桌面环境。
2. `Q_onAssert()` 必须被当成 never-return 的系统保护入口。
3. assertion handler 本身必须做 fault-injection 测试，因为它运行在系统已受损的条件下。
4. 硬件 fault handler 与软件断言在工程角色上是一致的，完全可以统一处理。
5. 高密度 assertions 会把大量 bug 变成更透明、更早暴露的 assertion failures。
6. 经过设计和验证的断言机制，原则上不应在发布版里被轻易整体关闭。

从课程主线看，47 和 48 两课其实是在为后面的 testing 主题铺路。因为如果没有稳定的 contracts、assertions 和 fault handling，后续的测试体系就很难真正有效地约束复杂软件系统。