# 连接与烧录恢复

按从小到大的影响排查：

1. 探针是否枚举；USB 数据线、驱动/udev、访问权限是否正确。
2. 目标供电、GND、VTref、SWDIO、SWCLK 和 NRST 是否连接；不要把探针电压参考当成供电能力。
3. 是否有 CubeProgrammer/GDB server/IDE 独占探针。只停止已确认属于本次任务的服务，不结束所有调试程序。
4. 降低 SWD 频率，例如 `freq=1000` 或更低的工具支持值，再普通连接。
5. 固件进入低功耗、修改调试引脚或快速复位时，在确认 NRST 连接后使用 under-reset。

示例变量 `$sn` 来自[枚举步骤](program.md)：

```powershell
cube programmer -c port=SWD "sn=$sn" freq=1000 mode=UR reset=HWrst
```

连接成功后先读取状态，确认是否需要重新下载；不要顺手执行全片擦除。UR 不是绕过访问保护的通用方法。

| 失败 | 进一步证据 |
| --- | --- |
| 无 ST-Link | 系统 USB 枚举、驱动、权限、线缆 |
| 有探针无目标 | VTref、SWD/NRST、电路、频率、低功耗 |
| 不支持器件 | Programmer/DFP 版本和精确 C5 型号 |
| 写入地址错误 | ELF/BIN 类型、链接区域、bootloader |
| 写入成功校验失败 | 供电、Flash 保护、地址、外部 loader |
| 校验成功不启动 | 启动配置、向量、时钟、Fault，转[启动诊断](../../stm32c5-debug/references/startup-faults.md) |

解除读保护、全片擦除、修改选项字节或生命周期可能清空数据或改变后续访问能力。执行前说明具体器件、当前状态、拟改字段及后果，取得该动作的授权；此前的“烧录应用”授权不能自动扩大到这些操作。
