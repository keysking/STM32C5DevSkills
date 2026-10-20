---
name: stm32c5-configure
description: 用 CubeMX2 配置 STM32C5 引脚、时钟、GPIO/EXTI、UART、I2C、SPI、TIM/PWM、ADC、DMA、中断和 FreeRTOS。用于修改 ioc2、查询实际参数、保存重载、生成以及配置不生效的排查。
license: MIT
---

# STM32C5 配置

确认完整型号、引脚接线、工作频率和当前 `.ioc2`；已有工程先检查修改。C5 的默认驱动是 HAL2，命令入口是 `cube mx`。

执行闭环：**查询当前状态 → 查询可选值 → 修改 → 回读 → 保存 → 关闭重载 → 回读 → 导出空目录 → 比较生成代码 → 构建**。按[配置操作基础](references/operations.md)建立 `$p`、`$s` 等变量并执行闭环。退出码为零且响应 `status=success` 才继续。

只读取当前任务对应的资料：

| 任务 | 参考 |
| --- | --- |
| GPIO 输出、外部中断 | [GPIO/EXTI](references/gpio-exti.md) |
| 系统或外设时钟、外部晶振 | [时钟](references/clocks.md) |
| 串口、接收中断、接收 DMA | [UART](references/uart.md) |
| I2C 或 SPI 通信 | [I2C/SPI](references/i2c-spi.md) |
| 定时器、PWM、ADC | [TIM/ADC](references/tim-adc.md) |
| DMA 和 NVIC 共性检查 | [DMA/中断](references/dma-irq.md) |
| FreeRTOS、HAL 时间基准 | [RTOS 配置](references/freertos.md) |

新工程先指定所需引脚，再启用外设，避免自动分配到别的引脚。已有配置冲突时先查占用，不通过全局清空或反复启停外设解决。参数不存在时查当前资源与条件参数，不把另一个系列的参数名作为替代。

重新生成必须执行[暂存导出与合并](../stm32c5-project/references/regenerate.md)。完成时分别说明配置回读、生成代码、构建和实际信号测试结果。
