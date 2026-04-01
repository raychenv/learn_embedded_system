# 9 递归函数、多文件项目与栈帧嵌套

## 引言

第8课讲解了单个函数的栈帧管理。现实的应用程序通常包含多个函数的相互调用，其中一些函数可能是**递归的**（即函数调用自己）。递归函数引入了栈帧的**嵌套**现象，这在调试时可能让人困惑，但对理解栈的工作机制至关重要。

本课展示如何使用递归（以阶乘函数为例），讲解栈帧如何嵌套，以及如何将项目分解为多个源文件。这为后续课程（特别是第13课）中讨论的`.data`初始化提供了基础。

## 递归函数的基本概念

递归函数是指在其定义中调用自己的函数。最经典的例子是**阶乘**（Factorial）：

$$n! = \begin{cases} 1 & \text{if } n = 0 \\ n \times (n-1)! & \text{if } n > 0 \end{cases}$$

C语言实现：

```c
unsigned fact(unsigned n) {
    if (n == 0U) {
        return 1U;
    } else {
        return n * fact(n - 1U);
    }
}
```

这个函数：

1. **基础情况**（Base Case）：当 `n == 0` 时，返回1，不做进一步递归调用
2. **递归情况**（Recursive Case）：计算 `n * fact(n-1)`，即调用自己，参数更小

调用 `fact(5)` 会导致一系列嵌套调用：

```
fact(5)
  ← 需要 fact(4)
    ← 需要 fact(3)
      ← 需要 fact(2)
        ← 需要 fact(1)
          ← 需要 fact(0)
            ← 返回 1
          ← 返回 1*1 = 1
        ← 返回 2*1 = 2
      ← 返回 3*2 = 6
    ← 返回 4*6 = 24
  ← 返回 5*24 = 120
```

每一层调用都有自己的栈帧，参数 `n` 是独立的。

## 递归栈帧的嵌套

当执行 `fact(5)` 时，栈会经过以下变化。

**初始状态**（在 `main()` 中）：

```asm
main:
    MOV     R0, #5          ; 参数 n=5
    BL      fact            ; 调用 fact(5)
    ...
```

**第一层：fact(5) 的序言**

```asm
fact:
    CMP     R0, #0          ; if (n == 0)
    BEQ     base_case
    
    ; 递归情况
    PUSH    {R4, LR}        ; 保存寄存器和返回地址
```

栈状态（假设初始 SP = 0x20003000）：

```
地址        内容
0x20002FF8  R4 保存值
0x20002FFC  返回地址（指向 main 中的下一条指令）← SP
```

此时 R0 = 5，所以跳过基础情况，执行递归：

```asm
    MOV     R4, R0          ; R4 = n（保存原值）
    SUB     R0, R0, #1      ; n = n - 1 = 4
    BL      fact            ; 递归调用 fact(4)
```

**第二层：fact(4) 的序言**

再次 `BL` 时，R14（LR）被更新为新的返回地址（指向 `fact(5)` 中的后续指令）。这就是为什么 `fact(5)` 在序言中执行了 `PUSH {LR}`。

```asm
fact:                       ; 这次是 fact(4)
    CMP     R0, #0
    BNE     recursive_case
    
recursive_case:
    PUSH    {R4, LR}        ; 保存新的 LR 和 R4
    MOV     R4, R0          ; R4 = 4
    SUB     R0, R0, #1      ; R0 = 3
    BL      fact            ; 调用 fact(3)
```

栈现已为（SP = 0x20002FF4）：

```
地址        内容
0x20002FF4  R4 = 4（或之前值）
0x20002FF8  LR = 0x00008888（fact(5) 的返回）← SP
0x20002FFC  R4 = 5（保存值）
0x20003000  LR = 0x00001234（main 的返回）
```

**第三层：fact(3) 的序言**

```asm
fact:
    PUSH    {R4, LR}        ; SP←0x20002FF0
    MOV     R4, R0          ; R4 = 3
    SUB     R0, R0, #1      ; R0 = 2
    BL      fact            ; 调用 fact(2)
```

**继续到底层：fact(0)**

当参数最终变为0时：

```asm
fact:
    CMP     R0, #0          ; 条件满足
    BEQ     base_case
    
base_case:
    MOV     R0, #1          ; 返回值 = 1
    MOV     PC, LR          ; 直接返回（无 PUSH/POP，因为未调用其他函数）
```

此时 LR 包含 `fact(1)` 中 `BL fact` 的返回地址。直接返回，R0 = 1。

## 返回链的逐层展开

从 `fact(0)` 返回，R0 = 1：

**fact(1) 的后续**：

```asm
MUL    R0, R4              ; R0 = 1 * 1 = 1
POP    {R4, PC}            ; 恢复 R4，PC ← fact(2) 的返回地址
```

**fact(2) 的后续**：

```asm
MUL    R0, R4              ; R0 = 2 * 1 = 2
POP    {R4, PC}            ; 恢复，返回到 fact(3)
```

**...依次向上...**

**fact(5) 的后续**：

```asm
MUL    R0, R4              ; R0 = 5 * 24 = 120（fact(4) 的结果）
POP    {R4, PC}            ; 返回到 main()，R0 = 120
```

## 递归的危险：栈溢出

递归函数的重大风险是**栈溢出**。如果递归太深，栈会溢出 RAM 的边界。

例如，调用 `fact(10000)` 会产生10000个嵌套的栈帧，每个占用8字节（在上面的例子中）。总共需要 80KB 栈空间。如果只配置了 16KB，则会导致栈溢出。

这在第10课中详细讲述。

## 多文件项目的组织

实际项目中，函数通常分散在多个源文件中。例如，来自present lesson-09的源代码结构：

**main.c**:
```c
#include "delay.h"

unsigned fact(unsigned n);  // 函数声明（原型）

int main(void) {
    unsigned volatile x;
    
    x = fact(0U);
    x = 2U + 3U*fact(1U);
    (void)fact(5U);
    
    SYSCTL_GPIOHBCTL_R |= (1U << 5);
    SYSCTL_RCGCGPIO_R |= (1U << 5);
    
    GPIO_PORTF_AHB_DIR_R |= (LED_RED | LED_BLUE | LED_GREEN);
    GPIO_PORTF_AHB_DEN_R |= (LED_RED | LED_BLUE | LED_GREEN);
    
    GPIO_PORTF_AHB_DATA_BITS_R[LED_RED | LED_BLUE | LED_GREEN] = 0;
    GPIO_PORTF_AHB_DATA_BITS_R[LED_BLUE] = LED_BLUE;
    
    while (1) {
        GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = LED_RED;
        delay(1000000);
        GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = 0;
        delay(500000);
    }
    
    return 0;
}
```

**delay.c**（单独的源文件）:
```c
#include "delay.h"

void delay(int iter) {
    int volatile counter = 0;
    while (counter < iter) {
        ++counter;
    }
}
```

**delay.h**（头文件）:
```c
#ifndef DELAY_H
#define DELAY_H

void delay(int iter);  // 函数原型

#endif
```

**fact() 的实现**（可以在单独的 fact.c 中，或内联在 main.c）：

如果分离到 `fact.c`：

```c
unsigned fact(unsigned n) {
    if (n == 0U) {
        return 1U;
    } else {
        return n * fact(n - 1U);
    }
}
```

## 编译与链接过程

**编译阶段**：
1. 编译器分别编译 `main.c`, `delay.c` 为目标文件（`.o`）
2. 每个目标文件包含已编译的机器代码，但对外部符号的引用未解析

```
main.o：
    { 代码包含调用 delay() 和 fact() 的指令，但这些地址未知 }

delay.o：
    { delay() 的完整定义 }

fact.o：（如果分离）
    { fact() 的完整定义 }
```

**链接阶段**：
1. 链接器读取所有目标文件和库
2. **符号解析**：`main.o` 中对 `delay` 和 `fact` 的引用被映射到实际地址
3. **搬迁**（Relocation）：更新指令中的地址常数
4. **生成可执行文件**：将所有代码段和数据段按顺序连接

```
可执行文件:
    [Text]   main.o 代码 + delay.o 代码 + fact.o 代码
    [Data]   初始化全局变量
    [BSS]    未初始化全局变量
    调用指令中的地址现已计算出来
```

## IAR Embedded Workbench 工作流程

在 IAR 中（本课程使用的IDE），项目配置自动处理编译和链接：

```
lesson-09/
  ├── tm4c123-keil/
  │   ├── main.c
  │   ├── delay.c
  │   ├── delay.h
  │   └── [project.ewp]  # IAR 工作空间配置
```

**IDE 的工作流程**：

1. **项目文件**（.ewp）定义源文件列表和编译选项
2. **预处理**：展开 `#include` 指令
3. **编译**：生成 `.o` 目标文件
4. **链接**：整合所有目标文件
5. **调试符号**：生成 `.elf` 文件（包含调试信息）
6. **设备编程**：将 `.hex` 或 `.bin` 下载到 MCU

IDE 自动发现所有源文件，调用编译器和链接器，管理符号表。

## 函数原型的重要性

在 C 中，函数原型提示编译器**函数签名信息**：

```c
unsigned fact(unsigned n);  // 原型：参数类型和返回类型
```

如果缺少原型，编译器会做**假设**，通常导致问题：

```c
// 没有原型的调用（危险！）
int main(void) {
    x = fact(5);  // 编译器不知道 fact 的参数和返回类型
                  // 可能误解为 int 返回值类型等
}

unsigned fact(unsigned n) { ... }  // 定义在后面
```

这可能导致：

1. **参数类型不匹配**：编译器可能对 `5` 的解释与实际不符
2. **返回值误解**：假设 `int` 而非 `unsigned`
3. **栈帧大小计算错误**：参数和返回值的处理方式不同
4. **运行时崩溃**：例如，如果被调用函数期望 `unsigned` 但收到有符号值

现代编译器（带足够的警告级别，如 `-Wall -Wextra`）会标记这种错误。最佳实践是在头文件中声明所有公开函数。

## 头文件的结构与最佳实践

好的头文件组织：

```c
/* delay.h */
#ifndef DELAY_H
#define DELAY_H

/**
 * 延迟指定的循环次数。
 * 这是一个忙等待型延迟，阻塞CPU。
 *
 * 注意：延迟时间取决于CPU时钟频率且不精确。
 * 仅用于演示目的。
 *
 * @param iter 循环迭代次数
 */
void delay(int iter);

#endif  /* DELAY_H */
```

**头文件保护**（Header Guard）：

```c
#ifndef DELAY_H
#define DELAY_H

// 头文件内容

#endif  /* DELAY_H */
```

这防止了多个包含（inclusion）时的重复定义。另一种现代方式是使用 `#pragma once`，但跨平台兼容性略差。

## 递归与迭代的权衡

考虑迭代实现的阶乘：

```c
unsigned fact_iter(unsigned n) {
    unsigned result = 1U;
    while (n > 0U) {
        result *= n;
        --n;
    }
    return result;
}
```

对比递归版本：

| 特性 | 递归 | 迭代 |
|------|------|------|
| 代码可读性 | 高（符合数学定义） | 中等 |
| 栈使用 | $O(n)$ | $O(1)$ |
| 执行效率 | 低（函数调用开销） | 高 |
| 调试难度 | 高（栈帧嵌套） | 低 |
| 实现复杂性 | 低 | 低 |

在嵌入式系统中，由于栈空间有限，迭代通常优先。但对于自然递归结构（如树遍历），递归仍然是合适的。有些编译器支持**尾部递归优化**（Tail Recursion Optimization），在某些情况下可以将递归转换为迭代形式，节省栈空间。

## 栈帧的可视化调试

在 IAR Debugger 中，调试器提供了**栈视图**（Stack View）和**调用栈视图**（Call Stack View）：

**Call Stack View** 显示：

```
fact(0)     ← 当前执行位置
  called by fact(1) at main.c line 42
  called by fact(2) at main.c line 42
  called by fact(3) at main.c line 42
  called by fact(4) at main.c line 42
  called by fact(5) at main.c line 42
  called by main() at main.c line 20
```

这直观地展示了嵌套的栈帧链。

**Stack View** 显示内存中的栈内容：

```
地址        内容        描述
0x20002FF0  R4_value    fact(0) 的栈帧
0x20002FF4  LR_value_1
0x20002FF8  R4_value    fact(1) 的栈帧
0x20002FFC  LR_value_2
...
```

调试器可以导航到不同的栈帧，查看局部变量和寄存器值。

## 整个项目的编译流程示例

对于 lesson-09，完整的编译过程：

```bash
# 1. 预处理
cpp main.c -o main.i          # 展开 #include, #define 等

# 2. 编译
arm-none-eabi-gcc main.i -c -o main.o
arm-none-eabi-gcc delay.c -c -o delay.o

# 3. 链接
arm-none-eabi-ld main.o delay.o -o lesson09.elf \
  -T linker_script.ld           # 使用链接脚本
  
# 4. 生成镜像文件
arm-none-eabi-objcopy lesson09.elf -O hex lesson09.hex
arm-none-eabi-objcopy lesson09.elf -O binary lesson09.bin
```

IAR 自动完成这些步骤，用户只需点击"Build"按钮。

## 项目增长的管理

随着项目规模扩大，文件组织变得关键：

```
firmware/
  ├── src/
  │   ├── main.c
  │   ├── delay.c
  │   ├── led.c
  │   └── timer.c
  ├── inc/
  │   ├── delay.h
  │   ├── led.h
  │   └── timer.h
  ├── lib/
  │   ├── CMSIS/
  │   └── peripheral/
  └── linker/
      └── tm4c123.ld
```

**源文件组织**：

- `main.c`：程序入口和高级逻辑
- `delay.c`：延迟函数实现
- `led.c`：LED控制函数
- `timer.c`：定时器驱动

**头文件组织**：

- Separate `inc/` 目录存放所有头文件
- 每个 `.c` 文件有对应的 `.h` 文件

**库文件**：

- 标准库（CMSIS、PeripheralLib）在 `lib/` 目录
- IAR 工作空间配置引用这些库

这种组织方式促进了**代码重用**、**并行开发**和**维护**。

## 小结与知识体系的构建

关键要点：

1. **递归栈帧**是普通栈帧的嵌套应用
2. **每层递归**有自己的参数、局部变量和返回地址副本
3. **基础情况**必须存在，否则无限递归导致栈溢出
4. **多文件项目**需要通过编译和链接整合
5. **函数原型**是编译器类型检查的关键
6. **栈深度**的管理对于安全、可靠的嵌入式代码至关重要

这三课（7、8、9）共同构建了对函数调用、栈管理和递归的深入理解。在下一课（第10课）中，我们将看到当栈滥用时会发生什么——栈溢出如何导致灾难性故障，以及硬件异常机制如何捕获这些错误。这个知识链条最终在第13课链接到程序启动和数据初始化，闭合了从硬件启动到高级编程抽象的完整循环。

