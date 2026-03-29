# 43 Active Object 与硬实时调度

这一课，Miro Samek 暂时从状态机语义回到 active objects 和 real-time programming 的结合点上。前面从 lesson 34 开始，你已经多次使用 active object，但它们的优势主要是从 event-driven programming、并发封装和避免共享状态的角度来解释的。这一课则专门回答一个很多嵌入式开发者最关心的问题：active object 到底能不能像传统 RTOS 线程那样满足 hard real-time deadline，尤其是在 Rate Monotonic Scheduling / Analysis 这种经典实时理论框架下。

为此，Samek 没有重新发明一个新例子，而是回到 lesson 27 的 QXK RTOS 示例。那一课原本就是为了展示 preemptive priority-based scheduling 和 RMS/RMA 原理，因此正好适合作为对照组，把传统线程版本和 active object 版本直接放在一起比较。

lesson 43 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-43

## 从 lesson 27 的 RTOS 示例重新出发

课程回顾的原始程序包含两个传统线程：

1. `blinky-1`
2. `blinky-2`

其中：

1. `blinky-1` 周期运行，周期约为 2 毫秒，是高优先级任务。
2. `blinky-2` 优先级较低，等待按钮 `SW1` 触发后执行。
3. 空闲时则由 QXK 的 idle thread 运行。

这个例子本身就是一个典型的 RMS 案例：运行频率更高的任务拥有更高优先级，因此它总能及时抢占其他就绪线程，从而满足自己的 2ms deadline。

Samek 先用 logic analyzer 重现这一点，让你再次看到：

1. `blinky-1` 周期稳定。
2. `blinky-2` 只有在按钮按下时才运行。
3. 即便 `blinky-2` 就绪，高优先级的 `blinky-1` 仍然总能按期执行。

## 核心问题：Active Object 也能满足 RMS 吗

在引出问题时，Samek 的表述非常直接：既然 active object 不像传统线程那样“永久占有自己的上下文并阻塞等待”，那它还能不能像 RTOS thread 一样满足 RMS/RMA 要求？

这个问题之所以重要，是因为很多人会误以为：

1. Active Object 的 RTC 语义意味着它一旦开始处理事件，就会独占 CPU 直到处理完成。
2. 因而它可能不适合 hard real-time。

这一课的目的，就是要拆穿这个误解。

## 第一步：把两个线程改造成两个 Active Object

课程接着按照 lesson 34 的方式，把 `blinky-1` 和 `blinky-2` 分别改写成 `Blinky1` 和 `Blinky2` 两个 active object。

### Blinky1 AO

`Blinky1`：

1. 继承 `QActive`。
2. 内部需要一个 `QTimeEvt`。
3. 初始 transition 中启动周期 time event，每 2 个 tick 超时一次。
4. 只有一个简单状态 `active`，在 `TIMEOUT_SIG` 上执行原先线程体中的工作负载。

这实际上就是把原来“线程 + 阻塞 delay”改写成了“active object + 周期 time event”。

### Blinky2 AO

`Blinky2`：

1. 同样继承 `QActive`。
2. 不需要 time event。
3. 只在 `BUTTON_PRESS_SIG` 上执行原先线程体中的工作负载。

也就是说，它把原来阻塞等待 semaphore 的模式，改成了等待队列中的按钮事件。

## Active Object 不再需要每个对象独立的私有栈

在迁移过程中，Samek 特别强调了一点：与传统 extended thread 不同，这里的 active object 在 QXK 内核下被当作 basic thread 来执行。

basic thread 的特点是：

1. 事件处理是 run-to-completion activation。
2. 本身不会在处理中途 block。
3. 因此不需要长期保有一个独立栈来等待恢复上下文。

这意味着多个 active object 可以共用同一个执行栈，而不必像传统线程那样每个线程单独分配一个大栈。

Samek 认为这是一项非常实际的收益：

1. event queue 往往比 thread stack 小得多。
2. 把多个线程改写成 active object 后，通常可以显著节省 RAM。

这也是 active object 在嵌入式环境中特别有吸引力的地方之一。

## 把 semaphore 改成 post event

迁移完成后，原来在 BSP 中用于唤醒 `blinky-2` 的 semaphore 也被替换成了事件投递：

1. BSP 定义 `BUTTON_PRESS_SIG`。
2. 按钮按下时，post 一个 immutable button press event 到 `Blinky2`。

这一步延续了前面多课反复强调的思想：从 blocking synchronization 转到 asynchronous event communication。

## 运行结果：Active Object 行为与原线程版等价

完成修改后，Samek 再次用 logic analyzer 观察运行结果。结论非常清楚：

1. `Blinky1` 的周期行为与原先线程版本一致。
2. `Blinky2` 在按钮按下时响应，行为与原先一致。
3. 高优先级的 `Blinky1` 仍然会抢占 `Blinky2`。
4. 因而 `Blinky1` 依然稳定满足其 2ms deadline。

换句话说，从实时调度效果看，active object 版本和传统线程版本在这个例子上没有本质差别。

## 本课最重要的结论：RTC 并不意味着独占 CPU 到事件处理结束

Samek 在这里特别强调一个常见误解。

run-to-completion 的含义是：

1. 一个 active object 在处理某个事件的过程中，
2. 不会在逻辑上开始处理第二个事件，
3. 必须先把当前事件完整处理完。

但这并不意味着它在物理时间上不能被更高优先级任务抢占。

在 preemptive kernel 下：

1. RTC step 仍然可以被更高优先级 active object 打断。
2. 被打断之后，再回来继续完成这个 RTC step。
3. 只要在逻辑上没有开始处理下一事件，RTC 语义就没有被破坏。

因此：

1. RTC 是事件处理语义。
2. preemption 是调度机制。
3. 这两者并不冲突。

这是本课最关键的认识。

## Active Object 与 RMS/RMA 完全兼容

基于上面的观察，Samek 得出结论：

1. Active Object 加 state machine
2. 再配合 preemptive、priority-based RTOS kernel

完全可以像传统线程一样适用于 Rate Monotonic Scheduling / Analysis。

只要：

1. 任务优先级分配合理，
2. 每个 active object 的执行时间可分析，
3. 调度器本身支持抢占，

那么 AO 系统同样可以满足 hard real-time requirements。

## Active Object 甚至可能比传统线程更适合硬实时

Samek 在课程结尾还多推进了一步：active object 不只是“也能做硬实时”，很多时候它实际上更适合硬实时。

原因并不是本课例子本身，而是 active object 更倾向于：

1. 不共享资源。
2. 用异步事件而非阻塞同步来交互。
3. 因此避免很多 shared-state concurrency 带来的时序问题。

不过他也承认，这一点需要更完整的例子来证明，所以把这个话题留到了下一课，也就是 lesson 44。

## 总结

43 课的核心，是证明 active object 并不会因为 RTC 语义而失去 hard real-time 能力。

这一课最重要的结论可以概括为：

1. 传统线程示例中的 `blinky-1` 和 `blinky-2` 可以较直接地改写成两个 active object，分别由 time event 和 button event 驱动。
2. 在 QXK 这样的 preemptive priority-based 内核下，active object 被作为 basic thread 运行，因此通常不需要像传统 extended thread 那样为每个对象单独保留私有栈。
3. Active Object 的 run-to-completion 语义只约束“一个对象一次只处理一个事件”，并不意味着该对象不能被更高优先级任务抢占。
4. 因此，在抢占式内核中，active object 的 RTC steps 完全可以在时间上相互交错，而高优先级 AO 依然能像高优先级线程那样满足 deadline。
5. 从 RMS/RMA 的角度看，active object 与传统线程一样可以用于 provably hard real-time scheduling。
6. Active Object 还有进一步优势：由于它天然倾向于减少共享和 blocking，它在实时性分析上往往比传统 shared-state threads 更干净。

如果说前面的课程主要从并发封装和事件驱动角度介绍 active object，那么第 43 课则第一次明确回答了“它是否适合硬实时”这个工程问题。Samek 给出的答案是肯定的，而且这只是开端。下一课会进一步说明：一旦涉及 active object 之间的交互，为什么“不要共享资源、改用事件传参”不仅是架构偏好，更是对实时性的直接保护。