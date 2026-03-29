# 49 测试在嵌入式软件开发中的核心地位

这一课从 assertions/DBC 自然过渡到 testing。前两课已经说明，高质量软件必须把 bug 尽早暴露出来；而这一课进一步把视角拉高：如果软件开发本质上是在构造复杂系统，那么 testing 并不是项目尾声的验收步骤，而是驱动整个软件演化的核心机制。

Samek 这一课并不只是演示某个测试框架怎么用，而是想建立一个更根本的认识：复杂软件不可能靠一次性设计直接做对，它只能从可工作的简单系统逐步演化出来，而 testing 就是这个演化过程里的“人工选择”机制。

lesson 49 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-49

## 复杂系统只能从可工作的简单系统演化而来

课程开头借用了 Darwin 的进化论来讨论软件开发。Samek 想表达的是：

1. 一切复杂系统都不是凭空一次性造出来的。
2. 它们必须经历 incremental、cumulative 的演化过程。
3. 这个过程必须伴随持续不断的 selection。

对应到软件开发，就是那句课程里反复强调的工程规律：

1. 能工作的复杂系统，几乎总是从能工作的简单系统演化而来。
2. 试图从零设计一个复杂系统，通常不会成功。
3. 在 embedded systems 里，经常是“everything works 之前，nothing works”。

这里其实是在批评传统 waterfall 或 big-bang integration 思路。因为如果直到最后才整体测试，就等于放弃了复杂系统赖以形成的渐进筛选机制。

## testing 不是收尾动作，而是软件演化的选择机制

Samek 把 agile、TDD、CI、CD 这些现代工程实践统一到一个更本质的框架里理解：

1. testing 负责不断淘汰坏适应，也就是 bug。
2. testing 不只是验错，更是在指导开发方向。
3. 持续集成和持续交付的本质，是让系统持续保持“活着并可工作”。

换句话说，测试不是开发完成后的附加活动，而是整个 development loop 的核心反馈回路。

如果把软件看成一个不断演化的系统，那么没有测试的开发，其实就像没有选择压力的进化，最终只会失控堆积复杂度。

## unit testing 需要人为构造一个“生存环境”

自然选择发生在自然环境里，而人工选择需要人为构造环境。Samek 把 testing harness 或 testing framework 比喻成这种人工环境。

这一课聚焦的是 unit testing，也就是对最小可隔离的软件单元做测试，例如：

1. 单个函数。
2. 一个 C module。
3. 某个小范围的可独立逻辑。

unit testing 的关键不是“把代码跑一遍”，而是给 Code Under Test（CUT）构造一个可控、可重复、可验证的环境。只有这样，测试结果才真正有选择作用。

## Embedded-Test（ET）为什么值得看

课程没有选择讲一个最流行的测试框架，而是展示了一个非常轻量的 harness：Embedded-Test（ET）。Samek 选择它的原因很务实：

1. 它足够简单。
2. 它是 open source。
3. 它能跑书籍《Test-Driven Development for Embedded C》里大部分典型案例。
4. 它同时支持 host 和 embedded target。

和 Unity、CppUTest、gtest 相比，ET 的卖点不是“生态最大”，而是更适合拿来直观理解 embedded unit testing 的本质结构。

## ET 的基本组织方式

课程用最基础的 `sum()` 例子说明了 unit test 的典型目录划分：

1. `src` 放 Code Under Test。
2. `test` 放测试代码。

测试文件通常包括：

1. `setup()` 和 `teardown()`，在每个测试前后执行。
2. test group，用来组织一组相关测试。
3. 若干 `TEST()` 块，每个块描述一个测试场景。
4. `VERIFY()` 宏，用来判断测试条件是否满足。

Samek 还特意指出，ET 使用 `VERIFY()` 而不是 `ASSERT()`，就是为了避免和 CUT 中真正的 assertions 混淆。这个细节很重要，因为测试环境里的检查和产品代码里的 contract check，虽然形式相似，但工程角色不同。

## ET 和传统 C 测试框架的一个关键区别

ET 与 Unity 这类 C testing harness 的一个明显差异，是它不需要 test runner。

1. 在很多 C 框架里，每个 test 都是一个独立函数。
2. 然后还需要额外生成 runner，把这些测试统一调度起来。
3. ET 则把 individual test 设计成 test-group 函数内部的代码块。

这样做减少了额外样板代码，也降低了 harness 本身的复杂度。Samek 选 ET，就是为了让注意力放在 testing workflow，而不是被框架仪式感分散。

## dual-targeting 是 embedded 测试能力的分水岭

这一课最重要的实践观点之一，是 James Grenning 提倡的 dual-targeting。

它的意思不是用 QEMU 之类的 emulator 去模拟目标板，而是：

1. 同一套 embedded code 从第一天起就能同时在 host 和 target 上构建。
2. 平时大量测试在 host 上跑。
3. 必要时再把同样的测试拿到真实板子上运行。

这种策略的优势非常大：

1. host 上测试周期更快，不被硬件烧录和连接流程拖慢。
2. 更容易自动化。
3. 反过来会逼着你把硬件边界和软件逻辑边界划清楚。

Samek 对这一点的态度也非常强：真正接受 dual-targeting 的团队，会明显快过不这么做的团队。

## host-based testing 的工具链和环境

课程里顺便讲了一个很现实的话题：做 embedded unit testing，经常需要脱离 IDE，直接在命令行里工作。

因此 Samek 介绍了常见工具来源：

1. Linux/macOS 自带或易于安装的 `make`、`gcc`。
2. Windows 下的 WSL。
3. Quantum Leaps 提供的 QTools，包括 `make`、GNU 工具链、ARM cross compiler 等。

这一段不是课程主旨，但很实用，因为它指出了一个事实：测试基础设施本身就是开发能力的一部分。没有可用的 build/test habitat，再好的测试理念都落不了地。

## 在真实板子上跑测试时，关键是怎么把结果送出来

课程后半段把同一套 ET 测试搬到 TivaC 和 NUCLEO 板上执行。这里最核心的技术点并不是“烧录脚本怎么写”，而是 embedded test 的共同难题：

1. 板子上测完之后，结果怎么回传给 host。

ET 的方案和第 45 课的软件 tracing 很像，也是通过 UART 输出测试结果。因此每个板级 BSP 需要提供几个 callback：

1. `ET_onInit()` 初始化 UART。
2. `ET_onPrintChar()` 逐字符输出。
3. `ET_onExit()` 在测试完成后执行收尾动作。

由于 embedded target 不能像 host 程序那样真正 `exit()`，所以 `ET_onExit()` 通常只是停在一个可观察状态，例如死循环闪灯。

这也再次说明：embedded testing 不只是“写几个 test case”，还必须考虑平台约束下的可观测性。

## 为什么测试环境本身也在被测试

课程结尾强调了一个初学者很容易忽略的点：测试从来不是只测 CUT。

1. 每次测试都同时在检验 CUT 和测试环境本身。
2. 因此 TDD 常常从“先写一个会失败的测试”开始。
3. 这个第一失败测试的意义，不只是证明 CUT 还没写，而是确认 test environment 真能发现失败。

这其实和前两课 assertions 的思想完全一致。无论是 contract 还是 unit test，关键都不是“看起来有检查机制”，而是这些机制本身必须被验证确实有效。

## regression testing 很重要，但测试集合也不能无限膨胀

最后，Samek 还提醒了一个非常务实的问题：测试也是代码。

1. 测试越多，回归保护越强。
2. 但测试越多，维护成本和运行成本也会上升。
3. 把所有历史测试永远无限保留，并不一定是最优策略。

真正成熟的做法不是“测试越多越神圣”，而是持续筛选哪些测试仍然高价值，哪些测试已经不再值得保留。这与整课“evolution + selection”的主题是完全一致的。

## 本课小结

第 49 课从更高层的工程视角解释了 testing 在 embedded software development 中的地位：

1. 复杂软件只能从简单且可工作的系统逐步演化出来。
2. testing 是这个演化过程里的人工选择机制，而不是项目末期的验收步骤。
3. unit testing 需要专门构造 testing harness，给 CUT 提供可控环境。
4. ET 展示了一种简洁的 embedded unit testing 组织方式。
5. dual-targeting 是提高嵌入式开发速度和设计质量的关键策略。
6. 在 target 上跑测试时，可观测性和结果回传机制是基础问题。
7. 测试不仅验证 CUT，也验证 test environment 本身。
8. regression testing 很重要，但测试集合也需要持续筛选和维护。

从课程节奏看，这一课把前面关于 assertions、fault handling、software tracing 的思想进一步整合到 testing 主线上，开始把“如何让系统更容易发现问题”提升为一整套开发方法论。