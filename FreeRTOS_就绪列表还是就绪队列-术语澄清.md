# FreeRTOS：是「就绪列表」还是「就绪队列」？（术语澄清）

## 问题

> FreeRTOS 中是叫就绪队列还是叫就绪列表？我在文档中只看到了「列表」和「列表项」这些概念。

## 回答

在 FreeRTOS 里，**官方/源码里叫「就绪列表」（ready list），不叫「就绪队列」**。你在文档里只看到「列表 / 列表项」是对的——FreeRTOS 调度用的底层数据结构本来就是**列表（List）**，不是队列。

### 源码里的实际名字

调度器里所有「任务集合」都是 `List_t`（列表），不是 `Queue_t`：

| 概念 | 源码里的变量 | 类型 |
|---|---|---|
| 就绪列表 | `pxReadyTasksLists[configMAX_PRIORITIES]` | `List_t` 数组（每个优先级一条） |
| 延时/阻塞列表 | `xDelayedTaskList1` / `xDelayedTaskList2` | `List_t` |
| 挂起列表 | `xSuspendedTaskList` | `List_t` |
| 待就绪列表 | `xPendingReadyList` | `List_t` |

任务（TCB）通过它的 `xStateListItem`（一个 `ListItem_t`，列表项）被**插入到**对应的那条 `List_t` 里。「这个任务现在是什么状态」=「它的 `xStateListItem` 挂在哪条 `List_t` 上」。

### 为什么不能叫「就绪队列」——会和另一个概念撞车

FreeRTOS 里**「队列（Queue）」是一个完全独立、专门的概念**：

- **队列 `Queue_t`**：任务间 / 中断与任务间**传数据**用的通信机制（`xQueueCreate`、`xQueueSend`、`xQueueReceive`…），也是信号量的底层实现。这是 **IPC（进程间通信）**。
- **就绪列表 `List_t`**：**调度器**用来管理「哪些任务就绪、按优先级排队等 CPU」的数据结构。

这俩是两套东西。如果把就绪列表叫成「就绪队列」，很容易和上面的消息队列搞混，所以 FreeRTOS 刻意用「列表」这个词。

### 那为什么有人会说「就绪队列 / 就绪表」？

- 概念上，就绪列表里的任务确实「排着队等 CPU」，所以口语上、有些中文教材/老师会随口叫它**就绪表**或**就绪队列**——意思你懂就行。
- 但**严格按 FreeRTOS 术语和源码**，正确叫法是 **就绪列表（ready list）**，底层是 **列表（`List_t`）+ 列表项（`ListItem_t`）**。

---

**一句话**：以 FreeRTOS 源码为准就是「就绪列表」（List），不是「队列」；「队列（Queue）」在 FreeRTOS 专指传数据的消息队列，别混。
