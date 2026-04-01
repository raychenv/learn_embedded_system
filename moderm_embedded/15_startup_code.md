# 15 把 vector table 真正做完整：栈顶、Reset 入口和异常处理函数该怎样初始化

lesson 14 已经把控制权从 IAR 默认 startup 手里接了过来。你已经能在工程里放进自己的 `startup_tm4c.c`，也知道为什么 `__vector_table` 会决定地址 0 处的内容。但那时的 startup 文件仍然只是个骨架：栈顶不可靠、Reset handler 只是占位、整张表也远没有按 TM4C123 手册补全。

所以 lesson 15 的任务很直接：把这张表真正做成可工作的工程版本。表面上这是一次“把数组填完整”的工作，实际上它牵涉三个更深的问题：

1. 栈指针初值到底该从哪里来。
2. Reset 之后 PC 应该跳到哪里。
3. 对于异常和中断，默认处理策略应该是什么。

这些问题一旦答清楚，你就不再只是“知道向量表存在”，而是真正掌握了 Cortex-M 启动入口最关键的那层控制权。

lesson 15 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-15

## lesson 14 已经完成了“替换”，lesson 15 要完成的是“正确初始化”

Samek 一开始就回顾上一课：你已经证明自己的 `startup_tm4c.c` 能被链接进来，并放到 `.intvec` section，也就是地址 0 的位置。但那还只是形式上的成功。

真正的问题是：CPU 上电或复位后，会把地址 0 处的第一个 word 当作初始 `SP`，把地址 4 处的第二个 word 当作初始 `PC`。如果这两个值不对，程序根本没有机会进入后面的世界。

所以 lesson 15 的第一件事，不是多加几个 handler，而是先把最前面的两个入口值做对。

## 初始栈指针为什么不能硬编码

最直观的做法，似乎是像演示代码那样直接写一个 RAM 顶地址进去，例如 `0x20004000`。但 Samek 立刻指出，这样做有明显问题：

1. 它和 IDE 里配置的 `CSTACK` 大小脱节。
2. 一旦链接脚本里栈大小变化，硬编码地址就可能失效。
3. 这会破坏与你当前工具链工作流的兼容性。

也就是说，如果你手写 startup，却把栈顶写死，那表面看起来是“自己掌控了向量表”，实质上却是失去了和 linker script/editor 的一致性。

## `CSTACK$$Limit`：让 linker 来告诉你真正的栈顶

Samek 的解决方案非常典型，也很值得记住：既然 `CSTACK` 本来就是链接器负责布局的 section，那最可靠的栈顶地址，理应也由链接器给出。

在 IAR 里，链接器会自动为 section 生成诸如：

1. `section-name$$Base`
2. `section-name$$Limit`

这样的符号。对于栈来说，最关键的是：

```c
extern int CSTACK$$Limit;
```

然后在向量表首项中使用它的地址：

```c
(int)&CSTACK$$Limit,
```

这里有几个很重要的知识点被串在了一起。

第一，ARM 栈是向低地址增长的，所以初始 `SP` 应该指向栈区的高端，也就是 `Limit`。

第二，这个符号是 linker 生成的，不是编译器自己知道的，所以你必须先用 `extern` 告诉编译器“这个名字会在后面被别人提供”。

第三，向量表数组元素被定义成 `int`，而 `&CSTACK$$Limit` 是指针，因此还需要显式类型转换。

这一段看似只是语法细节，实际上是 startup 代码中非常典型的“编译器世界”和“链接器世界”交汇点。

## 这一步最有价值的地方：你没有破坏 IDE 里对栈大小的配置

Samek 不是写完就算，而是专门去验证：如果你在 IAR 里修改 `CSTACK` 大小，对应的 `CSTACK$$Limit` 地址会不会变化？程序下载后 `SP` 寄存器会不会跟着变化？

实验结果证明，这种写法的好处就在于：

1. 栈大小仍由链接器脚本和 IDE 配置控制。
2. startup 文件只是读取最终结果，而不是自己另起炉灶。
3. 所以整个工程仍保持一致性。

这其实是 very important 的工程思维。真正好的 startup 代码，不是“我自己写了一份能跑的地址”，而是“我写的入口逻辑和工具链的内存配置保持同源”。

## Reset handler：为什么最终还是复用 `__iar_program_start`

解决完栈顶，lesson 15 接着处理地址 4 处的第二项，也就是 Reset handler。

从 CPU 的角度看，这一项的意义非常简单：复位后把这个值装进 `PC`，从这里开始执行。

那你是不是应该现在就自己写一套完整 reset 过程？Samek 的答案很务实：没必要。因为 IAR 的 `__iar_program_start` 已经把 startup code、数据初始化、FPU 初始化等流程做得很好，所以当前最合理的做法，是在自己的 vector table 里继续指向它。

代码形式大致是：

```c
void __iar_program_start(void);

(int)&__iar_program_start,
```

这里和 `CSTACK$$Limit` 类似，也要先声明符号，再把函数地址显式转成整型放进向量表。

这一步的思想很值得注意：自己接管 vector table，并不等于什么都得从零重写。你完全可以把“入口控制权”拿过来，但仍复用可靠的运行时初始化实现。

## 函数地址为什么也能放进表里

lesson 15 顺便也借机讲了一个 C 语言层面的细节：函数也是有地址的，因此可以把函数名当作入口地址使用。Samek 还特别提到，很多代码会省略 `&`，直接写函数名；语义上是成立的，但他更推荐显式写 `&`，因为这样更清楚地表达“这里放的是地址，而不是调用函数”。

这其实和整节课的风格一致：尽量用不会制造歧义的写法，把“入口地址表”这件事写得更明确。

## 向量表的后半部分：不是随便填函数名，而是必须严格跟 datasheet 对齐

做完栈顶和 Reset 之后，Samek 才继续往后填异常和中断入口。他特别强调一点：Cortex-M 的标准异常项并不是连续无缝排列的，中间会有保留槽位（reserved slots）。

这意味着：

1. 你不能“按看起来顺眼”的顺序自己重新排。
2. 你必须严格保留 datasheet 规定的布局。
3. 保留项就应该填 0。

这是 startup code 特别容易出错的地方。因为表面看你只是少填了几个没用的异常，但本质上却可能把后面所有 IRQ 项整体错位，导致 CPU 进入完全错误的入口。

## CMSIS 的价值：handler 名字不是你自己随意起的

课程还特别提到，像 `NMI_Handler`、`HardFault_Handler`、`MemManage_Handler`、`BusFault_Handler`、`UsageFault_Handler`、`SysTick_Handler` 这些名字，并不是随便写的，而是 CMSIS 约定的一部分。

这背后的价值不只是“命名统一”这么简单。更重要的是：

1. 第三方代码、RTOS、BSP 往往都假定这些标准名字存在。
2. startup code、头文件、工具链支持也围绕这些名字组织。
3. 遵守标准命名可以让你的工程更容易与外部组件协同。

所以 lesson 15 其实是在第一次认真把“CMSIS 不是只有寄存器定义，它还是接口约定”这件事落实下来。

## Samek 不喜欢异常处理里写死循环

到了具体 handler 实现，Samek 这里有一个很鲜明的工程立场。很多 startup 模板里，异常处理函数默认都是：

1. 进来后死循环。
2. 让 CPU 卡死在那里，方便调试器断进去看。

这种写法在实验阶段似乎方便，但 Samek 明确指出：很多产品把这种代码直接带进量产，结果一旦进入 HardFault 或其他异常，设备就完全假死，用户只能断电重启。这本质上就是 denial of service。

所以他的做法是：不要在每个异常处理里自己写死循环，而是统一调用一个不可恢复错误入口，例如 `assert_failed()`。

## `assert_failed()`：把异常处理从“卡死”改成“统一故障入口”

在 lesson 15 的工程里，startup 文件中定义了几个关键异常处理函数，例如：

```c
__stackless void HardFault_Handler(void) {
    assert_failed("HardFault", __LINE__);
}
```

对应的 `bsp.c` 里则提供了：

```c
__stackless void assert_failed (char const *file, int line) {
    NVIC_SystemReset();
}
```

这套写法背后的思路非常工程化：

1. 每个异常 handler 负责标识“发生了什么”。
2. 统一错误入口负责做 damage control。
3. 当前示例里 damage control 很简单，就是系统复位。
4. 实际产品里还可以加入日志、错误记录、故障上报等动作。

也就是说，lesson 15 不只是教你“异常发生时跳到哪里”，而是开始教你“异常发生后，系统该怎样有策略地收场”。

## `__stackless` 的含义：在异常上下文里尽量减少额外栈操作

源码里处理函数和 `assert_failed()` 前面都用了 `__stackless`。虽然 Samek 在这一课没有把它展开成主题，但它的意图很明确：

1. 当前你正在写的正是与栈、异常、启动直接相关的代码。
2. 此时尽量减少编译器额外生成的函数栈框架，会让行为更可控。
3. 尤其在 fault/exception 处理里，这样做更利于分析。

所以这一步和整个 lesson 15 的气质一致：在最底层的系统入口处，尽量把控制流写得简单、透明、可验证。

## weak handler：为什么整张表不需要你一次性写完所有函数体

TM4C123 的中断源很多，如果每一个都手写一个完整空函数，既啰嗦也不利于维护。Samek 的处理办法是定义一个 `Unused_Handler()`，然后用 `#pragma weak` 把大量暂时未实现的 IRQ handler 弱绑定到它：

```c
#pragma weak SysTick_Handler   = Unused_Handler
#pragma weak GPIOPortA_IRQHandler = Unused_Handler
```

这招很重要，因为它同时解决了两个问题：

1. 你的向量表可以先列出完整入口集合。
2. 暂时没实现的 handler 会统一落到一个默认处理路径。
3. 将来只要在别处提供一个同名强定义，就能自然覆盖默认弱定义。

这其实是 lesson 14 那套“symbol replacement”思想在 startup code 里的进一步应用。只不过上一课替换的是 `__vector_table`，这一课替换的是各个具体 handler 的默认实现。

## `Unused_Handler()` 的意义：把“未实现中断”变成可诊断故障

默认 handler 并不是“什么都不做”。在 Samek 的版本里，`Unused_Handler()` 同样走 `assert_failed()` 路径。这意味着：

1. 如果某个中断被意外使能且真的触发了，系统不会静默错过。
2. 它会立刻走到统一故障处理入口。
3. 在调试阶段，这比“悄悄返回”或者“莫名其妙跑飞”要有价值太多。

这又一次体现了 Samek 的工程偏好：宁可 fail fast，也不要悄悄把错误吞掉。

## SysTick handler 为什么此时仍然是空的

你会注意到，在 `bsp.c` 里已经出现了：

```c
void SysTick_Handler(void) {
}
```

这一步看起来很普通，但它其实是一个很重要的铺垫。因为这意味着：

1. 向量表已经为后面 lesson 16 的 SysTick 中断预留好了正确入口。
2. 现在还没填逻辑，只是因为本课的重点仍是 startup 和异常框架。
3. 接下来一旦开始讲中断，这个入口就可以立刻投入使用。

也就是说，lesson 15 并不是和 lesson 16 割裂的；它是在为“真正开始处理中断”先把最底层的 infrastructure 搭好。

## 这一课真正完成的事情：把 startup 从演示级别推进到工程级别

如果把 lesson 14 和 15 连起来看，区别会非常明显。

lesson 14 更像是在证明：“我能提供自己的 vector table，而且链接器会听我的。”

lesson 15 则是在把这张表真正做成能长期支撑工程演进的样子：

1. 栈顶来自 linker 生成符号，而不是手写常数。
2. Reset 入口复用可靠的 `__iar_program_start`。
3. 异常项和保留槽位严格遵守芯片手册布局。
4. handler 名称遵守 CMSIS 约定。
5. 默认 handler 和 fault handler 统一走 `assert_failed()`。
6. 未实现 IRQ 通过 weak alias 收口到 `Unused_Handler()`。

到这里，你的 startup code 才开始真正像一个可以继续扩展的底座，而不是一次性实验脚本。

## 总结

lesson 15 的主题，是把上一课刚接管到手的 vector table 真正初始化正确，并把异常/中断入口组织成可维护的工程结构。Samek 先解决了最关键的两个入口值：通过 `CSTACK$$Limit` 让初始 `SP` 与链接器的 `CSTACK` 配置保持一致，通过 `__iar_program_start` 继续复用 IAR 已验证的 reset 启动流程。

然后，这一课进一步把标准异常项、保留槽位和芯片级 IRQ 按照 datasheet 与 CMSIS 约定完整铺开，并用 `assert_failed()`、`Unused_Handler()` 和 `#pragma weak` 建立了一套统一的默认故障处理策略。与其让异常处理里塞满死循环，不如让系统在不可恢复错误时走统一入口，做复位、记录或其他 damage control。这是这一课非常明确的工程取向。

从课程结构上看，lesson 15 是 startup 主题的收束点，也是中断主题的真正起点。到这里，地址 0 处的向量表、Reset 入口、异常处理框架和默认 IRQ 路径都已经准备好了。所以下一课 Samek 才能顺理成章地开始讲真正的 interrupt handling：既然入口都已经搭好，现在终于可以让 SysTick 等中断源真正把程序“打断”起来了。
