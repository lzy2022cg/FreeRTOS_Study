# FreeRTOS TCB 任务控制块详解

## 问题

> 我在整个工程中搜索 `tskTaskControlBlock`，用 Ctrl+F 只能搜当前文档。如何在整个工程中搜索？

**回答**：在 VS Code 中，**Ctrl+F** 只搜索当前文件，要搜索整个工程，请使用 **Ctrl+Shift+F**（在文件中查找）。如果快捷键没反应，可能是因为中文输入法拦截或快捷键冲突，可以点击左侧活动栏的 🔍 搜索图标来打开。

搜索结果：`tskTaskControlBlock` 在整个工程中出现 3 次，都在 FreeRTOS 源码 `tasks.c` 和 `task.h` 中。

---

## `tskTaskControlBlock` 结构体详解

> 选自 `tasks.c` 第 252-329 行

`tskTaskControlBlock`（即 **TCB**，Task Control Block）是 FreeRTOS 中最核心的数据结构。每个任务在创建时都会分配一个 TCB，用于保存该任务的所有状态信息。

```c
typedef struct tskTaskControlBlock
{
```

### 1. 栈指针（必须放在第一个）

```c
volatile StackType_t *pxTopOfStack;
```

指向任务栈顶（最后入栈的数据位置）。**必须是 TCB 的第一个成员**，这样上下文切换时汇编代码能快速定位。

### 2. MPU 设置（条件编译）

```c
#if ( portUSING_MPU_WRAPPERS == 1 )
    xMPU_SETTINGS xMPUSettings;
#endif
```

如果启用了 MPU（内存保护单元），这是第二个成员，用于保存该任务的 MPU 区域配置。

### 3. 链表项 — 任务状态管理

```c
ListItem_t xStateListItem;   // 状态链表项
ListItem_t xEventListItem;   // 事件链表项
```

- **`xStateListItem`**：将任务挂到"就绪链表"、"阻塞链表"或"挂起链表"中，**链表所在位置决定了任务当前状态**。
- **`xEventListItem`**：将任务挂到某个事件链表上（如信号量、队列的等待列表）。

### 4. 优先级

```c
UBaseType_t uxPriority;
```

任务的当前优先级，0 为最低。调度器根据此值决定哪个任务运行。

### 5. 栈基址

```c
StackType_t *pxStack;
```

指向任务栈的**起始地址**（分配时的最底端），用于删除任务时释放栈内存。

### 6. 任务名称

```c
char pcTaskName[ configMAX_TASK_NAME_LEN ];
```

调试用，创建任务时指定的名字。

### 7. 栈顶地址（条件编译）

```c
#if ( ( portSTACK_GROWTH > 0 ) || ( configRECORD_STACK_HIGH_ADDRESS == 1 ) )
    StackType_t *pxEndOfStack;
#endif
```

指向栈的**最高有效地址**，用于栈溢出检测。

### 8. 临界区嵌套深度

```c
#if ( portCRITICAL_NESTING_IN_TCB == 1 )
    UBaseType_t uxCriticalNesting;
#endif
```

记录临界区嵌套层数，支持嵌套的 `taskENTER_CRITICAL()` / `taskEXIT_CRITICAL()`。

### 9. 跟踪调试编号

```c
#if ( configUSE_TRACE_FACILITY == 1 )
    UBaseType_t uxTCBNumber;   // TCB 序号，每次创建递增
    UBaseType_t uxTaskNumber;  // 第三方跟踪工具用的任务编号
#endif
```

### 10. 互斥量支持

```c
#if ( configUSE_MUTEXES == 1 )
    UBaseType_t uxBasePriority;  // 任务原始优先级（不含继承）
    UBaseType_t uxMutexesHeld;   // 当前持有的互斥量数量
#endif
```

用于**优先级继承机制**：当高优先级任务等待低优先级任务持有的互斥量时，低优先级任务会临时"继承"高优先级。

### 11. 任务标签（Hook）

```c
#if ( configUSE_APPLICATION_TASK_TAG == 1 )
    TaskHookFunction_t pxTaskTag;
#endif
```

用户可自定义的回调函数指针。

### 12. 线程本地存储

```c
#if( configNUM_THREAD_LOCAL_STORAGE_POINTERS > 0 )
    void *pvThreadLocalStoragePointers[...];
#endif
```

每个任务私有的存储指针数组。

### 13. 运行时间统计

```c
#if( configGENERATE_RUN_TIME_STATS == 1 )
    uint32_t ulRunTimeCounter;
#endif
```

记录任务累计运行时间（用于 `vTaskGetRunTimeStats()`）。

### 14. Newlib 重入支持

```c
#if ( configUSE_NEWLIB_REENTRANT == 1 )
    struct _reent xNewLib_reent;
#endif
```

为使用 Newlib C 库的任务提供线程安全的 `errno` 等。

### 15. 任务通知

```c
#if( configUSE_TASK_NOTIFICATIONS == 1 )
    volatile uint32_t ulNotifiedValue;   // 通知值
    volatile uint8_t ucNotifyState;      // 通知状态
#endif
```

FreeRTOS 的轻量级任务间通信机制。

### 16. 静态/动态分配标记

```c
#if( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 )
    uint8_t ucStaticallyAllocated;
#endif
```

记录任务是通过 `xTaskCreateStatic()` 还是 `xTaskCreate()` 创建的，删除时决定是否释放内存。

### 17. 其他

```c
#if( INCLUDE_xTaskAbortDelay == 1 )
    uint8_t ucDelayAborted;       // 延迟是否被强制中止
#endif

#if( configUSE_POSIX_ERRNO == 1 )
    int iTaskErrno;               // 每个任务独立的 errno
#endif
```

### 结尾

```c
} tskTCB;  // 类型别名

// 紧接着：
typedef tskTCB TCB_t;  // 新命名，TCB_t 是 tskTCB 的别名
```

---

## 总结

TCB 就像一个任务的"身份证"，记录了：

- 🧠 **执行上下文**（栈指针 `pxTopOfStack`）
- 📊 **调度信息**（优先级、状态链表）
- 🔧 **资源信息**（互斥量、通知、线程本地存储）
- 🐛 **调试信息**（名称、编号、运行统计）

调度器通过操作 TCB 中的链表项（`xStateListItem`），让任务在不同状态之间流转，实现多任务并发。
