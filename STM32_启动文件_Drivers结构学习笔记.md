# STM32 启动文件与 Drivers 结构学习笔记

> 日期：2026-06-03  
> 基于：正点原子探索者 STM32F407 开发板，HAL 库版本，实验5 串口通信实验

---

## 一、Drivers 目录结构

工程中 `Drivers/` 文件夹存放底层驱动代码，分为 4 个子目录：

### 1. BSP（板级支持包）
- 正点原子提供的板载外设驱动（原 `HARDWARE` 文件夹下的代码）
- 如：LED、BEEP、KEY、EXTI、TIMER、WDG 等
- 本工程中只有 `LED/`，提供 `led_init()`、`LED0_TOGGLE()` 等接口

### 2. CMSIS
- ARM 提供的 Cortex-M 核心支持库
- 包含：
  - `Include/`：内核寄存器定义头文件（`core_cm4.h` 等）
  - `Device/ST/`：启动文件（`.s`）、系统初始化文件等
- 作用：让 STM32 运行在 ARM CMSIS 标准框架下

### 3. STM32F4xx_HAL_Driver
- ST 官方提供的 HAL 库驱动
- `Inc/`：头文件（`stm32f4xx_hal_uart.h`、`stm32f4xx_hal_gpio.h` 等）
- `Src/`：源文件（对应的 `.c` 实现）
- 提供标准接口如 `HAL_UART_Transmit()`、`HAL_GPIO_WritePin()` 等

### 4. SYSTEM
- 正点原子提供的系统级核心驱动，包含三个子文件夹：
  - `delay/`：延时函数，支持在 OS 下使用
  - `sys/`：系统时钟初始化、IO 配置、中断管理
  - `usart/`：串口驱动，支持 `printf`，方便调试

### 调用关系示例（来自 `User/main.c`）

```c
#include "./SYSTEM/sys/sys.h"
#include "./SYSTEM/usart/usart.h"
#include "./SYSTEM/delay/delay.h"
#include "./BSP/LED/led.h"

int main(void) {
    HAL_Init();                             // HAL 库初始化
    sys_stm32_clock_init(336, 8, 2, 7);     // 设置时钟 168MHz
    delay_init(168);                        // 延时初始化
    led_init();                             // LED 初始化
    // ...
    printf("正点原子 STM32开发板 串口实验\r\n");
    HAL_UART_Transmit(&g_uart1_handle, ...);
}
```

> **一句话总结**：Drivers 是"硬件底盘"，User/main.c 是"驾驶员"，把底层能力拼成完整功能。

---

## 二、startup_stm32f407xx.s 启动文件详解

### 文件头部注释原文

```asm
;* Description : STM32F407xx devices vector table for MDK-ARM toolchain.
;*               This module performs:
;*               - Set the initial SP
;*               - Set the initial PC == Reset_Handler
;*               - Set the vector table entries with the exceptions ISR address
;*               - Branches to __main in the C library (which eventually calls main()).
;*               After Reset the CortexM4 processor is in Thread mode,
;*               priority is Privileged, and the Stack is set to Main.
```

### 关键概念辨析："复位后" vs `Reset_Handler`

| 概念 | 含义 |
|------|------|
| **"复位后"** | MCU 收到硬件复位事件（上电、NRST 拉低、看门狗复位等） |
| **`Reset_Handler`** | 复位后 CPU 执行的第一段启动代码（**不是普通的中断服务函数**） |

**正确理解**：
- 复位发生时，CPU 硬件自动从向量表取两个值：
  1. `__initial_sp` → 设置 SP（栈指针）
  2. `Reset_Handler` 地址 → 设置 PC（程序计数器）
- 然后 CPU 开始执行 `Reset_Handler` 中的代码
- `Reset_Handler` 不是"中断"，而是**复位入口函数**

### 完整启动流程

```
硬件复位
    │
    ▼
CPU 从向量表取 SP = __initial_sp, PC = Reset_Handler
    │
    ▼
Reset_Handler:
    1. 使能浮点运算 CP10, CP11
    2. 调用 SystemInit() —— 初始化时钟、FPU、向量表偏移等
    3. 调用 __main —— 进入 C 运行环境
    │
    ▼
__main (C 库初始化):
    - 初始化 RW/ZI 段（全局变量初始化）
    - 调用 main()
    │
    ▼
main() —— 用户应用程序入口
```

### 向量表结构

```asm
__Vectors   DCD     __initial_sp           ; 栈顶地址
            DCD     Reset_Handler          ; 复位入口（第 2 项）
            DCD     NMI_Handler            ; 不可屏蔽中断
            DCD     HardFault_Handler      ; 硬件错误
            DCD     MemManage_Handler      ; MPU 错误
            DCD     BusFault_Handler       ; 总线错误
            DCD     UsageFault_Handler     ; 用法错误
            ; ... 系统异常 ...
            DCD     SysTick_Handler        ; 系统滴答
            ; ... 外设中断（USART、TIM、DMA 等）...
```

向量表在复位时映射到地址 0x00000000，第一个字是栈顶，第二个字是复位入口地址。

### Reset_Handler 汇编代码

```asm
Reset_Handler   PROC
                EXPORT  Reset_Handler             [WEAK]
                IMPORT  SystemInit
                IMPORT  __main
                LDR     R0, =0xE000ED88           ; 使能浮点运算 CP10, CP11
                LDR     R1, [R0]
                ORR     R1, R1, #(0xF << 20)
                STR     R1, [R0]
                LDR     R0, =SystemInit           ; 调用系统初始化
                BLX     R0
                LDR     R0, =__main               ; 跳转到 C 库入口
                BX      R0
                ENDP
```

- `[WEAK]` 表示弱定义，用户可在其他地方重写
- `SystemInit` 在 `system_stm32f4xx.c` 中实现
- `__main` 是 MDK 编译器提供的 C 运行时初始化入口

### 默认中断处理函数

```asm
NMI_Handler     PROC
                EXPORT  NMI_Handler       [WEAK]
                B       .                  ; 死循环
                ENDP
```

所有未重写的中断处理函数都默认跳入死循环（`B .`），表示异常发生时会停在此处。用户需要在 `.c` 文件中实现同名函数来覆盖这些弱定义。

### 栈和堆配置

```asm
Stack_Size      EQU     0x00000400        ; 栈大小 1KB
Heap_Size       EQU     0x00000000        ; 堆大小 0（未使用 malloc）
```

---

## 三、常见疑问解答

**Q：`Reset_Handler` 是中断还是复位入口？**  
A：它是**复位入口函数**。复位事件触发后，CPU 自动跳到这里开始执行。它与 USART1_IRQHandler 等可屏蔽中断不同，复位是不可屏蔽的硬件事件。

**Q：注释中"设置初始 PC == Reset_Handler"是谁做的？**  
A：由 Cortex-M4 硬件复位逻辑自动完成，不是在 `Reset_Handler` 代码里做的。复位时 CPU 自动从向量表第 2 项取出 `Reset_Handler` 的地址赋给 PC。

**Q：`__main` 和 `main()` 有什么区别？**  
A：`__main` 是编译器提供的 C 运行时初始化函数，负责初始化全局/静态变量（RW/ZI 段），然后才调用用户写的 `main()`。

---

> **学习心得**：理解启动流程是嵌入式开发的基石。从向量表 → Reset_Handler → SystemInit → __main → main() 这条链路，就是 STM32 从"上电"到"运行你的代码"的完整路径。
