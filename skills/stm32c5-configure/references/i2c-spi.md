# I2C1 与 SPI1 参数实例

前提：C532CCT6、HAL2 2.1.0；先设置[工程变量](operations.md)。新工程先固定引脚再启用；已有工程先检查当前占用。

## I2C1：PB8/PB9，100 kHz

```powershell
cube mx pinout pin -p "$p" --pin PB8
cube mx pinout pin -p "$p" --pin PB9
cube mx pinout assign-signal -p "$p" --pin PB8 --signal I2C1_SCL
cube mx pinout assign-signal -p "$p" --pin PB9 --signal I2C1_SDA
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral I2C1 --mode I2C
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r I2C1 --parameter-path basic/speed_frequency_hz --value 100000
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r I2C1 --parameter-path basic/addressing_mode --value 7
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r I2C1 --parameter-path basic/timing --use-internal-value
```

检查开漏模式、外部上拉、电压、线长及从设备速度限制。`own_address1` 是 MCU 从机地址，不是传感器地址。主机目标地址在应用 API 中传入。

生成后检查 I2C kernel clock、时序值、模拟/数字滤波及 GPIO。切换 400 kHz 时先查询可选值并核对上升/下降时间；不能只改常量而保留与原时钟不匹配的 timing。

IRQ 路径是 `system/nvic/enable_event_interruption` 与 `system/nvic/enable_error_interruption`；对应的 handler 生成开关分别是 `event_irq_handler_generation`、`error_irq_handler_generation`。使用异步 API 时两类事件都要按驱动要求处理。

DMA 父项为 `system/dma/enable_dma_tx`、`enable_dma_rx`；启用后重新列参数，按[DMA 检查](dma-irq.md)确认请求、方向、计数单位和 IRQ。

应用使用 `HAL_I2C_MASTER_Transmit` 等 HAL2 API，见[代码实例](../../stm32c5-firmware/references/peripheral-code.md)。7 位器件地址如 `0x3C` 在该 HAL2 接口中左移一位传入；不要二次移位。

## SPI1：PA5/PA6/PA7，Mode 0、8 位

与 GPIO PA5 闪灯实例互斥；CS 在另一个经确认的空闲 GPIO 上配置输出，并在空闲时保持器件要求的电平。

```powershell
cube mx pinout assign-signal -p "$p" --pin PA5 --signal SPI1_SCK
cube mx pinout assign-signal -p "$p" --pin PA6 --signal SPI1_MISO
cube mx pinout assign-signal -p "$p" --pin PA7 --signal SPI1_MOSI
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral SPI1 --mode SPI
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/mode --value MASTER
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/direction --value FULL_DUPLEX
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/data_size/data_width --value 8
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/nss_pin_management --value INTERNAL
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/clock_polarity --value LOW
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/clock_phase --value 1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/baud_rate_prescaler --value 32
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/baud_rate
```

`LOW + 1` 对应 Mode 0；不是所有器件都用 Mode 0。实际 SCK 由 kernel clock 和分频决定，必须小于器件允许值。若用硬件 NSS，改为对应模式并分配 NSS 引脚，而不是同时假定软件 CS 生效。

SPI IRQ 父项为 `system/nvic/enable_interrupt`，与 UART 的拼写不同。DMA 开关为 `system/dma/enable_dma_tx`、`enable_dma_rx`；启用后查询动态路径。

## 完成检查

两种总线都要保存重载、暂存生成、检查差异并构建。实际测试用设备 ID 或已知寄存器值，避免只发送不检查响应。

I2C 无 ACK：检查地址格式、上拉、接线、器件供电和复位。SPI 返回全 0/全 1：查 CS、模式、MISO、时钟和器件命令。异步回调只到一次时检查 DMA 模式和传输重新启动，不能直接归因于线材。
