# 7 数组、指针算术与按位GPIO访问

## 引言

在嵌入式系统编程中，数组和指针算术是实现高效代码的核心概念。特别是在GPIO编程领域，现代微控制器芯片（如TivaC系列和STM32）提供了一种优雅的硬件设计方式——**按位屏蔽寄存器**（bit-masking register）。这种设计允许程序员通过简单的数组语法对单个GPIO引脚进行操作，而无需进行复杂的位掩码计算。

本课程重点讲解此工作原理，深入剖析ARM汇编代码如何实现数组索引寻址，以及这种语法糖背后的硬件机制。

lesson 7 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-07

## 数组的本质：内存顺序访问

在C语言中，**数组本质上是一块连续的内存区域**。当我们声明：

```c
unsigned int data[10];
```

编译器会为这10个`unsigned int`元素分配连续的内存。由于每个`unsigned int`占用4个字节（在32位ARM上），这10个元素占用总共40字节。

**关键原理**：数组元素 `data[i]` 的内存地址可以通过以下公式计算：

addr(data_i) = addr(data_0) + i * sizeof(elem)

## 指针算术的硬件实现

当我们访问 `data[i]` 时，编译器进行的操作包括：

1. **计算实际地址**：`base_address + i * element_size`
2. **生成ARM汇编指令**：使用寄存器间接寻址模式
3. **执行内存加载或存储操作**

以下是一个具体例子。若基址为R0，索引为R1（假设元素大小为4字节），访问 `data[i]` 需要：

```asm
LDR   R2, [R0, R1, LSL #2]    ; R2 = data[i], R1左移2位（×4）再加到R0
STR   R2, [R0, R1, LSL #2]    ; data[i] = R2
```

这里 `[R0, R1, LSL #2]` 表示：以R0为基址，加上R1左移2位（即R1×4）后的值作为偏移量。

## GPIO寄存器的位屏蔽技巧

TivaC微控制器的GPIOF模块的数据寄存器在高速AHB总线上实现了一个特殊的技巧。虽然从C代码的视角看起来我们在访问数组：

```c
GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = LED_RED;
```

但实际上这是一种**聪明的硬件设计**，利用了GPIO数据寄存器的位屏蔽原理。

### 硬件层面的原理

GPIOF的AHB接口包含：

- **基础寄存器地址**：如 `0x40025000`（GPIO Port F基址）
- **数据寄存器**：偏移量为 `0x000`
- **位屏蔽技巧**：寄存器的高8位用作"掩码"，低8位是实际的数据

当处理器向地址 `(base + mask) << 2` 写入数据时，硬件解释为：
- 低8位是要写入的数据
- 高8位（通过左移2位后作为地址偏移）决定哪些位有效

例如，要点亮红LED（对应位1），我们希望只修改位1，保持其他位不变。通过设置掩码为 `LED_RED`（值为2，二进制 `00000010`）：

```c
GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = LED_RED;  // 掩码 = LED_RED = 2
```

硬件层面的操作流程：

1. 开发者编写 `array[2] = 2`（如果 `LED_RED = 2`）
2. 编译器生成：`address = gpio_base + 2*4 = gpio_base + 8`
3. 处理器访问内存地址 `gpio_base + 8`
4. GPIO硬件单元识别出：掩码值为2，因此只允许修改位1
5. 写入的值来自寄存器的低8位

这种设计的优越性在于：**无需软件执行读-修改-写操作**。传统方式需要：

```c
// 低效的传统方式（需要硬件掩码技巧时）
unsigned char current = GPIO_PORTF_AHB_DATA_R;  // 读
current |= LED_RED;                             // 修改（可能影响其他位）
GPIO_PORTF_AHB_DATA_R = current;                // 写
```

相比之下，数组方式是原子的（atomic），且更简洁。

## 数组语法到汇编代码的转换

让我们分析实际代码如何转换：

```c
#define LED_RED   (1U << 1)  // 值为 2
#define LED_BLUE  (1U << 2)  // 值为 4
#define LED_GREEN (1U << 3)  // 值为 8

GPIO_PORTF_AHB_DATA_BITS_R[LED_RED | LED_BLUE | LED_GREEN] = 0;
GPIO_PORTF_AHB_DATA_BITS_R[LED_BLUE] = LED_BLUE;
GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = LED_RED;
```

第一行的编译过程：

1. **计算索引**：`LED_RED | LED_BLUE | LED_GREEN = 2 | 4 | 8 = 14`（二进制 `00001110`）
2. **计算地址偏移**：`14 << 2 = 56`
3. **生成汇编**（假设R0持有GPIO基址）:
   ```asm
   MOV    R1, #56        ; 偏移量（14*4）
   MOV    R2, #0         ; 要写入的值
   STRB   R2, [R0, R1]   ; 向 base+56 写入0，硬件掩码为14
   ```

GPIO硬件在收到对地址 `(base + 56)` 的写操作时：
- 提取高位作为掩码：14 → `00001110` → 表示要修改LED引脚（位1、2、3）
- 使用低8位的值（0）覆盖这些位
- 结果：所有LED位被清零，其他GPIO位不受影响

## 数组与指针的等价性

C语言规定：**数组访问 `a[i]` 等价于指针解引用 `*(a + i)`**。

在GPIO编程中，如果定义指针：

```c
volatile unsigned char *gpio_bits = (volatile unsigned char *)(0x40025000 + 0x3FC);
```

则以下两种写法等价：

```c
// 数组方式
GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = LED_RED;

// 指针方式
*(gpio_bits + LED_RED) = LED_RED;
```

编译器会生成相同的汇编代码。这种等价性是C语言强大的灵活性体现。

## 数组初始化与内存布局

在嵌入式系统中，常常需要初始化数组以预设值。例如：

```c
unsigned int lookup_table[] = {10, 20, 30, 40, 50};
```

编译器会：

1. 在`.data`或`.rodata`段中保留5个连续的32位槽
2. 用相应的值预填充这些槽
3. 计算对应的符号`lookup_table`指向第一个元素

链接器在最终生成ELF文件时，会在`.data`段中写入这5个32位值（以小端格式，如果是ARM）。

## Thumb指令与数组寻址

在Thumb-2指令集（常用于ARM Cortex-M系列）中，寻址模式受限。例如，`LDRB` 指令的寻址模式包括：

```asm
LDRB   R0, [R1]           ; 无偏移
LDRB   R0, [R1, R2]       ; 寄存器偏移
LDRB   R0, [R1, #imm8]    ; 立即数偏移（仅8位）
```

对于在编译时无法确定的数组索引，编译器通常生成：

```asm
MOV    R2, R3, LSL #(log2 sizeof(elem))  ; R2 = 索引 * 元素大小
LDRB   R0, [R1, R2]                       ; 使用计算出的偏移
```

这演示了ARM的强大：虽然立即数寻址有限制，但寄存器间接寻址配合移位操作提供了灵活性。

## TivaC与STM32的异同

在实际项目中，不同MCU系列的GPIO访问方式略有不同：

### TivaC（LM4F系列）

TivaC完全支持位屏蔽寄存器数组访问方式：

```c
// 原子的、高效的
GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = LED_RED;
GPIO_PORTF_AHB_DATA_BITS_R[LED_RED] = 0;  // 关闭
```

底层原理是硬件的位屏蔽逻辑。这种方式在中断驱动的系统中特别安全。

### STM32C031

STM32的GPIO设计不使用相同的位屏蔽技巧。取而代之，STM32提供了原子的设置/清除寄存器（BSRR）：

```c
// 使用专用的设置/清除寄存器
GPIOA->BSRR = (1U << LD4_PIN);           // 设置（置1）
GPIOA->BSRR = (1U << (LD4_PIN + 16U));   // 清除（置0），利用高16位
```

尽管语法不同，但目的相同——**避免竞态条件**（race condition）。在多线程或中断驱动的系统中，直接读-修改-写会导致数据竞争。

实际上，STM32的BSRR寄存器结构为：

```
位[15:0]  - 设置掩码：写1表示将对应GPIO位设置为1
位[31:16] - 清除掩码：写1表示将对应GPIO位清除为0
```

设置位4:
```c
GPIOA->BSRR = (1U << 4);  // 位[4] = 1，对应GPIO4 → 1
```

清除位4:
```c
GPIOA->BSRR = (1U << (4 + 16));  // 位[20] = 1，对应 GPIO4 → 0
```

硬件确保这些操作是原子的，无需软件同步。

## 多维数组与指针

对于多维数组，指针算术变得更复杂：

```c
unsigned int matrix[3][4];  // 3行4列的矩阵
```

内存布局为**行优先**（Row-Major Order）：前4个元素是第一行，接下来4个是第二行，等等。

访问 `matrix[i][j]` 的实际地址：

addr = base + (i * cols + j) * sizeof(elem)

编译器能够根据数组声明信息自动计算这个公式。例如，访问`matrix[2][3]`:

```asm
MOV    R1, #2           ; i = 2
MOV    R2, #3           ; j = 3
ADD    R3, R1, ASL #2   ; R3 = 2 * 4 = 8（行偏移，列数为4）
ADD    R3, R2           ; R3 = 8 + 3 = 11
ADD    R3, R3, LSL #2   ; R3 = 11 * 4 = 44（转为字节偏移）
LDR    R0, [R0, R3]     ; 加载 matrix[2][3]
```

在指针形式中：

```c
unsigned int *p = &matrix[0][0];  // 指向第一个元素
unsigned int value = *(p + 2*4 + 3);  // 等价于 matrix[2][3]
```

或者通过指针数组：

```c
unsigned int *rows[3] = {matrix[0], matrix[1], matrix[2]};
unsigned int value = rows[2][3];  // 等价于 matrix[2][3]
```

## 数组越界与内存安全

C语言对数组访问**不进行边界检查**。访问 `array[10]` 当数组仅有10个元素时（有效索引为0-9）：

```c
unsigned int array[10];
unsigned int x = array[10];     // undefined behavior!
array[-1] = 0;                  // undefined behavior!
```

这会导致未定义行为（UB）。在嵌入式系统中，这通常会：

1. 读/写到相邻的数据结构
2. 读/写到栈上的其他变量（包括返回地址）
3. 导致硬件地址异常（如果越界访问超出RAM范围）

这种行为也正是课程后期（特别是第10课）中讨论的栈溢出问题的根源。

## 指针运算的风险

当使用指针时，某些运算可能导致问题：

```c
volatile unsigned char *gpio = (volatile unsigned char *)0x40025000;
volatile unsigned char *out_of_bounds = gpio + 0x1000000;  // 超出地址空间
```

在32位ARM系统中，地址空间有限（通常映射为4GB）。写入超出范围的地址会导致：

- **公线（Bus Fault）异常**：CPU无法完成内存访问
- **栈损坏**：如果写入了栈区域
- **代码段修改**：如果写入了代码段（虽然通常受保护）

硬件保护机制（如MPU——Memory Protection Unit，在Cortex-M4+上可用）可以捕获某些这样的错误并触发异常。

## 小结：从数组到硬件

本课关键要点：

1. **数组本质**是连续内存加上编译器计算索引和偏移
2. **指针算术**直接映射到ARM汇编中的寻址模式
3. **GPIO位屏蔽**是硬件设计的优雅应用，避免了竞态条件
4. **不同MCU**可能采用不同的原子操作方式（TivaC的DATA_BITS vs STM32的BSRR）
5. **数组越界没有编译时检查**，导致运行时隐患

这些知识为后续课程奠定基础——第8课将深入讨论函数调用时的栈管理，这与数组和指针密切相关。在构建复杂系统时，理解这些低层机制能够帮助我们写出既高效又安全的代码。

