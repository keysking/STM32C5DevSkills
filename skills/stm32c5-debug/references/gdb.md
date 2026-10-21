# ST-Link GDB 会话

按保留现场的 attach、需要复位的调试或静态 ELF 分析选择操作，不能从头执行全部命令。序列号、端口和 ELF 路径均取实际对象，61234 仅为端口写法。使用与设备当前固件匹配的带符号 ELF。先检查[工具环境](../../stm32c5-env/references/tools.md)，再枚举探针：

```text
cube stlink-gdbserver --help
cube stlink-gdbserver -q
cube arm-none-eabi-gdb --version
```

仅在需要连接硬件且没有可复用的本次 server 时，在终端 A 设置实际序列号与未占用端口：

```powershell
$sn = Read-Host '目标 ST-Link 序列号'
$gdbPort = 61234
cube stlink-gdbserver -d -i "$sn" -p "$gdbPort" --frequency 1000 -g
```

`-g` 是 attach；连接与调试仍可能改变时序。需要保留 Fault 现场时不加复位参数，也不在 GDB 中执行 `load`。若 server 要求 Programmer 路径，用 `cube --json --resolve programmer` 找到实际安装并按 `-cp` 的帮助传入正确安装目录，不凭猜测拼路径。

自动化后台启动 Windows 服务时使用 `Start-Process -WindowStyle Hidden`、重定向日志并保存 PID。终端方式保持服务运行；不要在等待服务退出后才启动 GDB。

终端 B：

```text
cube arm-none-eabi-gdb ./build/debug_GCC_STM32C532CCT6/C5App.elf
```

在 GDB 中输入，端口与终端 A 一致：

```gdb
target extended-remote localhost:61234
monitor help
info registers
bt
x/8i $pc
```

需要暂停时使用当前 server 支持的 halt 命令或 GDB interrupt；读取 `monitor help` 后再用相应命令。不要把其他 server 的 monitor 语法直接套入。

## 从复位调试应用

需要从复位开始且已保存现场时，按 monitor 帮助复位并暂停。下面分别为下载、设断点及继续运行，按需求选择；无需更新目标固件时跳过 load：

```gdb
load
break main
continue
```

`load` 会写目标存储，不能当作只读检查。若固件已匹配则无需重复下载。逐步查看 `mx_system_init()` 返回值、RCC 状态和外设初始化失败位置。

## 常用证据

```gdb
info files
info registers pc lr sp xpsr msp psp
bt
x/16wx $sp
x/8i $pc
print SystemCoreClock
```

没有符号时用 `arm-none-eabi-addr2line -e firmware.elf -f -C 地址` 对照精确同版 ELF。优化会影响局部变量和单步行为；先保留可重现构建，再按需建立 Debug 配置。

结束时 `detach`、`quit`，按任务要求恢复目标运行；只关闭本次 PID 的 server。报告是否修改了目标固件、复位或暂停了设备。
