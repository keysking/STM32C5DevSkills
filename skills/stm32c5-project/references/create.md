# 新建 C5 工程与导出目标

本页说明创建、选择软件工程、按需配置和导出代码等操作，不指定一个串口或 LED 应用。名称、CPN、Pack 版本和目录须按实际任务确定；下列 `C5App`、`STM32C532CCT6`、2.1.0 只用于展示命令形式。已有工程进入[接手工程](existing-and-examples.md)，不重新创建。

## 查询创建能力及所需版本

工具未就绪时读[环境发现](../../stm32c5-env/references/tools.md)及[后端查询](../../stm32c5-env/references/mx-session.md)。完整 CPN 来自需求、芯片或板卡；Pack 取已确认的兼容版本。

```powershell
cube mx project create-from-mcu --help
cube mx finder device info --help
cube mx finder device info --cpn STM32C532CCT6
cube pack list packs
```

查询返回的型号、封装、容量须与目标一致。本文未覆盖的创建来源、组件及编译器选项，继续查当前叶子帮助和设备模型，不能把代表型号当成所有 C5。

## 从已确认的 MCU 新建工程

仅当任务要求新建时执行。假定目标确为 C532CCT6、DFP 2.1.0，且选择名称 C5App。在实际父目录执行 PowerShell：

```powershell
$projectName = 'C5App'
$cpn = 'STM32C532CCT6'
$packVersion = '2.1.0'
$root = Join-Path (Get-Location).Path $projectName
if (Test-Path -LiteralPath $root) { throw '目标目录已存在，先检查内容或另选新目录' }
New-Item -ItemType Directory -Path $root | Out-Null
cube mx project create-from-mcu --cpn "$cpn" --pack-name stm32c5xx_dfp --pack-version "$packVersion" --project-location "$root" --project-name "$projectName"
$p = (Resolve-Path -LiteralPath (Join-Path $root "$projectName.ioc2")).Path
cube mx project get-sw-project-list -p "$p"
```

每步检查退出码和 JSON `status`。Bash 使用经检查的新目录、`root="$(pwd)/C5App"` 及对应变量赋值；不要跨 shell 复制 PowerShell 语法。

从返回列表选择实际软件工程名。假定返回目标名为 C5App：

```powershell
$s = 'C5App'
cube mx sw-config list-configurations -p "$p" -s "$s"
cube mx peripherals list -p "$p"
```

未指定工具链和运行模型的新工程默认 CMake/Ninja/GCC、HAL2、裸机；明确需求优先。创建成功不隐含必须添加 UART、LED 或 RTOS。

## 按任务添加必要配置

只配置需求涉及的时钟、引脚和外设。新增外设先查询引脚支持与占用，再分配已确认信号并启用，见[配置命令用法](../../stm32c5-configure/references/operations.md)。只需空工程时跳过外设配置；需要 FreeRTOS 时再读[中间件配置](../../stm32c5-configure/references/freertos.md)。

创建或修改后的配置需要落盘时：

```powershell
cube mx project save -p "$p"
cube mx project close -p "$p"
cube mx project get-sw-project-list -p "$p"
```

重载后回读本次配置的具体对象；没有配置 USART1 就不查询或设置 USART1 波特率。

## 选择导出格式与构建目标

需要生成源代码时查询：

```powershell
cube mx ide-project list-format
cube mx ide-project build-target list -p "$p"
cube mx ide-project generate --help
```

`$target` 必须取返回列表中与软件工程、器件和编译器匹配的条目。假定选定的是 `C5App.debug_GCC+STM32C532CCT6`，且需求采用 CMake：

```powershell
$target = 'C5App.debug_GCC+STM32C532CCT6'
$out = Join-Path $root 'export-initial'
if (Test-Path -LiteralPath $out) { throw '导出目标必须是新的目录' }
cube mx ide-project generate -p "$p" -s "$s" --build-target "$target" --destination "$out" --format CMake --source include-packs-from-local
```

`include-packs-from-local` 的依赖路径须在当前主机可用。换平台时恢复对应依赖，不能复用另一主机的绝对路径。其他格式以实际支持为准，不为采用本页命令强行迁移工具链。

## 检查交付结果

配置工程检查 CPN、软件工程和本次设置已保存。生成 CMake 工程时检查 `CMakeLists.txt`、`CMakePresets.json`、`main.c` 及实际生成目录；外设文件只应对应已选择的资源。

需要交付可构建工程时进入 `$out`，按[构建方法](../../stm32c5-build/references/build.md)验证 ELF 和目标架构。应用实现按实际功能选取[HAL2 API 参考](../../stm32c5-firmware/references/peripheral-code.md)，不加入固定演示应用。

烧录或板端验证仅在任务包含这部分时按[下载指导](../../stm32c5-flash/references/program.md)执行。后续配置更新使用[暂存重新生成](regenerate.md)，不对已有应用目录直接加 `--force`。
