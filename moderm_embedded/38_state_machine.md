# 38 状态表与状态的 Entry/Exit Action

这一课，Miro Samek 继续讲 state machine implementation，这次的主题是 state tables。前几课里，状态机都是用最直接的 nested-switch 方式来实现：先按当前状态 `switch`，再在每个状态内部按事件 `switch` 或写 guard condition。Samek 一直强调，这种写法虽然最容易理解，但并不一定是最优雅或最可扩展的实现方式。

因此这一课要做两件事：

1. 引入另一种典型状态机实现策略，也就是基于 state table 的 data-driven 实现。
2. 在此基础上继续引入 entry/exit actions，把一些更适合归属于状态本身的动作，从 transition 上移到 state 上。

这节课的价值在于，它不只是又教一种写法，而是在帮助你建立“状态机实现策略”的视角：同一个 state diagram，可以映射成不同风格的代码结构；而不同实现方式，在可读性、性能、可维护性和扩展性上都有明显 trade-off。

lesson 38 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-38

## 从 nested-switch 到 state table：从 code-driven 转向 data-driven

课程一开始先回顾前两课的实现方式。

### 之前的实现：嵌套 switch

在 lesson 36 的 `TimeBomb` 里，整个状态机都写在一个 `dispatch()` 函数中：

1. 外层 `switch` 根据当前 `state` 分发。
2. 每个状态内部再按 `event signal` 分发。
3. guard condition 则出现在某些状态的事件处理分支里。

这种写法的优点很直接：

1. 逻辑集中。
2. 映射状态图比较直观。
3. 对小状态机最容易上手。

但它的缺点也明显：随着状态和事件增多，整个 `dispatch()` 会越来越膨胀，而且状态-事件组合的信息是“散落在代码分支里”的，不够结构化。

### state table 的思路：把状态机当作一个表

Samek 接着指出，其实你在 lesson 37 里已经见过类似的东西，只不过那时是在讲 Mealy/Moore 硬件状态机时看到的 truth table。

如果把 state machine 看成“当前状态 + 当前事件 -> 产生动作并进入下一状态”的映射，那么它天然就可以表示成表。

最开始你可以写成一维表：

1. 逐个状态列出它能处理的事件。
2. 每个表项写明 next-state 和 action。

但更自然的形式是二维表：

1. 一维是 states。
2. 一维是 events。
3. 每个 cell 表示“在某个状态收到某个事件时该做什么”。

这种表示方式和状态图是一一对应的，只是它把行为规则从分散代码中抽出来，放进了一个显式数据结构里。

## 用 TimeBomb 构造二维状态表

课程继续沿用 lesson 36 的 `TimeBomb` 例子来说明 state table。

### 先列出事件和状态

第一步是列出 `TimeBomb` 实际会处理哪些事件。根据 `bsp.h`，它的信号包括：

1. `BUTTON_PRESSED`
2. `BUTTON_RELEASED`
3. `TIMEOUT`

然后列出状态机中的状态：

1. `wait4button`
2. `blink`
3. `pause`
4. `boom`

### 填表时的基本规则

对每个状态，逐个检查它是否处理某个事件：

1. 如果处理，就把 next-state 和对应动作填进该单元格。
2. 如果不处理，可以填 `ignore`。
3. 如果理论上不该收到该事件，也可以填 `error`。

Samek 特别指出，`pause` 状态会有一个带 guard 的转移，因此对应的表项不是单一 next-state，而是“根据 guard 决定走向 blink 还是 boom”。这说明 guard 依旧存在，只是它在表中的表达方式可能不如在图里那样自然。

## 在 C 中如何表示 state table

讲完概念后，课程进入实现。第一件事就是决定：state table 在 C 里应该用什么形式表示。

### 用二维数组表示 state table

C 原生支持二维数组，因此最直接的做法就是定义一个二维数组，例如：

1. 第一维对应 state。
2. 第二维对应 signal。

为了让维度与枚举保持一致，Samek 建议在 state 枚举的最后放一个 `MAX_STATE`，在 signal 枚举的最后放一个 `MAX_SIG`。这样只要枚举有变化，数组尺寸会自动跟着更新。

这是个很实用的小技巧，因为它避免了手工同步状态数量和事件数量带来的错误。

## 表项里到底放什么：不要把表项设计得过重

这是本课最有意思的设计点之一。

有些 state table 实现，会把每个 cell 设计成一个复杂结构体，里面可能包含：

1. guard condition
2. next state
3. action function
4. 各种附加元数据

Samek 明确不推荐这种过重的表项设计，原因包括：

1. 初始化变得非常繁琐。
2. 内存占用变大。
3. 代码会被切碎成大量小函数。
4. 整体实现反而更难维护。

### Samek 选择的简化方案：表项只存 action handler

在这节课里，他选择了一种更简单的方案：二维表的每个 cell 只存一个函数指针，也就是 `TimeBombAction`。

这个 action handler：

1. 接收 `me` 指针，用于访问对象数据。
2. 接收 `e` 指针，用于访问事件参数。
3. 在函数内部自行决定是否检查 guard。
4. 也在函数内部自行修改 `me->state`。

这样一来，state table 不再显式保存“next-state + guard + action”的完整描述，而只保存“这个状态-事件组合对应哪段处理逻辑”。

这让表更简单，也避免了把状态机拆成过多的独立元件。

## state table 应该是 const，并尽量放进 ROM

Samek 反复强调，state table 是设计时完全确定、运行时不应修改的数据结构。因此它理应声明为 `const`。

这样做的好处有两个：

1. 编译器可以防止你在运行时误改状态表。
2. 常量表可以放到 ROM，而不是占用 RAM。

对嵌入式系统来说，这是一个非常实际的优势。

当然，`const` 对象必须在定义处初始化，所以 state table 需要直接用 C 的数组初始化器完整写出来。

## 处理 signal 索引时要注意：枚举值未必从 0 开始

这里有一个实现细节非常关键。

二维数组的索引必须是：

1. 连续的。
2. 从 0 开始。

但在这套代码里，应用级 signal 并不是从 0 开始的，因为前面还有框架保留的 `INIT_SIG`、`USER_SIG` 等值。所以 Samek 特别提醒，state table 的信号维度必须把这些保留信号也一起考虑进去，否则数组索引就会错位。

这也是 data-driven 实现常见的问题：图上看起来很干净，但落到数组索引上时，你必须非常小心数值编码与枚举布局。

## 把原来的 TimeBomb dispatch 迁移到 state table

课程接着把 `TimeBomb` 的行为逐步迁移到 state table 版本。

### 先定义各个 action handler

原来 nested-switch 中每个状态-事件分支里的处理逻辑，现在都被拆成独立函数，例如：

1. `TimeBomb_init`
2. `TimeBomb_wait4button_pressed`
3. `TimeBomb_blink_timeout`
4. `TimeBomb_pause_timeout`
5. `TimeBomb_ignore`

这些函数大体上就是把原来的 case 代码直接拷过去，只不过现在它们需要：

1. 通过 `me` 修改状态变量。
2. 通过 `e` 获取事件信息。
3. 在必要时处理 guard。

与此同时，构造函数里要把 `me->state` 初始化成一个已知值，因为它之后要直接作为 state table 的索引。

### 新的 dispatch 变得非常短

一旦表和 action handler 都准备好，`dispatch()` 的实现就会变得很短：

1. 用 `me->state` 和 `e->sig` 去索引 `TimeBomb_table`。
2. 取出相应的函数指针。
3. 调用这个函数。

Samek 认为，这种写法的一个明显优点是：运行期分派逻辑非常规则，代码也更“整齐”。

同时，他也建议在索引前加断言，确保 `state` 和 `sig` 都在合法范围内。

## state table 的本质：实现从 code-driven 变成 data-driven

这一步之后，状态机实现的重心已经发生了变化。

在 nested-switch 中：

1. 行为规则主要编码在控制流结构里。
2. 代码本身就是状态机的主载体。

而在 state table 中：

1. 行为规则主要体现在表这个数据结构中。
2. `dispatch()` 只是一个通用的解释器或执行器。

Samek 把这种风格称为 data-driven implementation。这是理解本课的一个关键点。

## 接下来进一步改造：把重复动作移入状态的 entry/exit action

在 `TimeBomb` 状态机中，Samek 发现有些动作在多个 transition 上重复出现。

最典型的是进入 `blink` 状态时，总要做两件事：

1. 打开红灯。
2. 启动半秒 time event。

无论是从 `wait4button` 进入 `blink`，还是从 `pause` 经 guard 回到 `blink`，这两步都必须执行。这说明它们逻辑上属于“进入 blink 状态时必须做的准备”，而不是某条特定 transition 独有的动作。

这就是 entry action 的用武之地。

### entry/exit action 的意义

如果某个动作：

1. 无论从哪条转移进入某个状态都必须执行，

那么它更适合写成该状态的 `entry` action。

反过来，如果某个动作：

1. 无论通过哪条路径离开某个状态都必须执行，

那么它更适合写成该状态的 `exit` action。

Samek 由此对 `TimeBomb` 做了进一步整理：

1. `wait4button` 的绿灯点亮移到 entry，熄灭移到 exit。
2. `blink` 的红灯点亮和 arm timeout 移到 entry，红灯熄灭移到 exit。
3. `pause` 的 arm timeout 移到 entry。
4. `boom` 的所有 LED 点亮移到 entry。

这样一来，更多动作被归属于状态本身，而不是散落在 transition 上。

## 这相当于让软件状态机更偏向 Moore 风格

Samek 顺带借用上一课的硬件术语说明：

1. 如果动作主要挂在 transition 上，状态机更像 Mealy 型。
2. 如果动作更多归属到 state 本身，状态机就更偏向 Moore 型。

他也明确提醒，不要把 Mealy/Moore 的硬件分类硬套到软件中，因为大多数软件状态机其实是混合型的。但从多年经验来看，动作偏向写在 state 上的软件状态机，通常会更清晰、更健壮、更容易维护。

## 如何在 state table 中支持 entry/exit action

为了让 state table 也支持 entry/exit，Samek 采用了一个很巧妙也很直接的方法：给框架再增加两个保留 signal。

也就是：

1. `ENTRY_SIG`
2. `EXIT_SIG`

这样一来，entry 和 exit action 也能像普通事件处理一样，成为 state table 中的一列。

换句话说：

1. state table 的“事件维度”不再只包含业务事件。
2. 还包含两个特殊的伪事件：进入状态和退出状态。

然后，对需要 entry/exit 的状态，就在对应单元格里放上相应的 action handler；对不需要的状态，则放 `ignore`。

## 为了正确触发 entry/exit，需要 action handler 返回状态码

这时又出现一个实现问题：`dispatch()` 怎么知道某个 action handler 执行后是否真的发生了 transition？

Samek 的方案是让每个 action handler 返回一个 status，表示本次处理结果。例如：

1. `TRAN_STATUS`：发生了状态转移。
2. `HANDLED_STATUS`：事件被处理，但没有转移。
3. `IGNORED_STATUS`：事件被忽略。
4. `INIT_STATUS`：执行了初始转移。

有了这个返回值，`dispatch()` 就能在调用某个 action handler 之后判断：

1. 如果发生 transition，就先调用旧状态的 `EXIT_SIG`。
2. 再调用新状态的 `ENTRY_SIG`。
3. 如果是 `INIT_STATUS`，则只做进入，不做退出。

这相当于把 entry/exit 调度逻辑集中在通用 dispatch 里，而不是让每个 transition handler 自己负责。

## 改造后的 dispatch 更像一个通用事件处理器

加入 status 之后，新的 `dispatch()` 逻辑大致变成：

1. 保存 `prev_state`。
2. 调用 `state_table[me->state][e->sig]` 对应的 handler。
3. 读取返回的 status。
4. 若 status 表示发生 transition，则调用旧状态的 exit action，再调用新状态的 entry action。
5. 若 status 表示初始化，则直接进入初始状态。

此时的 `dispatch()` 已经不再只是“查表并调用一下”，而更像一个小型、可复用的 event processor。Samek 也在结尾暗示，下一课会继续沿着这个方向发展。

## state table 的优点：结构规整、性能稳定、覆盖性强

课程最后对这种实现方式做了总结。

其主要优点包括：

1. 结构非常规整，强迫你系统地考虑每个 `state-signal` 组合。
2. 运行期分派逻辑比较简单直接，性能相对稳定可预期。
3. 状态机规则被显式放进表中，具有很强的结构化特征。

对某些追求规则性和可分析性的场景来说，这确实是很大的优点。

## state table 的缺点：碎片化、稀疏、扩展时有“表格痛感”

Samek 同样非常坦率地指出了其缺点。

### 1. 代码碎片化

原来集中在一个 `dispatch()` 函数里的逻辑，现在会被拆成很多小型 action handler。对人脑阅读来说，整体跳转感会变强。

### 2. 表通常很稀疏

很多状态并不处理很多事件，因此表里会有大量 `ignore` 空洞。尤其事件枚举值本身还可能存在数值间隔，这会进一步让表显得“浪费”。

### 3. 新增状态和事件的心理门槛更高

这也是 Samek 认为最严重的问题之一。因为只要新增：

1. 一个状态，
2. 或一个事件，

往往就意味着要增加整整一行或一列。开发者会本能地觉得这很麻烦，于是为了避免“动表”，就更倾向于在现有 action handler 里加内部变量、加 guard condition、加特殊情况判断。

结果反而又慢慢滑回了 spaghetti code，而这正是使用 state machine 本来要避免的东西。

## lesson 38 关键代码（节选）

下面几段代码对应本课实现层面的三个关键点：

1. 用二维 `const` 表保存 `state x signal` 映射。
2. 通过 handler 返回 `Status`，由统一 `dispatch()` 触发 entry/exit。
3. 通过保留信号把 `ENTRY` / `EXIT` 也纳入同一套事件分发机制。

代码来源：`lesson-38/tm4c123-uc_ao-keil`。

### 1. 状态表只保存 action handler（`main.c`）

```c
typedef Status (*TimeBombAction)(TimeBomb * const me, Event const * const e);

static TimeBombAction const TimeBomb_table[MAX_STATE][MAX_SIG] = {
			   /*   INIT            | ENTRY                      | EXIT                      | PRESSED                      | RELEASED        | TIMEOUT */
/* wait4button */ { &TimeBomb_init,   &TimeBomb_wait4button_ENTRY, &TimeBomb_wait4button_EXIT, &TimeBomb_wait4button_PRESSED, &TimeBomb_ignore, &TimeBomb_ignore },
/* blink       */ { &TimeBomb_ignore, &TimeBomb_blink_ENTRY,       &TimeBomb_blink_EXIT,       &TimeBomb_ignore,              &TimeBomb_ignore, &TimeBomb_blink_TIMEOUT },
/* pause       */ { &TimeBomb_ignore, &TimeBomb_pause_ENTRY,       &TimeBomb_ignore,           &TimeBomb_ignore,              &TimeBomb_ignore, &TimeBomb_pause_TIMEOUT },
/* boom        */ { &TimeBomb_ignore, &TimeBomb_boom_ENTRY,        &TimeBomb_ignore,           &TimeBomb_ignore,              &TimeBomb_ignore, &TimeBomb_ignore }
};
```

这里的表项没有硬塞 next-state/guard 结构体，而是只放函数指针，guard 与状态切换由 handler 内部处理，代码体量更可控。

### 2. `dispatch()` 根据 `Status` 统一调度 entry/exit（`main.c`）

```c
static void TimeBomb_dispatch(TimeBomb * const me, Event const * const e) {
	Status stat;
	int prev_state = me->state; /* save for later */

	Q_ASSERT((me->state < MAX_STATE) && (e->sig < MAX_SIG));
	stat = (*TimeBomb_table[me->state][e->sig])(me, e);

	if (stat == TRAN_STATUS) { /* transition taken? */
		Q_ASSERT(me->state < MAX_STATE);
		(*TimeBomb_table[prev_state][EXIT_SIG])(me, (Event *)0);
		(*TimeBomb_table[me->state][ENTRY_SIG])(me, (Event *)0);
	}
	else if (stat == INIT_STATUS) { /* initial transition? */
		(*TimeBomb_table[me->state][ENTRY_SIG])(me, (Event *)0);
	}
}
```

这正是本课最核心的“框架化”动作：每个 handler 不必自己调用 entry/exit，只需返回状态码即可。

### 3. `ENTRY_SIG` / `EXIT_SIG` 作为保留信号（`uc_ao.h`、`bsp.h`）

```c
/* uc_ao.h */
enum ReservedSignals {
	INIT_SIG,
	ENTRY_SIG,
	EXIT_SIG,
	USER_SIG
};

/* bsp.h */
enum EventSignals {
	BUTTON_PRESSED_SIG = USER_SIG,
	BUTTON_RELEASED_SIG,
	TIMEOUT_SIG,
	MAX_SIG
};
```

保留信号和业务信号共用同一个 `sig` 空间，因此二维表的列索引可统一处理，这也是 `MAX_SIG` 必须覆盖保留信号的原因。

## 总结

38 课的核心，是引入 state table 这种典型的 data-driven 状态机实现方式，并在其基础上进一步加入 entry/exit action。

这一课最重要的结论可以概括为：

1. nested-switch 是最直接的状态机实现方式，而 state table 则代表了一种 data-driven 的实现思路。
2. state table 本质上是一个“状态 x 事件 -> 行为”的二维映射，可以直接用 C 的二维数组表达。
3. 一个务实的实现策略，是让表项只保存 action handler 指针，而把 guard 判断和状态修改放进 handler 内部处理。
4. state table 应该声明为 `const`，尽量放入 ROM，并用 `MAX_STATE`、`MAX_SIG` 这类技巧保持维度与枚举同步。
5. entry/exit action 能把一些逻辑上属于状态本身的动作，从 transition 中提炼出来，使状态机更清晰，并使软件状态机更偏向 Moore 风格。
6. 在实现上，可以通过增加 `ENTRY_SIG` 和 `EXIT_SIG` 两个保留信号，再配合 handler 返回 status，来统一调度 entry/exit 调用。
7. state table 的优点是结构规整、覆盖性强、运行期分派稳定；缺点则是代码碎片化、表稀疏，以及扩展时容易给开发者带来“加一整行/列”的心理负担。

如果说第 35-36 课重点在于“如何把行为建模成状态和转移”，第 37 课在于“状态机类型及其运行环境”，那么第 38 课则开始进入“状态机到底应该怎样在代码里实现”。这一步很重要，因为从这里开始，状态机不再只是图上的设计符号，而逐渐变成一套可以复用、可以抽象、可以比较不同实现策略优劣的工程技术。
