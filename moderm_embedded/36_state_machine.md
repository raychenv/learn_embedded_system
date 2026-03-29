# 36 状态机之 Guard Condition

这一课，Miro Samek 继续讲 state machines，主题是 guard conditions。第 35 课已经说明了 state machine 为什么适合事件驱动系统，以及它如何替代杂乱的 flags 来保存 relevant history。但上一课的状态机仍然有一个隐含前提：状态和状态转移的整体结构，在设计时就已经完全确定。

现实系统往往没有这么“静态”。很多行为不仅依赖当前状态和事件类型，还依赖运行时才知道的数据，例如用户输入、计数值或某个条件表达式的结果。也就是说，某次事件到来时，“接下来该转到哪个状态”有时在设计阶段并不能完全写死。这正是 guard condition 要解决的问题。

这节课的重点是：在不破坏 state machine 结构清晰性的前提下，如何引入有限的运行时决策能力；同时也强调 guard 虽然有用，但必须谨慎使用，否则状态机会重新退化成充满 if/else 的 spaghetti code。

lesson 36 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-36

## 上一课的状态机为什么还不够灵活

课程先回顾第 35 课的 `blinky button` 状态机。在那个设计里：

1. `TIMEOUT` 在 `Off` 状态会让绿灯亮起，并转到 `On`。
2. `TIMEOUT` 在 `On` 状态会让绿灯熄灭，并转回 `Off`。

这很好地展示了状态机的基本价值：同一个事件 signal，会因为当前状态不同而触发不同反应。

但 Samek 随后指出，这种机制仍然不够灵活。因为 state machine 的状态结构和状态转移网络是在设计时固定下来的。如果某个事件到来后“究竟去哪个状态”取决于运行时才能知道的条件，那么单靠静态画好的状态和普通转移箭头就不够了。

换句话说：

1. state machine 负责表达设计时已知的行为结构。
2. 但有些决策要到 runtime 才能做出。

这时就需要在状态机中引入一种“受控的运行时分支机制”。

## Choice pseudo state 与 guard condition 的基本概念

UML 为此提供了两个相关机制：

1. choice pseudo state
2. guard condition

### Choice pseudo state 是状态图里的决策点

choice pseudo state 通常画成一个菱形，看起来很像流程图里的 decision point。某条 transition 可以先进入这个菱形节点，然后再从它分出多条后续路径。

这些后续路径并不是随便选，而是由 guard condition 来决定。

### Guard condition 是运行时求值的布尔表达式

每条从 choice 节点出去的路径上，都可以标一个方括号括起来的 guard，比如：

1. `[count > 0]`
2. `[error_detected]`
3. `[else]`

guard 本质上就是一个在 runtime 求值的布尔表达式：

1. 如果 guard 为真，对应路径可走。
2. 如果 guard 为假，这条路径就被禁用。

其中 `[else]` 是补充 guard，只有当前面所有其他 guard 都为假时才成立。

因此，choice + guards 的整体作用，就是把“流程图式的条件分支”安全地嵌入状态机转移之中。

## 一个更合适的例子：Time Bomb 控制器

为了展示 guard condition 的实际用途，课程重新构建了一个新的示例：把 TivaC LaunchPad 变成一个“定时炸弹控制器”。

行为要求如下：

1. 上电或复位后，绿灯亮起，系统等待按钮按下。
2. 按下 `SW1` 后，绿灯熄灭，系统开始倒计时。
3. 红灯闪烁三次。
4. 三次闪烁结束后，三色灯全亮，表示“爆炸”。

从行为序列上看，就是：

1. reset 后绿灯亮。
2. button press 后进入 `blink -> pause -> blink -> pause -> blink -> pause -> boom`。

这个例子之所以好，是因为它正好包含一个典型问题：从 `pause` 状态超时之后，到底应该回到 `blink`，还是进入 `boom`，取决于“还剩多少次闪烁”，而这个条件不适合简单地为每次闪烁都展开一组全新状态。

## 没有 guard 时的笨办法：硬编码重复状态

Samek 先指出，如果不用 guard，最粗暴的办法就是把状态图手工展开：

1. `blink1 -> pause1`
2. `blink2 -> pause2`
3. `blink3 -> pause3`
4. 然后再到 `boom`

这种做法当然能工作，但问题非常明显：

1. 状态数量被无意义地复制。
2. 这些状态在行为上高度重复。
3. 一旦闪烁次数从 3 改成 10，状态图会立刻膨胀。
4. 如果闪烁次数需要由用户运行时设定，那么设计时甚至根本不知道应该展开多少组状态。

这说明：某些行为差异不是“值得单独建模成不同状态”的设计时差异，而更像是运行时的条件判断。

## 更合理的设计：用计数器配合 choice 和 guard

课程给出的改进方法是：

1. 保持 `blink` 和 `pause` 这两个状态不变。
2. 增加一个私有变量 `blink_counter`。
3. 每次经过 `pause` 状态时，先把计数器减一。
4. 然后通过 choice pseudo state 根据 `blink_counter` 的值决定下一步去哪里。

具体来说：

1. 如果 `blink_counter > 0`，说明还需要继续闪烁，于是回到 `blink`。
2. 否则走 `[else]` 分支，进入 `boom`。

这样一来：

1. 状态结构保持简洁。
2. 重复行为不会被复制成大量状态。
3. 闪烁次数可以通过运行时变量灵活控制。

这正体现了 guard 的价值：它不是为了取代状态，而是为了在少量必要的位置给状态机增加运行时弹性。

## Time Bomb 状态机的结构

课程最终构造出的状态机大致包含四个状态：

1. `WaitForButton`
2. `Blink`
3. `Pause`
4. `Boom`

### 初始状态：WaitForButton

initial transition 会执行以下动作：

1. 打开绿灯。
2. 进入 `WaitForButton`。

在这个状态下，系统只关心一个事件：按钮按下。

收到 `BUTTON_PRESSED` 后：

1. 关闭绿灯。
2. 打开红灯。
3. 启动一个半秒的 time event。
4. 初始化 `blink_counter`。
5. 转移到 `Blink`。

### Blink 状态

在 `Blink` 状态中，只关心 `TIMEOUT` 事件。

收到超时后：

1. 关闭红灯。
2. 再启动一个半秒 timeout。
3. 转移到 `Pause`。

### Pause 状态与 choice 节点

`Pause` 状态同样只关心 `TIMEOUT` 事件。

收到超时后：

1. 先将 `blink_counter` 递减。
2. 进入一个 choice pseudo state。

然后：

1. 若 `[blink_counter > 0]` 为真，则打开红灯、重新启动 timeout，并回到 `Blink`。
2. 若 `[else]` 为真，则点亮全部 LED，进入 `Boom`。

### Boom 状态

`Boom` 是终止性状态，不再处理其他事件，至少在这一课的设计里是如此。

## 在 C 代码中，choice 和 guard 最终就是 if/else

完成状态图设计后，课程继续沿用第 35 课的最直接实现方式：

1. 用枚举表示状态。
2. 用 `dispatch()` 中的双层 `switch` 实现状态机。

新的状态枚举包括：

1. `WAIT_FOR_BUTTON_STATE`
2. `BLINK_STATE`
3. `PAUSE_STATE`
4. `BOOM_STATE`

私有数据则包含：

1. time event
2. `blink_counter`

这里最值得注意的，不是普通状态和事件分支，而是 choice pseudo state 的实现方式。Samek 明确指出：在 C 代码里，它最终就会变成 `if/else`。

例如在 `PAUSE_STATE` 处理 `TIMEOUT` 时：

1. 先执行 `blink_counter--`。
2. 然后 `if (blink_counter > 0)`，执行回到 `Blink` 的那段动作并改状态。
3. `else`，执行“爆炸”的动作并改状态到 `Boom`。

因此，guard 本质上并不神秘，它只是把必要的运行时条件判断显式纳入状态机设计。

## 但这也意味着：guard 用多了，状态机会重新退化成 if/else 程序

Samek 在这里强调了全课最重要的设计提醒：虽然 guard 很有用，但它们必须用得非常克制。

原因很直接。state machine 原本存在的一个重要价值，就是把行为结构从散乱的条件分支中抽出来，用状态和转移来组织。如果你在每个地方都塞大量 guard，那么：

1. 代码里还是会充满 if/else。
2. 设计复杂度会重新转移到条件判断里。
3. 状态图会失去“状态驱动行为”的清晰性。
4. 最终又会走回第 35 课批评过的 spaghetti code。

也就是说，guard 不是状态机的主角，而只是补充手段。

## 如何把握 state 与 guard 的边界

Samek 认为，成为一个成熟的 state machine 设计者，关键能力之一就是判断：

1. 哪些差异应该在设计时建模为状态和转移。
2. 哪些差异确实需要作为运行时 guard 来处理。

一个很实用的原则是：尽量少用 guard。

更具体地说：

1. 如果某个条件长期影响系统未来行为，往往更适合建模成状态。
2. 如果某个条件只是某个局部决策点上的临时运行时判定，可以考虑用 guard。
3. 如果你发现状态图越来越像“每条边都是复杂条件表达式”，那通常说明设计可能正在退化。

这条原则和第 35 课的思想是一致的：state machine 的目标是把 relevant history 和行为结构组织清楚，而不是换个图形化外壳继续堆 if/else。

## 运行结果验证了 guard 带来的灵活性

课程最后在板子上验证 `Time Bomb` 示例：

1. reset 后绿灯亮起。
2. 按键后红灯开始按节奏闪烁。
3. 倒计数结束后进入 `Boom`。

然后 Samek 把 `blink_counter` 从 3 改成 10，再重新编译运行。状态机本身的结构没有变化，但行为立刻变成了“闪 10 次再爆炸”。

这恰好说明 guard + runtime variable 的价值：状态机框架保持不变，只通过运行时条件就能表达一类可调行为。

## 总结

36 课的核心，是在第 35 课状态机基础上引入“有限的运行时灵活性”。

这一课最重要的结论可以概括为：

1. 普通 state machine 的状态和转移结构通常在设计时固定下来，但某些行为分支要到 runtime 才能决定。
2. UML 用 choice pseudo state 和 guard condition 来表达这种运行时分支。
3. guard 是运行时求值的布尔条件，只有 guard 为真时，对应转移路径才可用；`[else]` 表示补充分支。
4. 在设计上，guard 适合处理“局部的、必要的运行时决策”，例如计数值是否大于零。
5. 在实现上，choice 和 guards 最终通常会落成普通的 `if/else`。
6. 正因为如此，guard 必须谨慎使用；一旦滥用，状态机就会退化回充满条件判断的 spaghetti code。
7. 好的状态机设计需要在“状态建模”和“guard 判断”之间取得平衡，并尽量让大部分行为仍由状态结构本身表达。

如果说第 35 课解决的是“如何用 state 替代零散 flags”，那么第 36 课解决的就是“当设计时无法完全写死后续转移时，如何在 state machine 中安全地引入运行时分支”。这一步很重要，因为它为后续更复杂的状态机变体打下了基础，也让状态机从纯静态模型走向更接近真实系统需求的表达能力。
