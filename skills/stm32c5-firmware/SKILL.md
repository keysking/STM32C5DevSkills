---
name: stm32c5-firmware
description: 编写 STM32C5 裸机或 FreeRTOS 应用，使用 HAL2/LL 外设接口、移植 HAL1 驱动、实现中断和 DMA 数据处理。用于功能开发、外设库集成、任务队列同步及 CMSIS-RTOS 现有工程维护。
license: MIT
---

# STM32C5 固件

按本次需求选择下列场景及参考章节，不把整篇资料当成执行清单。命令中的型号、引脚、数值和名称用于说明语法，须替换为真实需求及当前工程的值；只执行所选操作和必要依赖，查询任务不顺带修改配置。

1. 识别完整型号、实际 HAL2/LL 版本、生成文件与应用文件边界、运行模型。HAL1 示例不能仅把头文件改成 C5 后继续使用。
2. 读取[HAL2 与 LL](references/hal2-ll.md)，从工程实际头文件确定签名、单位、状态和生命周期；不要凭训练记忆构造 API。
3. 按实际传输方式选取[API 用法](references/peripheral-code.md)，不加入无关演示功能。中断/DMA 要同时检查回调开关、向量、缓冲区寿命、重入和错误路径。
4. 移植第三方库读[驱动移植](references/porting.md)。将协议逻辑与 C5 的 GPIO、总线、时间和并发适配分开，避免无关架构改写。
5. FreeRTOS 读[任务、队列与同步](references/freertos.md)；新建默认原生 API，已有 CMSIS-RTOS v1/v2 则沿用适配层。
6. 通过[构建技能](../stm32c5-build/SKILL.md)确认编译、链接和资源占用；有硬件时按实际接线验证超时、错误及恢复路径。

应用实现放在现有应用区或自行维护的源文件中；新增文件须进入构建目标。修改生成器所有的初始化时，同步 `.ioc2` 并执行[重新生成流程](../stm32c5-project/references/regenerate.md)。不能仅依赖 `USER CODE` 标记推断 CubeMX2 的保留行为。

完成标准：功能代码及错误处理明确、构建通过、说明已执行的测试与仍需硬件确认的行为。
