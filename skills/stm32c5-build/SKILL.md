---
name: stm32c5-build
description: 构建 STM32C5 固件并处理 GCC/CMake/Ninja、已有 Makefile、IAR 或 Keil 工程的编译链接问题。用于查找编译器、选择预设、检查 ELF/map 和 Flash/RAM 占用，以及 HAL2、HSE、RTOS 依赖错误。
license: MIT
---

# STM32C5 构建

按本次需求选择下列场景及参考章节，不把整篇资料当成执行清单。命令中的型号、引脚、数值和名称用于说明语法，须替换为真实需求及当前工程的值；只执行所选操作和必要依赖，查询任务不顺带修改配置。

1. 先识别工程根目录、工具链、目标型号和依赖；沿用已有构建体系。CubeMX2 导出格式与编译目标是两项设置，不能把 AC6 目标误当成 Keil 工程导出已存在。
2. CMake/Ninja/GCC 按[构建与产物](references/build.md)执行：定位真实 GCC/Ninja → 列出预设 → 配置 → 构建 → 检查 ELF 和内存区域。
3. 编译或链接失败，按[错误定位](references/errors.md)从第一条有效错误开始。不要靠删除大量代码或关闭告警掩盖根因。
4. 已有 Makefile、IAR、Keil，按[已有工具链](references/other-toolchains.md)调用原生工具。无法在当前平台运行时明确限制，保留工程原配置。
5. 成功后给出实际 ELF、构建配置和资源占用。烧录需要另外进入[烧录技能](../stm32c5-flash/SKILL.md)。

编译器能运行不等于工程构建成功；ELF 存在不等于本次构建成功；构建成功不等于板端功能正确。判断产物时间、命令退出码和错误日志，避免使用陈旧 ELF。
