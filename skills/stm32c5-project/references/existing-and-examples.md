# 接手工程、官方例程与无生成器工程

## 识别已有工程

从工程根目录进行有界搜索：

```text
rg --files -g "*.ioc2" -g "*.ioc" -g "*Presets.json" -g "*CMakeLists.txt" -g "*csolution.yml" -g "*cproject.yml" -g "cube-dcm.json" -g "*.ld" -g "*.icf" -g "*.sct" -g "*.uvprojx" -g "*.ewp"
rg -n "STM32C5|mcpu|mfloat-abi|mfpu|FreeRTOS|cmsis_os2?\.h" .
```

排除 build、供应商缓存等大目录，必要时只搜索应用和构建目录。随后检查：

| 证据 | 判断 |
| --- | --- |
| `.ioc2`、软件工程列表 | CubeMX2 配置来源，可恢复生成 |
| `generated/hal/`、`hal_uart_handle_t` | HAL2 线索，以头文件版本为准 |
| `LL_*` | 某些模块使用 LL，不能据此认定全工程无 HAL |
| `main.c` 调用链 | 初始化、应用入口、调度器启动位置 |
| 启动文件、链接脚本、设备宏 | 精确型号、内存布局、向量 |
| `FreeRTOS.h` / `cmsis_os.h` / `cmsis_os2.h` | 原生 / CMSIS v1 / CMSIS v2 API |

`.ioc` 可能是被移植项目的遗留文件，先核对型号与实际构建，不能交给传统 CubeMX 生成 C5。

## 从官方例程开始

1. 在 [STM32CubeC5 官方仓库](https://github.com/STMicroelectronics/STM32CubeC5)或 [C5 软件文档](https://dev.st.com/stm32cube-docs/stm32c5-mcu-package/2.1.0/en/)选择匹配设备、开发板、编译器和驱动版本的例程。
2. 克隆源码时包含子模块：

```text
git clone --recursive https://github.com/STMicroelectronics/STM32CubeC5.git
git -C STM32CubeC5 submodule status
```

3. 已有克隆缺文件时使用 `git submodule update --init --recursive`，不从另一个版本复制随机驱动补缺口。需要可复现版本时固定官方 tag/commit。
4. 读例程 README、`cube-dcm.json`、预设和 board/application 配置。Example Library、Package Creator、Pack 导出的目录可能不同，不能强制改成 `Core/Src`。
5. 先构建未修改的例程，再替换引脚、时钟和应用逻辑。板载 LED、晶振、串口、收发器与自制板不同的地方逐项改。

例程的选项、板级覆盖与文件组织以其说明为准：[官方例程配置入口](https://dev.st.com/stm32cube-docs/examples/2.0.0/en/configure/config_toc.html)。例程不是完整订货号兼容性的自动证明。

## 无生成器工程

已有手写 C5 工程可以继续使用。确认以下内容已纳入构建：

- C5 设备头文件、CMSIS Core、匹配启动文件、`SystemInit`。
- 当前 HAL2/LL 源码和配置宏，唯一的系统时钟与 tick 实现。
- 精确内存布局、向量地址、FPU ABI、链接库与 syscalls。
- 主时钟、外设时钟、GPIO 复用、DMA 和 NVIC 初始化调用顺序。

应用改动直接维护这些文件；不要伪造 `.ioc2`。如果用户希望改为生成器管理，先在独立目录建同型号工程，逐项迁移并比较行为，不覆盖原工程。
