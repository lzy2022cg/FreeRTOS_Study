# vTaskDelay() vs vTaskSuspend()

## vTaskDelay() 函数会进入阻塞态，vTaskSuspend() 会进入挂起状态，对吗？

**完全正确！**

### FreeRTOS 任务状态对比

| | `vTaskDelay()` | `vTaskSuspend()` |
|---|---|---|
| 状态 | **阻塞态（Blocked）** | **挂起态（Suspended）** |
| 谁来唤醒 | **自动**：延时到期后自动回到就绪态 | **手动**：必须用 `vTaskResume()` 恢复 |
| 有超时吗 | ? 有（指定 tick 数） | ? 没有，可能永远挂起 |
| 操作对象 | 只能是**调用者自己** | 可以是**任何任务**（通过句柄） |

### 在课堂源码中的体现

```c
// task3 中的两行：
vTaskSuspend(task1_handler);   // task1 → 挂起态（Suspended）
                                // 必须等 KEY1 按下，vTaskResume(task1_handler) 才能恢复

vTaskDelay(10);                // task3 自己 → 阻塞态（Blocked）
                               // 10ms 后自动恢复，不用任何人唤醒
```

### 为什么挂起后 `vTaskDelay` 也救不了 task1？

```
task1 被 vTaskSuspend → 挂起态（Suspended）
                         ↓
                   调度器完全忽略它！
                   即使 vTaskDelay(500) 超时了也没用
                   因为它在挂起态，不在阻塞态等待
                         ↓
                   只有 vTaskResume() 能把它拉回就绪态
```

**一句话**：阻塞态是"设了闹钟睡觉"，挂起态是"被打晕了，只能靠别人叫醒"。
