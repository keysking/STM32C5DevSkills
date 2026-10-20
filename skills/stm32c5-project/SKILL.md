---
name: stm32c5-project
description: 创建或接手 STM32C5 工程，识别完整型号、CubeMX2 的 ioc2、HAL2/LL、软件工程、构建目标和依赖。用于 C5 新建、官方例程导入、无生成器工程整理及重新生成时保留应用代码。
license: MIT
---

# STM32C5 工程

1. 从 `.ioc2`、设备头文件、启动文件、链接脚本和构建参数确认完整 C5 型号；不凭文件夹名称选芯片。型号未知时先询问型号；现有工程可以继续只读检查。
2. 新建默认 CubeMX2 + HAL2 + CMake/Ninja/GCC + 裸机。依[新建实例](references/create.md)建立 `STM32C532CCT6` 示例；换型号前读[型号边界](../stm32c5-env/references/c5-basics.md)。
3. 接手工程或从例程开始，读[工程识别与其他入口](references/existing-and-examples.md)。已有工具链、布局、RTOS API 保持原选择，除非任务要求迁移。
4. `.ioc2` 是复合工程容器，使用 `cube mx` 读写；不做 JSON 反序列化后整体重写。
5. 引脚和外设任务进入[配置技能](../stm32c5-configure/SKILL.md)。生成前必须读[安全重新生成](references/regenerate.md)：导出到新的空目录，检查完整差异并合并。
6. 用[构建步骤](../stm32c5-build/references/build.md)取得 ELF，再说明是否具备烧录和功能测试条件。

完成标准：型号与内存布局明确、依赖可恢复、生成或导入的工程能构建、应用代码得到保留。创建工程不自动授权烧录连接中的其他板卡。
