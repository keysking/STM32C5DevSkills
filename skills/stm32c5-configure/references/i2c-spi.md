# I2C 与 SPI 配置参考

本文按 I2C、SPI 及其传输方式分别使用，不要求同时配置两种总线。路径依据 C532CCT6、HAL2 2.1.0；先设置[工程变量](operations.md)。I2C1、SPI1、引脚和数值用于说明命令语法，实际参数来自板卡、器件协议和现有工程。

## 查询实例、模式及参数

本文不覆盖所有选项。以实际目标替换 I2C1：

```powershell
cube mx peripherals view -p "$p" --peripheral I2C1
cube mx peripherals enable --help
cube mx sw-config list-parameters -p "$p" -s "$s" -r I2C1
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r I2C1 --parameter-path basic/addressing_mode
```

未启用时先查设备能力，不能为查询而启用。新建分配时先核对引脚、固定需要的信号再启用；已有实例只修改需求涉及的项。

## 新增 I2C 总线

仅当实际需要 I2C1 且选择 PB8/PB9 时，参考以下新增分配形式；已有总线跳过此块。

```powershell
cube mx pinout pin -p "$p" --pin PB8
cube mx pinout pin -p "$p" --pin PB9
cube mx pinout assign-signal -p "$p" --pin PB8 --signal I2C1_SCL
cube mx pinout assign-signal -p "$p" --pin PB9 --signal I2C1_SDA
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral I2C1 --mode I2C
```

## 修改 I2C 速度与地址模式

分别按对端要求选择。下列数值仅适用于目标为 100 kHz、7 位寻址；只改速度时不要顺带改寻址模式。

```powershell
# 目标总线速度为 100 kHz 时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r I2C1 --parameter-path basic/speed_frequency_hz --value 100000
# 目标寻址模式为 7 位且需要改变该项时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r I2C1 --parameter-path basic/addressing_mode --value 7
# 读取模型计算出的时序
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r I2C1 --parameter-path basic/timing --use-internal-value
```

检查开漏模式、外部上拉、电压、线长及从设备速度限制。`own_address1` 是 MCU 从机地址，不是传感器地址。主机目标地址在应用 API 中传入。

生成后检查 I2C kernel clock、时序值、模拟/数字滤波及 GPIO。切换 400 kHz 时先查询可选值并核对上升/下降时间；不能只改常量而保留与原时钟不匹配的 timing。

## I2C 中断或 DMA

轮询传输不需要本节。只有选择异步 API 或需处理相应事件时配置；IRQ 路径是 `system/nvic/enable_event_interruption` 与 `system/nvic/enable_error_interruption`；对应的 handler 生成开关分别是 `event_irq_handler_generation`、`error_irq_handler_generation`。使用异步 API 时两类事件都要按驱动要求处理。

DMA 父项为 `system/dma/enable_dma_tx`、`enable_dma_rx`；启用后重新列参数，按[DMA 检查](dma-irq.md)确认请求、方向、计数单位和 IRQ。

应用使用 `HAL_I2C_MASTER_Transmit` 等 HAL2 API，见[API 用法](../../stm32c5-firmware/references/peripheral-code.md)。7 位器件地址如 `0x3C` 在该 HAL2 接口中左移一位传入；不要二次移位。

## 新增 SPI 总线

仅当实际选择 SPI1、PA5/PA6/PA7 且需要这些信号时分配；已有 GPIO 占用 PA5 时不能直接覆盖。先查询各引脚支持和占用。半双工或仅发送时按实际方向选择信号，不固定要求 MISO/MOSI 都存在。

```powershell
cube mx pinout assign-signal -p "$p" --pin PA5 --signal SPI1_SCK
cube mx pinout assign-signal -p "$p" --pin PA6 --signal SPI1_MISO
cube mx pinout assign-signal -p "$p" --pin PA7 --signal SPI1_MOSI
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral SPI1 --mode SPI
```

## 修改 SPI 协议与时钟

先查实际 `list-parameters` 和各项 `get-parameter-possible-values`。下面分别列出主机、全双工、8 位、软件片选、Mode 0 及分频 32 的写入形式；每行只在需求对应时使用，不作为统一配置组合。

```powershell
# 需要选择主机角色时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/mode --value MASTER
# 需要全双工时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/direction --value FULL_DUPLEX
# 器件协议要求 8 位帧时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/data_size/data_width --value 8
# 选择软件片选、内部 NSS 管理时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/nss_pin_management --value INTERNAL
# 器件协议要求 Mode 0 时，CPOL 与 CPHA 成对设置
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/clock_polarity --value LOW
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/clock_phase --value 1
# 已按实际 kernel clock 和目标 SCK 算得分频为 32 时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/baud_rate_prescaler --value 32
# 读取实际 SCK
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r SPI1 --parameter-path basic/timing/baud_rate
```

`LOW + 1` 对应 Mode 0；不是所有器件都用 Mode 0。实际 SCK 由 kernel clock 和分频决定，必须小于器件允许值。选择软件 CS 时在实际空闲 GPIO 上配置输出和正确空闲电平，并由应用控制事务边界。若用硬件 NSS，选择对应模式并分配 NSS 引脚，而不是同时假定软件 CS 生效。

## SPI 中断或 DMA

仅在需求采用对应异步方式时启用。SPI IRQ 父项为 `system/nvic/enable_interrupt`，与 UART 的拼写不同。DMA 开关为 `system/dma/enable_dma_tx`、`enable_dma_rx`；启用后查询动态路径。

## 完成检查

只对本次实际修改的总线保存重载；需要更新固件时暂存生成、检查差异并构建。纯查询无需这些操作。实际测试用设备 ID 或已知寄存器值，避免只发送不检查响应。

I2C 无 ACK：检查地址格式、上拉、接线、器件供电和复位。SPI 返回全 0/全 1：查 CS、模式、MISO、时钟和器件命令。异步回调只到一次时检查 DMA 模式和传输重新启动，不能直接归因于线材。
