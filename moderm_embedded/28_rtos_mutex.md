# 28 RTOS 互斥机制

28课，Miro Samek 讲解 RTOS 中线程共享资源时会遇到的问题，以及 RTOS 常见的互斥机制。前一课已经把 blinky 线程移植到 QP/C 框架中的 QXK 内核，并引入了信号量做线程同步；这一课继续往前走，讨论线程之间真正开始共享变量、函数和外设之后，为什么会出现竞争、冲突、优先级反转，以及该如何选择合适的保护手段。

lesson 28 工程参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/blob/main/lesson-28/tm4c123-qxk-keil/bsp.c

## 为什么需要互斥机制

在 RTOS 中，并发线程通常运行在同一个地址空间里。和桌面操作系统里彼此隔离的进程不同，RTOS 线程可以直接访问相同的全局变量、函数以及内存映射外设寄存器。编译器并不知道线程切换在底层是如何发生的，因此从 C 语言角度看，各个线程函数和普通函数没有本质区别，它们都能访问自己“看得见”的资源。

这带来一个直接后果：**只要资源被多个线程共享，就有可能发生冲突**。这可以类比十字路口上的车辆交汇。只要路径相交，就必须有规则，否则就会出事故。软件里的这些规则，统称为 mutual exclusion mechanisms，目标就是保证共享资源在任意时刻只被一个执行单元使用。

![十字路口](images/28_cross_road.png)

## 从 GPIO 竞争重新理解 race condition

这一课首先复用了第 20 课里讲过的 GPIO 竞争案例。之前 LED 的 `on/off` 操作分别访问不同寄存器，而 `toggle` 操作会对同一个 `GPIOF_AHB->DATA` 寄存器做“读-改-写”，这就让多个线程之间共享了同一个寄存器。

为了更容易观察问题，课程把 blinky1 和 blinky2 中原本的 `ledOn/ledOff` 改成 `ledToggle`。这样一来：

1. blinky1 会切换绿色 LED。
2. blinky2 会切换蓝色 LED。
3. 两个线程实际都在读写同一个 `GPIOF_AHB->DATA` 寄存器。

逻辑分析仪显示：当高优先级 blinky1 运行时，绿色 LED 本应翻转，但在某些时刻它会“看起来没有变化”。继续放大时序后会发现，问题不是 blinky1 没有执行，而是 blinky2 在被抢占后恢复执行时，把自己先前读到的旧寄存器值写了回去，结果把 blinky1 刚刚做的修改也覆盖了。

这说明：

1. race condition 不只会发生在主程序和中断之间。
2. 在抢占式 RTOS 中，任意两个并发线程之间同样会发生 race condition。
3. 只要存在“读-改-写”共享资源，线程切换就可能把状态搞乱。

## 第一种办法：临界区 critical section

最直接的互斥手段，是在访问共享资源前后临时关闭中断。这样 CPU 就不会被外界打断，自然也不会发生抢占。但简单粗暴地“关所有中断”在 RTOS 里有两个明显问题：

1. 它不符合 RTOS 的中断策略。QXK 采用的是“zero-latency”思想，只屏蔽特定优先级以下的中断；而无差别关中断会额外增加本不该受影响的中断延迟。
2. 它通常不支持嵌套。如果某个函数内部已经建立临界区，而外层又再次进入临界区，就可能因为提前开中断而破坏保护。

为了解决这些问题，QXK 提供了自己的临界区机制：

1. 先定义一个自动变量保存中断状态。
2. 用 `QF_CRIT_ENTRY()` 保存当前中断状态并进入临界区。
3. 用 `QF_CRIT_EXIT()` 恢复原来的中断状态。

这种实现可以嵌套，也和 QXK 的中断屏蔽策略保持一致。把 LED toggle 代码放进这个临界区以后，前面 `GPIOF_AHB->DATA` 寄存器的竞争现象就消失了。

但临界区虽然强力，也有明确边界：**它只适合非常短的代码段**。如果受保护代码持续几十、几百微秒，甚至更长，那么中断延迟就会明显变差，系统实时性会受到影响。

## 长时间共享资源：Morse code 示例

为了说明“临界区不适合保护长操作”，Samek 设计了一个摩尔斯码发送例子。共享资源还是绿色 LED，但这次不是简单翻转，而是通过 `BSP_sendMorseCode()` 按位输出消息：

1. 高优先级 blinky1 周期性发送紧急消息 `SOS`。
2. 低优先级 blinky3 在系统空闲时发送 `TEST`。
3. 发送摩尔斯码时需要精确的微秒级 busy-wait，因此整个发送过程会持续接近 1ms。

如果不做保护，两个线程输出到同一个绿色 LED 上，逻辑分析仪会看到 `SOS` 和 `TEST` 相互打断、相互污染，最后拼出完全错误的内容，紧急消息甚至会错过自己的截止时间。

这时就进入了本课的核心：**访问共享资源的时间已经太长，不能再靠临界区硬挡住一切。**

## 第二种办法：二值信号量做互斥

课程先用一个 binary semaphore 来保护 `BSP_sendMorseCode()`：

1. 定义 `Morse_sema`。
2. 初始化计数为 1，表示资源一开始可用。
3. 进入共享资源前 `wait()`。
4. 使用完资源后 `signal()`。

这样做之后，`SOS` 和 `TEST` 不再在时间上互相碰撞，消息内容看起来是完整的。但新的问题很快出现了：

1. blinky3 先拿到信号量，开始发送 `TEST`。
2. 此时按下按钮，blinky2 被唤醒并由于优先级高于 blinky3 而抢占它。
3. 下一次 tick 到来后，更高优先级的 blinky1 被唤醒，准备发送 `SOS`。
4. 但 blinky1 发现信号量还被 blinky3 占着，于是自己阻塞。
5. 结果 blinky2 继续长时间运行，最高优先级的 blinky1 反而被拖住，最终错过硬实时截止时间。

这就是 unbounded priority inversion，即“无界优先级反转”。

它的本质是：**经典 semaphore 不理解线程优先级，只知道资源是否可用，因此并不适合在优先级调度系统里承担互斥职责。**

所以这一课给出的结论很明确：

1. semaphore 很适合做同步。
2. semaphore 不适合直接拿来解决优先级系统中的互斥问题。

## 第三种办法：选择性 scheduler lock

接下来 课程引入 QXK 的 selective scheduler locking。它的思路不是“让线程阻塞在资源前”，而是“在访问共享资源期间，暂时禁止某一优先级以下的线程参与调度”。

它的使用方式是：

1. 进入共享代码前调用 `QXK_schedLock(ceiling)`。
2. 退出共享代码后调用 `QXK_schedUnlock()`。
3. `ceiling` 必须至少等于访问该资源的最高优先级线程。

例如本例中访问共享 LED 的最高优先级线程是 blinky1，优先级为 5，因此 ceiling 至少设为 5。

这种机制的关键点：

1. 它是 non-blocking 的互斥机制。
2. 它不会关闭中断，因此不会影响 ISR 延迟。
3. 它只阻止优先级小于等于 ceiling 的线程被调度，更高优先级线程和中断依然照常运行。
4. 它也支持嵌套，因为 API 会返回先前的调度器锁状态。

在逻辑分析仪中可以看到：当 blinky3 正在发送 `TEST` 且持有 scheduler lock 时，即使按钮触发了 blinky2，blinky2 也不会立刻运行；直到 blinky3 结束临界共享操作后，调度器才重新选择最高优先级就绪线程。这样就避免了前面的无界优先级反转。

不过 scheduler lock 也有自己的限制：**持锁期间线程不能阻塞**。如果在锁保护的代码里调用延时、等待信号量等阻塞 API，QXK 会直接断言失败。也就是说，它适合“执行时间较长，但内部不会阻塞”的共享资源访问。

## 第四种办法：mutex

最后一类机制是 mutex。Samek 强调，不要把 mutex 仅仅理解为“另一种 semaphore”，而应该把它看成 RTOS 专门为共享资源保护设计的对象。

QXK 提供的是 priority-ceiling mutex，使用方式如下：

1. 定义一个 `QXMutex` 对象。
2. 初始化时指定 ceiling priority。
3. 访问资源前调用 `QXMutex_lock()`。
4. 访问完成后调用 `QXMutex_unlock()`。

和 scheduler lock 类似，这个 ceiling 必须至少高于所有会访问该资源的线程优先级。但在 QXK 中，mutex 的 ceiling 必须使用一个**未被线程占用的独立优先级**，因此本例中设为比 blinky1 更高一级的 6。

priority-ceiling mutex 的核心效果是：

1. 低优先级线程一旦拿到 mutex，会立即被提升到 ceiling priority。
2. 在它释放 mutex 之前，中间优先级线程不能把它抢走。
3. 因而高优先级线程不会再被“间接卡死”在低优先级线程后面。

从本课这个不发生阻塞的示例来看，scheduler lock 和 priority-ceiling mutex 的外部执行效果几乎一样；但 mutex 的适用范围更一般，因为它是为“线程可能在共享资源访问期间发生阻塞”的场景准备的。

## Priority Ceiling 与 Priority Inheritance

本课还顺带比较了两种经典 mutex 协议：

1. **Priority-Ceiling Protocol**：线程一拿到 mutex，就立刻提升到预设天花板优先级。
2. **Priority-Inheritance Protocol**：线程起初不升优先级，只有当更高优先级线程真的被它阻塞时，它才临时继承对方优先级。

Samek 认为 priority ceiling 更适合硬实时系统，原因包括：

1. 行为更直接，时序分析更简单。
2. 通常上下文切换更少。
3. RTOS 内部实现复杂度更低。

相比之下，priority inheritance 自动化更强，但往往引入更多上下文切换，也更难做严格时序分析。

## 总结

28课的主线非常清楚：

1. 只要线程共享状态，就一定会引入竞争条件和时间上的冲突。
2. 临界区可以解决极短代码段的互斥问题，但不能滥用。
3. semaphore 适合同步，不适合在优先级 RTOS 中直接承担互斥职责。
4. selective scheduler locking 适合“不能被中等优先级线程打断，但又不会阻塞”的共享操作。
5. mutex 是更通用、更现代的共享资源保护机制，用于处理优先级反转等问题。

传统 RTOS 的 shared-state concurrency 模型会天然带来 race condition、collision、priority inversion、deadlock 等一系列连锁问题。互斥机制虽然能缓解这些问题，但也会引入新的复杂性和实时性代价。因此，后续课程会继续讨论那些尽量避免共享状态的更现代软件架构。