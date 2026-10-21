# CubeMX2 配置命令用法

本目录按场景提供命令参考，各节不是待依次执行的脚本。先确定本次需要查询或改变的对象，再选择相关操作。参数由用户需求、原理图和现有工程确定；固定名称及数值仅说明语法，不构成默认配置。

## 选择工程与软件配置

工具未就绪时读[启动会话](../../stm32c5-env/references/mx-session.md)。以下为 PowerShell，将路径替换为实际 `.ioc2`，不要因文件名不同而创建新工程：

```powershell
$p = (Resolve-Path -LiteralPath './C5App.ioc2').Path
cube mx project get-sw-project-list -p "$p"
```

从返回列表选择本次目标软件工程并赋给 `$s`。假定实际名称为 `C5App`：

```powershell
$s = 'C5App'
cube mx peripherals list -p "$p"
cube mx sw-config list-configurations -p "$p" -s "$s"
```

显式后端的每条命令追加 `--port "$mxPort"`。Bash 对应 `p="$(pwd)/C5App.ioc2"`、`s='C5App'`，同样替换真实值。

## 只读查询参数及命令形式

本文不能列出所有设备和版本的参数。资源名取自 `list-configurations`，路径取自 `list-parameters`；资源也可能是 `HALCore`、`FreeRTOS`。假定目标为已启用 USART1 的波特率：

```powershell
$resource = 'USART1'
$parameter = 'basic/baud_rate'
cube mx sw-config list-parameters -p "$p" -s "$s" -r "$resource"
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r "$resource" --parameter-path "$parameter" --use-internal-value
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r "$resource" --parameter-path "$parameter"
cube mx sw-config get-parameter-value --help
cube mx sw-config set-parameter-value --help
```

纯查询取得结果后结束，不执行设置、保存、关闭、生成或构建。资源不存在时先确认任务是否要求启用它；不要为查参数改动现有工程，可改查设备能力、帮助或隔离工程。

## 修改已确定的参数

仅在需求要求修改时执行。先读当前值，核对类型、单位、范围及枚举 `const`。显示标签未必是内部写入值。以下表示“将已选参数改为需求值”；115200 只适用于需求确实为该波特率时：

```powershell
$value = '115200'
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r "$resource" --parameter-path "$parameter" --value "$value"
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r "$resource" --parameter-path "$parameter" --use-internal-value
```

含空格、`@` 或 `/` 的路径整体引用。启用 DMA、IRQ、通道等父项后重新列出参数。数组先读父项确认元素和实际索引；仅需新增对象时才添加，不把索引 0 当成当前目标。

长参数列表在本地按当前外设和关键词筛选，不载入所有 schema。只修改本次要求的字段及必要依赖，已有正确值不必重设；轮询、中断、DMA 等按需求选择，不逐节升级。

## 修改后的保存与重载

确认本次修改回读正确后保存；关闭前处理好同一会话的其他变更：

```powershell
cube mx project save -p "$p"
cube mx project close -p "$p"
cube mx project get-sw-project-list -p "$p"
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r "$resource" --parameter-path "$parameter" --use-internal-value
```

重新访问会从磁盘加载。回读本次变更项及关键依赖，确认不仅改变后端内存。`set` 成功不等于已保存或已更新固件。

需要把配置落实到代码时，执行[暂存导出与合并](../../stm32c5-project/references/regenerate.md)，检查差异并构建。若只要求修改配置文件，保存重载后交付并说明生成代码尚未更新。构建、烧录和功能分别验证，不自动扩大任务范围。

## 命令失败或配置不生效

1. 查询失败叶子命令的 `--help`，核对版本、必选参数和路径。
2. 检查资源及父项，区分未启用、枚举无效、索引错误、引脚冲突和后端问题。
3. 需要 GUI 解冲突时保留可恢复状态，再打开同一工程，不反复全局 disable/enable 清空资源。
4. GUI 修改保存后，用 CLI 重载、回读本次目标。

CLI 无法表达某设置时可用 GUI 或维护现有手写初始化，明确配置所有者及生成边界。不直接编辑 `.ioc2` 的嵌套字符串绕过校验。
