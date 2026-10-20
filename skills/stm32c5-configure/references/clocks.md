# C5 时钟与 HSE

先确认板上晶振/外部时钟频率、连接方式和供电条件。新建无板卡约束的实例先用内部时钟完成构建；不能因为生成器显示 HSE=8 MHz 就认为实物也有 8 MHz 晶振。

变量 `$p` 按[操作基础](operations.md)设置：

```powershell
cube mx clock get-clocks -p "$p"
cube mx clock get-clock-status -p "$p"
cube mx clock get-clock-items --help
cube mx clock list-possible-values --help
cube mx clock set-value --help
```

从 `get-clocks` 获取实际 ID；`get-clock-items` 需要指定 ID，不能省略然后假定返回整棵树。C5 时钟模型含 PSI 等专用路径，不套用 F1 的 PLL 倍频公式。

以 C532 模型中存在的 `PSI_Clock_Mux` 为例：

```powershell
cube mx clock list-possible-values -p "$p" --id PSI_Clock_Mux
cube mx clock get-clock-items -p "$p" --ids 'PSI_Clock_Mux,PSI,HSI,HCLK_Frequency'
```

`get-clocks` 中可能显示内部值 `HSI_DIV_18_inputid`，而 `get-clock-items` 和 `list-possible-values` 对应的写入值是 `HSI_DIV_18`。不能直接把不同接口的显示字段互相复制。

## 内部时钟 144 MHz 示例

适用于新的 C532 示例：HSI 已启用、无需外部晶振，目标与当前电源/Flash 条件允许 144 MHz。查询确认后：

```powershell
cube mx clock set-value -p "$p" --id PSI_Clock_Mux --value HSI_DIV_18
cube mx clock set-value -p "$p" --id PSI --value 144000000
cube mx clock get-clock-items -p "$p" --ids 'PSI_Clock_Mux,PSI,HCLK_Frequency'
cube mx clock get-clock-status -p "$p"
```

预期 PSI 源为 `HSI_DIV_18`，PSI 为 144 MHz；HCLK 还取决于下游分频，本实例为 144 MHz。仍须保存重载、生成并检查 RCC 代码。某些节点列出的候选频率可能高于该型号最高频率，不能仅凭候选列表就使用，需同时满足器件限制和 clock status。

## 修改流程

1. 读取当前 SYSCLK/HCLK/PCLK、外设 kernel clock 及 clock status。
2. 查询目标节点的可选值；设置振荡器启用状态、频率和源，再设置下游选择/分频。每一步回读。
3. 保证目标频率符合该器件、电源档和 Flash latency 限制；若状态报告约束错误，先解决再生成。
4. 保存重载，导出暂存工程；检查 `mx_system`、RCC 初始化、外部环境配置和外设时序。
5. 构建后，硬件验证 `SystemCoreClock`、tick 周期和可测输出；若切换失败保留返回状态定位。

外部 HSE 的状态位于 `RCC/basic/hse/state`。先用 `sw-config get-parameter-possible-values -r RCC --parameter-path basic/hse/state` 查询当前支持的关闭、晶振或 bypass 值，再按板卡设置；频率通过 clock 模型核对。切换 PSI 到 HSE 之前，必须确认 HSE 已启用，避免“选中了 HSE 但振荡器仍关闭”。

## HSE 编译失败与启动失败分开处理

如果 `HAL_RCC_HSE_Enable`、`HAL_RCC_HSE_GetState` 或 HSE 宏未声明，先检查：

- 构建命令中的 `USE_EXTERNAL_ENV`。
- 设备头文件是否因此包含 `stm32_external_env.h`。
- `stm32_external_env.h` 的 include 路径，以及其中 `HSE_VALUE` 等实际配置。
- 切换配置或导入源文件时是否遗漏预包含/环境链。

不要用手工函数声明、删除 HAL 检查或无依据的 `-DHSE_VALUE=8000000` 掩盖缺失链路。若编译通过但 HSE 超时，再检查晶振、bypass 模式、频率和供电。

RTOS 的 SysTick/HAL 时间基准冲突按[FreeRTOS 配置](freertos.md)处理，不通过随意重写系统时钟函数解决。
