# GPIO 输出与 PB0 外部中断

前提：C532CCT6，HAL2 2.1.0，变量 `$p`、`$s` 按[操作基础](operations.md)设置。PA5 输出与 SPI1 SCK 的 PA5 方案是两个独立实例，不能同时使用同一引脚。

## PA5 输出

```powershell
cube mx pinout pin -p "$p" --pin PA5
cube mx pinout assign-signal -p "$p" --pin PA5 --signal GPIO
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/mode --value OUTPUT
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/output_init_state
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r GPIO --parameter-path pins/PA5/basic/output_type
```

按负载选择初始电平、推挽/开漏、上下拉和速率，再回读。LED 有效电平由电路决定；PA5 不自动意味着开发板 LED。

生成后确认 GPIOA 时钟、PA5 模式和初始电平。`mx_system_init()` 成功后，在裸机主循环中：

```c
HAL_GPIO_TogglePin(HAL_GPIOA, HAL_GPIO_PIN_5);
HAL_Delay(500);
```

应观察到约 1 秒完整闪烁周期。不要在中断中使用此延时。HAL 时间基准不工作时，先修复 tick，不能据此判定 GPIO 配置失败。

切换此 GPIO 资源为 LL：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path info/layer --value LL
```

生成文件仍可能在 `generated/hal/`；检查内容而非目录名。LL 的复位默认配置可能被省略，不能把生成 init 当成任意运行状态下的完整寄存器复原操作。

## PB0 按键，下降沿 EXTI

示例接法：PB0 通过按键接地，选择上拉；确认电压与引脚限制。其他板卡按原理图调整。

```powershell
cube mx pinout assign-signal -p "$p" --pin PB0 --signal GPIO
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PB0/basic/mode --value INPUT
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PB0/basic/pull --value UP
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path pins/PB0/basic/enable_exti --value true
cube mx sw-config list-parameters -p "$p" -s "$s" -r GPIO
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/trigger_detection' --value FALLING
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/interruption/enable_interruption' --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r GPIO --parameter-path 'pins/PB0/basic/exti_config@/Exti line/basic/interruption/irq_handler_generation' --value true
```

回读 `.../nvic@/EXTI0/priority`，按是否调用 RTOS API 选择优先级。同一个 EXTI 线可能有多个 GPIO 来源，不能把 PA0 和 PB0 都独占绑定到 EXTI0。

保存重载、暂存生成并构建。检查 `EXTI0_IRQHandler`、HAL EXTI handle、触发配置和回调；示例生成 getter 为 `mx_gpio_default_exti0_gethandle()`。通过实际头文件确认 `HAL_EXTI_RegisterTriggerCallback` 的签名及配置开关。

回调只置标志、计数或发送合法的 ISR 通知；在主循环/任务里消抖和处理业务。观察按键稳定输入、IRQ 计数和消抖后的事件数，区分机械抖动与重复清标志错误。

失败时依次查 GPIO 电平 → EXTI 来源与边沿 → pending 位 → NVIC enable/priority → 向量和回调，见[中断诊断](../../stm32c5-debug/references/peripherals.md)。
