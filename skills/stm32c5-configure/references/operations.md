# CubeMX2 配置操作基础

先读[启动会话](../../stm32c5-env/references/mx-session.md)。以下示例假定当前目录有 `C5App.ioc2`：

```powershell
$p = (Resolve-Path -LiteralPath './C5App.ioc2').Path
cube mx project get-sw-project-list -p "$p"
$s = 'C5App'
cube mx peripherals list -p "$p"
cube mx sw-config list-configurations -p "$p" -s "$s"
```

选择实际返回的软件工程名替换 `$s`。如果正在使用显式后端，每条命令追加 `--port "$mxPort"`；不用端口时由 CLI 自动连接。Bash 对应 `p="$(pwd)/C5App.ioc2"`、`s='C5App'`。

## 查询与修改一个参数

以已启用 USART1 的工程为例：

```powershell
cube mx sw-config list-parameters -p "$p" -s "$s" -r USART1
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --value 115200
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
```

先确认类型、枚举 `const`、范围和单位。显示名称不一定是内部值。路径包含空格、`@` 或 `/` 时整体引用。资源名来自 `list-configurations`；无绑定硬件的组件可能使用 `HALCore`、`FreeRTOS`，不是外设名。

启用 DMA、中断、通道、数组元素等父项后，重新 `list-parameters` 才能看到新增子路径。参数列表可能较长，在本地保存 JSON 并筛选当前关键字，不把所有外设的完整 schema 一次装入上下文。

## 保存、重载、生成

```powershell
cube mx project save -p "$p"
cube mx project close -p "$p"
cube mx project get-sw-project-list -p "$p"
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r USART1 --parameter-path basic/baud_rate --use-internal-value
```

将最后一条换成刚修改的参数。随后执行[暂存导出与合并](../../stm32c5-project/references/regenerate.md)，查看初始化、时钟、IRQ、DMA 和构建差异，再构建。`set` 返回成功只代表后端接受了修改。

## API 不支持或模型冲突

1. 查询该叶子命令 `--help`，确认当前版本、资源和参数路径。
2. 读取完整错误和当前配置，判断是父项未启用、枚举无效、引脚冲突还是后端问题。
3. 保存可恢复状态后，在 CubeMX2 GUI 的同一工程检查冲突；不反复 disable/enable 试图清空复杂资源。
4. GUI 保存后关闭并用 CLI 重载回读，再生成。

CLI 无法表达某项设置时可使用 GUI 或维护现有手写初始化；明确配置所有者和后续生成边界。不静默修改 `.ioc2` 的嵌套字符串来绕过校验。
