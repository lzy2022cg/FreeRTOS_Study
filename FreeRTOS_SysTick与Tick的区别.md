# FreeRTOS：SysTick 与 Tick 的区别（以及它们的中断）

## 问题

> - SysTick 是系统滴答定时器，Tick 是系统时钟节拍
> - SysTick 中断是系统滴答定时器中断，Tick 中断是系统时钟节拍中断
>
> 对吗？

## 回答

基本对，但**第 4 句有个容易踩的坑**——别把「SysTick 中断」和「Tick 中断」理解成两个不同的中断。先逐条判：

| 说法 | 对错 | 说明 |
|---|---|---|
| SysTick 是系统滴答定时器 | ✅ 对 | **硬件**——Cortex-M 内核自带的 24 位递减计数定时器 |
| Tick 是系统时钟节拍 | ✅ 对 | **软件概念**——FreeRTOS 的时间基准单位（一个 tick = 一拍） |
| SysTick 中断是系统滴答定时器中断 | ✅ 对 | SysTick 这个硬件产生的中断（`SysTick_Handler`，异常号 15） |
| Tick 中断是系统时钟节拍中断 | ⚠️ 字面对，但**它和 SysTick 中断是同一个中断**，不是两个 |

### 关键：它俩不是两个中断，而是「一个中断、两层叫法」

在 STM32F407 + FreeRTOS 里：

> **SysTick 硬件定时器产生的那个中断，被 FreeRTOS 直接拿来当「节拍中断」用。** 所以「SysTick 中断」和「Tick 中断」指的是**同一个物理中断**，只是站在不同层看叫法不同。

```
硬件层:  SysTick 定时器 ──每隔 1/configTICK_RATE_HZ 秒──▶ 产生 SysTick 中断 (SysTick_Handler)
                                                                  │  同一个中断
内核层:  FreeRTOS 在这个中断里做 xTaskIncrementTick() → xTickCount++ ◀┘
         └─ 这一下,就是"一个系统时钟节拍 / 一个 tick"
```

- **从硬件角度**叫它：SysTick 定时器中断。
- **从 RTOS 角度**叫它：节拍中断（tick interrupt）——因为 FreeRTOS 在这个中断里把节拍计数 +1。

所以正确的关系是：**SysTick 中断 = Tick 中断**（在 Cortex-M 上），一次 SysTick 中断 = 走一个 tick。

### 一个层次上的区别（这才是 SysTick 和 Tick 的真正不同）

- **SysTick** 是**具体硬件**；**Tick** 是**抽象的 RTOS 概念**。
- 「节拍」这个概念**不一定非得由 SysTick 提供**——换到别的架构（比如某些 ARM Cortex-A、RISC-V），FreeRTOS 会用那个平台上的别的定时器当节拍源。只是在 Cortex-M（STM32）上，**约定俗成用 SysTick** 来做节拍源。
- 所以准确说法是：**「Tick 是节拍这个概念，SysTick 是在 STM32 上实现这个节拍的硬件」**，而不是「两个并列的中断」。

---

**一句话**：前 3 句都对；第 4 句要修正成——「Tick 中断」不是独立的另一个中断，它就是 SysTick 中断（SysTick 硬件的中断被 FreeRTOS 用作节拍中断）。SysTick 是硬件、Tick 是概念，二者是「实现 与 被实现」的关系，不是两个东西。
