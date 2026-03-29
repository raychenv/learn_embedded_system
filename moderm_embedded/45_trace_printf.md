# 45 用 printf 做软件追踪

这一课，Miro Samek 把话题转向 software tracing。到目前为止，这门课里调试和理解程序运行过程的主要手段，一直是单步调试器：下断点、停住、看寄存器、看变量、看内存。这种方法当然很重要，但它也有一个根本缺陷：它会停住系统，从而改变系统原本的实时行为。

这在嵌入式实时系统里尤其致命。你真正想知道的，往往不是“静止时系统长什么样”，而是“系统高速运行时，内部到底发生了什么”。因此，这一课引入最朴素也最常见的软件追踪方式：`printf` debugging，也就是在代码里插入 instrumentation，让程序自己实时报告它正在做什么。

Samek 对这类方法的评价很务实：它原始、简单、几乎人人都用，但也有明显缺点。所以这节课的目标，一方面是教你如何在 TivaC LaunchPad 上把 `printf` tracing 真正跑起来；另一方面，也会让你看到这种 primitive tracing 的短板，为下一课引出更成熟的 tracing 系统做准备。

lesson 45 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-45

## 为什么单步调试器不够

课程一开始先指出单步调试器的局限。

传统调试器的问题不是“看不到变量”，而是它只能在系统停下来时看变量。真正麻烦的是：

1. 程序一停，原本的时序关系就被破坏了。
2. 很多实时 bug 恰恰依赖特定时序，停下来后可能完全消失。
3. 在 event-driven 系统中，call stack 的价值也会下降，因为每次事件处理都会返回，执行轨迹不会长时间保留在栈上。

Samek 用了一个很形象的比喻：用单步调试器研究实时系统，就像先把生物杀死，再研究它是怎么活着的一样。

因此，更合适的方法是：让系统在几乎不被打断的情况下，自行把运行信息吐出来。这就是 software tracing。

## `printf` tracing：最原始但最通用的软件追踪方式

这一课使用的 tracing 方法，就是最朴素的 `printf` instrumentation。

具体做法非常直接：

1. 在关键函数里插入 `printf()`。
2. 当代码运行到这里时，把当前动作、参数或状态打印出来。
3. 通过这些文本输出拼出程序的实时执行轨迹。

Samek 把这种方式归类为 software tracing 的最 primitive 版本。它不是最聪明的方案，但在理解 tracing 的基本需求时很有帮助。

## 回到 lesson 41 的 TimeBomb 示例

为了让 tracing 有明确对象，这一课回到 lesson 41 的 `TimeBomb` active object。这个例子很合适，因为：

1. 状态机行为已经成熟稳定。
2. 它本来就包含明显的动作，例如 LED 开/关、按钮按下、time event 等。
3. 很适合通过 trace 观察状态机运行轨迹。

Samek 的任务是：给这份代码加入 instrumentation，使它除了控制 LED 之外，还能实时输出“刚才做了什么”。

## 直接加 `printf()`：代码能编译，但一运行就卡在库函数断点

最自然的第一步，当然就是：

1. `#include <stdio.h>`
2. 在 LED 控制函数和 `SysTick_Handler` 里加入 `printf()`。

代码甚至可以顺利编译、链接通过。但运行时立刻出问题：程序在进入 `main()` 之前就卡到某个硬编码断点里。

启用 MicroLIB 后虽然有所进展，程序能进入 `main()`，但一调用 `printf()` 又会掉进 `fputc()` 里的断点。

Samek 用这个现象解释了一个嵌入式 C 库常见事实：

1. 标准库会提供 `printf()` 的上层实现。
2. 但涉及具体硬件 I/O 的底层函数，例如 `fputc()`，库无法替你决定具体怎么输出。
3. 所以很多工具链会给这类函数放一个 dummy 实现，并在里面打死断点提醒你：“这里该你自己补”。

## 自己实现 `fputc()`：让 `printf` 有地方输出

要让 `printf()` 真正工作，就必须由应用代码自己提供 `fputc()`。

`fputc()` 的职责其实很简单：

1. 接收一个字符。
2. 把它发送到某个输出设备。
3. 成功后返回该字符。

一开始 Samek 只是写了一个空壳 `fputc()`，确认：

1. `printf()` 确实会逐字符调用它。
2. call stack 里也能看到 `printf_core()` -> `fputc()` 的路径。

这一步的意义在于先把链路打通，再决定这些字符最终送到哪里。

## 第一个输出选项：ITM

Samek 先介绍了一个比较“高级”的方式：把 `printf` 输出发到 Cortex-M 内建的 ITM（Instrumentation Trace Macrocell）。

这个方案的优点是：

1. 不需要自己占用普通串口。
2. 与调试器集成度高。
3. 是 Cortex-M 为 tracing 专门设计的通道。

但它有现实限制：

1. 需要支持 trace 的调试探头。
2. TivaC LaunchPad 板载的 Stellaris ICDI 不支持 ITM tracing。
3. 因而要想用 ITM，需要更现代一点的 probe，比如 ST-Link。

Samek 用 STM32 Nucleo 板快速演示了这种方式，说明只要调试器和 IDE 设置正确，ITM 确实能直接把 `printf` 输出显示在调试器窗口里。

## 更通用的方案：用 MCU 自带 UART 输出到虚拟串口

回到 TivaC LaunchPad，Samek 选择了更普适的方案：用芯片自带 UART0，把 `printf` 字符流直接送到虚拟 COM 口。

这也是 TivaC 板子很适合教学的原因之一：UART0 已经通过板载 USB 桥接到 PC，不需要额外飞线。

### `fputc()` 的 UART 实现

这里的 `fputc()` 做两件事：

1. 轮询 UART 状态寄存器，等 UART 不 busy。
2. 把当前字符写入 UART 数据寄存器。

这个实现很原始，但足够让 `printf` 真正把字符送出芯片。

### UART 初始化也必须补上

为了让 UART 工作，还需要：

1. 打开 UART0 和 GPIOA 的时钟。
2. 配置对应引脚的复用功能。
3. 设置波特率，例如 115200。
4. 配置 8-N-1 等串口格式。
5. 在 `BSP_init()` 中显式调用 `uart_init()`。

完成这些后，程序即使不进入调试器，也能单独通过串口对外输出 trace。

## 用串口终端查看输出：`printf` tracing 真正跑起来了

完成 UART 版本的 `fputc()` 后，只要在 PC 上打开串口终端，配置成与板子一致的波特率，就能看到 `TimeBomb` 的 trace 输出。

当你按下 reset 或按钮时，会看到与 ITM 方案本质相同的文本追踪信息。这说明：

1. `printf` tracing 在嵌入式板上完全可行。
2. 而且不一定依赖调试器。
3. 只要有一个可用输出通道，就能把程序内部行为“直播”出来。

这是本课第一个重要收获。

## 但 `printf` tracing 不应永久裸露在代码里，需要能方便开关

接下来，Samek 马上讨论一个现实问题：调试 instrumentation 不可能一直裸写在代码中，不加管理。

但“关闭 tracing”也不意味着以后把所有 `printf()` 删掉，因为：

1. 以后调试、维护时仍然可能需要它们。
2. instrumentation 本身是一种很有价值的长期资产。

因此更合理的做法是：保留 tracing 代码，但能轻松整体打开或关闭。

### 朴素做法：到处写 `#ifdef SPY`

最直接的做法，就是把每个 `printf()`、`uart_init()` 之类的 tracing 代码块，都包在：

1. `#ifdef SPY`
2. `#endif`

之间。

这样定义 `SPY` 宏时 tracing 启用，不定义时 tracing 关闭。

但 Samek 认为这种方式太笨重，因为：

1. 每一处 instrumentation 都要手动加条件编译。
2. 很容易漏掉。
3. 随着代码增多，可维护性很差。

### 更好的做法：把 tracing 自身封装成宏

Samek 推荐的方法是：定义 tracing abstraction macro，例如：

1. `MY_PRINTF(...)`
2. `MY_PRINTF_INIT()`

在 `SPY` 打开时：

1. `MY_PRINTF(...)` 展开成真正的 `printf(...)`
2. `MY_PRINTF_INIT()` 展开成 `printf_init()`

在 `SPY` 关闭时：

1. `MY_PRINTF(...)` 展开成不产生代码的空表达式
2. `MY_PRINTF_INIT()` 也展开成 `(void)0`

这里还用到了 variadic macro，解决 `printf()` 参数数量不固定的问题。

这样以后代码里就只保留 `MY_PRINTF()` 这层抽象，而不再到处散落着 `#ifdef`。

## 进一步改进：把 tracing 宏集中到单独头文件

如果 tracing 不只用于 `bsp.c`，还想在 `main.c` 或其他模块里复用，那么这些宏定义也不应在每个文件里重复。

Samek 因此又做了一步重构：

1. 把 tracing 宏与初始化接口统一放到一个头文件，例如 `my_printf.h`
2. 各个需要 instrumentation 的模块只要 include 这个头即可

同时，他还把 `uart0_init()` 重命名为更抽象的 `printf_init()`，因为在别的板子上，输出设备未必还是 UART0。这样命名更利于跨平台和复用。

## 最专业的方式：把 tracing 放进单独 build configuration

课程最后进一步把 tracing 管理提升到构建层面。

Samek 认为，在真实工程里，最合理的方式不是手工改宏，而是使用独立 build configuration。例如：

1. `dbg`：普通调试配置
2. `spy`：软件追踪配置
3. `release`：最终发布配置

这样：

1. tracing 是否启用
2. 编译宏定义
3. 优化选项
4. 相关源文件

都可以跟随配置切换，而不需要开发者手工反复修改代码。

这一点很重要，因为一旦 tracing 成为日常工具，就必须把它纳入构建体系，而不是把它当作临时 hack。

## 这节课真正要建立的认识

如果只看表面，第 45 课像是在教你“怎么把 printf 打到串口”。但 Samek 真正想建立的是对 software tracing 的整体直觉：

1. tracing 是观察实时系统活体行为的重要手段。
2. `printf` tracing 是最容易上手的 tracing。
3. 它非常有用，但也非常 primitive。
4. 真正可用的 tracing 应该具备抽象层、开关机制和构建层集成。

这也正是下一课会进一步转向更成熟 tracing 方案的原因。

## 总结

45 课的核心，是把“debugging by printf”作为最基础的软件追踪方式真正跑通，并顺带建立对 tracing 工程化管理的基本要求。

这一课最重要的结论可以概括为：

1. 单步调试器在实时系统中会破坏时序行为，因此仅靠断点和 call stack 很难理解系统在 full-speed 下的真实运行过程。
2. `printf` instrumentation 是最原始但最常见的软件追踪方式，它通过在代码中嵌入输出语句，让程序在运行中自己报告所做的动作。
3. 在嵌入式环境里，要让 `printf` 工作，通常必须自己实现底层输出函数，例如 `fputc()`，并配置具体输出通道。
4. 对 TivaC LaunchPad 来说，使用 UART0 输出到板载虚拟 COM 口是一种简单、通用且不依赖调试器的实现方式。
5. tracing 代码不应该永久裸露在工程中，而应通过统一宏封装、头文件复用以及 build configuration 来进行可控管理。
6. `printf` tracing 虽然非常有用，但它仍然只是 software tracing 的 primitive form，后续需要更低侵入、更适合 real-time 的方案来替代。

如果说前面几课一直在强调事件驱动、active object 和 state machine 的结构化优势，那么第 45 课则开始解决另一个现实问题：系统写出来以后，怎么观察它。`printf` tracing 给了一个最容易理解的起点，但 Samek 也已经清楚地为下一课埋下伏笔：真正适合实时系统的 tracing，不能停留在文本打印层面。