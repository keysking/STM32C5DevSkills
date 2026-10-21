# CMake / Ninja / GCC 构建与产物

本页分别说明工具选择、构建和产物检查；只检查现有 ELF 时无需重新配置或增加 map/BIN 输出。版本、preset、路径和目标名仅展示语法，取实际工程值。在包含 `CMakeLists.txt`、`CMakePresets.json` 的导出目录执行。`.ioc2` 所在目录不一定是构建根目录。

## 找到并配置工具

```powershell
$bundles = cube --json bundle list | ConvertFrom-Json
$bundles | Where-Object { $_.details.name -in @('gnu-tools-for-stm32', 'ninja') } | Select-Object path, details
```

按项目要求选择输出中的版本。下方版本仅演示筛选语法，先替换为项目要求的版本；匹配不到时返回[环境恢复](../../stm32c5-env/references/tools.md)，不继续使用空路径：

```powershell
$gcc = @($bundles | Where-Object { $_.details.name -eq 'gnu-tools-for-stm32' -and $_.details.version -eq '14.3.1+st.2' })
$ninja = @($bundles | Where-Object { $_.details.name -eq 'ninja' -and $_.details.version -eq '1.13.2+st.1' })
if ($gcc.Count -ne 1 -or $ninja.Count -ne 1) { throw '请选择工程要求的唯一已安装版本' }
$gccBin = Join-Path $gcc[0].path 'bin'
$ninjaBin = Join-Path $ninja[0].path 'bin'
$env:PATH = "$gccBin;$ninjaBin;$env:PATH"
& (Join-Path $gccBin 'arm-none-eabi-gcc.exe') --version
cube cmake --version
cube ninja --version
```

无需永久改 PATH。Linux/macOS 从相同 Bundle 输出取得实际目录：

```bash
read -r -p 'GCC Bundle 的 bin 绝对目录: ' gccBin
read -r -p 'Ninja Bundle 的 bin 绝对目录: ' ninjaBin
test -x "$gccBin/arm-none-eabi-gcc"
test -x "$ninjaBin/ninja"
export PATH="$gccBin:$ninjaBin:$PATH"
arm-none-eabi-gcc --version
```

预设明确要求 `GCC_TOOLCHAIN_ROOT` 等变量时再按其含义设置；有的要求 Bundle 根，有的要求 bin，先读工具链文件。不要为解决 PATH 错误改写编译器 ABI。

## 配置与构建

先用 `cube cmake --list-presets` 查询名称，选项以 `cube cmake --help` 为准。仅当实际存在并选择下列 preset 时：

```text
cube cmake --preset debug_GCC_STM32C532CCT6
cube cmake --build --preset debug_GCC_STM32C532CCT6 --parallel 4
```

预设名从当前文件取得，不能用于所有 C5 型号。没有预设的现有 CMake 工程，先读 toolchain，再使用其规定的 `-S/-B/-G/-DCMAKE_TOOLCHAIN_FILE`。不要用主机 x86 编译器代替 ARM 编译器。

换工具链或目标时新建独立 build 目录，保留原目录便于比较；不直接删除整个工程目录。

## ELF、map 与占用

PowerShell 找到当前目标产物：

```powershell
Get-ChildItem -LiteralPath './build' -Filter '*.elf' -Recurse | Select-Object FullName, LastWriteTime, Length
$elf = (Resolve-Path -LiteralPath './build/debug_GCC_STM32C532CCT6/C5App.elf').Path
& (Join-Path $gccBin 'arm-none-eabi-size.exe') "$elf"
& (Join-Path $gccBin 'arm-none-eabi-objdump.exe') -h "$elf"
& (Join-Path $gccBin 'arm-none-eabi-readelf.exe') -h -l "$elf"
```

`C5App.elf` 是语法示意名称；用上一步实际列出的文件替换。Bash 对应 `find build -name '*.elf'` 及 `arm-none-eabi-size "$elf"`。

检查 ELF 为 ARM、段地址与实际 Flash/RAM/bootloader 分区一致。`text+data`、`data+bss` 只是常用估算，还需计入向量、对齐、堆、栈、加载地址及特殊内存区域。

默认导出不一定生成 map。需要分析时，在目标定义之后加入对应选项，目标名取实际 `add_executable`：

```cmake
target_link_options(C5App PRIVATE
  "-Wl,-Map=${CMAKE_CURRENT_BINARY_DIR}/C5App.map"
  "-Wl,--print-memory-usage")
```

在新一次链接中检查 map 的 MEMORY、段分布、最大对象及 discarded sections。报告 Flash/RAM 时写清容量来自哪个订货号/链接区域，而不是全系列上限。

需要 BIN 时：

```powershell
& (Join-Path $gccBin 'arm-none-eabi-objcopy.exe') -O binary "$elf" './C5App.bin'
```

BIN 不含下载地址，交付时附工程的真实起始地址。优先烧录带地址的 ELF/HEX。任务包含板端下载时，再按[烧录流程](../../stm32c5-flash/references/program.md)验证目标。
