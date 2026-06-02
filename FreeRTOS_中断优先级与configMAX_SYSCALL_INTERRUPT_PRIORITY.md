# FreeRTOS 中断优先级与 configMAX_SYSCALL_INTERRUPT_PRIORITY

## 问题

> 「低于 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 优先级的中断里才允许调用 FreeRTOS 的 API 函数」——解释一下。

## 回答

先说一个**最容易被这句话坑到的点**：Cortex-M（STM32F407）的中断优先级是「**数值越小，优先级越高**」。所以书里说的「优先级低于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`」，指的是**逻辑上的紧急程度低**，对应的是**数值上 ≥ 这个阈值**。这个反过来的关系不理清，整句话就会读反。

### 这句话到底在说什么

`configMAX_SYSCALL_INTERRUPT_PRIORITY` 是一条**分界线（优先级阈值）**，把所有中断分成两类：

```
数值:   0   1   2   3   4 │ 5   6   7 ...  14  15
        ├─────────────────┤ ├──────────────────┤
逻辑:   最高/最紧急         │      较低/不紧急      最低
        ✗ 不能调 API       │      ✓ 能调 FromISR API
                          ↑
        configMAX_SYSCALL_INTERRUPT_PRIORITY = 5(举例)
```

- **数值 ≥ 阈值（逻辑优先级 ≤ 阈值）的中断**：归 FreeRTOS「管」，里面**才允许调用 `xxxFromISR()` 这类 API**。
- **数值 < 阈值（逻辑优先级更高、更紧急）的中断**：FreeRTOS **完全不碰它们**，里面**绝对不能调用任何 FreeRTOS API**。

### 为什么有这条规定（关键原理）

FreeRTOS 要保护它的内核数据（就绪表、链表等）不被中途打断破坏，靠的是**临界区**（`taskENTER_CRITICAL()` / `portSET_INTERRUPT_MASK_FROM_ISR()`）。

在 Cortex-M 上，FreeRTOS 的临界区**不是简单的关总中断（CPSID i）**，而是设置 **`BASEPRI` 寄存器 = configMAX_SYSCALL_INTERRUPT_PRIORITY**。`BASEPRI` 的效果是：

> 屏蔽掉所有**数值 ≥ BASEPRI** 的中断，但**数值更小（更高优先级）的中断照样能打断**。

由此推出两类中断的差别：

| 中断类型 | 临界区里会被屏蔽吗 | 能否安全调 API |
|---------|----------------|--------------|
| 数值 ≥ 阈值（受管） | **会被屏蔽** → 不会在内核改数据时插进来 | ✅ 安全，可调 `FromISR` |
| 数值 < 阈值（高于阈值，更紧急） | **不会被屏蔽** → 随时能打断临界区 | ❌ 一旦调 API 就可能在内核改到一半时插进来，破坏数据结构 |

所以那条「更高优先级」的中断换来的好处是：**永远不会被 FreeRTOS 延迟**（零延迟中断，实时性最好），代价就是**不能用任何 FreeRTOS 功能**（不能通知任务、不能发信号量/队列）。

### STM32F407 上的具体数值

F407 的 NVIC 优先级是 **4 位（16 级）**，存在 8 位寄存器的**高 4 位**里。所以 `FreeRTOSConfig.h` 里通常这样配：

```c
#define configPRIO_BITS                          4
/* SysTick/PendSV 用最低优先级 15 */
#define configKERNEL_INTERRUPT_PRIORITY          (15 << 4)   // 0xF0

/* 阈值:数值 5 */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY  5
#define configMAX_SYSCALL_INTERRUPT_PRIORITY     (5 << 4)    // 0x50
```

对应到实操，**抢占优先级数字**：

- `0 ~ 4`：高于阈值，**不能调用任何 FreeRTOS API**；
- `5 ~ 15`：等于或低于阈值，**可以调用 `xxxFromISR()` API**。

> 还有个前提：必须把 NVIC 优先级分组设成 **Group 4**（`HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)`），让 16 级全部是抢占优先级、没有子优先级。否则子优先级会让上面的比较失效。HAL 默认就是 Group 4，一般不用改。

### 实操规则 + 违反后果

1. **凡是中断服务函数里要调 `FromISR` API（比如 `xSemaphoreGiveFromISR`、`xQueueSendFromISR`、`vTaskNotifyGiveFromISR`），该中断的抢占优先级数字必须 ≥ 5**（即逻辑优先级 ≤ 阈值）。
2. CubeMX/HAL 默认很多外设中断优先级是 **0**——直接在 0 优先级的中断里调 API 就违规了，**记得手动把它的抢占优先级改成 5 或更大**。
3. 如果违反，FreeRTOS 在 Debug 下会通过 `vPortValidateInterruptPriority()` 里的 **`configASSERT` 直接断言报错**（程序停在断言处）。这是最常见的「一调 API 就卡死/进 HardFault」原因之一。

---

**一句话记忆**：阈值以下（更紧急、数值更小）的中断 = 内核管不着 = 不准碰 API；阈值及以上（数值更大）的中断 = 内核能屏蔽 = 才准调 `FromISR`。
