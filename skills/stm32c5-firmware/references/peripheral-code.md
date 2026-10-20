# HAL2 应用片段

以下针对 C5 HAL2 2.1.0，放在应用维护的源文件中并加入构建目标。先完成外设配置与 `mx_system_init()`，包含实际生成头文件。例中 getter 名需与生成的 `.h` 对照；不要重复调用 init 代替取 handle。

## USART1 轮询发送

```c
#include "mx_usart1.h"

hal_status_t app_uart_hello(void)
{
    static const char text[] = "C5 ready\r\n";
    return HAL_UART_Transmit(mx_usart1_uart_gethandle(),
                             text, sizeof text - 1U, 100U);
}
```

在系统初始化成功后调用，检查 `HAL_OK`；失败时记录状态并检查串口时钟、初始化和 tick。超时单位为毫秒。

## UART 固定长度接收中断

此片段只使用一字节 IT 接收；不要同时在同一 handle 上启动 DMA。启用 UART IRQ 和注册回调开关，按[UART 配置](../../stm32c5-configure/references/uart.md)生成。

```c
#include "mx_usart1.h"

static uint8_t rx_byte;
static volatile uint32_t rx_count;
static volatile hal_status_t rx_restart_status;

static void app_rx_done(hal_uart_handle_t *uart, uint32_t size_byte,
                        hal_uart_rx_event_types_t event)
{
    (void)event;
    if (size_byte == 1U) {
        ++rx_count; /* 最小观察点；业务数据应送入环形缓冲或队列。 */
    }
    rx_restart_status = HAL_UART_Receive_IT(uart, &rx_byte, 1U);
}

hal_status_t app_uart_rx_start(void)
{
    hal_uart_handle_t *uart = mx_usart1_uart_gethandle();
    hal_status_t status = HAL_UART_RegisterRxCpltCallback(uart, app_rx_done);
    if (status != HAL_OK) {
        return status;
    }
    return HAL_UART_Receive_IT(uart, &rx_byte, 1U);
}
```

应用周期检查 `rx_restart_status` 和错误回调；overrun 等错误后按实际 HAL 状态恢复接收。`volatile` 仅用于观察，不是多生产者队列或原子同步的替代。

DMA/IDLE 路线使用长期有效的静态缓冲区：

```c
static uint8_t rx_dma[128];
/* 在回调、DMA 绑定和两个 IRQ 就绪后调用，并检查返回值。 */
hal_status_t status = HAL_UART_ReceiveToIdle_DMA(
    mx_usart1_uart_gethandle(), rx_dma, sizeof rx_dma);
```

DMA 模式决定回调数据表示和重启策略。普通模式在传输完成后消费并重新启动；循环模式根据事件位置计算增量，防止覆盖未处理数据。不要无条件在每个半满/IDLE 事件里重新启动 DMA。业务设计需明确生产者、消费者和溢出处理。

## I2C 主机发送

```c
#include "mx_i2c1.h"

hal_status_t app_i2c_write(uint8_t address7,
                            const uint8_t *data, uint32_t size)
{
    if (address7 > 0x7FU || data == NULL || size == 0U) {
        return HAL_ERROR;
    }
    return HAL_I2C_MASTER_Transmit(mx_i2c1_i2c_gethandle(),
                                    (uint32_t)address7 << 1U,
                                    data, size, 100U);
}
```

调用者传 7 位地址；器件寄存器协议由调用者组织。要求 repeated-start 的寄存器读取，应选实际 HAL2 MemRead/SEQ 接口并核对停止条件，不能盲目拆成两次独立传输。

## SPI 全双工

```c
#include "mx_spi1.h"

hal_status_t app_spi_exchange(const uint8_t *tx, uint8_t *rx, uint32_t count)
{
    return HAL_SPI_TransmitReceive(mx_spi1_gethandle(), tx, rx, count, 100U);
}
```

这里配置为 8 位帧，`count` 才与字节数相同。调用前后用板级 CS 函数选择/释放器件，包括错误路径；CS 引脚由实际板卡决定。中断/DMA 发送时不得在启动函数返回后立即释放 CS，应在实际传输结束且驱动确认总线完成后释放。

## PWM 启动

```c
#include "mx_tim1.h"

hal_status_t app_pwm_start(void)
{
    hal_tim_handle_t *tim = mx_tim1_gethandle();
    hal_status_t status = HAL_TIM_OC_StartChannel(tim, HAL_TIM_CHANNEL_1);
    if (status != HAL_OK) {
        return status;
    }
    status = HAL_TIM_Start(tim);
    if (status != HAL_OK) {
        (void)HAL_TIM_OC_StopChannel(tim, HAL_TIM_CHANNEL_1);
    }
    return status;
}
```

比较单元和通道参数来自[配置实例](../../stm32c5-configure/references/tim-adc.md)。动态占空比使用 `HAL_TIM_OC_SetCompareUnitPulse` 并确认 preload/更新时机，不直接照搬 HAL1 宏。

## ADC 单次读取

```c
#include "mx_adc1.h"

hal_status_t app_adc_begin(void)
{
    hal_adc_handle_t *adc = mx_adc1_gethandle();
    hal_status_t status = HAL_ADC_Start(adc);
    if (status != HAL_OK) {
        return status;
    }
    return HAL_ADC_Calibrate(adc);
}

hal_status_t app_adc_read(int32_t *sample)
{
    hal_adc_handle_t *adc = mx_adc1_gethandle();
    if (sample == NULL) {
        return HAL_ERROR;
    }
    hal_status_t status = HAL_ADC_REG_StartConv(adc);
    if (status != HAL_OK) {
        return status;
    }
    status = HAL_ADC_REG_PollForConv(adc, 100U);
    if (status == HAL_OK) {
        *sample = HAL_ADC_REG_ReadConversionData(adc);
    }
    hal_status_t stop = HAL_ADC_REG_StopConv(adc);
    return status == HAL_OK ? stop : status;
}
```

初始化后调用一次 `app_adc_begin()`，成功才周期调用读取函数。HAL2 的校准要求 ADC 已处于 ACTIVE 状态，不能直接沿用 HAL1 的校准顺序。转换超时仍尝试停止转换；持续失败再检查时钟、触发和 handle 状态。

## EXTI 回调

```c
#include "mx_gpio_default.h"

static volatile uint32_t button_edges;

static void app_button_edge(hal_exti_handle_t *exti, hal_exti_trigger_t trigger)
{
    (void)exti;
    (void)trigger;
    ++button_edges;
}

hal_status_t app_button_start(void)
{
    return HAL_EXTI_RegisterTriggerCallback(
        mx_gpio_default_exti0_gethandle(), app_button_edge);
}
```

GPIO/EXTI 初始化成功后注册；若头文件宏限制此 API，先在实际 HALCore 配置中开启对应注册选项并生成。业务层观察计数并消抖，ISR 不延时。

每段代码只证明接口用法。依实际外设完成构建、烧录和信号验证后，才能判断应用功能正确。
