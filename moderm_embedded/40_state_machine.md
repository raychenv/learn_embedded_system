# 40 层次状态机初探

这一课，Miro Samek 开始正式引入 modern hierarchical state machines，也就是通常所说的 statecharts。前几课已经把传统 finite state machine 的概念、guard、state table，以及基于 state-handler 的“较优实现”都铺垫好了。这一课的任务，就是进一步说明：为什么传统平面状态机会在稍复杂的问题上迅速失控，以及层次状态机如何通过状态嵌套和行为继承，把这些重复结构优雅地压缩掉。

这节课还有一个很重要的工程目标：不是停留在概念图示上，而是直接把 `TimeBomb` 应用扩展为层次状态机，再把它从教学用的 `Micro-C-AO` 框架迁移到正式的 QP/C 框架。Samek 这样安排，是为了同时说明两件事：

1. 层次状态机并不是“纸上模型”，它完全可以手工实现。
2. 但一旦你需要完整支持层次状态语义，框架支持就会变得非常重要。

lesson 40 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-40

## 从一个新需求开始：给 TimeBomb 增加“拆弹”功能

课程先用一个非常具体的新需求来暴露传统状态机的根本问题。

在原来的 `TimeBomb` 里，炸弹一旦开始计时，就只能一路走向 `boom`。现在加入新需求：

1. 第二个按钮 `SW2` 用来 defuse 炸弹。
2. 无论炸弹处于哪个阶段，只要按下 `SW2`，都必须进入 `defused` 状态。
3. 进入 `defused` 后，蓝灯点亮，表示炸弹已解除。

这个需求听上去并不复杂，但它对传统平面状态机的影响非常典型：如果 `BUTTON2_PRESSED` 必须在任何阶段都生效，那就意味着你必须在所有已有状态里都添加这条转移。

也就是说，在原来的 `wait4button`、`blink`、`pause`、`boom` 等每个状态中，都要复制一条指向 `defused` 的 `BUTTON2_PRESSED` transition。

## 这暴露了传统有限状态机的根本问题：state-transition explosion

Samek 说明，这种重复不是偶然的编码坏习惯，而是传统平面状态机的结构性问题。代码里所有的复制粘贴，其实只是忠实反映了状态图本身的重复。

这违反了一个很基本的软件原则：DRY，也就是 Don't Repeat Yourself。

其后果很直接：

1. 调试困难，因为相同逻辑分散在多处。
2. 维护困难，因为修改时必须保证所有副本都一致。
3. 很容易遗漏某一处，造成行为不一致。
4. 状态机复杂度增长速度，快于问题本身复杂度的增长速度。

Samek 把这种现象称为 state-transition explosion。也就是说，随着问题规模扩大，平面状态机里重复出现的状态和转移会爆炸式增长，最终使得传统 FSM 难以支撑更复杂的行为建模。

这也是很多团队虽然知道状态机概念，却始终觉得“状态机一复杂就不好用”的根本原因。

## David Harel 的解决方案：层次状态机

Fortunately，课程接着引出 1980 年代末 David Harel 提出的 statecharts。其最关键的创新，就是允许 state 嵌套 state，也就是 hierarchical state nesting。

在这种形式中：

1. 高层状态可以包含低层状态。
2. 高层状态定义的行为，会自动适用于其内部的所有子状态。
3. 子状态如果自己显式处理了某事件，则会覆盖高层行为；否则事件会向上“冒泡”到 superstate。

这就是层次状态机最核心的语义，也是它能够消除重复的根本原因。

## 在 TimeBomb 中引入 superstate：把四个状态包进 `armed`

对 `TimeBomb` 来说，最自然的层次化方式是：

1. 把原来的 `wait4button`
2. `blink`
3. `pause`
4. `boom`

这四个状态全部包进一个新的高层状态 `armed`。

这样一来：

1. `armed` 是 superstate。
2. 里面的四个状态是其 substates。
3. `defused` 则与 `armed` 并列，不属于它的子状态。

接下来，只需要把 `BUTTON2_PRESSED -> defused` 这条转移定义在 `armed` 上，就能自动让它适用于 `armed` 的所有子状态。

于是原来必须写四遍的转移，现在只剩一条。重复被直接消除了。

## 层次状态机的语义：事件未在子状态处理时，向上交给 superstate

Samek 特别强调，state nesting 不是一种“画图时的分组美化”，而是有明确行为语义的。

例如，当前如果机器正处于 `wait4button` 子状态：

1. 收到 `BUTTON2_PRESSED`。
2. `wait4button` 自己没有处理这个事件。
3. 事件不会像传统 FSM 那样直接被忽略。
4. 它会向上交给 `armed` superstate。
5. `armed` 处理该事件，并触发到 `defused` 的转移。

反过来，如果子状态自己已经处理了某事件，那么它就不会再上传到 superstate。这种机制使得：

1. superstate 可以定义共同行为。
2. substate 可以在需要时 override 这些共同行为。

Samek 把这种关系类比为 behavioral inheritance，也就是行为层面的继承。这和前面课程中的 OOP inheritance、Win32 里的 Programming by Difference，有非常强的对应关系。

## 层次状态机本质上是“行为继承”

Samek 在这里把层次状态机和前面几课串了起来。

他指出，你其实已经见过类似思想：

1. Win32 GUI 中，窗口自己不处理的消息会交给默认窗口过程，这就是 Ultimate Hook / Programming by Difference。
2. OOP 中，子类会继承父类的共同行为，再在必要时 override。

层次状态机做的事情，本质上也是一样：

1. higher-level state 捕获公共行为。
2. lower-level state 继承这些行为。
3. 只有差异化部分才在子状态中显式定义。

因此，hierarchical state machine 并不是“在传统 FSM 上随便加个分层结构”，而是把 inheritance 的思想引入到了行为建模中。

## 在已有的 state-handler 实现上，层次状态其实很容易表达

这一课特别重要的一点是：Samek 用第 39 课建立的“较优”状态机实现方式作为起点，说明层次状态的表达其实并不难。

关键做法是：让每个 substate 在“未处理事件”的默认路径中，显式指定自己的 superstate。

### 用 `SUPER()` 宏表达父状态

这和前一课 `TRAN()` 宏的思路类似。现在新增一个 `SUPER()` 宏，其参数就是当前 state 的父状态 handler，比如：

1. `TimeBomb_armed`
2. 或最顶层的 `Hsm_top`

因此：

1. 如果某状态处理了事件，就返回 handled/transition 状态。
2. 如果它没处理，就执行 `SUPER(parent)`。
3. 这表示：把同一个事件继续交给 parent state 去试着处理。

这正是事件向 superstate 传播的代码实现。

### `defused` 的父状态是隐含的 `top`

由于 `defused` 不属于 `armed`，所以它的 superstate 不是 `armed`，而是隐式的顶层 `top`。

这也是 Samek 提醒的一个很有用的实现概念：把整个状态机都看成嵌套在一个顶层 `top` 里，往往会使实现更整齐。

## 把 `armed` 写成一个真正的 superstate-handler

在代码层面，`armed` 现在也变成了一个真正的 state-handler function。它的职责是：

1. 处理共同行为，例如 `BUTTON2_PRESSED -> defused`。
2. 作为其所有 substates 的上层 fallback。
3. 在没有进一步父状态可用时，再通过 `SUPER(Hsm_top)` 把事件交给顶层。

这样一来，原来散落在四个子状态里的 `BUTTON2_PRESSED` 处理代码就都可以删掉，整个应用代码结构反而变得更简洁。

这也正是 Samek 所说的：第 39 课的“较优实现”之所以优，是因为它很容易向 hierarchy 扩展。层次关系只需要在状态处理函数的默认路径里补上一层 superstate 说明即可。

## 但完整的层次状态语义并不止是事件向上冒泡

到这里，层次状态机已经能工作一部分了，但 Samek 很快指出：真正困难的部分，不在于“把事件沿 superstate 链往上试”。这一点只需要一个 while 循环就能处理。

真正复杂的是 transition chains，也就是：

1. 从当前嵌套状态配置中正确退出哪些状态。
2. 再进入目标嵌套状态配置中的哪些状态。
3. 以及如何处理 nested initial transitions。

这正是层次状态机和传统平面 FSM 的关键差别。因为在平面 FSM 里，一次 transition 只需要：

1. 退出当前状态。
2. 进入目标状态。

而在 HSM 中，一次 transition 可能涉及：

1. 退出多个层次的源状态。
2. 进入多个层次的目标状态。
3. 自动执行目标 superstate 内部的初始转移。

Samek 在这一课并没有完全自己从零实现这整套语义，而是借机说明：这部分复杂性应当被下沉到框架，而不是暴露给应用层。

## 先在 `armed` 上加入更完整的层次语义元素

为了展示这一点，课程继续给 `TimeBomb` 增加两个重要元素。

### 1. `armed` 的 exit action

之前从 `boom` 转到 `defused` 时，LED 没有被正确清理，视觉效果混乱。Samek 借此说明：在层次状态机里，清理动作往往更适合挂在高层状态的 exit action 上。

因此，他把“退出 armed 时要关掉相关 LED”的动作放到 `armed` 的 exit action 中。这样无论是从哪个子状态离开 `armed`，都能统一完成清理。

### 2. `armed` 的 nested initial transition

此外，`armed` 还可以拥有自己的 initial transition，指向其默认子状态 `wait4button`。

这意味着：

1. 如果从别处直接转移到 `armed`，
2. 那么不是停留在抽象的 `armed`，
3. 而是必须继续执行它的内部 initial transition，
4. 最终进入 `wait4button`。

这就是 nested initial transition 的语义，也是 hierarchical state machine 中非常典型的机制。

## 实现复杂性应该下沉到框架：从教学框架转向专业的 QP/C

Samek 到这里做出一个非常重要的工程决策：不继续在教学用的 `Micro-C-AO` 里手搓完整层次状态机语义，而是把应用 port 到专业的 QP/C 框架。

这个决定本身很有启发性，因为它说明：

1. 简化版框架适合教学和理解原理。
2. 但一旦进入完整 HSM 语义，自己维护全部细节并不划算。
3. 更合理的工程做法，是用成熟框架承接这部分复杂性。

Samek 还顺带类比到前面 RTOS 课程：当初从教学用 `MiROS` 迁移到 QP/C 的 `QXK` 时，真正复杂的也不是上下文切换，而是线程通信机制。这一课同样如此：层次状态机的难点不在“状态嵌套”本身，而在“正确处理 entry/exit/initial transition 链条”。

## 把应用 port 到 QP/C：行为不变，基础设施替换

课程把这次迁移明确称为 porting，也是一种 refactoring。形象地说，就是在不大动房子本体的前提下，把地基换掉。

### 框架替换的总体思路

迁移步骤大致包括：

1. 删除 `Micro-C-AO` 和相关教学框架代码。
2. 引入 QP/C 的 `QF`、`QV` 和 ARM Cortex-M 对应 port。
3. 修改 include path。
4. 把 BSP 和应用代码中的 API 名字替换为 QP/C 风格。

这其中最有意思的一点是，应用级状态机代码本身并没有被推倒重写。很多改动只是名字替换，例如：

1. `Active` -> `QActive`
2. `TimeEvent` -> `QTimeEvt`
3. `State` -> `QState`
4. `Event` -> `QEvt`
5. `TRAN` -> `Q_TRAN`
6. `SUPER` -> `Q_SUPER`
7. `ENTRY_SIG` -> `Q_ENTRY_SIG`
8. `EXIT_SIG` -> `Q_EXIT_SIG`
9. `INIT_SIG` -> `Q_INIT_SIG`

这恰恰证明，QP/C 使用的正是 Samek 前一课称为“较优”的那套 state-handler 实现范式，只不过它已经把层次语义、事件处理器和 active object 基础设施都做成了完整框架。

## QP/C 的意义：把复杂性封装到框架中

迁移完成后，最关键的收获不是“代码终于能编译”，而是：

1. hierarchical state machine 的完整语义 now works correctly。
2. 包括 superstate 处理、nested initial transition、entry/exit 链条，都由框架正确处理。
3. 应用层仍然保持了比较清晰、接近状态图的 state-handler 风格。

这正是 Samek 这节课反复强调的设计方向：复杂性不是不能处理，而是应该被封装在正确层次上。对于 HSM 来说，这个层次就是框架，而不是每个应用状态机自己重复发明一套算法。

## 这一课也在反驳一个常见误解：层次状态机并没有“难到无法手工编码”

Samek 在结尾特别说明，他花了较多时间讲实现细节，是为了反驳一个常见误解：hierarchical state machine 太复杂，不适合手工实现。

他的观点更细致：

1. 如果你用错实现策略，确实会很快陷入复杂度泥潭。
2. 但如果前面先建立了合理的 state-handler 风格和 reusable event processor，向 hierarchy 扩展其实是可控的。
3. 真正复杂的部分，主要集中在框架内的 transition semantics，而不是应用级状态逻辑。

所以，HSM 不是“不能手写”，而是“不该每次都从零手写一整套运行语义”。这也是为什么成熟框架和自动代码生成会在后续课程中变得越来越重要。

## 第40课关键代码（简化融入）

下面这 3 段代码，基本对应本课最核心的 3 个语义点：

1. `armed` 作为 superstate，统一处理 `BUTTON2_PRESSED_SIG -> defused`。
2. 子状态默认 `Q_SUPER(TimeBomb_armed)`，把未处理事件向上冒泡。
3. `defused` 与 `armed` 并列，父状态是 `QHsm_top`，并支持二次按键回到 `armed`。

### 1) `armed` superstate：公共行为 + exit/init 语义

```c
QState TimeBomb_armed(TimeBomb * const me, QEvt const * const e) {
	QState status;
	switch (e->sig) {
		case Q_EXIT_SIG: {
			BSP_ledRedOff();
			BSP_ledGreenOff();
			BSP_ledBlueOff();
			status = Q_HANDLED();
			break;
		}
		case Q_INIT_SIG: {
			status = Q_TRAN(TimeBomb_wait4button);
			break;
		}
		case BUTTON2_PRESSED_SIG: {
			status = Q_TRAN(TimeBomb_defused);
			break;
		}
		default: {
			status = Q_SUPER(QHsm_top);
			break;
		}
	}
	return status;
}
```

### 2) 子状态默认冒泡到 `armed`

```c
QState TimeBomb_wait4button(TimeBomb * const me, QEvt const * const e) {
	QState status;
	switch (e->sig) {
		case BUTTON_PRESSED_SIG: {
			me->blink_ctr = 5U;
			status = Q_TRAN(TimeBomb_blink);
			break;
		}
		default: {
			status = Q_SUPER(TimeBomb_armed);
			break;
		}
	}
	return status;
}
```

### 3) `defused`：与 `armed` 并列，父状态为 top

```c
QState TimeBomb_defused(TimeBomb * const me, QEvt const * const e) {
	QState status;
	switch (e->sig) {
		case Q_ENTRY_SIG: {
			BSP_ledBlueOn();
			status = Q_HANDLED();
			break;
		}
		case BUTTON2_PRESSED_SIG: {
			status = Q_TRAN(TimeBomb_armed);
			break;
		}
		default: {
			status = Q_SUPER(QHsm_top);
			break;
		}
	}
	return status;
}
```

你可以把这三段当成第40课的最小可复用模板：

1. 先抽出 superstate 处理公共事件。
2. 子状态默认向 superstate 冒泡。
3. 需要并列状态时，父状态回到 top。

## 总结

40 课的核心，是从传统有限状态机正式迈入层次状态机，并通过真实工程迁移说明：HSM 的关键价值在于消除重复、表达共同行为，而其实现复杂性应由框架承担。

这一课最重要的结论可以概括为：

1. 传统平面 FSM 在稍复杂问题上会遭遇 state-transition explosion，重复状态与转移迅速堆积，违反 DRY 原则。
2. 层次状态机通过 superstate/substate 关系，把共同行为提升到高层状态中定义，从而用 behavioral inheritance 消除重复。
3. 在语义上，未被子状态处理的事件会向 superstate 传播，而子状态也可以 override 父状态行为。
4. 在基于 state-handler 的实现里，层次关系可以通过 `SUPER()` 宏和隐式 `top` 状态非常自然地表达出来。
5. 但完整 HSM 语义不仅仅是事件向上冒泡，更难的是 transition chain、entry/exit 链和 nested initial transition 的正确处理。
6. 这部分复杂性应当下沉到框架层，而不是让每个应用手工重复实现。
7. 将 `TimeBomb` 从 `Micro-C-AO` 迁移到 QP/C 说明：一旦采用合适的 state-handler 范式，应用级代码可以较平滑地移植到支持完整 HSM 语义的专业框架。

如果说第 39 课解决的是“怎样实现一个更优的平面状态机”，那么第 40 课解决的就是“为什么平面状态机终究不够，以及如何通过 hierarchy 把状态机从能用推进到真正可扩展”。这一步非常关键，因为从这里开始，状态机不再只是描述若干离散状态，而是开始具备真正的软件工程抽象能力。下一步，自然就是把这种层次状态机的代码生成自动化。
