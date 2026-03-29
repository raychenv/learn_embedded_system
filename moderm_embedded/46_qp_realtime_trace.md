# 46 用 QP/Spy 做实时软件追踪

这一课，Miro Samek 继续讲 software tracing，但重点已经从上一课的 primitive `printf` tracing 升级到一个成熟得多的 tracing system：QP/Spy，也就是 QSPY。第 45 课已经证明，软件追踪对理解 embedded system 的实时行为非常有价值，但 `printf` 方法也暴露出了明显缺陷：它把“生成 trace 数据”和“把 trace 发送出去”绑在一起，而且输出还是人类可读的 ASCII 文本，导致时间开销巨大、侵入性很强。

这一课的任务，就是说明一个更适合 real-time software 的 tracing 方法应该是什么样，并且通过 QP/Spy 展示：软件追踪完全可以做到既高价值，又低侵入。Samek 还会进一步解释 QP/Spy 的二进制协议、字典机制以及为什么 event-driven framework 天然特别适合被 tracing。

lesson 46 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-46

## 为什么 `printf` tracing 不够聪明

课程一开始就先总结了上一课 `printf` tracing 的问题。Samek 的核心批评并不是“输出 trace 不好”，而是：`printf` 把太多事情都放在了 time-critical path 上做。

例如一条简单 trace 可能花掉数百微秒，几条连续输出甚至能消耗数毫秒，而且这些开销还可能发生在：

1. 中断上下文里
2. 关键状态机动作里
3. 本该尽快返回的实时路径上

Samek 用消防员的比喻来解释这一点：

1. 你需要的是在救火的同时快速记录现场。
2. `printf` 却相当于让消防员停下来，先写一篇详细的书面报告。
3. 更糟的是，这篇报告还要先翻译成人类可读文本。

也就是说，真正荒谬的不是“记录”，而是“在最不该慢的时候做慢事”。

## 更合理的 tracing 策略：快速记录，延迟发送

Samek 据此提出一个更合理的软件追踪基本架构：

1. 在 time-critical path 中，只做非常快的记录。
2. 记录内容应是 native binary，而不是格式化成文本。
3. 这些记录先写入 MCU 内部 RAM buffer。
4. 真正慢的“往外发送”动作，应当延后到 CPU 空闲时再做。

这就是 QP/Spy 的核心思路。

与上一课相比，它最大的变化是把两件事彻底解耦：

1. record trace data
2. transmit trace data

在实时系统里，这种解耦非常关键。

## 为什么 idle thread 是发送 tracing 数据的理想时机

为了实现“延后发送”，系统需要一个合适的执行点。对大多数 RTOS 或事件驱动 kernel 来说，这通常就是 idle thread 或 idle callback。

Samek 提醒，当前示例里使用的是 QV kernel。虽然它不是完整 RTOS，但同样具有“系统空闲时调用 `QV_onIdle()`”的能力。因此：

1. 关键路径里只往 buffer 里塞 trace record。
2. 真正把 buffer 中数据发到 UART 的动作，放在 `QV_onIdle()` 里做。

这正好实现了“快路径只记录，慢路径再发送”的目标。

## QP/Spy 已经内建在 QP/C 里，只需激活即可

课程接着开始具体替换上一课的 `MY_PRINTF` tracing。

Samek 先强调一个很实际的点：QSPY 并不是额外拼凑的第三方工具，而是 QP/C framework 自带的 tracing subsystem。因此你不需要再去写一套新的 tracing framework，只需要：

1. 在工程中打开 `Q_SPY` 宏
2. 引入 QS 相关源文件
3. 在 BSP 中提供板级初始化和传输代码

即可把 tracing 能力接入现有 QP/C 工程。

## QSPY 的 trace record 不是文本，而是二进制记录

在代码层面，QSPY 最大的变化是：你不再写 `printf("...")` 这种 format string，而是显式插入二进制字段。

典型结构是：

1. `QS_BEGIN_ID(...)`
2. 若干 `QS_str()`、`QS_u8()` 等字段输出宏
3. `QS_END()`

这些宏直接把原始数据写入内部 tracing buffer，而不是先格式化成字符串。

Samek 这里强调了两个优点：

1. 生成 binary data 比生成 ASCII 文本快得多。
2. 不同数据类型被保留为 native representation，而不是先被迫“翻译成人类语言”。

这一步正是从 primitive tracing 到 real-time tracing 的关键跨越。

## 板级支持：初始化 buffer、时间戳、UART 和 host 输入

为了让 QSPY 真正工作，BSP 还需要补齐几部分支持代码。

### 1. RAM tracing buffer

QSPY 首先需要一个内部 buffer，记录 binary trace records。这是整个系统的“黑匣子”。

### 2. host 输入缓冲

QSPY 不只是 target 往 host 单向输出，它还支持 host 向 target 发送命令，因此还需要一小块输入缓冲区和 UART RX 处理。

### 3. 时间戳源

QSPY 会给 trace record 加精确时间戳，因此 BSP 还要初始化一个硬件 timer，并在相应 callback 中读取它。

### 4. 在 idle callback 里发送数据

最关键的是：真正发 UART 的代码移到了 `QV_onIdle()` 中，并且不再 busy-wait，而是只在 UART ready 时才发一个或几个字节。

这就使得发送过程被切碎并分散到系统 idle 时段，而不会在关键路径中卡住 CPU。

## 与上一课的 `printf` 版本对比：核心不是“能不能输出”，而是“输出代价落在哪”

Samek 特别强调，这里的代码表面上和上一课也有些相似：

1. 都需要 UART。
2. 都要做板级初始化。
3. 都会最终把数据送到 host。

但真正本质的差别在于：

1. `printf` 是在关键路径里做格式化 + 发送。
2. QSPY 是在关键路径里仅做极快的 binary record，发送被延迟到 idle。

这就是两种 tracing 方法在实时性上差异如此巨大的原因。

## 先替换应用级 user trace record，再接上 QS 源文件

课程中的迁移步骤很直接：

1. 先把上一课的 `MY_PRINTF()` 替换成 QS trace record。
2. 再修改 QM 模型，把自动生成的 `main.c` 中残留的 tracing 也改成 QS 风格。
3. 随后把 `qpc/src/qs` 目录下的 QS 源文件加入工程。
4. 只在 `spy` build configuration 中包含它们，而不在 `dbg` 中启用。

Samek 借此再次体现了 build configuration 的价值：成熟 tracing 系统不该靠“手动注释代码”开关，而应当成为构建体系的一部分。

## 用普通串口终端看 QSPY 数据会是乱码，因为它本来就是二进制

迁移完成后，Samek 先故意用上一课的普通串口终端去看输出。结果当然是一堆乱码。

这一步非常关键，因为它再次说明：

1. QSPY 不是“更高级的 `printf`”。
2. 它输出的不是人类可直接读懂的文本。
3. 它输出的是 binary protocol，需要专门的 host-side parser。

因此要查看它，必须使用专门的 host utility，也就是 `qspy` 程序，而不是普通串口终端。

## QSPY host utility：把 binary records 还原成人类可读 trace

当用 `qspy -c COMx` 连接到目标板后，重置板子并按下 SW1，就能看到和上一课 `printf` 类似、但更加丰富的 trace 输出。

这说明：

1. target 侧只需要发紧凑二进制。
2. host 侧再负责把它解析成人类可读文本。
3. 这种“把解释成本转移到 host” 的做法，对 MCU 来说极其有利。

Samek 还指出，这些 trace record 都带有精确时间戳，这是上一课 `printf` 版很难优雅做到的。

## 性能对比：QSPY 的开销比 `printf` 小近两个数量级

课程用 logic analyzer 做了非常有说服力的对比：

1. 在 `SysTick_Handler` 中，单次 QSPY 记录开销只有几微秒量级。
2. 同等信息用 `printf` 输出则接近 0.8 毫秒，也就是约 800 微秒。

Samek 给出的数字大意是：QSPY 在这个例子上比 `printf` tracing 快近 100 倍。

这正是它“更适合 real-time”的最直接证据。

## 应用级 trace 只是冰山一角，框架内建 trace 更强大

到这里，QSPY 还只是拿来做 user-defined trace record，也就是你自己在应用里插入的记录。

但 Samek 认为，真正强大的地方在于：QP framework 自带大量 predefined trace records，而这些 records 对 event-driven system 特别有价值。

原因在于 inversion of control：

1. 在 event-driven framework 中，应用的大量关键行为都必须经过 framework。
2. 例如 event dispatch、state entry、state exit、transition 等。
3. 因此只要 framework 本身被正确 instrumentation，很多行为根本不需要应用自己再加 trace，也能自动被观测到。

这正是 QP/Spy 与传统 RTOS tracing 的本质区别之一。RTOS 当然也能 trace task、semaphore、context switch，但它不理解 state machine 和 event dispatch 的语义；而 event-driven framework 天生就理解这些更高层行为。

## 要把地址解析成符号，QSPY 需要 dictionary records

启用 framework 内建 trace 后，输出中会出现很多更底层的信息，例如：

1. event dispatch
2. state exit
3. state entry
4. transition

但一开始这些记录里的对象和函数常常只是地址值，例如：

1. RAM 地址对应某个 active object
2. ROM 地址对应某个 state-handler

为了把这些地址转成可读符号名，QSPY 需要 dictionary records。

Samek 介绍了三类核心字典：

1. object dictionary：把对象地址映射到对象名
2. signal dictionary：把 signal 值映射到信号名
3. function dictionary：把函数地址映射到函数名

有了这些字典，host 端就能把“原始二进制地址”翻译成更可读的 symbolic trace。

### QM 还能自动生成 function dictionaries

其中一个很实用的点是：QM 可以自动为状态机生成 function dictionaries。只要在属性里勾选 соответствующие 选项，QM 就会在生成代码时自动插入这些字典初始化记录。

这再次体现了 QM 与 QP/C 生态的协同：建模工具不仅能生成状态机代码，也能顺手把 tracing 所需的符号信息一并接好。

## QSPY 的全局过滤器：不要所有 trace record 都一股脑打开

当启用全部预定义 trace record 后，输出很快会变得过于密集，例如每个 tick 都有记录，导致信息洪流淹没有用内容。

因此 Samek 引入了 QS global filter：

1. 可以精确启用某类 record
2. 也可以单独禁用某类 record

例如把高频 tick record 过滤掉，就能让输出立即清爽很多。

这说明成熟 tracing 系统不只是“能记录”，还必须“能筛选”。否则信息再丰富，也无法高效使用。

## QSPY 的协议本身也有工程价值：二进制更紧凑，还能检测错误与丢包

课程最后一段开始“掀开盖子”，解释 QSPY 为什么不仅快，而且协议设计本身也很聪明。

### 1. Binary format 本身就是压缩

Samek 把同一段 trace 保存成：

1. 原始二进制数据
2. host 端格式化后的可读文本

并比较文件大小，发现 binary data 只有文本版的大约三分之一左右。这说明 binary format 本身就是天然压缩，相比 `printf` 文本输出，通信带宽利用率更高。

### 2. 协议能检测错误和丢失记录

Samek 还故意删掉二进制 trace 文件中的一个字节，再让 `qspy` 以 playback 模式解析它。结果：

1. `qspy` 检测到了 checksum 错误
2. 还能指出 sequence discontinuity，也就是哪条记录丢了

这表明 QSPY 不是“盲目读一串字节”，而是有自同步、自校验能力的 tracing protocol。

### 3. 允许按任意 chunk 从缓冲区取数据

QSPY 协议的另一个关键优势是：target 可以在 idle 时每次只取出 buffer 中任意一小段数据发送，而不必恰好对齐 record 边界。host 端仍然能通过协议 framing 和 checksum 重新找回正确 record 边界。

这对 real-time 系统非常重要，因为它意味着：

1. 发送过程可以被切得很碎。
2. 每次只占用很短 idle 时间。
3. 不需要为了“发完整一条记录”而长时间占用 CPU。

## 这节课真正建立的 tracing 思维

如果把第 46 课压缩成一个最核心的工程认知，那就是：

1. tracing 对 real-time system 是必要的。
2. 但 tracing 必须尊重系统的实时性。
3. 所以 tracing 系统的好坏，关键不在“打印得多漂亮”，而在：
   - 是否把记录与发送解耦
   - 是否使用紧凑的 binary format
   - 是否能利用 idle time 慢慢输出
   - 是否能用 framework 内建语义自动捕捉高层行为

Samek 实际上是在把 software tracing 从“程序员的小技巧”提升成“框架级观测基础设施”。

## 总结

46 课的核心，是用 QP/Spy 展示一种真正适合实时嵌入式系统的软件追踪方式，并把 tracing 从 `printf` 这种原始方法提升到成熟的框架级方案。

这一课最重要的结论可以概括为：

1. `printf` tracing 的根本问题，在于它把 trace 生成、格式化和发送都放进了 time-critical path，导致高延迟和强侵入性。
2. 更合理的 tracing 架构应当在关键路径中只快速记录 native binary 数据到 RAM buffer，再在系统 idle 时异步发送给 host。
3. QP/Spy 正是基于这种思想：记录和发送解耦，发送发生在 `QV_onIdle()` 之类的空闲点上，从而显著降低对实时性的破坏。
4. QSPY 的 binary trace record 比文本输出紧凑得多，也避免了 MCU 在关键路径里做昂贵的字符串格式化。
5. 相比 `printf`，QSPY 在示例中的 tracing 开销降低了近两个数量级，更适合实时嵌入式软件。
6. 除了应用级自定义 trace，QSPY 最大的优势在于 QP framework 内建了大量预定义 trace records，能够自动观察 event-driven system 的 dispatch、entry/exit、transition 等高层行为。
7. 通过 object/signal/function dictionaries，QSPY 可以把 target 发来的原始地址与编号还原成可读的 symbolic trace，而 QM 还能帮助自动生成其中一部分字典。
8. QSPY 的协议本身也非常关键：binary format 更紧凑，还能检测错误、丢包，并允许 target 按任意 chunk 发送 buffer 数据而不必对齐记录边界。

如果说第 45 课给了一个最容易理解的软件追踪起点，那么第 46 课则展示了 tracing 真正走向工程化时应该具备的样子。它不再只是“输出几行日志”，而是开始成为事件驱动框架的一部分，成为观察、测试、诊断和后续更高级工具链的基础。