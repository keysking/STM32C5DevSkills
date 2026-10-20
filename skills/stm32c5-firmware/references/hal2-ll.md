# HAL2 / LL 的实际使用

## 从工程获得 API，而非套 HAL1

查找本工程实际使用的头文件和配置：

```text
rg --files -g "stm32c5xx_hal_*.h" -g "stm32c5xx_ll_*.h" -g "*hal_conf*.h" -g "*external_env*.h"
rg -n "HAL_UART_Init|HAL_UART_SetConfig|hal_uart_handle_t" stm32c5xx_drivers
rg -n "USE_HAL_UART|USE_EXTERNAL_ENV" .
```

如果源码通过 Pack 绝对路径引用，先用 `cube --get-current-value cmsis_pack_root` 获得根目录，再定位 `STMicroelectronics/stm32c5xx_hal_drivers/实际版本`。不要只搜索另一个工程或最新缓存版本。

HAL2 2.1.0 的典型差异：

| 需求 | C5 HAL2 要点 |
| --- | --- |
| UART 初始化 | `hal_uart_handle_t`，`HAL_UART_Init(handle, HAL_USART1)` 与 `HAL_UART_SetConfig` 分开 |
| UART 完成回调 | RX 回调含 handle、`size_byte` 和事件类型 |
| I2C 主机发送 | `HAL_I2C_MASTER_Transmit`，检查目标地址左移约定 |
| SPI 长度 | `count_packet`；8/16/32 位帧与字节数不能混淆 |
| ADC | 启动 ADC 与启动 REG/INJ 转换分开 |
| PWM | TIM 计数器与 OC 输出通道分开启动 |
| DMA 绑定 | 明确 `SetRxDMA`、`SetTxDMA` 或外设专用绑定接口 |

使用顺序：资源时钟与 GPIO → handle 初始化 → 配置 → 中断/DMA/回调连接 → 启动外设活动 → 启动传输。生成器可能已完成前几项，先读调用链，避免重复初始化清掉运行状态。

对每次传输检查返回状态，区别参数错误、忙、超时和外设错误。超时依赖 HAL 时间基准；tick 未运行时不要在高优先级中断中等待超时退出。

## 注册回调与弱回调

先检查对应 `USE_HAL_*_REGISTER_CALLBACKS` 和头文件条件。开启注册后，在初始化完成、异步传输启动前注册；未开启时按当前源码的弱回调签名实现。不要同时提供不匹配的 HAL1 回调并假定会自动调用。

`USE_HAL_*_DMA` 控制 DMA API 是否编译。宏不存在、源码未入目标与函数名错误是不同问题，分别排查。

## LL 与 HAL 混用

可以让 GPIO 使用 LL、串口使用 HAL2。FDCAN 等模块不一定提供同等 LL API；先检查实际头文件，不能凭命名规律发明 `LL_FDCAN_*`。

同一外设的状态由一个明确的实现负责。直接 LL 改动 HAL 管理的 DMA、状态或中断后，需要同步其状态机；避免在 HAL 正在传输时强改寄存器。

CubeMX2 的 `info/layer` 控制对应资源生成层，`info/init_type` 的 `called` 表示生成并调用，`enabled` 仅生成，`disabled` 不生成。读回设置并检查 `mx_system_init()` 调用链。LL 文件不一定位于名为 `ll` 的生成目录。

依据：[ST HAL/LL 分层说明](https://dev.st.com/stm32cube-docs/stm32c5xx-hal-drivers/2.1.0/en/docs/overview/hal_ll_layered_drivers.html)。具体签名与生命周期以项目实际版本头文件和源文件为准。
