# 37 其他类型的状态机：从硬件起源到 Input-Driven State Machine

这一课，Miro Samek 继续讲 state machines，不过重点从前两课的 event-driven state machine 扩展到了“软件里其他常见的状态机类型”。Samek 这一课做了两件很重要的事：

1. 从历史上把状态机的来源追溯回硬件设计，解释 Mealy 和 Moore 状态机为什么会是今天很多软件状态机概念的源头。
2. 把 event-driven state machine 与另一大类软件状态机作对比，也就是 input-driven、polled 或 periodic state machine，并分析它们各自的特点与风险。

这节课的价值在于，它让你看到“状态机”并不是单一技术，而是一整个谱系。前面几课中 Samek 明显偏向事件驱动状态机，但这一课并没有简单地否定其他变体，而是更细致地讨论：不同状态机形式适合怎样的运行环境，以及为什么环境本身会影响状态机的可靠性和鲁棒性。

lesson 37 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-37

## 状态机最早来自硬件设计，而不是软件工程

课程一开始先回到历史。Samek 提醒，状态机并不是先在软件领域兴起的，而是源于 1950 年代的数字电路设计，尤其是 George Mealy 和 Edward Moore 在 Bell Labs 的经典论文。

这一点非常关键，因为它解释了为什么今天很多“软件状态机”概念看起来总带着一点硬件味道。很多术语、图示和思维方式，其实最早都是用来描述带内部状态的数字电路，而不是描述现代软件对象的行为。

要理解这一点，Samek 先区分了两类电路：

1. combinational circuit
2. circuit with internal state

## 组合逻辑没有状态，因此不需要记忆历史

像 `AND`、`OR`、`XOR` 这样的数字电路，都属于 combinational logic。它们的输出完全由当前输入决定，不依赖过去发生过什么。

比如一个 half-adder：

1. 输入是两个比特 `A` 和 `B`。
2. 输出是和与进位。
3. 对于每一种输入组合，输出都唯一确定。

这种系统可以完全由 truth table 描述，因为系统不需要记住过去历史。换句话说，没有内部 memory，就没有 state machine 的必要。

## 一旦系统要记住“之前发生过什么”，状态机就出现了

Samek 接着举了一个更能体现“状态”必要性的例子：检测一个输入信号的 rising edge。

如果要求输出 `Y` 只在输入 `A` 从 0 变成 1 的那个瞬间为高，那么系统就不能只看当前输入值，还必须知道“前一个时刻 `A` 是多少”。

这就意味着：

1. 系统必须保存过去的一点历史。
2. 必须有某种 memory element，例如 flip-flop。
3. 也就引入了 state。

从这里开始，就进入了真正的 state machine 世界。

## 硬件状态机通常是同步的，而这深刻影响了状态机形式

在 Samek 给出的电路例子中，状态是由 D flip-flop 保存的，而 flip-flop 又受时钟驱动。这个时钟决定系统什么时候采样输入、什么时候更新状态。

这种电路叫 synchronous circuit，也就是同步电路。

其关键特征是：

1. 输入与状态的变化被统一绑定到时钟节拍。
2. 系统在离散时刻更新，而不是任意时刻随便变化。
3. 这样可以显著降低硬件 race condition 风险。

Samek 顺带提到，asynchronous circuit 当然也可能存在，但它们更容易遭遇由于传播延迟引起的硬件竞态，所以现实中的数字系统几乎都采用同步设计。

这一点对后面理解软件状态机很重要，因为很多“传统状态机”的形式，其实都隐含继承了这种“由统一时钟驱动”的硬件世界观。

## Mealy 状态机：输出依赖当前状态和当前输入

课程接着介绍了 Mealy state machine。

在 Mealy 机中：

1. 当前输入参与决定 next state。
2. 当前输入也直接参与决定输出。

这意味着输出可能对输入变化做出更快反应，因为不必等状态真正更新后再输出。

在状态图上，Mealy 机通常表现为：

1. 状态画成圆圈。
2. transition 上同时标注输入和输出。

也就是说，输出被看成“转移的一部分”。

Samek 用 rising edge detection 电路说明了这种形式的优势和代价：它可以更快地对输入变化作出响应，但也因为输出直接依赖输入，更容易受到同步边界附近竞争关系的影响。

## Moore 状态机：输出只依赖当前状态

与 Mealy 相对的是 Moore state machine。

在 Moore 机中：

1. 输入只参与决定 next state。
2. 输出只由 current state 决定。

这意味着：

1. 输出变化通常要慢一个节拍。
2. 但输出更稳定，因为它不直接受输入瞬时变化影响。

在状态图上，Moore 机通常表现为：

1. transition 上只写输入。
2. 输出写在状态内部。

Samek 特别指出，Moore 机往往需要更多状态，才能完成与 Mealy 机相同的功能。但它的分离性更好，在硬件中也常常更稳健。

## 从硬件视角看，Mealy/Moore 的差别本质是“输出到底依赖谁”

课程通过 rising edge detector 的两个电路版本说明了最核心差别：

1. Mealy：输出路径上有输入的直接影响。
2. Moore：输出路径上只有状态，没有输入的直接影响。

这种区分后来被大量照搬到软件状态机中。哪怕今天的软件状态机不会真的画出 flip-flop 和组合逻辑块，但很多图和术语其实仍然在沿用这套思想。

## 很多软件里的“状态机”其实不是 event-driven，而是 input-driven

讲完历史背景后，Samek 开始提出本课真正要澄清的问题：如果你去网上搜“software state machine”，经常会看到很多图，其 transition 上写的不是事件 signal，而是变量条件、逻辑表达式、比较式，比如：

1. `x > 10`
2. `button == pressed`
3. `true`
4. `always`

这些看上去当然也像状态机，但它们和前几课讲的 event-driven state machine 明显不同。区别不在于图是圆圈还是圆角矩形，而在于：这些状态机并不是被离散事件驱动的，而是被 inputs 驱动的。

这里的 input，在软件里通常就是一些会变化的变量：

1. 按钮电平。
2. 某个计数器值。
3. 外设寄存器内容。
4. 中断更新的时间戳。

这类状态机的 transition 上写的其实就是 guard condition，而这些 guard 直接建立在输入变量之上。

## Input-Driven State Machine 的运行方式：不断轮询输入

Samek 接着提出一个关键问题：既然 event-driven state machine 只在新事件到来时运行，那么 input-driven state machine 是什么时候运行的？

答案是：它们通常是“总在运行”或“周期性运行”的。

常见方式包括：

1. 在 `while(1)` superloop 中一遍遍执行。
2. 由某个周期调度器定期调用。
3. 在控制系统或仿真环境中按固定 frame rate 执行。

因此这类状态机常被称为：

1. input-driven state machine
2. periodic state machine
3. polled state machine
4. controller state machine

Samek 自己倾向于用 input-driven 或 polled 这类称呼，因为它们更准确地反映了其本质：状态机本身必须去 poll 输入，并在 guard condition 中“发现”真正有意义的事件。

## 这类状态机把“事件发现”与“状态机逻辑”耦合在一起

这是 input-driven 状态机与 event-driven 状态机最本质的区别之一。

在 event-driven 模式中：

1. interesting event 通常已经在状态机之外被识别出来。
2. 状态机拿到的是一个稳定的 event object。
3. 然后只负责根据当前状态处理这个事件。

而在 input-driven 模式中：

1. 状态机每次运行时，要自己去读输入变量。
2. 再通过 guard expressions 判断“有没有发生值得响应的事”。
3. 也就是说，event discovery 和 behavior logic 混在一起。

Samek 把这些原始输入称为 proto-events，这个说法很贴切：它们还不是被整理好的事件，只是状态机需要不断筛选的原材料。

## Input-Driven 状态机的主要风险：输入是异步变化的共享变量

课程接着指出了 input-driven 状态机最棘手的问题。它们所依赖的输入变量，通常来自状态机外部，比如：

1. GPIO 输入寄存器。
2. 中断中更新的计数器。
3. 某些全局共享变量。

这些输入会相对于状态机执行过程异步变化。于是它们本质上就是 shared data。

这就带来了和前面 RTOS、shared-state concurrency 类似的问题：

1. 输入可能在状态机运行过程中变化。
2. 同一个 guard 判断中的多个读取可能彼此不一致。
3. 某些边沿或瞬时条件可能被漏掉。
4. 程序行为会依赖时序细节，而不再完全可预测。

Samek 因此把它们类比为硬件里的 asynchronous circuit，而把 event-driven state machine 类比为 synchronous circuit。这个类比不是严格等价，但它很好地抓住了核心：前者直接暴露于异步变化的输入，后者则通过事件队列获得更稳定的处理边界。

## 一个典型问题：guard 判断与保存前一值之间，输入可能已经变了

课程举了一个飞机起落架控制状态机的例子。某个 guard 的意图是检测一个 lever 输入的上升沿，因此它要同时检查：

1. 当前 `gearLever` 值。
2. 之前保存的 `prevGearLever` 值。

问题在于，如果 `gearLever` 在 interrupt 中异步变化，那么可能发生这种情况：

1. guard 条件刚读到它还是旧值。
2. 还没来得及把新值保存到 `prevGearLever`，它又变了。
3. 结果某个边沿被错过。

Samek 用一句很戏剧化的话形容：如果这个输入边沿丢了，飞机可能就永远起飞不了。

这个例子说明，input-driven 状态机的正确性，不只取决于状态图画得对不对，还非常依赖输入采样方式是否可靠。

## 最直接的补救方法：先缓冲输入，再在一次 RTC 步骤中只读快照

针对上述问题，课程给出的第一个改进措施是 buffering inputs。

做法是：

1. 在进入状态机本轮执行前，先把所有关心的外部输入复制到局部变量。
2. 状态机在整个 Run-to-Completion 步骤中，只使用这些局部副本。

这样做的好处是：

1. 同一轮处理里，输入不会前后不一致。
2. guard 判断基于一致快照。
3. 至少不会因为同一次处理里多次读取异步输入而产生自相矛盾。

这虽然不能解决所有问题，但已经比“在状态机内部随时直接读外设或共享变量”稳健得多。

## 用 Lesson 21 的 superloop blinky 作为 input-driven 状态机例子

为了把这些概念落到代码上，Samek 回到第 21 课的 foreground/background blinky。那时他在 `while(1)` superloop 中写了一个 non-blocking blinky，它本质上正是一个 input-driven state machine。

其特征很典型：

1. 代码中有 `switch(state)`。
2. 每个状态分支里又有若干 `if`。
3. 这些 `if` 正是 guard conditions。
4. 输入来自一个异步变化的 tick counter。

Samek 还指出，这个状态机在图论上属于 Mealy 型，因为动作都挂在转移上，而不是状态内部。

## 第一步改进：把 tick counter 先缓存到 `now`

原始代码里，在同一个状态里可能会多次调用 `BSP_tickCtr()`。由于该计数器在 SysTick ISR 中不断更新，所以这些调用返回值在同一次状态机执行中可能已经不同。

改进方式很简单：

1. 在状态机入口先读一次 tick counter。
2. 把它保存到局部变量 `now`。
3. 后面一轮 RTC 处理都只使用 `now`。

这就实现了最基本的输入快照。

Samek 特别提醒，这个问题与输入是“直接读寄存器”还是“通过函数返回”无关。只要底层数据是异步变化的，包装成函数并不会自动让它变得安全。

## 加入按钮输入后，buffering 仍然只是第一步

课程接着给这个 superloop 状态机加上 `SW1` 按钮输入，用来控制蓝灯亮灭。做法包括：

1. 新增一个 BSP 函数读取按钮。
2. 在状态机入口把当前按钮值缓冲到局部变量 `button`。
3. 用一个 `static` 变量保存前一次按钮值 `prevButton`。
4. 通过 guard condition 检测按钮下降沿和上升沿。

这样，状态机就能：

1. 按下按钮时亮蓝灯。
2. 松开按钮时灭蓝灯。

逻辑分析仪还能看到按钮抖动，但状态机有时能“跟上”，有时会漏掉某些 bounce。这再次说明：仅仅做了一层 buffering，并不能保证输入处理稳定可靠。

## 更深层的问题：输入采样频率与状态机执行速率耦合在一起

Samek 认为，这才是 input-driven 状态机真正麻烦的地方。因为状态机既负责“做业务逻辑”，又负责“采样和判断输入”，于是输入采样时机完全受状态机执行速度影响。

而状态机执行时间又可能变化很大：

1. 某些轮次 guard 都为假，很快就结束。
2. 某些轮次需要执行较多动作，会慢很多。

这样一来，输入采样频率就不稳定了。

如果输入变化速度比状态机最差采样速率还快，那么你就可能丢事件。按钮抖动只是最温和的例子，其他更快的输入会让问题更严重。

## 想进一步提高可靠性，就必须把 polling 与状态机逻辑解耦

当系统对输入处理更严格时，Samek 建议把输入 polling 从状态机本体中拆出来。这种解耦通常意味着：

1. 用固定频率去采样输入。
2. 在采样阶段做去抖、滤波或预处理。
3. 再把处理结果缓冲或排队。
4. 最后把真正的 event 交给状态机。

一旦你开始这样做，系统其实就已经越来越接近 event-driven architecture 了。

## 把 buffered inputs 包装成结构体，本质上就在向 event 靠近

Samek 做了一个非常有启发性的观察：如果你把状态机每轮读取到的输入快照包成一个结构体，并命名为 `evt`，那么你其实就已经在构造一个 event object。

这时：

1. 原来的输入变量变成了 event parameters。
2. 这些参数一起组成一个一致快照。
3. 状态机通过 `e->param` 访问它们。

虽然此时这个“事件”还可能只是 generic sample，而不是来自真正队列中的离散事件，但从结构上讲，它已经越来越像前几课中的 event-driven dispatch 了。

这恰好说明，两类状态机并不是完全断裂的，而更像一个连续谱：

1. 最左边是直接在 superloop 里轮询、几乎没有 buffering 的 input-driven 状态机。
2. 中间是加入输入快照、generic sample event、guard condition 的半事件化形式。
3. 最右边则是运行在 active object 中、拥有完整队列和稳定 RTC 边界的 fully event-driven state machine。

## 状态机的可靠性不仅取决于类型，也取决于运行环境

这是全课最关键的总结之一。Samek 明确指出：状态机是否健壮，不只取决于你说它是 Mealy、Moore、input-driven 还是 event-driven，更取决于它运行在什么样的环境里。

例如：

1. 同一个状态图，放在裸 `while(1)` 里不断轮询，可靠性可能一般。
2. 如果给它加上稳定采样、输入缓冲和适当队列，鲁棒性会明显提高。
3. 如果再放进 active object 和真正的事件队列中，它的行为边界会更清晰、更适合 RTC 假设。

因此，状态机设计不能只看“图画得好不好”，还必须同时考虑输入来源、采样方式、并发环境和事件交付机制。

## Input-Driven 状态机并不是“坏”，它们在很多场景仍然非常合适

虽然 Samek 一直在强调 event-driven 的优势，但这一课并没有把 input-driven 状态机一棍子打死。相反，他明确给出了几个很适合它们的场景。

### 1. 游戏

游戏逻辑通常按 frame 执行，比如每秒 30 或 60 帧。很多“有趣事件”本来就需要在程序内部根据当前位置、对象距离、碰撞检测等条件实时计算出来，不适合在外部先离散化成事件再交给状态机。

### 2. 机器人

机器人软件同样常常周期运行。来自传感器的大量原始输入，需要结合当前任务和环境持续分析，这种场景也很适合 input-driven 或 periodic state machine。

### 3. 作为 event-driven 系统前端的输入发现器

Samek 还举了一个非常重要的例子：按键去抖。去抖算法本身就很适合写成一个小型 input-driven Moore state machine，周期性运行在 tick interrupt 中，然后再把整理好的按键事件发给后面的 event-driven 系统。

这说明 input-driven 状态机并不一定要直接承担最终业务逻辑，它也非常适合作为“事件发现与清洗前端”。

## 总结

37 课的核心，是把前两课介绍的 event-driven state machine 放回更大的状态机谱系中重新理解。

这一课最重要的结论可以概括为：

1. 状态机最早源于数字硬件设计，因此 Mealy、Moore 等概念天然带有硬件背景。
2. Mealy 状态机的输出依赖状态和输入，反应更快，但耦合更紧；Moore 状态机的输出只依赖状态，更稳定但通常需要更多状态。
3. 软件中的很多状态机并不是 event-driven，而是 input-driven、polled 或 periodic 的，它们通过持续轮询输入并在 guard 中发现“真正事件”。
4. input-driven 状态机把事件发现和行为逻辑耦合在一起，因此更容易受到异步输入变化、共享变量和采样时序问题的影响。
5. 对 input-driven 状态机，最基本的改进是先对外部输入做 buffering，使一次 RTC 处理中使用一致快照。
6. 如果进一步把输入快照封装成结构体，并把 polling、滤波、缓冲与业务状态机分离，系统就会逐渐向 event-driven architecture 演进。
7. 状态机的可靠性不仅取决于类型，还取决于它所处的运行环境，包括采样方式、并发边界和事件交付机制。
8. input-driven 状态机并不是过时或错误的技术，它们在游戏、机器人、去抖等场景里仍然非常有价值。

如果说第 35 课讲的是“状态机是什么”，第 36 课讲的是“如何通过 guard 增加有限灵活性”，那么第 37 课讲的则是“状态机并不只有一种形态，真正重要的是把状态机放到合适的运行环境里”。这也为后面继续讨论状态机实现方式、去抖算法以及更高级事件驱动架构，打下了很好的基础。
