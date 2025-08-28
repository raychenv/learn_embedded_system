# 22 RTOS 什么是实时操作系统

基于Miro Samek现代嵌入式系统编程课程第22课内容，本文将开始介绍实时操作系统（RTOS）的概念。
需要说明的是，Samek讲解的RTOS，特指RTOS中的实时内核（Real-Time Kernel）部分，也就是负责多任务调度的部分。不包括硬件抽象层、设备驱动、文件系统、网络协议栈等有时也被归为RTOS的组件。

## 什么是RTOS内核？

21课介绍了前后台基本架构。它既可以用顺序阻塞代码实现，也可以用事件驱动非阻塞代码实现。RTOS属于顺序架构。这里再一次放上前后台编程模型图，如下，图中可以清晰的找到RTOS属于的编程范式：

![programming model](./images/programming_patterns.drawio.svg)

22课中，编程挑战让LaunchPad板上的绿色LED和蓝色LED独立且同时闪烁。最直接的尝试可能是复制粘贴绿色LED的代码，改成蓝色LED和不同的延时。但编译和下载后会发现，两个LED确实都在闪烁，但不是同时，而是依次亮灭——先绿后蓝。这是顺序代码的必然结果。
```c
#include <stdint.h>
#include "bsp.h"

int main(void) {
    BSP_init();
    while (1) {
        BSP_ledGreenOn();
        BSP_delay(BSP_TICKS_PER_SEC / 4U);
        BSP_ledGreenOff();
        BSP_delay(BSP_TICKS_PER_SEC * 3U / 4U);
        
        BSP_ledBlueOn();
        BSP_delay(BSP_TICKS_PER_SEC / 2U);
        BSP_ledBlueOff();
        BSP_delay(BSP_TICKS_PER_SEC / 3U);
    }
    //return 0; // unreachable code
}
```
要让LED真正独立且同时闪烁，同时又保持简单的顺序结构，需要让两个后台循环同时运行。为此，可以创建两个主函数——`main_blinky1`和`main_blinky2`，每个都有自己的while(1)后台循环。在原始的`main()`函数中引用它们，以防止编译器优化掉未使用的代码。但要注意：如果直接顺序调用它们，编译器会发现第一个调用永远不会返回，从而优化掉第二个。为避免这种情况，可以用一个`volatile`变量来条件分支调用。对应的工程实现在lesson-22/tm4c123-keil，main.c代码如下：
```c
#include <stdint.h>
#include "bsp.h"

void main_blinky1() {
    while (1) {
        BSP_ledGreenOn();
        BSP_delay(BSP_TICKS_PER_SEC / 4U);
        BSP_ledGreenOff();
        BSP_delay(BSP_TICKS_PER_SEC * 3U / 4U);
    }
}

void main_blinky2() {
    while (1) {
        BSP_ledBlueOn();
        BSP_delay(BSP_TICKS_PER_SEC / 2U);
        BSP_ledBlueOff();
        BSP_delay(BSP_TICKS_PER_SEC / 3U);
    }
}

int main(void) {
    volatile uint32_t run = 0;
    BSP_init();
    if (run) {
        main_blinky1();
    } else {
        main_blinky2();
    }
    //return 0; // unreachable code
}
```

运行代码前，配置工程选项：
1. 关闭浮点硬件，以简化中断处理，编译的寄存器。
2. 同时确保堆大小非零，因为Keil调试器可能需要它来支持“semihosting”等功能。

在调试器中发现只有`main_blinky2()`在运行并闪烁蓝色LED。为了更好地理解机制，在`bsp.c`中`SysTick`中断处理函数结尾处设置断点（该中断每秒触发100次）。

断点命中后，打开Memory1视图，将其停靠在侧边，滚动到堆栈指针（SP）位置。参考TivaC数据手册中的中断堆栈帧布局。由于ARM堆栈向低地址增长，所以要倒过来看。会发现从顶部数第7项是程序计数器（PC），中断返回时会加载它。可以通过单步执行`BX lr`指令来验证这一点。
https://www.ti.com/tool/EK-TM4C123GXL

![TI arm exception model](./images/arm_exception_model.png)

现在来个小技巧：当`SysTick_Handler`再次命中时，把堆栈上的返回地址（PC项）改为`main_blinky1()`的地址（比如0x7C6）。然后单步执行中断返回指令，你会发现跳转到了`main_blinky1()`，绿色LED开始闪烁。示意图如下。这说明通过中断和堆栈操作可以在后台循环间切换。但这种方法**不合法**，在更复杂的程序中**不可靠**，后面会解释原因。
![simple switch](./images/switch_thread_simple.drawio.svg)

上述操作的验证了：  
1）CPU在多个循环间切换是可行的；  
2）可以用中断硬件实现上下文切换；  
3）这就是多任务的基本思想——让CPU在独立线程间切换执行。  
这个过程可以由实时操作系统内核（RTOS Kernel）自动完成。示意图如下，相当于在任务线程和CPU间加了一层（RTOS），让任务线程可以合理分享CPU计算资源。

![task switch](./images/context_switch.drawio.svg)

这样，RTOS内核可以被简单定义为：一种软件，它扩展了前后台模型，使得多个后台循环（称为线程或任务）能在单个CPU上运行。多线程或多任务就是频繁切换CPU上下文，让每个线程看起来都独占CPU。**注意**：线程本质上就是前后台模型中的后台循环。

## 手动修改返回地址（PC）的问题
再回到手动修改返回地址（PC）的问题。普通中断时，保存和恢复的是同一个线程的上下文。但如果你从`blinky1`切换到`blinky2`，实际上是恢复了`blinky1`的寄存器，却在执行`blinky2`，示意图如下。这显然不对。简单情况下能用，但复杂代码（用到更多寄存器）就会出错。
![task switch problem](./images/switch_thread_problem.drawio.svg)
解决办法是**为每个线程维护独立的堆栈**。在C语言中很简单：为每个线程分配一个`uint32_t`数组作为堆栈，并有一个指向栈顶的指针。初始化时让堆栈指针指向数组末尾（ARM堆栈向下增长）。然后，在主代码中，不直接调用线程函数，而是**伪造一个堆栈帧**，让它看起来像刚被抢占时的状态。包括设置xPSR（第24位为THUMB状态）和函数地址（强转为`uint32_t`）作为PC。

其他寄存器暂时无关紧要，可以填上易于调试的值。设置好两个线程的堆栈（`blinky1`和`blinky2`）后，让`main()`进入死循环，初始时什么都不执行。在调试器中检查堆栈和指针。然后在`SysTick`结尾设置断点，命中后手动把SP设为`sp_blinky1`，就切换到了绿色LED线程。运行后你会看到绿色LED闪烁。

要切换到`blinky2`，再次在`SysTick`中断下，**先保存当前SP到`sp_blinky1`**，再把SP设为`sp_blinky2`。继续运行，现在蓝色LED开始闪烁。你刚刚手动完成了一次**上下文切换**。可以反复切换。关键是，每次返回线程时，能从被抢占的地方继续执行，因为堆栈被完整保存了。示意图如下。
![task switch](./images/switch_thread_stack1.drawio.svg)

更新后的时序图显示了正确的上下文切换。寄存器都从各自线程的堆栈保存和恢复。但**还没完**。你的切换还没有保存所有CPU状态。ARM异常帧只保存易失性寄存器（R0–R3, R12, LR, PC, xPSR），**没有**保存非易失性寄存器R4–R11，这些也必须保存。

![task switch](./images/switch_thread_stack2.drawio.svg)

如果`blinky1`用到了R7但没保存，`blinky2`可能会覆盖它。解决方法：在`SysTick`结尾切换上下文前，保存R4–R11，恢复时再还原。这需要多做几步：每个线程堆栈多分配8个寄存器空间，切换时更新SP，并确保寄存器状态正确。

## 总结
手动管理很繁琐——这正是自动化的意义所在。下一篇将介绍如何自己构建RTOS内核，自动处理这些细节。本文的核心要点是：理解RTOS线程是什么，以及内核如何在它们之间切换CPU上下文。