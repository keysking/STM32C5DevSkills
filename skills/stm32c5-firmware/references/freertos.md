# FreeRTOS 任务、队列与同步 API

按实际功能选择 API，本页不是一个要求创建两个任务或周期传递计数的完整应用。已有工程保持原生/CMSIS v1/v2、对象所有者、tick 和内存方案。只有缺少所需内核能力时才进入[生成器配置](../../stm32c5-configure/references/freertos.md)，新增任务不等于重配内核。

## 确认 API、类型和启用条件

在工程实际使用的 `FreeRTOS.h`、`task.h`、`queue.h`、`semphr.h` 和 `FreeRTOSConfig.h` 查询。源码位于 Pack 时从 `cube --get-current-value cmsis_pack_root` 定位工程选定版本。本文不覆盖所有 API；检索所需函数声明、配置条件及 port 实现，不凭训练知识换用另一版本签名。

```text
rg -n "xTaskCreate|xTaskCreateStatic|xTaskNotifyFromISR" .
rg -n "configSUPPORT_DYNAMIC_ALLOCATION|configSUPPORT_STATIC_ALLOCATION|configTICK_RATE_HZ" .
```

下文 C 语法片段放在相应任务函数、ISR 或初始化函数内；其中句柄、缓冲区、名称、容量及超时由当前应用提供，不把所有片段拼成启动程序。

## 创建任务

仅在需要新增任务且尚未存在时创建。动态创建用 `xTaskCreate`，静态创建用 `xTaskCreateStatic`，按已有内存策略选择，不能为了照抄 API 就开启动态分配。

动态创建的调用形式：

```c
/* entry: void (*)(void *)；name: 名称；argument: 生命周期足够的参数。
 * stack_words、priority 由任务需求、配置范围和测量确定。
 * task_handle 类型为 TaskHandle_t。 */
BaseType_t created = xTaskCreate(entry, name, stack_words,
                                argument, priority, &task_handle);
if (created != pdPASS) {
    /* 返回初始化失败或进入现有故障处理，不能当作创建成功。 */
}
```

包含 `FreeRTOS.h` 和 `task.h`。栈深度单位为 `StackType_t` 个数，不是字节；用栈水位验证余量，不统一写成 256。任务入口采用循环和适当阻塞，任务函数不得直接返回，需要结束时使用项目允许的删除机制。已有调度器运行时新增任务可能立即运行，先准备好参数和共享对象。

静态创建还须提供长期有效的 `StaticTask_t` 及栈数组，并检查返回 handle。一个任务只由生成器或应用的一处创建，避免重复。

## 启动调度器

仅对尚未接入启动链的新 RTOS 应用执行：先系统初始化，再创建必需对象、检查结果，最后 `vTaskStartScheduler()`。已有调度器不重复启动。启动函数返回通常表示内核任务分配或 port 失败，应进入现有故障处理。

对象创建部分失败时，不直接反复重试并泄漏已分配对象。需要可恢复初始化时逐项清理并维护状态；否则保持可诊断的失败状态。构建通过不能证明调度器已运行。

## 创建及使用队列

已有目标队列时先复用其 handle、元素类型和生产者约定；增加处理任务不要求新建队列。仅当需要传递固定大小消息且当前缺少所需队列时创建。包含 `queue.h`；队列元素类型由业务定义，下面假定已定义 `message_t`，容量为实际预算 `queue_length`：

```c
QueueHandle_t queue = xQueueCreate(queue_length, sizeof(message_t));
if (queue == NULL) {
    /* 执行初始化失败处理。 */
}
```

动态分配不可用时查 `xQueueCreateStatic` 并提供长期有效的存储。发送和接收是不同调用点，不要求放在同一任务中：

| 操作 | 调用形式 | 参数与失败处理 |
| --- | --- | --- |
| 任务发送 | `xQueueSend(queue, &message, wait_ticks)` | `message` 为有效消息；队列复制元素，满队列/超时由业务选择丢弃、重试或上报。 |
| 任务接收 | `xQueueReceive(queue, &message, wait_ticks)` | 接收存储至少容纳一个元素；仅成功时消费消息。 |
| ISR 发送 | `xQueueSendFromISR(queue, &message, &higher_woken)` | 不阻塞，检查返回值及是否需要切换任务。 |

任务发送/接收成功返回 `pdPASS`。传值队列允许复用发送局部变量；指针队列只复制指针，必须另行管理缓冲区寿命和所有权。不要因某段代码演示消息发送就创建额外的生产者/消费者任务。

## 等待、周期与超时

原生 API 使用 tick，毫秒用 `pdMS_TO_TICKS(timeout_ms)` 转换。短于一个 tick 的转换可能为 0，是否至少等待一个 tick 由需求决定。`0` 表示不等待，长期等待还须满足 API 和配置条件。

- 相对延时：`vTaskDelay(delay_ticks)`，需启用对应 API。
- 固定周期：查询实际版本的 `xTaskDelayUntil` 或对应接口，维护上次唤醒时刻，避免把工作时间累加到周期中。
- 条件等待：优先等待队列、通知或信号量，使任务在事件到达时运行。

周期来自业务，不统一插入 100 ms、500 ms 或闪灯循环。

## 互斥与事件通知

| 需求 | 选择与约束 |
| --- | --- |
| 多任务独占 I2C/SPI | mutex；明确锁覆盖的完整事务及所有失败出口的释放，ISR 不获取 mutex。 |
| 一个 ISR 唤醒一个任务 | 任务通知或二值信号量；按计数、置位或值覆盖语义选择接口。 |
| 传递数据 | 队列；明确容量、元素类型和溢出处理。 |
| 多条件事件 | event group；核对 ISR 接口及延迟处理条件。 |

互斥量包含 `semphr.h`，按分配策略选择创建 API；`xSemaphoreTake(mutex, wait_ticks)` 成功后执行受保护事务，再 `xSemaphoreGive(mutex)`。异步总线事务不能在刚启动 DMA 时就释放本应覆盖整个传输的锁。

## ISR 与任务交接

仅在中断确需通知任务、对象已创建且优先级合法时使用 `FromISR`。以下为向既有队列提交既有 `message` 的调用形式：

```c
BaseType_t higher_woken = pdFALSE;
BaseType_t sent = xQueueSendFromISR(queue, &message, &higher_woken);
if (sent != pdPASS) {
    /* 使用 ISR 专属计数或既有错误通道记录丢失。 */
}
portYIELD_FROM_ISR(higher_woken);
```

NVIC 数值越小越紧迫。调用这些 API 的 IRQ，其未移位的库级优先级数值须 **>= `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`**；不直接拿移位后的内核值比较。ISR 中不调用阻塞传输、延时或 mutex；HAL 回调可能在中断上下文执行，先确认上下文。

HAL OS port 可能依赖动态 mutex，内核仅允许静态分配时须核对实现。`volatile` 不是并发队列或原子同步的替代。

## 维护已有 CMSIS-RTOS 工程

确认实际适配层与 FreeRTOS 版本兼容。存在 FreeRTOS 内核不等于已有 CMSIS 适配，不只添加头文件就切换 API。

| 项目 | 原生 | CMSIS-RTOS v1 | CMSIS-RTOS v2 |
| --- | --- | --- | --- |
| 头文件 | `FreeRTOS.h`、`task.h` | `cmsis_os.h` | `cmsis_os2.h` |
| 创建线程 | `xTaskCreate` | `osThreadDef` + `osThreadCreate` | `osThreadNew` |
| 延时单位 | tick，毫秒用 `pdMS_TO_TICKS` | `osDelay` 毫秒 | `osDelay` tick |
| 队列发送 | `xQueueSend` 复制元素 | `osMessagePut` 的 32 位消息值 | `osMessageQueuePut` 复制 `msg_size` 字节 |
| 栈单位 | `StackType_t` 个数 | 查实际适配宏 | `osThreadAttr_t.stack_size` 字节 |

仅在需求确为 100 ms 且使用 v2 时，下式说明按当前 tick 频率换算，不能假定一 tick 等于一毫秒：

```c
uint32_t ticks = (osKernelGetTickFreq() + 9U) / 10U;
osDelay(ticks == 0U ? 1U : ticks);
```

v1 对应 `osDelay(100U)`。v1 `osMessagePut(queue, value, timeout_ms)` 与 v2 `osMessageQueuePut(queue, &value, 0U, timeout_ticks)` 的值/指针不同；v2 ISR 队列操作 timeout 须为 0，仍受底层优先级限制。

不同时启动 `osKernelStart()` 和 `vTaskStartScheduler()`。来源：[CMSIS v1](https://arm-software.github.io/CMSIS_5/RTOS/html/cmsis__os_8h.html)、[v2 队列](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html)、[v2 线程](https://arm-software.github.io/CMSIS_6/latest/RTOS2/group__CMSIS__RTOS__ThreadMgmt.html)。

## 检查结果

只检查实际新增或修改的任务和对象：创建成功、调度可达、阻塞与唤醒符合需求、栈和堆有余量、溢出和超时可观察。新增任务后还应确认现有任务及中断行为未被优先级或资源占用改变。
