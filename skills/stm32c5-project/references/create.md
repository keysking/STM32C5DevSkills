# 新建 C532 工程：串口输出

目标：`STM32C532CCT6`，USART1 TX=PA9、RX=PA10，115200 baud，HAL2，裸机，CMake/Ninja/GCC。这是软件实例；连接 USB 串口时确认 3.3 V 电平、共地和交叉 TX/RX。

## 1. 准备环境与空目录

先执行[环境发现](../../stm32c5-env/references/tools.md)和[后端查询](../../stm32c5-env/references/mx-session.md)。此处示例 Pack 为 2.1.0；已有项目使用其约束版本。

在希望放置工程的父目录执行 PowerShell：

```powershell
if (Test-Path -LiteralPath './C5App') { throw 'C5App 已存在，请先检查内容或另选新目录' }
New-Item -ItemType Directory -Path './C5App' | Out-Null
Set-Location -LiteralPath './C5App'
$root = (Get-Location).Path
cube mx project create-from-mcu --cpn STM32C532CCT6 --pack-name stm32c5xx_dfp --pack-version 2.1.0 --project-location "$root" --project-name C5App
$p = (Resolve-Path -LiteralPath './C5App.ioc2').Path
cube mx project get-sw-project-list -p "$p"
$s = 'C5App'
```

Bash 的目录准备使用 `mkdir C5App && cd C5App`，`root="$(pwd)"`；其余创建命令相同。每个命令成功后再执行下一步。已有同名工程不覆盖。

## 2. 先分配引脚，再启用外设

```powershell
cube mx pinout pin -p "$p" --pin PA9
cube mx pinout pin -p "$p" --pin PA10
cube mx pinout assign-signal -p "$p" --pin PA9 --signal USART1_TX
cube mx pinout assign-signal -p "$p" --pin PA10 --signal USART1_RX
cube mx peripherals view -p "$p" --peripheral USART1
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral USART1 --mode Async
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --value 115200
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
cube mx project save -p "$p"
cube mx project close -p "$p"
cube mx project get-sw-project-list -p "$p"
cube mx pinout pin -p "$p" --pin PA9
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
```

预期 PA9 对应 `USART1_TX`，PA10 对应 `USART1_RX`，`pinned=true`，波特率 115200。关闭后再次访问会从磁盘载入工程。若自动分配到了 PA15，先检查原先是否已经启用 USART1，不继续套用“全新工程”的步骤。

## 3. 选择目标并导出

```powershell
cube mx ide-project list-format
cube mx ide-project build-target list -p "$p"
$target = 'C5App.debug_GCC+STM32C532CCT6'
$out = Join-Path $root 'export-initial'
if (Test-Path -LiteralPath $out) { throw '导出目录必须是新的空目录' }
cube mx ide-project generate -p "$p" -s "$s" --build-target "$target" --destination "$out" --format CMake --source include-packs-from-local
```

`$target` 必须在上一步列表中存在；换软件工程名称、编译器或型号后重新选择。Bash：`target='C5App.debug_GCC+STM32C532CCT6'`，`out="$root/export-initial"`，先 `test ! -e "$out"`。

导出应含 `CMakeLists.txt`、`CMakePresets.json`、`main.c` 和 `generated/hal/`。检查 `mx_usart1.*` 的实际函数和 handle；在 `mx_system_init()` 成功之后按[HAL2 串口实例](../../stm32c5-firmware/references/peripheral-code.md)发送数据。

## 4. 构建与验证

进入 `$out`，按[构建步骤](../../stm32c5-build/references/build.md)配置、构建。先确认空应用可构建，再加入应用代码。

预期取得对应 C532 的 ELF。连接硬件并得到授权后，按[烧录步骤](../../stm32c5-flash/references/program.md)下载；串口终端使用与生成配置一致的数据位、校验、停止位和 115200 波特率，观察发送字符串。

完成后更新配置时使用[暂存重新生成](regenerate.md)，不再把 `--force` 加到原导出目录。
