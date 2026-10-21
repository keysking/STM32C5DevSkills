# HAL2 外设 API 用法

本文按传输、启动和事件处理分别列出 API，用于选择当前任务需要的操作，不是要求全部加入工程的演示应用。依据 C5 HAL2 2.1.0；实例、数据、长度、超时和通道按真实需求确定。USART1 等名称仅说明如何找到生成 handle，不要求新增对应外设。

## 取得声明和已初始化的 handle

先检查工程实际 HAL 头文件、配置宏及生成的 `mx_*.h`。下表的 getter 名须与当前生成文件核对；不要重复 init 代替取 handle。

| 资源写法 | 生成头文件 / getter 的一种实际形式 |
| --- | --- |
| USART1 | `mx_usart1.h` / `mx_usart1_uart_gethandle()` |
| I2C1 | `mx_i2c1.h` / `mx_i2c1_i2c_gethandle()` |
| SPI1 | `mx_spi1.h` / `mx_spi1_gethandle()` |
| TIM1 | `mx_tim1.h` / `mx_tim1_gethandle()` |
| ADC1 | `mx_adc1.h` / `mx_adc1_gethandle()` |
| GPIO EXTI0 | `mx_gpio_default.h` / `mx_gpio_default_exti0_gethandle()` |

调用前确认初始化成功，例如 `mx_system_init()` 返回成功。下文 C 片段位于应用函数内，参数及 handle 由应用准备；检查每个 `hal_status_t`，失败后按状态处理，不继续假定成功。

本文不覆盖所有接口。查询实际头文件及源文件中的函数声明、注释、条件宏和状态要求；Pack 定位见[HAL2/LL 查询方法](hal2-ll.md)。例如：

```text
rg -n "HAL_UART_Transmit|HAL_UART_Receive_IT|HAL_UART_ReceiveToIdle_DMA" .
rg -n "HAL_ADC_Start|HAL_ADC_Calibrate|HAL_ADC_REG_StartConv" .
```

## UART 阻塞发送或接收

仅在需求选择轮询阻塞方式时使用，不需要额外启用 DMA/回调。包含实际生成头文件，取得 `hal_uart_handle_t *uart`；`data` 指向发送数据，`buffer` 为可写接收空间，`size_byte` 与 `timeout_ms` 由协议确定。

| 操作 | 调用形式 |
| --- | --- |
| 发送 | `HAL_UART_Transmit(uart, data, size_byte, timeout_ms)` |
| 接收 | `HAL_UART_Receive(uart, buffer, size_byte, timeout_ms)` |

长度单位为字节，阻塞超时单位为毫秒。不要插入固定的“C5 ready”输出或固定延时。失败时记录状态，查时钟、初始化、tick 和对端；ISR 中不使用阻塞等待。

## UART 中断接收

仅在选择 IT 接收时使用；同一接收 handle 不同时启动 DMA。先确认 UART IRQ、handler 和相应回调路径，见[UART 配置](../../stm32c5-configure/references/uart.md)。

固定长度接收的形式为 `HAL_UART_Receive_IT(uart, buffer, size_byte)`。缓冲区必须持续有效直至传输结束，不能传入马上离开作用域的栈数组。长度来自协议，不统一设为一字节。

采用注册回调且 `USE_HAL_UART_REGISTER_CALLBACKS=1` 时，在接收启动前调用 `HAL_UART_RegisterRxCpltCallback(uart, rx_done)` 并检查结果。回调签名：

```c
static void rx_done(hal_uart_handle_t *uart, uint32_t size_byte,
                    hal_uart_rx_event_types_t event)
{
    /* 根据 uart、事件类型和实际长度，记录完成或提交已接收数据。
     * 数据已移交且需要下一次传输时，按接收状态安排重新启动。
     * 不在这里无条件重启其他模式，也不执行阻塞业务。 */
    (void)uart;
    (void)size_byte;
    (void)event;
}
```

这是回调接口形式，不是已实现的数据处理器。应用须填入其缓冲区所有权及错误恢复逻辑。沿用弱回调时查当前源码签名，不为使用此形式强行切换注册机制。每次重新接收都检查返回值；overrun 等错误按实际 HAL 状态恢复。

## UART DMA 与 IDLE 接收

仅当需求选择 DMA 时使用。确认外设与 DMA handle 绑定、宽度、模式及必要 IRQ；IDLE 通知还需要 UART IRQ。

假定 `uart` 是既有 handle，`buffer` 是长期有效的缓冲区，`capacity_bytes` 为真实容量：

```c
hal_status_t status = HAL_UART_ReceiveToIdle_DMA(uart, buffer, capacity_bytes);
```

检查 `status`。正常模式按传输结束状态消费并重启；循环模式根据事件位置计算增量，不在每次半满/IDLE 事件无条件重启。固定长度 DMA 或 TX DMA 使用对应实际头文件中的接口，不因本页列出 IDLE 就改变协议。

## I2C 主机传输

仅在 MCU 为主机且协议要求对应操作时使用。`hal_i2c_handle_t *i2c` 来自现有初始化；数据区、长度和超时由器件协议给出。

7 位地址的发送形式：

```c
/* address7 是未移位的 7 位地址，范围 0..0x7F。 */
hal_status_t status = HAL_I2C_MASTER_Transmit(
    i2c, (uint32_t)address7 << 1U, data, size_byte, timeout_ms);
```

先校验参数并检查状态。若上层已传入 HAL 所需移位地址，不再左移；10 位寻址按实际接口约定处理。`own_address1` 不是主机目标地址。

接收、寄存器访问、repeated-start 或序列传输分别检索当前 HAL2 MASTER/内存/SEQ 接口及停止条件，不能盲目把寄存器读取拆成两个带 STOP 的传输。中断/DMA 路径额外确认事件及错误 IRQ、缓冲区寿命，不为阻塞发送添加它们。

## SPI 全双工传输

仅适用于实际选择全双工的事务。`hal_spi_handle_t *spi` 来自初始化；`tx`、`rx` 的存储容量须匹配帧宽与包数：

```c
hal_status_t status = HAL_SPI_TransmitReceive(spi, tx, rx, count_packet, timeout_ms);
```

`count_packet` 是帧/包数，只有 8 位帧时才等于字节数。单向需求使用对应方向 API，不固定改成全双工。软件 CS 由板级操作选择/释放器件，错误路径也须处理；异步启动返回不表示传输已结束，不能立即释放 CS。硬件 NSS 按其时序验证。

## 启动 PWM 输出

仅在初始化已配置为 PWM、应用需要开始输出时操作。下列说明 TIM1 CH1 的启动依赖；其他实例/通道替换为实际 handle 和宏。

```c
hal_tim_handle_t *tim = mx_tim1_gethandle();
hal_status_t status = HAL_TIM_OC_StartChannel(tim, HAL_TIM_CHANNEL_1);
if (status == HAL_OK) {
    status = HAL_TIM_Start(tim);
    if (status != HAL_OK) {
        (void)HAL_TIM_OC_StopChannel(tim, HAL_TIM_CHANNEL_1);
    }
}
```

检查最终状态。已经运行的共用定时器须检查状态及其他通道，不重复初始化或随意停整个计数器。配置依据[定时器参考](../../stm32c5-configure/references/tim-adc.md)；改变占空比查 `HAL_TIM_OC_SetCompareUnitPulse` 与 preload/更新时机，不默认改频率。高级定时器还应检查输出使能和 break。

## ADC 启动、校准及规则组转换

先区分 ADC 外设启动与转换启动。HAL2 2.1.0 校准要求 ACTIVE 状态：初始化后先 `HAL_ADC_Start(adc)`，成功才 `HAL_ADC_Calibrate(adc)`。不沿用 HAL1 的校准顺序，不每次读取都重复启动/校准。

按需求选择一种转换方式：

| 方式 | 启动调用 | 必要条件 |
| --- | --- | --- |
| 轮询 | `HAL_ADC_REG_StartConv(adc)` | 配置好触发及序列；之后 `HAL_ADC_REG_PollForConv(adc, timeout_ms)`，成功时 `HAL_ADC_REG_ReadConversionData(adc)`。 |
| 中断 | `HAL_ADC_REG_StartConv_IT(adc)` | IRQ、handler 及相应完成/错误回调。 |
| DMA | `HAL_ADC_REG_StartConv_DMA(adc, buffer, size_byte)` | DMA 绑定、方向、宽度、模式、IRQ 和长期有效缓冲区。 |

轮询返回样本类型为 `int32_t`。单次读取结束或超时后按状态调用 `HAL_ADC_REG_StopConv`，同时保留原始错误；持续/外部触发模式按其生命周期停止，不直接套用单次流程。

DMA 参数为 `uint8_t *` 和字节数；对于实际存在的 `uint16_t samples[32]`，指针转为 `(uint8_t *)samples`，长度用 `sizeof samples`，不是直接用 32。尺寸由真实采集需求确定。多通道消费顺序须与 rank 一致。

## EXTI 事件回调

仅当需要处理已配置 EXTI 的事件时注册。初始化成功后，按项目实际开关使用 `HAL_EXTI_RegisterTriggerCallback(exti, on_edge)` 并检查返回值；弱回调路线按头文件条件使用。

```c
static void on_edge(hal_exti_handle_t *exti, hal_exti_trigger_t trigger)
{
    /* 按实际来源和边沿记录事件或通知任务，不在 ISR 内延时。 */
    (void)exti;
    (void)trigger;
}
```

片段只展示签名，应用补入其事件处理。输入可能是传感器、同步信号或按键，不默认增加按键计数与消抖；仅机械输入需要时才安排消抖。

## 验证所选操作

在应用源文件中实现选定逻辑并加入当前构建目标。检查 API、宏、长度和生命周期；异步操作还要观察完成、错误、溢出及后续传输。构建、烧录及真实信号验证分别给出结果，不把片段可编译当成功能已验证。
