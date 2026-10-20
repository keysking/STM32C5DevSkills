# TIM1 PWM 与 ADC1 采样

示例器件 `STM32C532CCT6`，HAL2 2.1.0。先设置[工程变量](operations.md)。以下是两个可分别执行的实例。

## TIM1 CH1 → PA8，1 kHz、50%

先确认 TIM1 kernel clock 为 144 MHz。边沿对齐向上计数时：
`fPWM = fTIM / ((PSC + 1) × (ARR + 1))`。
本例 PSC=143、ARR=999、CCR1=500。更换时钟、中心对齐或抖动模式后重新计算。

```powershell
cube mx pinout assign-signal -p "$p" --pin PA8 --signal TIM1_CH1
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral TIM1
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path basic/time_base/clock_source_configuration/kernel_clock_frequency
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels
```

**数组路径要用实际元素索引。** 此版本 `list-parameters` 显示 `channels/CC1/...`，但读写对象数组实际使用 `channels/0/...`。先检查 `channels` 返回的 `_itemTitle` 和 `_rank`，确认索引 0 是 Channel 1。

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/use_channel --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/channel_direction --value Output
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path basic/time_base/clock_source_configuration/prescaler --value 143
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path basic/time_base/counter/period --value 999
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/compare_unit_configuration/mode --value PWM1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/compare_unit_configuration/pulse --value 500
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/ocx_channel_configuration/use_channel --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/ocx_channel_configuration/chx_gpio --value true
```

最后一个 GPIO 开关不可省略：有 compare unit 和输出通道，不代表生成器一定生成对应 GPIO 初始化。

保存重载、生成后检查 PA8 AF、prescaler、period、PWM1、pulse 和输出极性。应用还需按[HAL2 PWM 启动](../../stm32c5-firmware/references/peripheral-code.md)开启计数器和输出通道。TIM1 属于高级定时器，检查输出使能和 break 状态。

中断与 DMA：先列 TIM1 参数，区分 update、capture/compare 等事件，各自的 DMA 请求不同。不要把 UART 的 RX 请求用于 TIM。需要周期回调时使用实际 HAL2 `HAL_TIM_Start_IT` 及回调配置，IRQ 中不做长时间阻塞。

验证使用示波器/逻辑分析仪测 PA8 的周期和高电平宽度。单看 CNT 递增不能证明 PWM 引脚输出正确。

## ADC1 IN0 → PA0，12 位单次轮询

确认输入电压在该引脚允许范围内，参考电压与电路匹配；先选择足够长的采样时间，兼顾信号源阻抗。

```powershell
cube mx pinout assign-signal -p "$p" --pin PA0 --signal ADC1_IN0
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral ADC1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/channels/ADC1_IN0/use_channel --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/resolution --value 12
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/group_panel/reg_config_t/reg_sequencer
```

新工程规则序列为空时添加一个 rank：

```powershell
cube mx sw-config add-array-element -p "$p" -s "$s" -r ADC1 --parameter-path basic/group_panel/reg_config_t/reg_sequencer -e '{}'
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/group_panel/reg_config_t/reg_sequencer
```

本例只有 IN0 已分配时，新元素应绑定 `ADC1_IN0`、`_rank=0`。已有序列不能重复添加；若绑定了别的通道，查询数组元素的实际 channel 可选值并调整，不手工伪造包含内部 queryId 的对象。

核对 `basic/channels/ADC1_IN0/sampling_time`、规则组触发源、`continuous=false`、overrun 策略；采样时间先执行 `get-parameter-possible-values`。生成代码应包含 analog GPIO、ADC kernel clock、规则序列和采样配置。

HAL2 需要区分 ADC 启动与转换启动。按[ADC 应用片段](../../stm32c5-firmware/references/peripheral-code.md)校准、启动 ADC、触发规则转换、等待并读取。

## ADC 中断与 DMA 扩展

`system/nvic/enable_interruption` 打开 ADC IRQ；生成对应 handler，使用 `HAL_ADC_REG_StartConv_IT`。

DMA 父项为 `system/dma/dma_enable`。启用后查询实际绑定、方向、宽度、模式和 IRQ。HAL2 `HAL_ADC_REG_StartConv_DMA` 接收 `uint8_t *` 和 **字节数**；若缓冲区是 `uint16_t samples[32]`，传 `(uint8_t *)samples` 与 `sizeof samples`，不能直接传 32 充当字节数。

多通道要检查 rank 顺序、每个样本宽度和缓冲区索引。连续采样还需检查触发周期、DMA 循环模式和消费速度。用 GND、已知电压和重复采样确认转换范围、噪声及超时，不把构建成功作为模拟性能验证。
