# FreeRTOS 任务、队列与同步

先完成[生成器配置](../../stm32c5-configure/references/freertos.md)：正确 CM33 port、HAL 定时器 tick、堆、优先级和 FPU。新工程用原生 API；已有工程保持原生/CMSIS v1/v2 的当前选择。

## 原生 API：两个任务传递计数

将下面代码放进自行维护的 `app.c`，加入当前 CMake target。前提：动态分配开启，heap_4 已纳入构建，`INCLUDE_vTaskDelay=1`，至少支持优先级 1。应用自己创建任务时不再调用生成器的 `app_synctasks_init()`。

```c
#include <stdint.h>
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

static QueueHandle_t samples;
volatile uint32_t app_last_value;
volatile uint32_t app_queue_drops;

static void producer(void *argument)
{
    (void)argument;
    uint32_t value = 0U;
    for (;;) {
        ++value;
        if (xQueueSend(samples, &value, 0U) != pdPASS) {
            ++app_queue_drops;
        }
        vTaskDelay(pdMS_TO_TICKS(100U));
    }
}

static void consumer(void *argument)
{
    (void)argument;
    uint32_t value;
    for (;;) {
        if (xQueueReceive(samples, &value, portMAX_DELAY) == pdPASS) {
            app_last_value = value;
        }
    }
}

int app_rtos_create(void)
{
    samples = xQueueCreate(4U, sizeof(uint32_t));
    if (samples == NULL) {
        return -1;
    }
    if (xTaskCreate(producer, "producer", 256U, NULL, 1U, NULL) != pdPASS) {
        return -1;
    }
    if (xTaskCreate(consumer, "consumer", 256U, NULL, 1U, NULL) != pdPASS) {
        return -1;
    }
    return 0;
}
```

队列复制 `uint32_t` 值，生产者局部变量可以复用。若队列元素改成指针，必须重新设计缓冲区寿命。256 的单位是 `StackType_t`，在该 port 通常为 32 位字；通过栈水位测量调整，不能只依赖示例值。

在 `main.c` 中增加声明和头文件，并在系统初始化成功后启动：

```c
#include "FreeRTOS.h"
#include "task.h"
int app_rtos_create(void);

/* 放入 main 中，位于 mx_system_init() 成功之后。 */
if (app_rtos_create() != 0) {
    for (;;) { /* 初始化失败；保持可调试状态，不启动半成品任务集。 */ }
}
vTaskStartScheduler();
for (;;) { /* 调度器意外返回，检查 idle/timer 任务分配和 port。 */ }
```

此初始化函数只调用一次；失败后不反复重试并泄漏已分配对象。产品需要可恢复初始化时，应增加逐项清理和状态管理。

构建后在 GDB 观察 `app_last_value` 增长和 `app_queue_drops`，再加入 UART/LED 等实际输出。构建成功只说明应用已链接，不证明调度器已在硬件运行。

## 同步选择

| 需求 | 选择与约束 |
| --- | --- |
| 一个 ISR 唤醒一个任务 | 任务通知或二值信号量；使用 `FromISR` 版本 |
| 传递固定大小数据 | 队列，检查满队列/超时 |
| 多任务独占 I2C/SPI | mutex，明确持有范围与失败释放；ISR 不获取 mutex |
| 多条件事件 | event group，确认允许的 ISR 接口及延迟处理 |
| 周期任务 | `xTaskDelayUntil`/版本对应 API；防止累积漂移 |

典型 ISR 队列发送，要求队列已创建、优先级合法：

```c
BaseType_t higher_woken = pdFALSE;
uint32_t event = 1U;
BaseType_t sent = xQueueSendFromISR(samples, &event, &higher_woken);
if (sent != pdPASS) {
    /* 使用 ISR 专属计数器记录丢失，避免与任务无保护读改写同一变量。 */
}
portYIELD_FROM_ISR(higher_woken);
```

NVIC 数值越小越紧迫。调用 `FromISR` 的 IRQ，其库级优先级数值必须 **>= `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`**。例如阈值为 5 时，0–4 的 IRQ 不调用这些 API。不能仅因为 API 名带 FromISR 就忽略优先级限制。

HAL OS port 若使用 FreeRTOS mutex，要检查其动态分配需求；“内核只允许静态分配”与该实现可能不兼容。HAL 阻塞传输放在任务中，回调和 ISR 中只做短工作。

## 维护已有 CMSIS-RTOS 工程

先确认工程实际提供兼容 FreeRTOS 版本的适配层。C5 Pack 提供 FreeRTOS 内核不等于自动包含全部 CMSIS 适配。不要只添加 `cmsis_os2.h` 就认为实现已存在。

| 项目 | 原生 | CMSIS-RTOS v1 | CMSIS-RTOS v2 |
| --- | --- | --- | --- |
| 头文件 | `FreeRTOS.h`、`task.h` | `cmsis_os.h` | `cmsis_os2.h` |
| 创建线程 | `xTaskCreate` | `osThreadDef` + `osThreadCreate` | `osThreadNew` |
| 延时参数 | tick，使用 `pdMS_TO_TICKS` | `osDelay` 的毫秒 | `osDelay` 的 tick |
| 队列发送 | `xQueueSend`，复制元素 | `osMessagePut` 的 32 位消息值 | `osMessageQueuePut`，复制 `msg_size` 字节 |
| 栈单位 | StackType_t 个数 | 检查当前适配宏定义 | `osThreadAttr_t.stack_size` 字节 |

v2 的 100 ms 延时，不能假定 tick 总是 1 ms：

```c
uint32_t ticks = (osKernelGetTickFreq() + 9U) / 10U;
osDelay(ticks == 0U ? 1U : ticks);
```

v1 可用 `osDelay(100U)` 表示 100 ms；v1 `osMessagePut(queue, value, timeout_ms)` 与 v2 `osMessageQueuePut(queue, &value, 0U, timeout_ticks)` 的值/指针含义不同。v2 在 ISR 调队列时 timeout 必须为 0，仍受底层 FreeRTOS 的中断优先级限制。

不要同时启动 `osKernelStart()` 和 `vTaskStartScheduler()`；也不要混用两个适配层创建同一对象。来源：[CMSIS v1](https://arm-software.github.io/CMSIS_5/RTOS/html/cmsis__os_8h.html)、[CMSIS v2 队列](https://arm-software.github.io/CMSIS_6/main/RTOS2/group__CMSIS__RTOS__Message.html)、[v2 线程](https://arm-software.github.io/CMSIS_6/latest/RTOS2/group__CMSIS__RTOS__ThreadMgmt.html)。
