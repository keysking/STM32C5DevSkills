# 工具发现与依赖恢复

按工具发现、缺失依赖恢复或原生程序定位选择章节。路径和版本用于说明查找方式，项目约束优先；找到所需工具后不继续安装无关组件。

## 找到实际程序

Windows PowerShell：

```powershell
Get-Command cube -All -ErrorAction SilentlyContinue
where.exe cube
Get-ChildItem -LiteralPath "$env:USERPROFILE\.local\stm32cube\bin" -Filter 'cube*' -ErrorAction SilentlyContinue
```

最后一条只检查常见位置，找不到并不代表未安装。继续检查 ST 官方安装器的安装目录、系统已安装应用及用户提供的工具目录。不要递归扫描整个系统盘。找到后将其所在目录加入当前会话 PATH，或使用 `& $cubeExe ...` 调用经过确认的绝对路径。

Linux/macOS：

```bash
command -v cube
type -a cube
ls "$HOME/.local/stm32cube/bin"
```

`$HOME` 只读使用，不重新赋值。PATH 修复后在工程目录执行：

```text
cube --version
cube --list
cube --json --resolve mx
cube mx --version
cube --json bundle list
cube --get-current-value cmsis_pack_root
cube pack --help
```

`--resolve mx` 的 `command` 是实际命令链，`bundle` 是选中组件，导出的环境字段解释运行时依赖。安装文件夹名称可能保留旧版本，不能代替实际版本输出。`cube mx` 不存在时，先检查 CubeMX2 是否安装并向 cube 注册了它的 Bundle，再按官方安装方式补装。

## 检查或恢复 Bundle 与 Pack

Bundle 是主机工具，如 CubeMX2、GCC、CMake、Ninja。Pack 是固件依赖，如 C5 DFP、HAL2、CMSIS、FreeRTOS。缓存中有 `.pdsc` 元数据并不代表 `.pack` 已安装。

```text
cube bundle --help
cube bundle list
cube bundle install --help
cube pack list --help
cube pack list packs
cube pack install --help
```

先从 `cube-dcm.json`、工程组件和预设取得要求的版本，再按本机 `bundle install --help` 提供的仓库/标识参数安装。Bundle 的标识与 Pack 标识不同，不把 `VENDOR.NAME.VERSION` 套给 Bundle。

仅当当前任务缺少对应 Pack 且已确定使用 2.1.0 时，分别使用以下安装形式；不作为每次环境检查的必执行列表：

```text
cube pack install STMicroelectronics.stm32c5xx_dfp.2.1.0
cube pack install STMicroelectronics.stm32c5xx_hal_drivers.2.1.0
cube pack install STMicroelectronics.stm32cubemx2_templates_no_os_stm32c5.2.1.0
```

仅当工程需要 FreeRTOS、缺少该 Pack 且约束为 2.2.0 时：

```text
cube pack install STMicroelectronics.freertos.2.2.0
```

若版本不可用，检查官方 Pack 索引、网络/代理和工程版本约束；不要无声替换成最新版本。CubeMX2 创建/启用组件也可能自动下载依赖，完成后回读 Composer 的实际组件和 Pack。安装软件附带的许可仍由各供应商管理。

## 找到编译器及底层 exe

`cube --list` 只列出可通过 cube 转发的命令。GCC Bundle 可能仅导出 `bin` 到运行环境，**没有 `cube arm-none-eabi-gcc` 这个命令**。

```powershell
$bundles = cube --json bundle list | ConvertFrom-Json
$bundles | Select-Object path, details
$gccBundles = @($bundles | Where-Object { $_.details.name -eq 'gnu-tools-for-stm32' })
$gccBundles | ForEach-Object { $_.details.version; $_.path }
```

按工程要求从结果选中一个版本；不要直接取第一项。将其 `path/bin` 中的 `arm-none-eabi-gcc`、`size`、`objdump`、`objcopy` 用于构建或检查，见[构建步骤](../../stm32c5-build/references/build.md)。

需要烧录或调试时，选择对应工具进行查询；仅配置工程无需检查全部工具：

```text
cube --json --resolve programmer
cube programmer --help
cube --json --resolve stlink-gdbserver
cube stlink-gdbserver --help
cube arm-none-eabi-gdb --version
```

需要 GUI 时，从 `--resolve mx` 定位到 CubeMX2 安装树，再检查同一安装树内实际 GUI 文件和版本。传统 CubeMX.exe 与 CubeMX2 后端不能互换。通常继续用 `cube mx` 操作，避免丢失由 cube 注入的 Pack 和 Bundle 环境。

完成修复后验证所修复的能力：CubeMX2 用[真实后端查询](mx-session.md)，编译环境用对应工程构建；只定位程序时报告实际路径与版本即可。

依据：[STM32CubeCLI 文档](https://dev.st.com/stm32cube-docs/stm32cubecli/latest/en/index.html)、本机各叶子命令的 `--help`。工具升级后重新检查叶子帮助，不将此版本列表视为永久全集。
