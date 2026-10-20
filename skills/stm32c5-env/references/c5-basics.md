# C5 知识与型号边界

STM32C5 是 Cortex-M33 系列，带 FPU/DSP，系列最高主频 144 MHz；容量、外设和封装随具体订货号变化。不要将它当成 C0/F1，也不要从 M33 推断一定存在 TrustZone 双镜像。[ST 系列页](https://www.st.com/en/microcontrollers-microprocessors/stm32c5-series.html)

本包命令实例以 `STM32C532CCT6`、C5 DFP/HAL2 2.1.0、CubeMX2 CLI 1.1.2 为基准。该订货号是 LQFP48、256 KiB Flash、64 KiB RAM，设备模型不实现 TrustZone。这里的参数是示例前提，不是所有 C5 的共同参数。

| 名称 | 用途 |
| --- | --- |
| `cube` / STM32CubeCLI | 查找并运行工具、管理主机 Bundle 和环境 |
| `cube mx` | CubeMX2 的命令入口，调用其后端操作工程 |
| CubeMX2 | C5 配置、组件依赖和代码生成，工程为 `.ioc2` |
| 传统 CubeMX | 旧 `.ioc` 与 Java 脚本接口；不用于新建 C5 |
| STM32CubeC5 / Pack | 设备头文件、启动文件、HAL2/LL、中间件和例程 |
| HAL2 / LL | 高层驱动 / 寄存器级辅助接口，可按外设选择 |

C5 对应 CubeMX2 的支持范围，不能将传统 `STM32CubeMX.exe -q script` 或 `java -jar STM32CubeMX.exe` 命令用来操作 `.ioc2`。[ST 生成器说明](https://www.st.com/content/st_com/en/stm32cubemx.html)

对每个实际任务确认：

1. 从芯片丝印、原理图或工程获得完整订货号；使用 `cube mx finder device info --cpn STM32C532CCT6` 检查设备信息。
2. 确认所用封装能引出目标信号。查看 `pinout pin` 的 `possibleConfiguration`，不能仅从系列数据手册猜引脚。
3. 查实际 Flash/RAM、启动文件、FPU ABI 和链接脚本。示例 Flash 起点 `0x08000000`；有 bootloader 时以分区布局为准。
4. 晶振频率、VDD、LED 极性、总线上拉、收发器、传感器地址来自板卡资料。生成器默认值不是板卡测量值。
5. 本包不改变已有工程的 API、工具链或 RTOS 选择。HAL1 来源代码通过适配迁移，不在 C5 工程中混入其他系列的驱动源文件。
