# UART：参数、轮询、中断与 DMA

各节对应不同需求，不是从轮询逐步升级到 DMA 的演示。以 USART1、PA9/PA10 说明 C532CCT6、HAL2 2.1.0 的命令路径；实例、引脚、方向和通信参数须按实际工程及对端确定。`$p`、`$s` 来自[操作基础](operations.md)。

## 查询能力及当前配置

本文不覆盖所有 UART 模式，用实际实例替换 USART1：

```powershell
cube mx peripherals view -p "$p" --peripheral USART1
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate
cube mx peripherals enable --help
```

未启用实例时先查询设备能力，不能为只读查询而启用。确认本次是新增串口、修改通信参数、轮询传输、IT 接收还是 DMA；已有工程沿用其他设置。

## 新增串口或更改引脚

仅当需求要求新增或重分配时操作。假定已确认使用 PA9 TX、PA10 RX：

```powershell
cube mx pinout pin -p "$p" --pin PA9
cube mx pinout pin -p "$p" --pin PA10
```

无冲突且需要对应方向时分配；只发送不因文档列出 RX 就分配 RX：

```powershell
# 需要 TX 时
cube mx pinout assign-signal -p "$p" --pin PA9 --signal USART1_TX
# 需要 RX 时
cube mx pinout assign-signal -p "$p" --pin PA10 --signal USART1_RX
```

实例尚未启用且目标为异步 UART 时：

```powershell
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral USART1 --mode Async
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
```

新建时先分配已确定的引脚再启用，避免自动分配与板卡不符。已有实例不重复 enable；局部引脚冲突用实际查询或 GUI 解决，不全局重置。

## 修改通信参数

参数由对端协议决定。仅当目标波特率为 115200 时，可使用：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --value 115200
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
```

数据位、奇偶校验、停止位、方向和流控从 `list-parameters` 取得路径，并查询内部可选值。只改变本次要求的项；不要为了改变波特率顺带重设其他参数。时钟必须能实现目标速率。

## 使用轮询传输

基础配置及 HAL tick 正常即可使用相应阻塞 API，见[固件调用参考](../../stm32c5-firmware/references/peripheral-code.md)。轮询需求不需要启用 UART IRQ、注册回调或 DMA；已有其他业务使用的 IRQ/DMA 也不因此被关闭。

## 使用中断传输

仅在选择 IT API 或其他功能需要 UART IRQ 时配置。假定生成器负责 handler：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path system/nvic/enable_interruption --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path system/nvic/irq_handler_generation --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
```

按返回的 NVIC 路径设置需求对应优先级。已有手写 handler 时保留其所有权，避免重复定义。USART1 的 `enable_interruption` 不可替换为其他外设的 `enable_interrupt`。

**仅当应用采用动态注册回调且当前开关未启用**时：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r HALCore --parameter-path uart_feature/use_register_callback --value true
```

该开关影响 UART 模块的回调机制，先检查其他 UART 用户。沿用弱回调时不必切换。生成后检查宏、向量以及 `HAL_UART_IRQHandler` 的实际 handle；应用启动接收后按完成或错误状态决定是否重新启动。

## 使用 RX DMA

仅当接收需求选择 DMA 时执行；TX DMA 应查询 TX 参数，不能复用 RX 请求。

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path system/dma/enable_dma_rx --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
cube mx dma get-allocations -p "$p" -s "$s"
```

C532 RX 参数前缀为 `system/dma/dma_config_rx@/Request: RX/basic/`。分别查询并按传输需求选择：

| 参数 | 选择依据 |
| --- | --- |
| `request`、`direction` | 实际 UART RX 请求，外设到内存。 |
| `src_inc`、`dest_inc` | 外设地址通常固定；连续缓冲区目标递增。 |
| `src_data_width`、`dest_data_width` | 实际 UART 数据格式及缓冲区布局，8 位数据通常为字节。 |
| `dma_mode` | 有限长度接收或持续流，明确正常/循环模式及消费策略。 |

需要 DMA 完成通知时配置 DMA IRQ，并查询实际 handler、通道及优先级：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path 'system/dma/dma_config_rx@/Request: RX/system/nvic/enable_interruption' --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
```

分配结果可能为 `LPDMA1_CH0`，不能写死所有工程均使用它。IDLE 接收还需要 UART 自身 IRQ；其他模式按实际驱动所需的完成/错误路径配置。DMA 不是“只开 DMA IRQ”即可完成。

## 修改后的检查与诊断

只读查询无需此流程。配置变更按[操作基础](operations.md)保存重载；需更新代码时暂存生成、比较并构建。

- 基础参数：检查 GPIO、时钟、方向和协议；用需求对应的数据与对端核对。
- IT：检查向量、handle、回调及后续接收是否被正确启动。
- DMA：检查时钟、实际通道、`HAL_UART_SetRxDMA`、相关 IRQ 和宏，参见[DMA 检查](dma-irq.md)。
- 接收一帧后停止：查传输结束、重启条件、error/overrun；不要直接改成循环 DMA 掩盖原因。
- 定位异步故障时可在隔离测试中用轮询缩小问题范围，测试结束恢复目标模式；这不是要求每个任务添加另一套传输实现。
