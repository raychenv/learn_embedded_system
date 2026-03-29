# 44 Active Object 的交互、共享状态与可变事件

这一课，Miro Samek 继续讨论 active objects for real-time，但重点从上一课“AO 能否满足 RMS”进一步推进到“AO 之间如何交互才不会破坏实时性”。第 43 课中的示例其实还比较理想化，因为两个 active object 几乎没有真正的数据交互：`Blinky1` 只是周期运行，`Blinky2` 只是响应按钮，然后两者彼此几乎独立。

现实系统当然不会这么简单。多个 active object 迟早要协作，而协作方式的选择，会直接影响：

1. 是否会引入 race condition。
2. 是否需要 mutual exclusion。
3. 是否会产生 priority inversion。
4. 最终是否还能满足 hard real-time deadline。

因此，这一课故意从一个“错误但常见”的做法开始：让两个 active object 通过共享全局变量交换 blinking pattern。然后 Samek 再一步步把它改造成基于 event parameter 的交互，从而说明为什么 active object 的真正优势，不只是“能跑 state machine”，而是“能把交互也变成异步消息”。

lesson 44 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-44

## 新需求：Blinky2 改变 Blinky1 的闪烁模式

课程首先扩展上一课示例：每按一次按钮，`Blinky2` 都要改变 `Blinky1` 的 blinking pattern。

`Blinky1` 的 pattern 由两个量决定：

1. `ticks`：time event 的 tick 数
2. `iter`：工作负载循环的迭代次数

因此，如果 `Blinky2` 能修改这两个值，就能让 `Blinky1` 在“慢闪/快闪”等不同模式间切换。

## 先故意做错：用全局变量共享 pattern

Samek 首先故意采用最传统、也最危险的方式：把这两个 pattern 参数做成全局变量，例如：

1. `g_ticks`
2. `g_iter`

然后：

1. `Blinky1` 在 timeout 时读取它们。
2. `Blinky2` 在按钮按下时修改它们。

为了让 pattern 多次切换，`Blinky2` 还用两个 const 数组和一个 sequence counter，在每次按键时切换到不同组合。

从功能角度看，这种写法很自然，甚至在实验室里跑起来也似乎“没问题”。但 Samek 立即指出，这里面已经埋下 race condition。

## Race condition 可能很隐蔽，但一定会发生

关键问题是：`g_ticks` 与 `g_iter` 是一组有语义关联的数据，但它们不是原子更新的。

因此完全可能发生：

1. `Blinky2` 先写了新的 `g_ticks`
2. 还没来得及写新的 `g_iter`
3. `Blinky1` 恰好在这个时间窗里运行
4. 它读到了“新 ticks + 旧 iter”的不一致组合

这就是典型的 race condition。Samek 特别提醒，这类错误最危险的地方在于：

1. 时间窗很窄。
2. 实验室里往往不容易复现。
3. 一旦量产部署，迟早会被用户踩到。

为了让问题更容易暴露，他故意在两次赋值之间加了一段 workload，把 race window 放大。但他也强调，这个 workload 不是问题的根源，它只是放大镜。真实系统里，串口接收、慢速总线、计算延迟都可能自然形成这种间隔。

## 结果：系统被拖进 denial of service

加入这段间隔后，logic analyzer 很快就显示出异常：

1. `Blinky1` 开始异常频繁地占用 CPU。
2. 系统几乎被它“锁死”。
3. 其他活动难以继续进行。

Samek 解释说，这并不是 `Blinky1` 自己突然变坏，而是因为不一致的 `g_ticks/g_iter` 组合导致：

1. `Blinky1` 仍然执行旧的大 workload。
2. 但 timeout 却按新值被设置得过短。
3. 结果它还没处理完当前事件，下一次 timeout 就已在队列中等待。
4. 最终 `Blinky1` 几乎连续运行，形成一种 denial of service 状态。

这个例子非常说明问题：共享状态不仅会带来“值不对”的 bug，还可能直接破坏系统的时序性质和实时性。

## 传统补救方式：互斥，但这会引入优先级反转

既然共享状态有 race，传统 RTOS 思路自然会想到：加 mutual exclusion。

由于 active object 不能使用 blocking mutex，Samek 选用了上一课提到过的 non-blocking mutual exclusion 机制，也就是 selective scheduler lock（QXK scheduler lock）。

这样做确实能解决一致性问题：

1. `Blinky2` 修改 `g_ticks/g_iter` 时，`Blinky1` 暂时不能打断它。
2. 因而 `Blinky1` 再也读不到半更新的数据。

从功能上看，问题被修好了。

## 但互斥修好正确性后，又破坏了实时性

logic analyzer 很快揭示出新问题：虽然系统不再失控，但 `Blinky1` 在模式切换期间出现了明显 hiccup，某些周期超过了其 2ms deadline。

原因很典型：bounded priority inversion。

即使：

1. scheduler lock 是非阻塞的，
2. 且 duration 有上界，

它仍然会让低优先级的 `Blinky2` 在持锁期间阻止高优先级的 `Blinky1` 运行。对 RMS/RMA 来说，这段时间必须计入高优先级任务的 effective CPU utilization。结果就是：

1. 系统在这段时间内的利用率暂时升高。
2. 超出了可调度边界。
3. 高优先级任务 deadline 被打破。

Samek 借此重申：

1. 不加互斥，correctness 错。
2. 加互斥，timing 可能错。

这正是 shared-state concurrency 最令人头疼的两难。

## 真正的事件驱动替代方案：不要共享，而是发带参数的事件

课程接着进入真正的 event-driven 解法。

既然 `Blinky2` 想让 `Blinky1` 改变 pattern，那么最自然的方式不是“改它的共享变量”，而是“给它发一个显式 change-pattern event”。

这就引出了本课第一次正式使用的 event parameter 技术。

### 定义带参数事件类

Samek 定义一个新的事件类 `BlinkPatternEvt`，继承自 `QEvt`，并添加两个参数：

1. `ticks`
2. `iter`

这样一来，原来那两个全局变量就不再是共享状态，而被封装成一次性发送给 `Blinky1` 的消息内容。

这非常符合 event-driven architecture 的精神：把“共享内存中的状态”变成“通过消息传递的信息”。

## 第一版看似正确的写法：static mutable event

接下来，Samek 故意展示了一种很多初学者都会写出来、但其实仍然有问题的做法：

1. 在 `Blinky2` 内部定义一个 `static BlinkPatternEvt`。
2. 每次按钮按下时，修改它的参数。
3. 然后把这个 event post 给 `Blinky1`。

表面上看，这种写法似乎已经避免了共享全局变量：

1. `Blinky1` 不再读全局变量。
2. `Blinky2` 也确实通过 event 把参数送了过去。
3. 运行时看起来 pattern 也能正确切换。
4. 而且 mutual exclusion 也不再需要，deadline hiccup 也消失了。

看上去这就是完美方案。

## 但这仍然是共享，只是换了个伪装形式

Samek 紧接着指出：这其实还是 sharing，只不过从“共享全局变量”变成了“共享可变事件对象”。

原因很简单：

1. `bpevt` 虽然是 `static` 局部变量，不是全局符号，
2. 但系统里依然只有这一份对象实例。
3. `Blinky2` 会反复修改它。
4. `Blinky1` 通过事件队列拿到的是指向它的指针。
5. 因此两边依然可能并发访问同一块可变内存。

这和 immutable event 完全不同。像 `BUTTON_PRESS` 这种 immutable event：

1. 可以声明为 `const`
2. 可放在 ROM
3. 不会被任何一方修改
4. 因而可以安全共享

而 mutable event 则完全不是这样。它的生命周期管理、所有权转移、并发安全要求都复杂得多。

## 本课真正要说明的是：mutable event 不能像 immutable event 那样随便复用

课程在这里没有给出完整的最终方案，但已经把关键认知立住了：

1. 事件驱动推荐用消息传递替代共享状态。
2. 但这并不意味着“只要把数据塞进 event 就自动安全”。
3. immutable event 和 mutable event 的并发语义差别极大。
4. mutable event 必须由框架提供更严格的内存管理和线程安全机制来支持。

Samek 明确表示，QP/C 这类 active object framework 会替你处理这部分复杂性，而且是 generic、thread-safe 的。但这一课的重点，先是让你真正意识到问题本身，而不是匆忙背下某个 API。

## 这一课的核心逻辑链

如果把本课压缩成一条最重要的逻辑线，其实就是：

1. 共享状态会导致 race condition。
2. 用互斥能修 correctness，但会引入 priority inversion，损害 real-time performance。
3. 用 event 传递参数能避免共享状态，是架构上更好的方向。
4. 但 mutable event 不是简单变量替换，也需要严肃的生命周期和所有权管理。

这条链非常重要，因为它把 Active Object 的真正价值从“框架风格”推进到了“实时系统设计方法”。

## 总结

44 课的核心，是通过一个具体交互场景，说明 active object 真正优于传统 shared-state concurrency 的地方，不只是“代码更整洁”，而是“可以同时保住正确性和实时性”。

这一课最重要的结论可以概括为：

1. 当多个 active object 需要协作时，如果仍然通过共享全局变量交换数据，就会重新掉回 shared-state concurrency 的老问题中。
2. 共享一组语义上相关但非原子更新的数据，会导致 race condition，而且这种错误往往在实验室里不容易稳定复现。
3. 即便使用 non-blocking mutual exclusion（如 selective scheduler lock）修复一致性，仍会引入 bounded priority inversion，从而破坏高优先级任务的 RMS/RMA 调度保证。
4. 更符合事件驱动思路的方案，是把交互数据封装进 event parameter，通过异步事件发送给接收方，而不是直接共享状态。
5. 但 mutable event 不能像 immutable event 一样被随意静态复用；把一个可变事件对象作为单实例在多个并发上下文间传递，本质上仍然是共享。
6. 因此，事件驱动真正成熟的做法，不只是“把变量搬进事件”，而是依赖框架提供对 mutable event 生命周期和并发安全的正确管理。

如果说第 43 课证明了 active object 与硬实时调度兼容，那么第 44 课则进一步说明：一旦涉及对象间交互，active object 的真正优势来自“用事件取代共享”。同样重要的是，这一课也提醒你，事件驱动不是简单换个语法，而是连数据所有权和对象交互方式都要一起重构。