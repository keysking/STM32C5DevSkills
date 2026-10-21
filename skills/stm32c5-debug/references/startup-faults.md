# 启动失败、HSE 与 HardFault

按实际现象选择分支，不把这些检查视为要求主动复位或触发故障。GDB 命令用于已连接且允许暂停的目标；具体地址除架构寄存器地址外均须取实际现场。未列出的命令用 GDB `help` 查询。

## 未到 main

1. 检查 ELF 型号、链接地址、向量表和实际下载地址。
2. 向量首项应为目标 RAM 内合法初始 SP，第二项为带 Thumb 位的 Reset_Handler 地址。
3. 跟踪 Reset_Handler → SystemInit → C runtime → main，检查 `.data` 复制、`.bss` 清零和栈范围。
4. 有 bootloader 时检查跳转 SP、PC、VTOR、中断清理和应用偏移，不能只改烧录地址。

## main 内初始化失败

在 `mx_system_init()` 返回前定位第一个失败模块。C5 默认模板可能直接 `return -1`，所以“卡住”也可能是初始化失败后的退出路径。

时钟失败先看 RCC 返回状态、HSE ready 和电源条件。编译期缺 HSE 声明时检查 `USE_EXTERNAL_ENV → stm32_external_env.h → HSE_VALUE`；运行期 HSE 超时才继续查晶振/bypass、频率和电路。不要通过删除检查强行继续运行。

`HAL_Delay` 不返回：观察 HAL tick 是否递增，再查 SysTick 或 TIM 时间基准的向量、IRQ、优先级和屏蔽状态。RTOS 中区分 HAL tick 与内核 tick。

## HardFault 现场

先暂停并保留寄存器。Cortex-M 系统控制块地址可读取：

```gdb
x/wx 0xE000ED28
x/wx 0xE000ED2C
x/wx 0xE000ED34
x/wx 0xE000ED38
info registers pc lr msp psp xpsr
bt
```

依次是 CFSR、HFSR、MMFAR、BFAR。只有有效位成立时才解释 fault address；不要把残留值当作本次访问地址。通过本机 CMSIS `SCB` 定义或对应 Cortex-M33 文档解码位。

根据异常入口保存的 EXC_RETURN 判断原线程使用 MSP/PSP，以及是否有浮点扩展帧。若是**基本异常帧且未被当前处理函数压栈改变位置**，堆栈依次是 r0、r1、r2、r3、r12、lr、pc、xPSR。不能对任意当前 `$sp` 无条件取第七个字当故障 PC。

从保存 PC 反汇编并关联源码：

```gdb
info line *0x08001234
x/8i 0x08001234
```

地址示例必须换成实际堆栈 PC。检查空/悬空指针、栈越界、非法指令/返回、未对齐访问、错误 FPU ABI、DMA 覆盖和外设访问时钟。

修复后使用原输入与运行条件复现；偶尔不再崩溃不足以证明根因消失。没有硬件时只能核对源码、向量和链接，明确现场证据缺失。
