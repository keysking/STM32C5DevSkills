# USART1：轮询、接收中断与 DMA

前提：C532CCT6，USART1 PA9 TX / PA10 RX，HAL2，变量 `$p`、`$s` 已按[操作基础](operations.md)设置。引脚方案先与板卡接线核对。

## 基础配置

新工程先固定引脚：

```powershell
cube mx pinout assign-signal -p "$p" --pin PA9 --signal USART1_TX
cube mx pinout assign-signal -p "$p" --pin PA10 --signal USART1_RX
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral USART1 --mode Async
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --value 115200
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
```

检查数据位、奇偶校验、停止位和方向的实际路径/内部值，与对端保持一致。已有工程不要重新执行 enable；先查现有引脚和参数，只修改需要的值。

## 接收中断

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path system/nvic/enable_interruption --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path system/nvic/irq_handler_generation --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
```

在返回的 `system/nvic/...` 路径中设置实际优先级。不要把另一个外设的 `enable_interrupt` 拼写照搬给 USART1。

回调采用注册方式时：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r HALCore --parameter-path uart_feature/use_register_callback --value true
```

生成后确认 `USE_HAL_UART_REGISTER_CALLBACKS=1`、USART1 向量处理函数调用了对应 handle 的 `HAL_UART_IRQHandler`。应用只启动一次接收，再在完成/错误处理后按设计重新启动；回调签名见[HAL2 代码](../../stm32c5-firmware/references/peripheral-code.md)。

## RX DMA

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path system/dma/enable_dma_rx --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
cube mx dma get-allocations -p "$p" -s "$s"
```

C532 的实际 RX 配置前缀为：

```text
system/dma/dma_config_rx@/Request: RX/basic/
```

其下核对 `request`、`direction`、`src_inc`、`dest_inc`、`src_data_width`、`dest_data_width`、`dma_mode`。UART 8 位数据的常见要求是外设到内存、源地址固定、目标递增、字节宽度；具体值用 `get-parameter-possible-values` 获取。

启用 DMA IRQ 的路径：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path 'system/dma/dma_config_rx@/Request: RX/system/nvic/enable_interruption' --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
```

实际分配可能为 `LPDMA1_CH0`，但不能硬编码所有工程都使用该通道。新返回的 NVIC 子路径包含实际通道名。IDLE 接收还需要 UART 自身中断，不是只开 DMA IRQ。

完成[保存重载](operations.md)后，生成代码应包括 DMA 时钟、通道配置、`HAL_UART_SetRxDMA`、DMA/USART IRQ 和 HAL 配置宏。应用使用长期有效的接收缓冲区；正常模式与循环模式的重启/处理策略不同。

## 验证与故障

- 先轮询发送已知字符串，确认接线、波特率和时钟。
- 再测试固定长度接收，最后测试 IDLE/DMA；不要一次把所有异步路径叠上去。
- 收到一帧后停止：查接收是否结束、是否重新启动、是否残留 error/overrun。
- DMA 数据不变：查实际请求、方向、地址、剩余计数和 IRQ，见[DMA 检查](dma-irq.md)。
- 模型在启用后移动引脚遇到重复资源：保留现有工程，使用 GUI 解决该局部冲突，不全局重置引脚。
