---
name: stm32c5-env
description: 准备或修复 STM32C5 开发环境，定位 cube、CubeMX2、C5 Pack、GCC、CMake、Ninja、烧录器和调试器。用于找不到 CubeMX2、工具启动失败、依赖缺失、版本冲突及三平台环境配置。
license: MIT
---

# STM32C5 环境

按本次需求选择下列场景及参考章节，不把整篇资料当成执行清单。命令中的型号、引脚、数值和名称用于说明语法，须替换为真实需求及当前工程的值；只执行所选操作和必要依赖，查询任务不顺带修改配置。

只处理 STM32C5。C5 使用 Cortex-M33、HAL2/LL 和 CubeMX2；不能套用 F1 的传统 CubeMX、HAL1 或 `.ioc` 脚本。芯片完整订货号和工程锁定版本优先于本文示例。

1. 检查工作目录、已有修改及工程依赖：`.ioc2`、`cube-dcm.json`、`*.csolution.yml`、`CMakePresets.json`、工具链文件。只读识别期间不升级依赖。
2. 按[工具发现与依赖](references/tools.md)定位实际可执行文件。区分主机 Bundle 和固件 Pack，查清版本及路径；不能把 `cube --list` 当成所有已安装可执行文件的列表。
3. 需要配置 C5 时，按[CubeMX2 启动会话](references/mx-session.md)完成一次真实设备查询。`--help` 成功不足以证明后端可用。
4. 缺工具时，先找现有安装或按工程版本恢复，再安装缺少的官方组件。使用本次会话的 PATH；不要无缘由修改用户全局环境。
5. 需要创建工程时转到[工程技能](../stm32c5-project/SKILL.md)，已有工程只缺编译器则转到[构建技能](../stm32c5-build/SKILL.md)。

完成时给出实际工具版本、路径、Pack 状态和成功执行的最小命令。区分“可执行”“配置成功”“构建成功”；不把安装成功表述为开发板已验证。

按需读取：安装、PATH 或权限问题见[平台差异](references/platforms.md)；型号、HAL2 和生成器概念不清时见[C5 知识与边界](references/c5-basics.md)。无需一次加载全部参考资料。
