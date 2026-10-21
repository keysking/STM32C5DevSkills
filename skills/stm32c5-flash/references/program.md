# 用 CubeProgrammer 下载与校验

按实际固件格式选择 ELF/HEX 或 BIN 路径，不依次下载两种文件。路径、SWD 频率及地址用于说明参数形式，须按构建产物、分区和连接条件确定。这里只列常用操作，其他参数查 `cube programmer --help`。

## 确认工具、固件和板卡

```text
cube --json --resolve programmer
cube programmer --version
cube programmer --help
cube programmer -l st-link-only
```

`st-link-only` 枚举探针，避免把枚举本身当作已连接验证。若 cube 未注册 Programmer，按[工具发现](../../stm32c5-env/references/tools.md)定位官方 `STM32_Programmer_CLI`，读取其版本/帮助后执行相同参数。

从枚举结果选择目标探针序列号；PowerShell：

```powershell
$sn = Read-Host '输入本次目标 ST-Link 序列号'
$firmware = (Resolve-Path -LiteralPath './build/debug_GCC_STM32C532CCT6/C5App.elf').Path
cube programmer -c port=SWD "sn=$sn" freq=4000
```

`$firmware` 使用本次成功构建的实际文件；Bash 可用 `read -r sn`、`firmware="$(pwd)/build/.../实际文件.elf"`，必须检查存在后执行。4000 是 SWD kHz 的一种写法，按板卡和当前稳定连接值选择；连接会影响运行状态，确认本次对象。只枚举探针时无需选择文件或连接目标。

检查连接输出中的器件 ID、容量、电压和探针序列号。目标与预期不一致时停止下载，先核实连线和固件型号。

## 下载带地址的 ELF 或 HEX

仅当所选产物为该格式时使用，`$firmware` 指向其中一个实际文件：

```powershell
cube programmer -c port=SWD "sn=$sn" freq=4000 -w "$firmware" -v
```

要求退出码成功且日志明确下载、校验成功。校验失败不继续复位，先查供电、保护、地址及工具版本。

## 复位启动

仅当本次任务要求下载后启动应用，且所选格式的下载校验成功后执行；只验证存储或保留现场时跳过：

```powershell
cube programmer -c port=SWD "sn=$sn" freq=4000 -rst
```

复位后观察应用串口、LED、计数器或协议响应。下载校验只验证存储内容，不证明时钟、引脚或任务行为正确。

## 下载不带地址的 BIN

BIN 无地址信息。从对应 ELF 段、链接脚本或明确的 bootloader 分区获得基地址。仅当实际布局为无 bootloader 且链接到内部 Flash 起点 `0x08000000` 时，下列地址才适用；这不是通用默认下载地址：

```powershell
$bin = (Resolve-Path -LiteralPath './C5App.bin').Path
cube programmer -c port=SWD "sn=$sn" freq=4000 -w "$bin" 0x08000000 -v
```

存在 bootloader 或偏移分区时，必须替换为实际布局，检查向量表与 VTOR。不能为了让校验通过改用任意地址。

## 完成报告

给出实际文件、目标型号、探针、地址来源、下载/校验/复位结果。没有硬件时只说明产物与命令准备就绪，不能输出虚构的 Device ID、校验成功或运行日志。
