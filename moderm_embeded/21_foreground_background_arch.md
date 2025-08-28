# 嵌入式RTOS之前后台架构

为了理解RTOS的设计思想，花了一些时间看了Miro Samek发布的*现代嵌入式系统编程*培训课程0～28课（共55课）。
- 系列课程可以通过[github代码库](https://github.com/QuantumLeaps/modern-embedded-programming-course)访问到，是英文的。
- 0～21讲了嵌入式编程的基础知识，包括嵌入式硬件架构、C语言、编译加载、ARM启动过程、中断等等。
- 22～28课讲解了RTOS基本结构和实现。
- 教学硬件是TI的[TivaC LaunchPad](https://www.ti.com/tool/EK-TM4C123GXL)开发板（EK-TM4C123GXL 核心架构**ARM Cortex-M4F**）。
- 软件开发环境支持了[KEIL MDK](https://www.keil.com)、[IAR Embedded Workbench for ARM (EWARM)](https://www.iar.com)和[Code Composer Studio (CCS)](https://www.ti.com/tool/CCSTUDIO)。

这是一个庞大而有趣的主题。我想整理出其中的要点，首先是21课（Foreground-Background Architecture）-- 前后台架构。前后台架构，也称为*超级循环*或*主函数+中断*。这种架构设计无处不在，是所有嵌入式软件架构的起点，也是理解实时操作系统（RTOS）的重要基础。

## 前后台架构 -- 阻塞版本
跳过课程里面[KEIL MDK](https://www.keil.com)软件配置部分，我们直接来到样例代码：
```c main
#include <stdint.h>
#include "bsp.h"

int main(void) {
    BSP_init();
    while (1) {
        BSP_ledGreenOn();
        BSP_delay(BSP_TICKS_PER_SEC / 4U);
        BSP_ledGreenOff();
        BSP_delay(BSP_TICKS_PER_SEC * 3U / 4U);
    }

    //return 0;
}
```
```c bsp.cpp

static uint32_t volatile l_tickCtr;

/* ISRs  ===============================================*/
void SysTick_Handler(void) {
    ++l_tickCtr;
}

uint32_t BSP_tickCtr(void) {
    uint32_t tickCtr;

    __disable_irq();
    tickCtr = l_tickCtr;
    __enable_irq();

    return tickCtr;
}

void BSP_delay(uint32_t ticks) {
    uint32_t start = BSP_tickCtr();
    while ((BSP_tickCtr() - start) < ticks) {
    }
}
```
这是一段常见的LED闪烁程序，完整的项目在[这里](https://github.com/QuantumLeaps/modern-embedded-programming-course/blob/main/lesson-21/tm4c123-keil)。加载到板子上，绿色LED闪烁--亮约1/4秒，灭约3/4秒。`BSP_delay()`基于`SysTick`中断实现精确计时。该中断以`BSP_TICKS_PER_SEC`（每秒100次）为频率触发，这个值在`bsp.h`中声明。 
`SysTick_Handler`会递增一个本地静态且`volatile`的变量`l_tickCtr`。`volatile`关键字告诉编译器该变量可能会被中断等意外修改。`BSP_tickCtr()`函数返回`l_tickCtr`的当前值，并且为了避免竞争条件，这个访问是在关闭中断的临界区内完成的。 
`BSP_delay()`函数将当前tick计数读入本地变量`start`，然后进入一个轮询循环，直到经过指定tick数。二进制补码运算可以无缝处理tick计数器回绕。然而，`BSP_delay()`是一个*阻塞*函数，会在循环中浪费CPU周期。 
不过，关键点是程序架构，它体现了*前台/后台架构*——在小型嵌入式系统中很常见。如下图所示，这种架构由两部分组成：`main()`中的后台循环和前台的中断服务程序（如`SysTick_Handler`）。前台代码会抢占后台代码，然后返回到被抢占的确切位置。两者通过共享变量通信，这些变量必须是`volatile`的，并在临界区访问以防止竞争条件。 

![priority time graph](./images/foreground_background.drawio.svg)

这种架构的一个问题是时序：后台循环中的执行时序不可预测，因为执行路径和中断活动会变化。对时序要求严格的操作应放在前台，而不是后台。但这样会增加中断的复杂性和持续时间，可能导致中断之间以及与后台的相互干扰。

![timing issue](./images/foreground_backgound_timing_issue.drawio.svg)

尽管有这些权衡，前台/后台架构在许多高出货量嵌入式应用中仍然很流行——如消费电子、家用电器、玩具、遥控器等。它也被Arduino等平台采用。虽然Arduino通过库隐藏了架构，但可以在主函数中找到它，主函数通过`setup()`初始化硬件，然后在无限循环中调用`loop()`。`for(;;)`和`while(1)`等价，前台的ISR（如系统时钟tick）与这里讨论的嵌入式结构类似。

这个LED闪烁程序大部分时间都在做什么呢？进入调试模式中断程序，会发现代码停在`BSP_tickCtr()`——它是被`BSP_delay()`调用的，而`BSP_delay()`又是被`main()`调用的。实际上，程序几乎所有时间都在等待。在流程图中，这些轮询循环用回退箭头表示，并标记为*阻塞*和*顺序*。
![flow chart](./images/sequence_blocking.drawio.svg)

代码之所以*阻塞*，是因为它在行内等待某个事件（如超时），在事件发生前不会继续执行。它是*顺序*的，因为期望的事件序列被硬编码在指令顺序中。例如，程序期望在点亮绿色LED后延时1/4秒，熄灭后延时3/4秒。

## 前后台架构 -- 非阻塞版本
也可以让代码*非阻塞*，不使用轮询循环，程序片段如下。调试时，LED依然会闪烁，但此时中断程序会发现总是在主循环中。
```c
/* background code: non-blocking version */
int main(void) {
    BSP_init();
    while (1) {
        /* Blinky polling state machine */
        static enum {
            INITIAL,
            OFF_STATE,
            ON_STATE
        } state = INITIAL;
        static uint32_t start;
        switch (state) {
            case INITIAL:
                start = BSP_tickCtr();
                state = OFF_STATE; /* initial transition */
                break;
            case OFF_STATE:
                if ((BSP_tickCtr() - start) > BSP_TICKS_PER_SEC * 3U / 4U) {
                    BSP_ledGreenOn();
                    start = BSP_tickCtr();
                    state = ON_STATE; /* state transition */
                }
                break;
            case ON_STATE:
                if ((BSP_tickCtr() - start) > BSP_TICKS_PER_SEC / 4U) {
                    BSP_ledGreenOff();
                    start = BSP_tickCtr();
                    state = OFF_STATE; /* state transition */
                }
                break;
            default:
                //error();
                break;
        }
    }
    //return 0;
}
```
非阻塞版本的流程图只有前进箭头——没有阻塞。主循环每秒可以运行几十万次，因此能立即响应到来的事件。这种结构是*事件驱动*的。但这种灵活性也带来代价：复杂度增加，因为事件序列不再固定在代码顺序中。  
![event driven](./images/event_driven.drawio.svg)

在本例中，非阻塞结构是通过轮询*状态机*实现的。这并不常见。在大多数实际项目中，非阻塞逻辑往往会退化成混乱、深层嵌套的`if-else`语句和纠缠的代码——通常被称为“意大利面条代码”或“泥球”。

## 总结
本文在21课的基础上，介绍了**前后台架构**，这是理解所有嵌入式软件架构的基础。它既可以用顺序/阻塞范式实现，也可以用事件驱动/非阻塞范式实现。这里一张图各种编程范式及其联系！
![programming diagram](./images/programming_patterns.drawio.svg)