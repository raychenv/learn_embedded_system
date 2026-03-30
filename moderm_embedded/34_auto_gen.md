# 34 事件驱动编程用于实时嵌入式系统

这一课，Miro Samek 继续讲 event-driven programming，但重点已经从上一课的桌面 GUI 转到 real-time embedded systems。上一课通过 Win32 GUI 解释了事件循环、消息队列、Run-to-Completion 和 inversion of control，但这些概念如果只停留在桌面窗口程序里，还不容易让人直觉地理解它们为什么对嵌入式同样重要。

所以这一课的切入方式很务实：先回到嵌入式开发者最熟悉的传统 RTOS 线程模型，从 superloop 和 blocking thread 出发，再一点点说明这些做法会带来什么问题，以及为什么 event-driven 的 active object 模型能更系统地解决这些问题。

这节课真正完成的，不只是“把线程改成消息驱动”这么简单，而是一次明确的范式转变：从 shared-state concurrency 的传统 RTOS 思路，转到以 event、message pump 和 active object 为核心的现代 reactive 设计。

lesson 34 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-34

## 从传统 RTOS 示例出发：线程、阻塞和共享状态

课程先给出一个非常典型的 RTOS 示例。程序运行在 TivaC LaunchPad 上，功能并不复杂：

1. 绿色 LED 按固定节奏闪烁。
2. 按下按钮 `SW1` 后，闪烁速度加快。
3. 蓝色 LED 用来显示按钮当前是否被按下。

这个例子包含两个传统 RTOS 线程：

1. `blinky` 线程负责绿色 LED 的闪烁。
2. `button` 线程负责处理按钮按下与释放。

在 `blinky` 线程中，主要逻辑是循环切换 LED 状态，然后调用 RTOS 的延时服务阻塞等待一段时间。这个等待时间来自一个共享变量 `blink_time`。

在 `button` 线程中，线程先阻塞等待“按钮按下”的 semaphore，被唤醒后点亮蓝灯并修改 `blink_time`，然后再次阻塞等待“按钮释放”的 semaphore，最后熄灭蓝灯。

这个例子很小，但已经完整展示了传统 RTOS 应用的典型结构：线程通过 blocking 等待某个事件发生，再在醒来后顺序执行一小段逻辑。

## 传统 RTOS 设计的核心问题：为了响应更多事件，就会不断增加线程

传统 RTOS 线程模型有一个非常根本的局限：一个线程在阻塞等待某个事件时，对其他事件是不响应的。

这意味着：

1. 如果一个线程只等按钮按下，它就不能同时等超时、串口数据、网络报文等其他事件。
2. 如果希望程序同时响应更多事件，最直接的办法通常就是增加更多线程。
3. 每增加一个线程，就增加了栈空间、调度开销和系统复杂度。

于是，系统往往会逐渐演化成“事件越多，线程越多”的形态，而这很快会变得昂贵、难维护，也难分析。

## 更多线程会进一步放大共享资源问题

线程数量一多，新的问题马上出现：这些线程很可能要访问相同的数据、外设或者其他资源。

课程里的 `blink_time` 就是最简单的例子：

1. `button` 线程负责写它。
2. `blinky` 线程负责读它。

这就是 shared-state concurrency。只要资源被多个线程共享，就会有 race condition 风险。

而线程切换本质上又和中断处理机制紧密相关，所以线程模型天然继承了与中断类似的问题：

1. 难以穷举所有竞态条件。
2. 一旦遗漏，可能直接导致系统错误。
3. 为了避免错误，就不得不引入各种 mutual exclusion 机制。

## mutex 虽然解决部分竞态，却会带来新的阻塞链条

为了解决共享变量的竞争，课程示例使用了 `mutex` 来保护 `blink_time`。但这其实只是把问题从“共享”转移到了“互斥和阻塞”。

因为 mutex 会进一步带来：

1. 额外 blocking。
2. 更复杂的时序关系。
3. 更困难的 timing analysis。
4. 更高的 missed deadline 风险。

也就是说，传统 RTOS 模型里常常会形成一个恶性循环：

1. 为了响应多个事件，不断增加线程。
2. 线程一多，就开始共享资源。
3. 共享资源引入竞态。
4. 为了解决竞态，再加入 mutex 等互斥机制。
5. 互斥又带来更多阻塞和时序不确定性。

## 经验法则：并发程序应尽量避免共享和阻塞

课程引用了 Herb Sutter 在并发编程方面的一组实践建议，用它们来承接从传统线程模型到 active object 的过渡。

这三条建议是：

1. 尽量把数据隔离到单个线程内部，不要共享。
2. 线程之间通过 asynchronous messages 通信。
3. 线程内部的工作围绕 message pump 组织。

这三条建议看似简单，但实际上正好对应着上一课讲过的事件驱动核心思想。

### 第一条：不要共享数据

如果数据尽量保持为“某个线程私有”，就不会出现 shared-state concurrency，也就不需要大量 mutex。

这条建议的本质，是让并发程序尽可能从“共享内存通信”转向“通过消息交互”。

### 第二条：通过异步消息通信

message 或 event 是专门为通信而设计的数据对象。它们表达“发生了什么”，必要时还可携带参数。

比如：

1. 按钮被按下。
2. 超时发生。
3. 收到一个以太网数据包。

更关键的是，这种通信必须是 asynchronous 的。也就是说，发送方只是把消息发给接收方，但不会阻塞等待对方处理完。

这条建议直接消除了“线程靠同步等待彼此协作”的那类阻塞链。

### 第三条：围绕 message pump 组织线程工作

这一条和上一课的 Win32 event loop 直接对应。线程内部不应该写成“处理一点业务，然后到处阻塞”等待不同事件”，而应该写成统一的消息循环：

1. 从队列取下一个事件。
2. 分派处理。
3. 返回消息循环。

这样 blocking 只允许发生在消息循环顶部，也就是“事件队列为空时等待下一个事件”这一处。

这正是课程反复强调的 non-blocking application-level code：应用逻辑本身不阻塞，只有框架级的等待队列操作可以阻塞。

## 这三条实践合在一起，就是 active object pattern

课程指出，上述三条最佳实践组合起来，形成的正是 active object pattern。

一个 active object 具备以下核心组成：

1. 私有数据。
2. 私有线程。
3. 私有事件队列。
4. 通过异步事件与外界交互。

外部代码与 active object 的交互方式只有一种：向它的事件队列 post event。

事件被 post 之后，发送方不会等待处理完成；而接收方会在自己的 event loop 中按顺序处理这些事件。

这使 active object 具有非常强的封装性。Samek 甚至认为，它才是“并发语境下真正的封装”。

## 为什么说 active object 才是真正的并发封装

课程这里做了一个非常重要的观点澄清：传统 OOP 语言里的 `private` 并不能自动解决并发问题。

即使一个对象的成员变量在语法上是 private：

1. 如果多个线程都能调用这个对象的方法，
2. 那么这些 private 数据依然可能被并发访问，
3. 依然可能出现 race condition。

所以常规 OOP 的 encapsulation，在并发意义上并不是真正封装。要想线程安全，仍然需要 mutex、monitor 等机制。

而 active object 不同。它的私有数据只允许在自己的线程里访问，因此天然不会被其他线程直接并发读写。

也就是说：

1. 它不依赖语言关键字来保证并发安全。
2. 它依赖的是“不共享，只发消息”的设计纪律。
3. 这样可以在不使用 mutex 的前提下，获得更强的并发封装。

## 在传统 RTOS 之上实现一个最小 active object 层

讲完原理后，课程开始做一件非常有启发性的事：不是直接切到现成框架，而是自己在传统 RTOS 之上手工搭一个最小 active object 层，名字叫 `uC/AO`。

这一步的目的很明确：让你看到 active object 并不是某个神秘库独有的特性，而是一组可以自己实现的设计机制。

### 先定义事件和事件信号

在这个最小实现里，首先要有 event signal。它本质上是一个小整数，用枚举值表示不同事件类型。

其中会包含一些保留信号，例如：

1. `INIT_SIG`，在 active object 进入事件循环前派发一次。
2. 用户自定义信号区间的起始值。

然后定义 `Event` 基类结构，它至少包含 signal，但结构本身必须支持后续扩展，以便派生出携带额外参数的事件。

这一步其实是把前面 OOP 课程里的 inheritance 用回来了：事件本身也可以形成类层次。

```c
/* uc_ao.h */
typedef uint16_t Signal; /* event signal -- 本质是个小整数 */

enum ReservedSignals {
    INIT_SIG, /* 框架在进入事件循环前派发一次，用于初始化 */
    USER_SIG  /* 用户自定义信号从这里开始编号 */
};

/* Event 基类：只含 signal，派生类可附加参数 */
typedef struct {
    Signal sig;
    /* 派生事件类型在这里追加字段，例如按钮坐标、传感器数值等 */
} Event;

```

### 定义 Active 基类

接下来定义 `Active` 基类。它至少需要有三部分：

1. 表示内部线程身份的信息，例如优先级。
2. 私有事件队列。
3. 一个 `dispatch` 虚函数指针，用来处理事件。

这里沿用了上一课 OOP in C 的技巧：通过函数指针实现 virtual dispatch。

派生出来的具体 active object 会：

1. 继承 `Active` 基类。
2. 在自身结构体中加入私有数据。
3. 提供自己的 `dispatch()` 实现。

```c
/* uc_ao.h */
typedef struct Active Active; /* 前向声明 */

/* dispatch 函数指针类型：处理一个事件 */
typedef void (*DispatchHandler)(Active * const me, Event const * const e);

/* Active 基类 */
struct Active {
    INT8U thread;             /* 线程优先级（也作 uC/OS-II task ID） */
    OS_EVENT *queue;          /* 私有消息队列 */
    DispatchHandler dispatch; /* 虚函数指针：对应派生类的事件处理函数 */
    /* 派生类在此之后追加私有成员 */
};

/* Active 的三个核心服务 */
void Active_ctor(Active * const me, DispatchHandler dispatch);
void Active_start(Active * const me,
                  uint8_t prio,
                  Event **queueSto, uint32_t queueLen,
                  void *stackSto,   uint32_t stackSize,
                  uint16_t opt);
void Active_post(Active * const me, Event const * const e);

```

### Active 的三个基本操作

基础的 active object 服务至少包括：

1. constructor：初始化 active object。
2. start：创建内部线程并启动事件循环。
3. post：异步向该 active object 的队列投递事件。

这里最关键的是 `start()` 的实现方式。

## inversion of control 再次出现：所有 AO 线程都运行同一个事件循环函数

在传统 RTOS 里，通常每个线程都有自己专门的线程函数，应用层为每个任务写不同的入口。

但在 active object 层里，让所有 active object 的内部线程都运行同一个通用 event loop 函数。

区别只在于：

1. 每个线程拿到的 `me` 指针不同。
2. 每个 `me` 指向不同的 active object 实例。
3. 每个实例绑定着自己的 `dispatch()` 虚函数和私有队列。

这正是事件驱动里的 inversion of control：

1. 应用不再控制线程主循环结构。
2. 主循环结构由 active object 框架统一规定。
3. 应用只负责提供事件处理逻辑。

从这个角度看，active object 层对传统 RTOS 做的最重要改造，并不是“又包了一层 API”，而是把线程从任意 superloop，收敛成统一的 message pump。

```c
/* uc_ao.c — 所有 AO 共享同一个线程入口函数 */
static void Active_eventLoop(void *pdata) {
    Active *me = (Active *)pdata; /* 每个 AO 实例通过 me 区分 */

    /* 进入循环前先派发一次 INIT_SIG，给 AO 初始化机会 */
    static Event const initEvt = { INIT_SIG };
    (*me->dispatch)(me, &initEvt);

    /* "message pump"：唯一允许阻塞的地方在循环顶部 */
    while (1) {
        Event *e;
        INT8U err;

        e = OSQPend(me->queue, 0U, &err); /* BLOCKING! 等待下一个事件 */
        Q_ASSERT(err == 0U);

        (*me->dispatch)(me, e); /* NO BLOCKING! 处理事件，必须 RTC */
    }
}

```

> 注意两行注释的对比：`OSQPend` 是唯一的阻塞点；`dispatch` 调用时**绝对不允许**再次阻塞。
## AO 事件循环与 Win32 message loop 的对应关系

课程特别指出，active object 的 event loop 和上一课 Win32 的 message loop 本质上是同一种结构：

1. 先等待队列里出现下一个事件。
2. 取出事件。
3. 调用 `dispatch()` 处理。
4. 返回循环顶部。

队列为空时，事件循环可以阻塞等待；但一旦取到事件，`dispatch()` 内部就不允许再阻塞。事件处理必须遵循 Run-to-Completion。

这正是上一课“GUI 里不要在 `WndProc` 中调用 `Sleep()`”的嵌入式版本。

## 先把 button 线程改造成 active object

为了逐步迁移，课程没有一上来重写整个程序，而是先把相对简单的 `button` 线程改造成 `Button` active object。

### Button AO 的结构

`Button` 类继承 `Active`，但它本身没有额外私有数据，因此结构体中只需要保留基类成员。

然后实现 `Button_dispatch()`，用 `switch(sig)` 处理几个事件：

1. `INIT_SIG`：初始化蓝灯状态。
2. `BUTTON_PRESSED_SIG`：点亮蓝灯，并执行原来按钮按下时的逻辑。
3. `BUTTON_RELEASED_SIG`：熄灭蓝灯，并执行原来按钮释放时的逻辑。

这个转换过程本质上就是：把原来“阻塞等待 semaphore 后执行的代码”，改写成“收到某个事件后执行的代码”。

### 关键差别：dispatch 里不再等待事件

特别强调，事件驱动代码里最重要的一点是：等待事件不发生在 `dispatch()` 内部，而只发生在 event loop 顶部。

也就是说：

1. `dispatch()` 只处理当前事件。
2. 处理完成后立即返回。
3. 下一次等待由框架统一完成。

这和传统顺序线程“在处理逻辑里写一半，再阻塞等下一个条件”完全不同。

### BSP 如何和 Button AO 交互

改造成 active object 之后，BSP 不再通过 semaphore 唤醒 `button` 线程，而是向 `Button AO` post event。

课程这里还强调了一个封装细节：对外暴露的是指向 `Active` 基类的全局指针，而不是 `Button` 的具体类型。

这样其他模块只知道“这里有一个 active object 可以收事件”，但看不到它的私有数据和具体实现，封装边界更清晰。

```c
/* bsp.c — App_TimeTickHook: ISR 里向 AO post 事件，而不是 V 一个 semaphore */
void App_TimeTickHook(void) {
    /* ... 按钮去抖逻辑 ... */
    if ((tmp & BTN_SW1) != 0U) { /* SW1 状态有变化？ */
        if ((buttons.depressed & BTN_SW1) != 0U) { /* 按下 */
            static Event const buttonPressedEvt = { BUTTON_PRESSED_SIG };
            Active_post(AO_BlinkyButton, &buttonPressedEvt); /* 异步投递 */
        } else {                                             /* 释放 */
            static Event const buttonReleasedEvt = { BUTTON_RELEASED_SIG };
            Active_post(AO_BlinkyButton, &buttonReleasedEvt);
        }
    }
}

```

> `AO_BlinkyButton` 是 `Active *` 类型的全局指针，bsp.c 完全不知道 `BlinkyButton` 的内部结构。传统方案此处会调用 `OSSemPost()`，区别在于接收方不再靠信号量阻塞唤醒，而是把事件放进队列，由 event loop 异步处理。
## 再把 blinky 线程改造成 active object：关键难点是“时间也是事件”

把 `button` 改完以后，最有挑战的是 `blinky` 线程，因为它原来的核心逻辑依赖 `delay()` 这样的 blocking time wait。

在这里提出一个非常关键的事件驱动视角：任何 blocking wait 背后，其实都对应着某种事件，只是传统顺序代码没有显式把它表达出来。

对 time delay 来说，这个事件就是 time event。

## 时间事件：把“等待一段时间”改写成“未来收到一个 timeout event”

为了支持 `blinky AO`，课程在 active object 层中增加了 `TimeEvent` 类。

它继承自 `Event`，并额外包含：

1. 目标 active object 指针。
2. timeout 计数器。
3. interval 值，用于支持周期性超时。

其语义是：

1. 当 timeout 递减到 0 时，就向目标 active object post 一个 timeout 事件。
2. 如果 `interval` 为 0，则该事件是 one-shot。
3. 如果 `interval` 非 0，则该事件会自动重装，实现 periodic timeout。

这一步非常关键，因为它把传统 RTOS 里的 `delay()` 重新表述成了“由框架稍后投递一个 timeout message”。

```c
/* uc_ao.h — TimeEvent 类（继承自 Event） */
typedef struct {
    Event super;       /* "继承" Event 基类，第一个成员 */
    Active *act;       /* 超时后投递给哪个 AO */
    uint32_t timeout;  /* 倒计时计数器，0 表示未被 arm */
    uint32_t interval; /* 周期性超时的重装值，0 表示 one-shot */
} TimeEvent;

void TimeEvent_ctor(TimeEvent * const me, Signal sig, Active *act);
void TimeEvent_arm(TimeEvent * const me, uint32_t timeout, uint32_t interval);
void TimeEvent_disarm(TimeEvent * const me);
void TimeEvent_tick(void); /* 在 SysTick/tick ISR 中调用 */

```

`TimeEvent_arm` 的调用示例：

```c
/* 启动一次 one-shot 超时：经过 blink_time 个 tick 后投递 TIMEOUT_SIG */
TimeEvent_arm(&me->te, me->blink_time, 0U);

/* 启动周期性超时：每 100 tick 投递一次 */
TimeEvent_arm(&me->te, 100U, 100U);

```
## TimeEvent 的内部实现也暴露了框架层的并发问题

课程实现 `TimeEvent` 时，用了一个模块内的静态数组来保存系统中所有 time event 实例，并在每个 tick 中遍历处理。

这里顺带指出一个现实问题：active object 框架本身运行在传统 RTOS 之上，因此框架内部仍然需要处理一些并发问题。例如：

1. 构造 time event 时会修改全局注册表。
2. arm/disarm time event 时会修改共享的内部状态。
3. 这些操作可能和系统 tick ISR 并发发生。

因此这里仍然需要短临界区保护。

这说明一件很有意思的事：虽然应用层 active object 尽量消除了共享和 blocking，但如果它是搭建在传统 RTOS 之上的扩展层，框架内部仍然不可避免会碰到传统并发问题。只是这些复杂性被局限在框架内部，而不再扩散到整个应用。

```c
/* uc_ao.c — TimeEvent_tick，在每个 SysTick 中断里调用 */
static TimeEvent *l_tevt[10]; /* 全局注册表，保存所有 TimeEvent 实例 */
static uint_fast8_t l_tevtNum;

void TimeEvent_tick(void) {
    uint_fast8_t i;
    for (i = 0U; i < l_tevtNum; ++i) {
        TimeEvent * const t = l_tevt[i];
        Q_ASSERT(t); /* 每个注册的 TimeEvent 必须有效 */
        if (t->timeout > 0U) {          /* 已 arm？ */
            if (--t->timeout == 0U) {   /* 本 tick 到期？ */
                Active_post(t->act, &t->super); /* 向目标 AO post 事件 */
                t->timeout = t->interval;       /* interval=0: 自动 disarm */
            }
        }
    }
}

```

```c
/* bsp.c — uC/OS-II App_TimeTickHook，在 SysTick ISR 上下文中调用 */
void App_TimeTickHook(void) {
    TimeEvent_tick(); /* 推进所有时间事件 */
    /* ... 按钮去抖逻辑 ... */
}

```

## 用 Blinky AO 处理 timeout 事件时，需要显式保存状态

有了 `TimeEvent` 之后，`Blinky AO` 就可以通过 `TIMEOUT_SIG` 来驱动 LED 状态切换。但新的问题也出现了：

同样一个 `TIMEOUT_SIG` 到来时，程序必须知道“这次超时之后应该把 LED 打开，还是关闭”。

在顺序代码里，这个信息隐含在程序执行位置上；但在事件驱动代码里，处理函数每次都是重新进入的，因此必须显式保存状态。

课程的做法是，在 `Blinky AO` 的私有数据里增加一个布尔标志，例如 `is_led_on`，用来记录当前 LED 状态。

这说明 event-driven 编程的一个常见变化：

1. 原来顺序代码靠“当前位置”隐含保存的状态，
2. 改造成事件驱动后，
3. 往往需要转成显式状态变量，放进对象私有数据中。

这也正是后面状态机设计会变得很重要的原因。

```c
/* main.c — BlinkyButton AO：合并了 blinky + button 两个线程的功能 */
enum { INITIAL_BLINK_TIME = (OS_TICKS_PER_SEC / 4) };

typedef struct {
    Active super;        /* 继承 Active 基类（必须是第一个成员） */
    TimeEvent te;        /* 私有 TimeEvent，用于驱动 LED 闪烁 */
    bool isLedOn;        /* 显式状态变量：当前 LED 是否亮着 */
    uint32_t blink_time; /* 私有数据：闪烁间隔（tick 数），不再共享 */
} BlinkyButton;

static void BlinkyButton_dispatch(BlinkyButton * const me,
                                  Event const * const e) {
    switch (e->sig) {
        case INIT_SIG:             /* AO 启动时由框架发送一次 */
            BSP_ledBlueOff();
            /* fall-through 到 TIMEOUT_SIG，执行首次 arm */
        case TIMEOUT_SIG: {
            if (!me->isLedOn) {    /* LED 当前关着 → 打开 */
                BSP_ledGreenOn();
                me->isLedOn = true;
                TimeEvent_arm(&me->te, me->blink_time, 0U);
            } else {               /* LED 当前亮着 → 关闭 */
                BSP_ledGreenOff();
                me->isLedOn = false;
                TimeEvent_arm(&me->te, me->blink_time * 3U, 0U);
            }
            break;
        }
        case BUTTON_PRESSED_SIG: {
            BSP_ledBlueOn();
            me->blink_time >>= 1;  /* 按一次，频率提高一倍 */
            if (me->blink_time == 0U) {
                me->blink_time = INITIAL_BLINK_TIME; /* 防止变为 0 */
            }
            break;
        }
        case BUTTON_RELEASED_SIG: {
            BSP_ledBlueOff();
            break;
        }
        default: break;
    }
    /* 注意：整个函数里没有任何 blocking 调用 */
}

```
## INIT 事件的作用：给 active object 一个统一的初始化入口

`Blinky AO` 还需要一个地方来“启动第一次 timeout”，否则永远不会收到第一个超时事件。

课程这里利用了保留事件 `INIT_SIG`：在 active object 进入事件循环前，框架先派发一次 `INIT_SIG`。这样对象就可以在 `INIT_SIG` 中：

1. 设置 LED 初始状态。
2. 初始化内部状态变量。
3. arm 第一次 time event。

这一步对应 GUI 里的 `WM_CREATE`，本质上是事件驱动对象的统一初始化钩子。

## 把 Button AO 和 Blinky AO 合并，展示事件驱动代码的可组合性

课程最后一个练习非常有启发性，让你重新审视：为什么最开始会有两个线程，后来又变成两个 active object？

答案是：在传统顺序线程模型里，之所以要拆多个线程，往往只是因为一个 superloop 无法在 blocking 过程中同时保持对多个事件的响应。RTOS 通过多线程恢复了“可组合性”，但代价是线程数量、共享状态和互斥复杂度急剧上升。

而事件驱动代码本身是可组合的，因为它不在处理逻辑中阻塞。既然一个 active object 能在自己的 `dispatch()` 中响应多种事件，那么原本拆开的 `button` 和 `blinky` 功能就可以重新合并成一个 `BlinkyButton AO`。

## 合并后的最大收益：共享变量不再共享，mutex 可以彻底删除

把 `button` 和 `blinky` 合并到同一个 active object 后，原来的 `blink_time` 就不再是“被两个线程共享的全局变量”了，而可以直接变成该 active object 的私有成员。

这带来了几个直接收益：

1. 不再需要 mutex。
2. 不再有围绕 `blink_time` 的 race condition。
3. 不再有 mutex 带来的额外 blocking。
4. 时序更可预测，更适合 hard real-time。
5. 还节省了一个线程栈和一个事件队列缓冲区。

这一步非常能体现 active object 的价值：不是单纯把“线程函数改写成事件处理函数”，而是通过更高层的组织方式，把原本不得不拆开的功能重新组合起来，并顺势消除共享状态。

```c
/* main.c — 构造函数和启动代码 */

void BlinkyButton_ctor(BlinkyButton * const me) {
    /* 绑定 dispatch 虚函数（C 版 vtable） */
    Active_ctor(&me->super, (DispatchHandler)&BlinkyButton_dispatch);
    /* 注册 TimeEvent，超时信号为 TIMEOUT_SIG，目标 AO 是自身 */
    TimeEvent_ctor(&me->te, TIMEOUT_SIG, &me->super);
    me->isLedOn = false;
    me->blink_time = INITIAL_BLINK_TIME;
}

/* 静态分配线程栈和事件队列缓冲区 */
static OS_STK stack_blinkyButton[100];
static Event *blinkyButton_queue[10];
static BlinkyButton blinkyButton;

/* 对外只暴露 Active* 指针，隐藏具体类型 */
Active *AO_BlinkyButton = &blinkyButton.super;

int main() {
    BSP_init();   /* 初始化板级外设 */
    OSInit();     /* 初始化 uC/OS-II */

    BlinkyButton_ctor(&blinkyButton);
    Active_start(AO_BlinkyButton,
                 2U,                          /* 优先级 */
                 blinkyButton_queue,
                 sizeof(blinkyButton_queue)/sizeof(blinkyButton_queue[0]),
                 stack_blinkyButton,
                 sizeof(stack_blinkyButton),
                 0U);

    BSP_start();  /* 配置并使能中断 */
    OSStart();    /* 启动调度器，不会返回 */
    return 0;
}

```
## 这节课真正完成的是一次并发设计上的范式转变

课程最后总结得很直接：这一课不是在教你“怎么在 RTOS 上多包一层 API”，而是在推动一次从 1980 年代传统 RTOS 设计，到现代事件驱动 active object 设计的范式转移。

这次转变包含几个层面：

1. 从 blocking threads 转向 message-driven active objects。
2. 从 shared-state concurrency 转向 share-nothing + asynchronous events。
3. 从应用自己控制 superloop，转向 framework 统一控制 event loop。
4. 从大量依赖 RTOS 原生同步原语，转向把这些复杂性尽量封装进框架层。

特别提醒：传统 RTOS 当然可以拿来实现 event-driven framework，但它并不是最匹配的基础设施。因为传统 RTOS 提供的大量机制，本来就是围绕 blocking 和 shared-state concurrency 设计的，而这些东西在 active object 应用层里不仅帮助不大，甚至可能有害。

## 总结

34 课的核心，是把上一课在 GUI 中讲清楚的事件驱动思想，真正落到实时嵌入式系统的并发设计上。

这一课最重要的结论可以概括为：

1. 传统 RTOS 线程模型虽然常见，但会自然导向“更多线程、更多共享、更多互斥、更多阻塞”的复杂性循环。
2. 避免共享、使用 asynchronous messages、围绕 message pump 组织代码，是更稳健的并发设计实践。
3. 这三条实践合在一起，就形成了 active object pattern。
4. active object 通过私有线程、私有事件队列和异步事件通信，实现了比传统 OOP 更强的“并发封装”。
5. 在 RTOS 之上完全可以手工实现一个最小 active object 层，其核心就是统一 event loop、virtual dispatch、event queue 和 time event。
6. 把顺序线程改造成 active object 时，原来的 blocking wait 需要被重新表达为显式事件，例如 timeout event。
7. 事件驱动代码天然更容易组合，因此可以把原本为响应性而拆开的多个线程重新合并，并进一步消除共享状态和 mutex。

如果说第 33 课建立了事件驱动的基本直觉，那么第 34 课则第一次把这种直觉真正变成了嵌入式并发设计方法。它让你看到，active object 并不是对 RTOS 线程的小修小补，而是另一种更适合 reactive system 的组织方式。后续课程如果继续沿着这条线展开，你会发现状态机、事件队列和 active object 会逐渐组合成一套非常完整的方法论。