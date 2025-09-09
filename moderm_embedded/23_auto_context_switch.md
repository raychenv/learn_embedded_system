# 嵌入式RTOS - 自动化上下文切换

Miro Samek博士在现代嵌入式系统编程课程23课中展示了如何**自动化上下文切换过程**，替换22课中手动完成的上下文切换过程。

22课中需要实现的功能：让 LaunchPad 开发板上的绿色和蓝色 LED **独立地闪烁**。无法在一个循环里完成这个任务，于是尝试让两个后台循环同时运行。具体来说，创建两个循环：`main_blinky1` 和 `main_blinky2`，然后尝试在它们之间切换 CPU 执行。

这其实就是 RTOS 的核心思想：**扩展前台/后台架构，能在一个 CPU 上运行多个后台循环（线程）**，并通过频繁切换 CPU 来制造“并发执行”的错觉。其中由 RTOS 管理的后台循环称为“线程（thread）”或“任务（task）”，而线程之间的切换称为“上下文切换（context switch）”。示意图如下：

![context_switch](./images/context_switch.drawio.svg)

为了演示上下文切换过程，Miro Samek通过MiROS（Minimal Real-time Operating System）的开发展示了上下文切换的简单实现。为什么要构建MIROS呢？
- 构建一个最小但可运行的 RTOS 内核比去逆向学习已有项目（如 FreeRTOS）更能加深理解
- 真实的 RTOS 内核非常复杂，包含许多初学者一时难以理解的特性，容易忽视整体架构而陷入细节迷宫

## 1. 创建**MiROS** 模块
添加两个文件：
一个头文件`miros.h`，包含 RTOS 的 **应用程序接口（API）**
一个 C 文件`miros.c`，包含 RTOS 的完整实现
## 2. 定义线程控制块（TCB）
在头文件中定义线程的表示方式。每个线程至少需要一个私有的堆栈指针 `SP`，将来可能还会扩展更多信息。可以用一个 `struct OSThread` 来表示线程控制块（TCB），并用 `OS` 前缀以避免与大型项目中其他模块命名冲突。 
定义好 `OSThread` 后，你需要将 `main.c` 中原本的堆栈指针变量改为 `OSThread` 类型，同时包含 `miros.h`。
```c
typedef struct {
    void *sp; /* stack pointer */
    /* ... other attributes associated with a thread */
} OSThread;
```
### 3. 实现 **线程创建函数**（如传统 RTOS 中的 thread\_create 或 thread\_start）
这里线程创建函数命名为 `OSThread_start()`。它接收以下参数：
* 指向线程控制块（TCB）的指针 `me`（我们将在面向对象 C 编程中解释此命名）
* 指向线程处理函数的函数指针（需要提前 `typedef`，我命名为 `OSThreadHandler`）定义 `OSThreadHandler` 为“无参数、返回 void 的函数指针类型”
* 一个堆栈缓冲区的指针
* 堆栈的大小
```c
void OSThread_start(
    OSThread *me,
    OSThreadHandler threadHandler,
    void *stkSto, uint32_t stkSize);
```
在 `miros.c` 实现文件中，包含 `stdint.h` 和 `miros.h`，实现OSThread_start函数逻辑：
 * 初始化堆栈指针从堆栈内存的高地址开始（因为 ARM Cortex-M 的堆栈向下增长）
 * 将堆栈指针对齐到 8 字节边界（用整数除法和乘法）
 * 构造初始堆栈帧，包括 PC、xPSR 等
 * 将线程的处理函数写入堆栈帧
 * 将最终的 `SP` 写入 `OSThread` 的 `sp` 字段
 * 可以用 `0xDEADBEEF` 预填充堆栈，方便调试时分析最大使用量
```c
void OSThread_start(
    OSThread *me,
    OSThreadHandler threadHandler,
    void *stkSto, uint32_t stkSize)
{
    /* round down the stack top to the 8-byte boundary
    * NOTE: ARM Cortex-M stack grows down from hi -> low memory
    */
    uint32_t *sp = (uint32_t *)((((uint32_t)stkSto + stkSize) / 8) * 8);
    uint32_t *stk_limit;

    *(--sp) = (1U << 24);  /* xPSR */
    *(--sp) = (uint32_t)threadHandler; /* PC */
    *(--sp) = 0x0000000EU; /* LR  */
    *(--sp) = 0x0000000CU; /* R12 */
    *(--sp) = 0x00000003U; /* R3  */
    *(--sp) = 0x00000002U; /* R2  */
    *(--sp) = 0x00000001U; /* R1  */
    *(--sp) = 0x00000000U; /* R0  */
    /* additionally, fake registers R4-R11 */
    *(--sp) = 0x0000000BU; /* R11 */
    *(--sp) = 0x0000000AU; /* R10 */
    *(--sp) = 0x00000009U; /* R9 */
    *(--sp) = 0x00000008U; /* R8 */
    *(--sp) = 0x00000007U; /* R7 */
    *(--sp) = 0x00000006U; /* R6 */
    *(--sp) = 0x00000005U; /* R5 */
    *(--sp) = 0x00000004U; /* R4 */

    /* save the top of the stack in the thread's attibute */
    me->sp = sp;

    /* round up the bottom of the stack to the 8-byte boundary */
    stk_limit = (uint32_t *)(((((uint32_t)stkSto - 1U) / 8) + 1U) * 8);

    /* pre-fill the unused part of the stack with 0xDEADBEEF */
    for (sp = sp - 1U; sp >= stk_limit; --sp) {
        *sp = 0xDEADBEEFU;
    }
}
```
在 `main.c` 中调用 `OSThread_start()` 来启动 blinky1 和 blinky2 两个线程。

### 4. 实现上下文切换 - PenSV配置
切换必须发生在**中断返回阶段**，通常选用 `SysTick` 中断。但如果在每个中断ISR中都手写切换代码将非常重复，并且破坏 ARM Cortex-M 中断ISR可用 C 编写的优点，因为上下文切换需要操作CPU寄存器，必须用汇编实现。
ARM Cortex-M 提供了一个专门用于上下文切换的中断：**PendSV**。几乎所有 Cortex-M RTOS 都使用它来实现上下文切换。可以通过写入 **ICSR 寄存器的 bit 28** 来触发 PendSV。

![context_switch](./images/context_switch_intr.drawio.svg)

为了保证SysTick中断服务不被PendSV嵌套中断，如上图所示。设置优先级（配置中断优先级寄存器SYSPRI3）：SysTick 设置为 0（高），PendSV 设置为 E0（低） —— 注意 ARM 的优先级数值越低，优先级越高。代码实现如下：
```c
void OS_init(void) {
    /* set the PendSV interrupt priority to the lowest level 0xFF */
    *(uint32_t volatile *)0xE000ED20 |= (0xFFU << 16);  // 0xE000ED20 is SYSPRI3
}
```

### 5. 实现上下文切换 - 调度函数 `OS_sched()`
调度算法主要有两个功能，一是决定下一个可运行的线程或者任务，另外一个功能是进行上下文切换。
Miro Samek侧重上下文切换的实现。下一个可运行的线程或者任务是人工选择的。实现如下：
```c
void OS_sched(void) {
    /* OS_next = ... */
    OSThread const *next = OS_next; /* volatile to temporary */
    if (next != OS_curr) {
        *(uint32_t volatile *)0xE000ED04 = (1U << 28);
    }
}
```
1. 全局变量 `OS_curr` 指向当前线程，`OS_next` 指向下一个线程，均为 `volatile OSThread*`
2. 判断是否需要切换：如果 `OS_next != OS_curr`，则触发 PendSV（设置 bit 28）
3. 为了避免竞争条件（race condition），调度器应在临界区中调用。可以在 `SysTick_Handler()` 中调用 `OS_sched()` 并用 `__disable_irq()` 和 `__enable_irq()` 包围。

### 6. 实现上下文切换 - 实现 PendSV_Handler
PendSV_Handler流程大致为：
1. 禁用中断
2. 如果当前线程不为 NULL，则将 r4–r11 入栈，并保存 SP
3. 加载 OS\_next 的 SP 到当前 SP
4. 设置 OS\_curr = OS\_next
5. 恢复 r4–r11，启用中断，返回
实现如下：
```c
__attribute__ ((naked))
void PendSV_Handler(void) {
__asm volatile (
    /* __disable_irq(); */
    "  CPSID         I                 \n"

    /* if (OS_curr != (OSThread *)0) { */
    "  LDR           r1,=OS_curr       \n"
    "  LDR           r1,[r1,#0x00]     \n"
    "  CMP           r1,#0             \n"
    "  BEQ           PendSV_restore    \n"

    /*     push registers r4-r11 on the stack */
    "  PUSH          {r4-r11}          \n"

    /*     OS_curr->sp = sp; */
    "  LDR           r1,=OS_curr       \n"
    "  LDR           r1,[r1,#0x00]     \n"
    "  MOV           r0,sp             \n"
    "  STR           r0,[r1,#0x00]     \n"
    /* } */

    "PendSV_restore:                   \n"
    /* sp = OS_next->sp; */
    "  LDR           r1,=OS_next       \n"
    "  LDR           r1,[r1,#0x00]     \n"
    "  LDR           r0,[r1,#0x00]     \n"
    "  MOV           sp,r0             \n"

    /* OS_curr = OS_next; */
    "  LDR           r1,=OS_next       \n"
    "  LDR           r1,[r1,#0x00]     \n"
    "  LDR           r2,=OS_curr       \n"
    "  STR           r1,[r2,#0x00]     \n"

    /* pop registers r4-r11 */
    "  POP           {r4-r11}          \n"

    /* __enable_irq(); */
    "  CPSIE         I                 \n"

    /* return to the next thread */
    "  BX            lr                \n"
    );
```

## 验证

* 手动设置 OS\_next 为 blinky1，触发上下文切换
* 验证进入 main\_blinky1，LED 变绿
* 再次手动设置 OS\_next 为 blinky2，LED 变蓝

上下文切换成功！现在调度仍是手动的，课程的24课自然过渡到了自动调度算法之一——轮转调度（Round-Robin）。