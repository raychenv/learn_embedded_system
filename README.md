# learn_embedded_system

记录嵌入式系统学习笔记，主要围绕现代嵌入式软件设计方法展开，包括 RTOS、面向对象编程、事件驱动、状态机、软件追踪、断言与测试等主题。笔记以中文整理为主，强调主线脉络、设计动机和工程取舍，尽量避免逐句翻译。

## 仓库结构

- `moderm_embedded/`：课程笔记主体，按 lesson 顺序整理。
- `moderm_embedded/images/`：部分笔记配图。

## 当前覆盖范围

目前已整理 lesson 21 到 lesson 49，内容主线大致如下：

1. RTOS 与基础并发模型
2. 面向对象编程在嵌入式中的应用
3. 事件驱动与 Active Object
4. 状态机与层次状态机
5. 软件追踪、断言与测试

## 课程索引

### RTOS 与并发基础

- [21 嵌入式RTOS之前后台架构](moderm_embedded/21_foreground_background_arch.md)
- [22 RTOS 什么是实时操作系统](moderm_embedded/22_what_is_rtos.md)
- [23 嵌入式RTOS - 自动化上下文切换](moderm_embedded/23_auto_context_switch.md)
- [24 嵌入式RTOS - 线程轮转调度器](moderm_embedded/24_rtos_round_robin.md)
- [25 嵌入式RTOS - 高效阻塞线程](moderm_embedded/25_rtos_delay.md)
- [26 什么是实时抢占和优先级调度](moderm_embedded/26_rtos_priority_based_scheduling.md)
- [27 RTOS 线程间同步通信机制](moderm_embedded/27_rtos_sync_com.md)
- [28 RTOS 互斥机制](moderm_embedded/28_rtos_mutex.md)

### 面向对象编程

- [29 面向对象编程与封装](moderm_embedded/29_oop_encapsulation.md)
- [30 面向对象编程之继承](moderm_embedded/30_oop_inheritance_c.md)
- [31 面向对象编程之多态](moderm_embedded/31_oop_poly1.md)
- [32 面向对象编程之多态（C 实现）](moderm_embedded/32_oop_poly2.md)

### 事件驱动与 Active Object

- [33 事件驱动编程入门](moderm_embedded/33_event_driven.md)
- [34 事件驱动编程用于实时嵌入式系统](moderm_embedded/34_auto_gen.md)
- [43 Active Object 与硬实时调度](moderm_embedded/43_active_object_real_time.md)
- [44 Active Object 的交互、共享状态与可变事件](moderm_embedded/44_active_object_interaction.md)

### 状态机与层次状态机

- [35 状态机入门](moderm_embedded/35_state_machine.md)
- [36 状态机之 Guard Condition](moderm_embedded/36_state_machine.md)
- [37 其他类型的状态机：从硬件起源到 Input-Driven State Machine](moderm_embedded/37_state_machine.md)
- [38 状态表与状态的 Entry/Exit Action](moderm_embedded/38_state_machine.md)
- [39 用状态处理函数实现状态机，并抽象出可复用的事件处理器](moderm_embedded/39_state_machine.md)
- [40 层次状态机初探](moderm_embedded/40_state_machine.md)
- [41 状态机代码自动生成](moderm_embedded/41_state_machine_codegen.md)
- [42 层次状态机语义深入理解](moderm_embedded/42_state_machine_semantics.md)

### 软件追踪、断言与测试

- [45 用 printf 做软件追踪](moderm_embedded/45_trace_printf.md)
- [46 用 QP/Spy 做实时软件追踪](moderm_embedded/46_qp_realtime_trace.md)
- [47 软件断言与 Design by Contract（上）](moderm_embedded/47_assert.md)
- [48 软件断言与 Design by Contract（下）](moderm_embedded/48_assert.md)
- [49 测试在嵌入式软件开发中的核心地位](moderm_embedded/49_testing.md)

## 阅读建议

如果按主线阅读，推荐顺序如下：

1. 先读 21 到 28，建立 RTOS、调度、同步与互斥的基本背景。
2. 再读 29 到 34，把 OOP、事件驱动和 Active Object 串起来。
3. 然后读 35 到 42，系统理解状态机、层次状态机和代码生成。
4. 最后读 43 到 49，把实时性、追踪、断言与测试放回完整工程实践中理解。

## 笔记风格说明

这些笔记不是英文讲稿直译，而是按以下原则整理：

1. 用中文提炼每课主线与关键结论。
2. 保留必要英文术语，如 Active Object、state machine、guard condition、Design by Contract。
3. 优先解释设计动机、问题来源与工程权衡。
4. 尽量把相邻课程之间的承接关系写清楚，方便连续阅读。

