# 42 层次状态机语义深入理解

这一课，Miro Samek 继续讲 hierarchical state machines，不过重点不再是“如何把 superstate/substate 写进代码”，而是更深入地解释 HSM 的语义到底是什么。第 40 课已经展示了层次状态机如何消除重复，并把 `TimeBomb` 从平面 FSM 扩展成 hierarchical state machine；第 41 课又引入了自动代码生成。但如果只停留在“会画、会生成、能运行”的层面，你其实还没有真正掌握 HSM，因为层次状态机最关键的价值，恰恰体现在它精确而丰富的运行语义上。

因此，这一课不再围绕某个真实嵌入式应用，而是专门拿出一个经过精心设计的教学状态机 `QHsmTst`，系统展示各种 transition topology、guard、internal transition、self-transition、嵌套 initial transition，以及它们对应的 entry/exit 执行顺序。Samek 的目标很明确：让你不只是“知道有这些概念”，而是通过 trace 真正看清楚 hierarchical state machine 在每一步到底做了什么。

lesson 42 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-42

## 为什么需要一个专门的语义示例

课程一开始先解释，前面的 `TimeBomb` 虽然已经引入了层次状态，但它只有一层简单嵌套，并不足以覆盖 HSM 的全部语义。比如：

1. 多层嵌套状态之间的转移。
2. 转移时 exit/entry 链的精确顺序。
3. guard 为假时事件如何继续向上传播。
4. internal transition 与 self-transition 的区别。
5. 不同层次状态间的 Least Common Ancestor（LCA）如何影响 transition chain。

为了把这些语义一次讲透，Samek 使用了 QP/C 框架自带的 `QHsmTst` 示例。这个例子不对应某个现实设备，但它被故意设计成包含各种典型 transition 形态，因此非常适合做“语义实验台”。

## 用软件追踪来学习 HSM 语义

课程这里再次强调了 software tracing 的重要性。`QHsmTst` 中的每个：

1. entry action
2. exit action
3. transition action

都会输出 trace，例如：

1. `s2-ENTRY`
2. `s21-EXIT`
3. `s1-A`

这意味着你可以把：

1. 当前状态图结构
2. dispatch 的事件
3. 运行时实际执行的动作顺序

三者精确对应起来。

Samek 还顺带提到一个个人经历：`QHsmTst` 最初是他为自己书中的例子手工写出来的，后来 QM 自动生成的代码产生了与手写版不同的 trace。最初他怀疑是代码生成器错了，但最终发现是自己手工画的图有误。这让他彻底意识到：手工实现即使对专家也不可靠，自动代码生成更值得信任。

## 初始转移：不仅进入显式目标，还会沿着嵌套 initial transition 一路钻到叶状态

课程先从 top-most initial transition 开始分析。

在 HSM 中，初始转移并不是“到达显式目标状态就结束”。它有两个关键语义：

1. 任何被 transition 穿过的状态都必须正确执行 entry action。
2. 一旦到达显式目标状态，如果该状态内部还有 nested initial transition，就必须继续沿着这些初始转移向下钻，直到到达 leaf state。

在 `QHsmTst` 中，顶层 initial transition 先进入 `s`，再进入 `s2`，但不会停在 `s2`，而是继续走 `s2` 内部的初始转移，最终进入 `s21`、`s211`，直到抵达稳定的 leaf state。

Samek 把这个最终状态称为 stable state configuration。也就是说，HSM 真正“停住”的位置总是某个叶状态，而不是一个仍有待展开的抽象父状态。

## 事件处理从当前 leaf state 开始，逐层向上尝试

HSM 的事件处理规则延续了第 40 课的核心思想：

1. 事件总是先交给当前 leaf state 处理。
2. 如果当前状态不处理，就交给它的 superstate。
3. 再不处理，就继续往上。
4. 一直上溯到最顶层。

例如当前处于 `s211` 时，若收到 `G` 事件：

1. `s211` 不处理。
2. 事件向上交给 `s21`。
3. `s21` 正好定义了 `G` 的 transition，于是由它处理。

这一点看似简单，但它是所有层次状态语义的出发点。因为只有明确了“事件处理从哪里开始、如何向上冒泡”，后面 transition chain、guard 和 inheritance 才有意义。

## LCA 决定退出链和进入链的边界

课程最重要的技术点之一，是 Least Common Ancestor，简称 LCA。

当某个 transition 被触发时，HSM 不是简单地“退出当前状态、进入目标状态”，而是需要：

1. 沿着当前状态配置向上退出到 LCA，但不退出 LCA 本身。
2. 再从 LCA 之下沿目标路径向下进入，直到目标状态配置。

这保证了：

1. 所有离开的状态都能正确清理。
2. 所有进入的状态都能正确初始化。
3. 共享的高层上下文不会被不必要地反复退出与进入。

例如，从 `s21` 经某条转移进入 `s1` 时，它们的 LCA 是 `s`，所以：

1. 退出 `s211`、`s21` 等位于 LCA 以下的状态。
2. 不退出 `s`。
3. 然后进入 `s1`，并继续按其 nested initial transition 进入 `s11`。

这就是 HSM 里 transition 之所以比平面 FSM 更复杂、但也更强大的原因。

## Internal transition：只执行动作，不改变状态配置

课程接着展示 internal transition 的语义。

如果某个事件在某层状态上被 internal transition 处理，那么：

1. 事件会被该状态处理。
2. 会执行关联动作。
3. 但不会发生任何状态变化。
4. 因此也不会执行任何 exit/entry action。

这和 regular transition 非常不同。即便 internal transition 是从 higher-level state 继承下来的，只要它被采用，本次 RTC 步骤也不会改变当前 leaf state。

Samek 用 `I` 事件举例说明：当前虽在 `s11`，但 `I` 由上层 `s1` 的 internal transition 处理。结果是：

1. 执行动作 `s1-I`。
2. 当前状态仍然保持 `s11`。

这再次体现了状态继承的意义：子状态可以使用父状态提供的处理逻辑，但这不等于离开当前子状态。

## Self-transition 在 HSM 中与 internal transition 完全不同

这是本课一个非常值得注意的点。

在很多非层次、非 entry/exit 风格的传统 FSM 中，self-transition 往往只是“在本状态里做点动作”的一种写法，因此容易和 internal transition 混淆。

但在 hierarchical state machine 中：

1. internal transition 不改变状态配置，不触发 exit/entry。
2. self-transition 虽然源和目标看起来是同一状态，但它仍然是 regular transition。

因此 self-transition 会：

1. 退出该状态配置直到 LCA。
2. 再重新进入该状态及其子状态。
3. 重新执行 entry、nested initial transition 等完整初始化链条。

Samek 指出，这使 self-transition 在 HSM 中具有一个很有用的语义：clean reset of state context。也就是“把当前状态上下文干净地退出并重新初始化一遍”。

这在实现 Cancel、Reset 之类功能时特别有价值。

## Guard 为假时，默认等价于“此转移不存在”，事件继续向上传播

课程随后利用 `D` 事件解释 guard condition 在层次状态机中的一个关键语义。

如果当前状态存在一条带 guard 的 transition，但 guard evaluates to FALSE，那么 UML 语义把它视为：

1. 这条 transition 当前不可用。
2. 对当前状态来说，等价于“没有这条转移”。
3. 因而事件不会被消费，而会继续向 superstate 传播。

在 `QHsmTst` 中，`s11` 有一条带 `foo` 条件的 `D` transition：

1. 当 `foo == 0` 时，guard 为假。
2. 于是 `D` 没有在 `s11` 被处理。
3. 事件继续向上传给 `s1`。
4. `s1` 上正好有 complementary guard 的 `D` transition，于是被那一层接住。

这说明 guard 并不是“事件被看到但忽略”，而是更精确地说：guard 为假时，当前这条 transition 根本不参与竞选。

## 如果不希望 guard 失败后继续向上传播，需要显式加 `[else]`

Samek 接着指出，这种“guard 失败 -> 事件继续往上找”的语义有时是你想要的，但有时恰恰不是。

如果你希望：

1. 当前状态看到某个事件后，
2. 即使 guard 为假，也不要再让 superstate 处理它，

那么就需要在当前状态显式加一个 complementary `[else]` 分支。这样就能把事件“截住”，即便它最后只是被当前状态忽略，也不会继续传播到更高层。

这是一个非常实用的设计细节，因为它决定了“guard 失败”到底是：

1. 继续给上层机会，
2. 还是在本层就终止处理。

## 同一条高层 transition，会因为当前子状态不同而产生不同 exit/entry 序列

课程还强调了一个非常体现 HSM 威力的现象：同一条定义在高层状态上的 transition，如果是从不同子状态配置继承触发的，实际执行的 exit/entry 序列可能不同。

换句话说：

1. transition label 相同。
2. action 也相同。
3. 但由于当前 leaf state 不同，LCA 不同，退出和进入链条也不同。

这说明 HSM 的行为语义并不只是“哪条边被选中”，而是“当前整个状态配置”共同决定了 transition 的具体执行过程。

这也是为什么 HSM 的能力远超平面 FSM：它不仅描述状态，还描述状态配置之间的层次关系。

## 退出程序本身也应该遵循状态机语义

课程最后还用 `TERMINATE` 事件做了一个很有意思的修正。原来的例子里，按下 Escape 直接退出程序，但那是通过 internal transition 实现的，因此不会触发任何 exit action，程序显得“戛然而止”。

Samek 认为更合理的做法是：

1. 增加一个 `final` 状态。
2. 把 `TERMINATE` 改成 regular transition，指向 `final`。
3. 把真正结束程序的动作放到 `final` 的 entry action 里。

这样，整个状态机在结束前就会先执行完整的 exit chain，做完应有的 cleanup，然后再退出。

这个例子非常好地说明：HSM 的 entry/exit 语义并不是“为了形式漂亮”，而是真正能把系统的初始化和清理行为组织得更严谨。

## `QHsmTst` 的价值：它几乎可以回答你对 HSM 语义的所有问题

Samek 在课程结尾特别强调，`QHsmTst` 之所以重要，不是因为它模拟了某个真实产品，而是因为它被精心设计成覆盖几乎所有典型 transition topology。

这意味着：

1. 如果你对某类 HSM transition 语义有疑问，
2. 很多时候不必靠猜，
3. 可以直接在 `QHsmTst` 中触发对应事件，
4. 观察 trace，验证状态机到底会怎么做。

这是一个很好的学习方法：不要只背概念，而是通过可运行例子和 software trace 把语义真正吃透。

## 总结

42 课的核心，是把 hierarchical state machine 从“会画、会写”进一步推进到“真正理解其运行语义”。

这一课最重要的结论可以概括为：

1. HSM 的 initial transition 不会停在显式目标状态，而会继续沿 nested initial transition 一路钻到 leaf state，形成 stable state configuration。
2. 事件总是先从当前 leaf state 开始处理，若本状态不处理，则逐层向 superstate 传播。
3. regular transition 会执行完整的 exit/entry chain，而这个链条的边界由源状态与目标状态的 Least Common Ancestor 决定。
4. internal transition 不改变状态配置，也不会触发任何 exit/entry action。
5. self-transition 在 HSM 中与 internal transition 完全不同，它会 cleanly exit 并 re-enter 当前状态，因此常可用来 reset 某个状态上下文。
6. guard condition 为假时，默认语义是“该转移不存在”，事件继续向上传播；如果不想传播，就需要显式加 `[else]`。
7. 同一条 inherited transition，会因当前子状态配置不同而产生不同的 exit/entry 序列，这正体现了层次状态机的丰富语义。
8. 终止、cleanup 这类行为也应遵循状态机语义，通过 regular transition 和 final state 来完成，而不是粗暴中断。

如果说第 40 课让你第一次看到 hierarchical state machine 能解决什么问题，那么第 42 课则让你真正理解它为什么能做到这一点，以及其精确语义到底是什么。这一步非常关键，因为只有吃透这些语义，后面在设计、调试、自动生成和测试更复杂的 statechart 时，才能心里有数。