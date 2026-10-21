# GPIO 输入、输出与外部中断

本文按场景提供命令和 API 参考，用于理解用途与语法。具体引脚、模式和参数须由用户需求、原理图及现有配置确定；PA5、PB0 仅用于说明语法，不代表目标板的 LED 或按键。各节独立选择，输出配置不要求启用 EXTI 或切换 LL。

路径依据 C532CCT6、CubeMX2 1.1.2、HAL2 2.1.0，其他封装或版本先核对能力。`$p`、`$s` 按[操作基础](operations.md)设置。命令为 PowerShell，C 调用用于固件。

## 查询引脚与可用参数

本文不覆盖所有模式及参数。操作前用实际引脚替换 PA5，查询当前模型：

```powershell
cube mx pinout pin --help
cube mx pinout pin -p "$p" --pin PA5
cube mx sw-config list-parameters -p "$p" -s "$s" -r GPIO
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/mode --use-internal-value
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/mode
cube mx sw-config set-parameter-value --help
```

未分配 GPIO 时，对应参数可能不存在。先通过 `pinout pin` 确认引脚存在、支持 GPIO 及当前占用；只有需求要求变更分配时才执行 `assign-signal`。已用于 SPI 等外设的引脚不能直接改占为 GPIO。

## 配置输出

仅当目标引脚应作为数字输出时执行。假定实际选择 PA5；已有 GPIO 输出只修改本次要求的参数，不重复分配。

```powershell
# 需要新建或变更 GPIO 分配时
cube mx pinout assign-signal -p "$p" --pin PA5 --signal GPIO
# 需要将模式设为输出时
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/mode --value OUTPUT
```

按负载确定初始电平、推挽/开漏、上下拉和速率。先查当前值、类型和枚举，再设置本次需要改变的字段：

| PA5 的参数路径 | 选择依据 |
| --- | --- |
| `pins/PA5/basic/output_init_state` | 初始化期间负载应保持的状态，考虑使能、复位或片选的有效极性。 |
| `pins/PA5/basic/output_type` | 推挽或开漏；开漏高电平依赖上拉。 |
| `pins/PA5/basic/pull` | 外部电路与空闲状态，不能自动假定上拉。 |
| 速率：从 `list-parameters` 取得路径 | 满足负载和边沿要求，不无依据选择最高速率。 |

例如要改变初始电平，先查询：

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/output_init_state --use-internal-value
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/output_init_state
```

按[参数修改方法](operations.md)写入需求对应的内部值并回读。修改后保存重载、暂存生成，确认 GPIOA 时钟、PA5 模式、电气参数和初始电平与需求一致。

## 在固件中写入、读取或翻转电平

适用于 HAL2 GPIO。先确认生成初始化成功，例如 `mx_system_init()` 返回成功；端口和引脚宏替换为实际对象。下表是各操作的调用形式，按需要选择，不按表逐项执行。

| 操作 | HAL2 调用（假定 PA5） | 条件与结果 |
| --- | --- | --- |
| 输出高电平 | `HAL_GPIO_WritePin(HAL_GPIOA, HAL_GPIO_PIN_5, HAL_GPIO_PIN_SET);` | 推挽驱动高；开漏释放输出，实际电平还取决于上拉及负载。 |
| 输出低电平 | `HAL_GPIO_WritePin(HAL_GPIOA, HAL_GPIO_PIN_5, HAL_GPIO_PIN_RESET);` | 将输出置低。 |
| 读取电平 | `hal_gpio_pin_state_t level = HAL_GPIO_ReadPin(HAL_GPIOA, HAL_GPIO_PIN_5);` | 读取输入数据中的电平，可观察输入或输出引脚，不等同于读取输出锁存值。 |
| 翻转输出 | `HAL_GPIO_TogglePin(HAL_GPIOA, HAL_GPIO_PIN_5);` | 仅在需要翻转时使用，不隐含周期、延时或闪灯功能。 |

在项目 `stm32c5xx_hal_gpio.h` 中检索上述函数声明；也可查 [HAL2 GPIO API](https://dev.st.com/stm32cube-docs/stm32c5xx-hal-drivers/2.1.0/en/docs/drivers/hal_drivers/gpio/api/hal_gpio_exported_functions.html)。`HAL_Delay` 不是 GPIO 操作的必要步骤；周期动作采用应用已有的调度或定时机制。

按需求测量引脚和负载的高/低电平。读值与实测不一致时，检查复用、开漏上拉、外部驱动和接线。

## 配置普通输入

仅当需要读取数字输入时执行，普通输入不需要 EXTI。假定选择 PB0，先查占用，再在需要时分配并设为输入：

```powershell
cube mx pinout pin -p "$p" --pin PB0
cube mx pinout assign-signal -p "$p" --pin PB0 --signal GPIO
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PB0/basic/mode --value INPUT
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path pins/PB0/basic/pull
```

按电路选择无上下拉、上拉或下拉，回读后保存并生成。固件用 `HAL_GPIO_ReadPin(HAL_GPIOB, HAL_GPIO_PIN_0)` 读取。悬空输入不能作为稳定逻辑值，有效极性由信号定义决定。

## 配置 EXTI 边沿中断

仅当输入需要边沿通知时执行。假定已配置 PB0 输入，上下拉沿用实际电路，不自动改为按键用途：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PB0/basic/enable_exti --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r GPIO
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/trigger_detection'
```

选择需求对应的上升沿、下降沿或双边沿。**仅当需求为下降沿**时，写入形式如下；其他边沿使用查询返回的对应内部值：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/trigger_detection' --value FALLING
```

需要 CPU 中断且由生成器维护 handler 时启用以下两项。已有手写 handler 时先确认所有者，避免重复定义：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/interruption/enable_interruption' --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/interruption/irq_handler_generation' --value true
```

回读边沿及 `.../nvic@/EXTI0/priority`，按是否调用 RTOS API 选择优先级。同一 EXTI 线的 GPIO 来源须唯一，不能把 PA0、PB0 同时独占绑定 EXTI0。仅需硬件事件而非 CPU 中断时查询事件参数，不套用中断使能。

保存重载并暂存生成；检查来源、触发配置、`EXTI0_IRQHandler` 和 handle。getter 可能为 `mx_gpio_default_exti0_gethandle()`，以实际头文件为准。回调注册见[固件 API 参考](../../stm32c5-firmware/references/peripheral-code.md)。

回调只记录事件或发送合法 ISR 通知；机械输入需要消抖时在主循环/任务处理。失败时依次检查电平、EXTI 来源和边沿、pending、NVIC、向量及回调，见[中断诊断](../../stm32c5-debug/references/peripherals.md)。

## 选择 HAL 或 LL 驱动层

普通 GPIO 操作沿用现有驱动层。仅当任务要求 LL，或已确定需要迁移驱动实现时修改。此项作用于 **GPIO 资源**，不能理解为只切换 PA5；先检查该资源其他引脚和调用代码。

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path info/layer --use-internal-value
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path info/layer
```

确认迁移范围且目标为 LL 时：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path info/layer --value LL
```

回读、保存重载、暂存生成，检查受影响的初始化及应用 API。生成文件仍可能在 `generated/hal/`，应检查内容。LL 初始化可能省略复位默认值，不能作为任意运行状态下的完整寄存器复原操作。
