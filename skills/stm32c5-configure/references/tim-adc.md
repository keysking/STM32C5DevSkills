# 定时器、PWM 与 ADC 配置参考

路径依据 `STM32C532CCT6`、HAL2 2.1.0。先设置[工程变量](operations.md)。按计时、PWM 或 ADC 的实际需求选节；它们不是一个应用的连续步骤。实例、引脚、频率、占空比和精度均须从需求确定。

## 查询能力与参数

本文不覆盖所有模式。将 TIM1 替换为实际目标，查询设备能力、当前参数与叶子帮助：

```powershell
cube mx peripherals view -p "$p" --peripheral TIM1
cube mx sw-config list-parameters -p "$p" -s "$s" -r TIM1
cube mx sw-config get-parameter-possible-values --help
cube mx sw-config get-parameter-value --help
```

未启用实例先查能力；纯查询不改变配置。只需内部定时或周期中断时，检查 time base、事件源和 NVIC，不分配 PWM 输出引脚或开启 OC 通道。

## 配置 PWM 输出

仅当需要 PWM 时配置输出通道。读取实际 timer kernel clock；边沿对齐向上计数时：
`fPWM = fTIM / ((PSC + 1) × (ARR + 1))`。
假定需求为 1 kHz、50%，且实际时钟为 144 MHz，可算得 PSC=143、ARR=999、CCR1=500。下列数值仅说明该组计算结果的写入形式。实际需求不同时重新计算；中心对齐或抖动模式不能直接套用此公式。

```powershell
# 仅在新增 PWM 分配、实际选择 PA8/TIM1_CH1 时执行；先查询 PA8 占用
cube mx pinout pin -p "$p" --pin PA8
cube mx pinout assign-signal -p "$p" --pin PA8 --signal TIM1_CH1
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral TIM1
```

已有实例跳过分配/启用，读取实际时钟和通道：

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path basic/time_base/clock_source_configuration/kernel_clock_frequency
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels
```

**数组路径要用实际元素索引。** 此版本 `list-parameters` 显示 `channels/CC1/...`，但读写对象数组实际使用 `channels/0/...`。先检查 `channels` 返回的 `_itemTitle` 和 `_rank`，确认索引 0 是 Channel 1。下面通道配置仅用于所选 PWM 通道；仅改变占空比时只改 pulse，不重设通道及定时器时基。改变 PSC/ARR 会影响同一定时器的其他通道，先检查共用需求。

### 新增 PWM 输出通道

需要建立 Channel 1 的 PWM1 输出时配置以下通道项；已有正确配置无需重设：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/use_channel --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/channel_direction --value Output
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/compare_unit_configuration/mode --value PWM1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/ocx_channel_configuration/use_channel --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/ocx_channel_configuration/chx_gpio --value true
```

最后一个 GPIO 开关不可省略：有 compare unit 和输出通道，不代表生成器一定生成对应 GPIO 初始化。

### 修改周期

需要建立或改变 PWM 频率时，按实际 kernel clock 计算 PSC/ARR。仅当计算结果为 143/999 时，写入形式为：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path basic/time_base/clock_source_configuration/prescaler --value 143
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path basic/time_base/counter/period --value 999
```

### 修改占空比

需要建立或改变占空比时，结合周期、PWM 模式和极性计算 pulse。仅当目标比较值为 500 时：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r TIM1 --parameter-path channels/0/output_configuration/compare_unit_configuration/pulse --value 500
```

### 检查 PWM 变更

保存重载、生成后检查 PA8 AF、prescaler、period、PWM1、pulse 和输出极性。应用还需按[HAL2 PWM 启动](../../stm32c5-firmware/references/peripheral-code.md)开启计数器和输出通道。TIM1 属于高级定时器，检查输出使能和 break 状态。

中断与 DMA：先列 TIM1 参数，区分 update、capture/compare 等事件，各自的 DMA 请求不同。不要把 UART 的 RX 请求用于 TIM。需要周期回调时使用实际 HAL2 `HAL_TIM_Start_IT` 及回调配置，IRQ 中不做长时间阻塞。

验证使用示波器/逻辑分析仪测 PA8 的周期和高电平宽度。单看 CNT 递增不能证明 PWM 引脚输出正确。

## 新增 ADC 输入或修改采样参数

仅当需要 ADC 输入时操作。假定目标选择 ADC1_IN0/PA0；确认输入电压、参考电压和源阻抗。新增时分配并启用，已有实例不重复 enable。12 位仅用于说明分辨率设置，实际精度按需求选择。

```powershell
cube mx pinout pin -p "$p" --pin PA0
cube mx pinout assign-signal -p "$p" --pin PA0 --signal ADC1_IN0
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral ADC1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/channels/ADC1_IN0/use_channel --value true
```

修改已有 ADC 精度时只处理分辨率及其必要依赖。目标确为 12 位时：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/resolution --value 12
```

## 添加或修改规则转换序列

先读取序列：

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/group_panel/reg_config_t/reg_sequencer
```

只有确需新增通道且当前序列缺少对应 rank 时添加；修改现有 rank 不执行添加：

```powershell
cube mx sw-config add-array-element -p "$p" -s "$s" -r ADC1 --parameter-path basic/group_panel/reg_config_t/reg_sequencer -e '{}'
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r ADC1 --parameter-path basic/group_panel/reg_config_t/reg_sequencer
```

若只有 IN0 已分配，新元素应绑定 `ADC1_IN0`、`_rank=0`。已有序列不能重复添加；若绑定了别的通道，查询数组元素的实际 channel 可选值并调整，不手工伪造包含内部 queryId 的对象。

按需求选择 `basic/channels/ADC1_IN0/sampling_time`、规则组触发源、连续模式和 overrun 策略；单次软件触发通常关闭 continuous，连续或外部触发须使用对应设置。采样时间先执行 `get-parameter-possible-values`。生成代码应包含 analog GPIO、ADC kernel clock、规则序列和采样配置。

HAL2 需要区分 ADC 启动与转换启动。仅使用轮询读取时，按[ADC API 顺序](../../stm32c5-firmware/references/peripheral-code.md)先启动 ADC、校准，再触发规则转换、等待并读取。轮询不要求启用 IRQ 或 DMA。

## 选择 ADC 中断或 DMA

需要中断转换时，`system/nvic/enable_interruption` 打开 ADC IRQ；生成对应 handler，使用 `HAL_ADC_REG_StartConv_IT`。

需要 DMA 采样时，DMA 父项为 `system/dma/dma_enable`。启用后查询实际绑定、方向、宽度、模式和 IRQ。HAL2 `HAL_ADC_REG_StartConv_DMA` 接收 `uint8_t *` 和 **字节数**；若缓冲区是 `uint16_t samples[32]`，传 `(uint8_t *)samples` 与 `sizeof samples`，不能直接传 32 充当字节数。

配置修改后保存重载；需要更新代码时暂存生成、比较并构建。多通道要检查 rank 顺序、每个样本宽度和缓冲区索引。连续采样还需检查触发周期、DMA 循环模式和消费速度。用 GND、已知电压和重复采样确认转换范围、噪声及超时，不把构建成功作为模拟性能验证。
