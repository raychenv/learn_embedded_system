# 8 C函数、调用约定与栈帧

## 引言

函数是高级编程语言的核心抽象。然而，大多数程序员习惯性地将函数视为"黑盒"——输入参数，执行代码，返回结果。在嵌入式系统和底层编程中，**必须理解函数在硬件级别的实现机制**。

本课深入讲解ARM体系结构下的函数传参、返回值处理、栈帧管理和调用约定。这对于调试、性能优化和理解异常处理至关重要。

lesson 8 工程可参考：
https://github.com/QuantumLeaps/modern-embedded-programming-course/tree/main/lesson-08

## 函数调用的高层观点

从C代码角度：

```c
void delay(int iter) {
    int volatile counter = 0;
    while (counter < iter) {
        ++counter;
    }
}

int main(void) {
    delay(1000000);
    delay(500000);
    return 0;
}
```

看起来很简单。但CPU如何知道：

1. 参数 `1000000` 传递给 `delay()` 的 `iter`？
2. 执行完 `delay()` 后如何返回到 `main()`？
3. `delay()` 中定义的 `counter` 变量存在哪里，如何不与 `main()` 的变量冲突？

答案涉及称为**调用约定**（Calling Convention）的协议。

## ARM调用约定（APCS / AAPCS）

ARM定义了多个标准调用约定。对于32位ARM核心（ARM7、Cortex-A、Cortex-M），常见约定是**AAPCS**（ARM Architecture Procedure Call Standard）。

### 寄存器规划

AAPCS将寄存器分配如下：

| 寄存器 | 名称 | 用途 |
|--------|------|------|
| R0-R3  | 参数/返回值 | 前4个参数通过R0-R3传递；返回值放在R0（或R0-R1用于64位） |
| R4-R11 | 被调用者保存 | 函数必须保存/恢复这些寄存器的原值（如果修改） |
| R12    | IP（Intra-Procedure-call Temp）| 临时寄存器，不需要保存 |
| R13    | SP (Stack Pointer) | 栈指针，必须对齐 |
| R14    | LR (Link Register) | 返回地址，调用指令 BL/BLX 自动填充 |
| R15    | PC (Program Counter) | 程序计数器（不能直接修改，但可通过 MOV PC, R14 返回） |

最关键的是：

- **参数传递**：前4个整型参数用R0-R3
- **返回值**：结果放在R0（或R0-R1）
- **返回地址**：存在R14（LR）

### 栈对齐要求

AAPCS要求**栈指针在函数入口处必须8字节对齐**。这允许编译器生成更高效的双字（64位）访问指令。

## 函数调用的汇编细节

让我们分析 `delay(1000000)` 调用的完整过程。

### 第1步：准备参数

在调用 `delay()` 前，`main()` 需要将参数放入R0：

```asm
MOVW   R0, #0x6400        ; R0 = 0x000F4240（1000000 的低16位）
MOVT   R0, #0x00F4        ; R0 = 0x000F4240（1000000 的完整值）
```

或者在优化后：

```asm
MOV    R0, #1000000       ; ARM编译器可能使用单条指令（带立即数旋转）
```

或使用特殊的立即数旋转编码。

### 第2步：执行跳转指令 BL（Branch with Link）

```asm
BL     delay              ; PC→R14（保存返回地址）, PC = delay的地址
```

`BL` 指令的硬件行为：

1. 计算目标地址：链接器已将符号`delay`解析为绝对地址（如 `0x00000200`）或相对偏移
2. **保存返回地址**：当前 PC + 4（下一条指令地址）复制到 R14
3. **跳转**：PC = 目标地址

此时 R14 包含返回地址（如 `0x00001234`）。

### 第3步：进入函数 delay() 的序言代码

编译器为每个函数生成序言（prologue）和尾声（epilogue）代码。对于 `delay()`：

```asm
delay:
    PUSH   {R4, R5, LR}    ; 保存寄存器（栈从高向低增长）
    ...函数主体...
    POP    {R4, R5, PC}    ; 恢复寄存器，从栈加载到 PC → 返回
```

**PUSH 指令的含义**：

```asm
PUSH {R4, R5, LR}  
; 等价于：
;   SP = SP - 12          ; 为3个寄存器预留12字节
;   [SP+8] = LR           ; 在栈上最高位置存放 LR
;   [SP+4] = R5
;   [SP+0] = R4
```

在ARM的内存模型中，**栈从高地址向低地址增长**（向下增长）。SP总是指向最后一个有效的栈帧元素。

执行 `PUSH {R4, R5, LR}` 后，栈状态如图所示：

```
地址        内容
...
SP+12      [上一个栈帧的数据]
SP+8       R4 value
SP+4       R5 value
SP+0       LR value (返回地址) ← SP 现在指向这里
...
```

### 第4步：分配局部变量

`delay()` 中有局部变量 `counter`：

```c
int volatile counter = 0;
```

编译器需要为这个变量分配栈空间。假设 `counter` 占用4字节：

```asm
PUSH   {R4, R5, LR}    ; SP已经减少了12字节
SUB    SP, #4          ; 再为 counter 分配4字节
MOV    R1, #0
STR    R1, [SP]        ; *SP = 0; 初始化 counter = 0
```

现在栈布局为：

```
SP+0       counter (初始值 = 0)
SP+4       LR 保存值
SP+8       R5 保存值
SP+12      R4 保存值
```

### 第5步：函数主体

```c
while (counter < iter) {
    ++counter;
}
```

转译为汇编（R0 = iter）：

```asm
LOOP:
    LDR    R1, [SP]        ; R1 = counter（从栈加载）
    CMP    R1, R0          ; 比较 counter 与 iter
    BGE    END_LOOP        ; 如果 counter >= iter，跳出循环
    
    ADD    R1, R1, #1      ; counter++
    STR    R1, [SP]        ; 存回栈
    
    B      LOOP            ; 回到循环开始
    
END_LOOP:
```

### 第6步：函数尾声与返回

```c
}  // delay() 结束
```

编译器生成尾声代码：

```asm
ADD    SP, #4          ; 弹出局部变量（counter）
POP    {R4, R5, PC}    ; 恢复保存的寄存器，**并从栈加载PC**
```

**POP {R4, R5, PC}** 的硬件行为：

```asm
; 等价于：
;   R4 = [SP]      ; SP 现在指向 R4 的保存值
;   R5 = [SP+4]
;   PC = [SP+8]    ; 跳回 BL 后的返回地址
;   SP = SP + 12
```

当 `PC = [SP+8]` 执行时，处理器下一个取指周期就会从返回地址（如 `0x00001234`）开始执行。

## 调用者保存 vs 被调用者保存

AAPCS规定：

**被调用者保存**（Callee-Saved）：R4-R11
- 如果函数使用这些寄存器，必须在函数序言中保存原值，在尾声中恢复

**调用者保存**（Caller-Saved）：R0-R3, R12
- 函数可以自由使用、修改这些寄存器而无需保存
- 调用者如果需要保留这些值，必须在 BL 前自行保存

例如：

```c
int x = 10;
delay(1000000);  // delay() 可能修改 R0-R3, R12
print(x);        // x 仍在 R1 中（如果编译器选择）
```

如果编译器在 `delay()` 调用前将 `x` 存储在 R1 中，则需要在调用后重新加载R1，因为R1是调用者保存的。如果取而代之将其存储在R4中，则因为R4是被调用者保存的，`delay()` 返回后R4的值不会被修改。

## 被调用函数如何保存寄存器

当 `delay()` 需要使用被调用者保存的寄存器（如R4）时：

```c
void delay(int iter) {
    static int saved_state = 0;  // 如果使用，会占用 RAM
    int volatile counter = 0;
    
    while (counter < iter) {
        ++counter;
    }
}
```

编译器会生成：

```asm
delay:
    PUSH   {R4, ..., LR}   ; R4 被函数使用，所以必须保存
    ... 使用 R4 ...
    POP    {R4, ..., PC}   ; 恢复 R4
```

如果函数不使用被调用者保存的寄存器，则不需要保存它们，从而节省栈空间和执行周期。

## 参数超过4个时的处理

当函数有5个或以上的参数时，前4个通过R0-R3传递，剩余的通过**栈**传递。

例如：

```c
void process(int a, int b, int c, int d, int e, int f) {
    // a=R0, b=R1, c=R2, d=R3, e=栈上, f=栈上
}

process(1, 2, 3, 4, 5, 6);
```

调用者在调用前需要：

```asm
; 准备调用者保存的参数（在栈上）
MOV    R0, #1   ; a
MOV    R1, #2   ; b
MOV    R2, #3   ; c
MOV    R3, #4   ; d
PUSH   {6}      ; e 和 f 需要在栈上
PUSH   {5}      ; 注意栈的增长方向
BL     process
ADD    SP, #8   ; 清理栈上的参数（调用者负责）
```

函数内部访问栈上的参数：

```asm
process:
    PUSH   {LR}     ; 如果需要保存 LR
    ; 访问第5个参数（e）：在 [SP+4]（因为 LR 占用4字节）
    LDR    R4, [SP, #4]  ; R4 = e
    LDR    R5, [SP, #8]  ; R5 = f
    ...
    POP    {PC}
```

这说明了**栈帧对理解参数传递的重要性**。

## 返回值处理

返回值放在R0中（或R0-R1用于64位/结构体小于8字节的情况）：

```c
unsigned int get_value(void) {
    return 42;
}

unsigned long long get_wide_value(void) {
    return 0x0123456789ABCDEF;
}
```

汇编形式：

```asm
get_value:
    MOV    R0, #42     ; 返回值放在 R0
    MOV    PC, LR      ; 返回

get_wide_value:
    MOVW   R0, #0xCDEF  ; 64位结果的低32位
    MOVT   R0, #0xAB89
    MOVW   R1, #0x4567  ; 高32位
    MOVT   R1, #0x0123
    MOV    PC, LR       ; 返回（R0-R1 包含结果）
```

如果返回值是结构体（超过8字节），AAPCS规定应返回一个指向内存中结果的指针。这通常在栈上分配内存，地址在R0中传递给调用者。

## 栈帧的完整生命周期

让我们追踪 `main() -> delay() -> 返回` 的完整过程：

**初始状态**（在 `main()` 中）：

```
地址        内容
...
0x20002FF8  旧的栈数据
0x20003000  ← SP（栈顶）
```

**执行 main() 的序言**：

```c
int main(void) {
    ...
}
```

假设 `main()` 的序言：

```asm
main:
    PUSH   {LR}         ; 保存 LR（如果 main 需要返回）
```

现在：

```
地址        内容
...
0x20002FF8  旧的栈数据
0x20002FFC  LR 值（main 的返回地址）
0x20003000  ← SP
```

**准备调用 delay()** 并 **执行 BL**：

```asm
    MOV    R0, #1000000
    BL     delay
```

硬件自动在R14中保存返回地址（`main()` 中 BL 的下一条指令地址）。

```
地址        内容
...
0x20002FFC  LR_main（main的返回地址）
0x20003000  ← SP（暂时未移动，取决于 main 的实现）
```

**执行 delay() 的序言**：

```asm
delay:
    PUSH   {LR}         ; 保存当前 LR（即 main 的返回地址）
    SUB    SP, #4       ; 为 counter 分配 4 字节
```

栈变化：

```
地址        内容
...
0x20002FFC  LR_main
0x20003000  LR_delay（delay 的返回地址）← SP+4
```

实际上，假设执行序言前 SP = 0x20003000：

```asm
PUSH   {LR}     ; LR 包含来自 BL 的返回地址（即 main 中调用点的下一条指令）
                ; SP现在 = 0x20002FFC
                ; [0x20002FFC] = LR_value
SUB    SP, #4   ; SP = 0x20002FF8
```

栈配置：

```
地址        内容
0x20002FF8  counter = 0 ← SP（当前）
0x20002FFC  LR_delay_return（返回地址）
0x20003000  [上一个栈帧的数据]
```

**delay() 执行主体**（很多次循环）：

对 `counter` 的读写发生在栈上的 `[SP]`。

**delay() 尾声**：

```asm
ADD    SP, #4       ; SP = 0x20002FFC
POP    {PC}         ; PC = [0x20002FFC] = LR_delay_return
                    ; SP = 0x20003000
```

**返回到 main()**：

PC现在指向 `main()` 中 `BL delay` 的下一条指令。执行继续进行。如果 `main()` 之后调用另一个函数或返回，则执行相应的倒序。

## 栈的实际大小限制

在嵌入式系统中，RAM是有限的。假设TCMp（Tightly Coupled Memory）为 `1MB = 0x100000` 字节：

```c
int get_stack_info(void) {
    return 0;
}
```

链接器配置 CSTACK（见课程15）为：`0x100000 - 大小（用于堆）`。假设 CSTACK 配置为 `0x10000` 字节（64KB）。

如果栈增长超过分配的区间，SP会指向无效的内存地址，从而导致：

1. **公线异常**（Bus Fault）
2. **或静默数据损坏**（如果写入了有效但不应写入的地址）

这就是第10课要重点讲述的栈溢出问题。

## 优化与栈大小

现代编译器使用各种技术优化栈使用：

1. **寄存器分配**：尽可能使用寄存器而非栈
2. **栈帧省略**（Frame Pointer Omission）：不使用 R11 作为基指针，直接相对 SP 访问
3. **局部变量重用**：同一个栈位置用于不同生命周期内的局部变量
4. **内联函数**：消除函数调用开销

例如，`delay()` 可以被声明为 `static inline`，编译器会直接将代码展开到调用点，完全消除函数调用开销。

## 小总结：从调用到返回

关键概念：

1. **参数传递**：前4个用R0-R3，超过4个时用栈
2. **返回地址**：BL 指令自动保存在 R14（LR）
3. **栈帧**：每个函数维护一个栈帧来管理局部变量和保存的寄存器
4. **被调用者职责**：函数必须保存/恢复被调用者保存的寄存器
5. **栈增长方向**：ARM 栈从高向低增长，SP递减

理解这些机制对于：

- **调试**：在调试器中检查栈帧、参数、返回地址
- **性能优化**：减少栈操作次数
- **异常处理**：因为异常处理涉及栈帧的动态修改

在下一课（第9课），我们会看到当函数相互调用、特别是递归调用时，这些栈帧如何嵌套。这种理解对于理解复杂控制流和调试困难的问题至关重要。

