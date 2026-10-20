# FreeRTOS 运行异常

## 任务完全不运行

1. 确认 main 实际调用对象创建函数和调度器启动，不只生成了 `mx_freertos_app.c`。
2. 检查 `xTaskCreate`、队列/互斥量创建返回值，堆实现只一份。
3. 调度器返回时检查 idle/timer 任务内存、port、断言；保留失败位置。
4. 检查 SysTick、SVC、PendSV 的实际链接符号。C5 的 CM33_NTZ port 已提供相应 handler，不额外复制其他系列的旧中断模板。
5. 核对 `configENABLE_FPU`、浮点 ABI、TrustZone/MPU 与实际器件和 port 组合。

## 某任务卡死或饿死

- 看任务状态：等待哪个队列、锁或通知；超时单位是否误用。
- 高优先级任务是否一直就绪，空循环是否没有阻塞。
- mutex 是否被遗忘释放，是否有循环锁依赖，是否在 ISR 误用。
- ISR 是否频繁触发或没有清 pending，导致线程得不到时间。
- HAL 阻塞调用所依赖的 tick 是否正常；HAL tick 与 RTOS tick 分开观察。

## 栈、堆和数据破坏

启用项目支持的 stack overflow/malloc failed hook，并实现相应函数。观察 `uxTaskGetStackHighWaterMark` 等接口前确认 include 配置宏；单位仍为 StackType_t。

比较最小剩余堆、失败分配和最大调用深度。printf、浮点格式化、协议解析、递归和大型局部数组可能扩大栈需求。增加栈可以用于验证假设，但最终要解释超出预期的原因。

DMA 缓冲不应指向已返回函数的局部数组；队列传指针时明确释放者。`volatile` 不能解决多个任务的读改写竞争。

## ISR 优先级与断言

检查 NVIC 库级数值、`configPRIO_BITS` 和最大 syscall 阈值。调用 `FromISR` 的 IRQ 数值不得小于 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`，并根据 `higherPriorityTaskWoken` 请求切换。

不要把所有 IRQ 都改成最高优先级；不要关闭 configASSERT 来掩盖非法调用。若故障在中断返回/任务切换时出现，检查栈破坏、FPU 上下文和 EXC_RETURN，而非直接认定 RTOS 内核缺陷。

## GDB 与任务视图

基础 GDB 先读当前线程栈、PC/LR、任务/队列计数和 `app_last_value` 等观察点。任务感知插件或 `cube rtos-proxy` 需要确认当前版本、配置与内核兼容性，按本机帮助接入；没有解析成功时不要展示虚构的任务列表。

修复后执行与问题匹配的压力、阻塞、溢出和恢复场景，明确测试时长与输入。配置入口见[RTOS 配置](../../stm32c5-configure/references/freertos.md)，应用与 API 约束见[RTOS 固件](../../stm32c5-firmware/references/freertos.md)。
