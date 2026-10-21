# DMA 与 NVIC：从配置到运行

先确定本次是查询、诊断还是新配置 DMA/IRQ。查询和诊断先读现状，不为使参数出现而修改工程。只有需求确需新增 DMA/IRQ 时启用外设自己的父项，再查询动态参数；不假定存在固定通用资源。各检查项用于当前传输，不要求给所有外设启用 DMA。

```powershell
cube mx dma get-allocations -p "$p" -s "$s"
cube mx nvic list-interrupts -p "$p" -s "$s" --used --full
# USART1 仅表示被检查的资源，替换为实际外设
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
cube mx dma --help
cube mx nvic --help
```

`$p`、`$s` 来自[操作基础](operations.md)。`dma bind-resource` 的用途与参数以叶子帮助为准，不能把为 MemToMem 准备的命令套用到所有外设请求。

每条 DMA 传输核对：

1. 请求源与实际外设实例，例如 USART1 RX。
2. 分配的控制器和通道，避免另一个驱动占用。
3. 方向、源/目标递增、源/目标宽度、长度单位。
4. 普通/循环模式、触发方式和链表模式是否与应用一致。
5. DMA 时钟、外设 handle 与 DMA handle 的绑定。
6. IRQ 的使能、优先级、向量和回调。某些传输结束还需外设 IRQ。
7. 缓冲区有效期、对齐、可访问内存及并发所有权。是否需要 cache 维护由实际器件和内存属性决定，不机械套用其他系列的 D-cache 操作。

配置路径可能含 `@`、空格或数组索引。复制完整查询结果并引用路径；`list-parameters` 中的数组标签未必可直接读写，需回读父数组确认索引。

NVIC 的数字越小，抢占紧迫度越高。是否可调用 FreeRTOS `FromISR` API 要比较**未移位**的库级优先级与 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`：允许调用的中断，其数值不得小于阈值。不要拿移位后的 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 直接与界面数值比较。

生成检查：搜索实际 IRQHandler、`HAL_*_IRQHandler`、DMA 绑定和 NVIC 设置。运行检查：依次观察外设状态、DMA 剩余计数、pending、IRQ 入口、回调和应用消费。这样可区分“没有请求”“DMA 已完成但没中断”“中断已到但应用丢数据”。
