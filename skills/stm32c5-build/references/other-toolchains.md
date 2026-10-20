# 沿用已有构建体系

## Makefile

读 README、Makefile 和环境要求，检查 CC、设备宏、链接脚本与输出目标：

```text
make --version
make -n
make -j4
```

`make -n` 仅预览，但递归 make 等规则可能仍执行命令；先读构建规则，不对陌生脚本假定绝对只读。按实际目标使用 `make target`，不要无条件执行 clean。

## IAR

使用工程当前支持的 IAR 产品版本，先定位 `IarBuild` 并读其帮助。典型已有工程命令：

```text
IarBuild app.ewp -build Debug
```

`app.ewp` 和 `Debug` 从实际工程取得。核对设备支持、C5 启动文件、ICF、runtime 和许可。EWARM 是 CubeMX2 可见的导出格式之一，但仍要在当前版本的 `ide-project list-format` 与 build target 中确认组合。

## Keil / Arm Compiler

已有 `.uvprojx` 工程沿用其设备包、scatter 和编译器设置。在支持的 Windows 主机定位 `UV4.exe`，读帮助后使用原生命令行构建；典型形式为：

```text
UV4.exe -b app.uvprojx -t Debug -o build.log
```

工程名和 target 来自实际工程，读取日志并确认生成的 AXF/ELF。不要将 CubeMX2 列出的 `debug_AC6` 编译目标解释为一定能直接导出 `.uvprojx`；导出格式列表与编译器目标列表必须分别查询。

## 何时迁移

只有用户要求迁移，或当前体系确实无法支持目标且已明确替代方案时，才建立新构建体系。迁移时逐项对照源文件、宏、include、启动/链接、优化、浮点 ABI 和库，先构建最小应用再恢复完整功能。

无法在当前主机运行原工具时，给出已完成的静态检查和所需原生主机条件；不把 CMake 构建通过冒充 IAR/Keil 验证。
