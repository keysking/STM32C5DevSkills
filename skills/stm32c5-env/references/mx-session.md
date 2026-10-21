# CubeMX2 启动、路径与会话

按正常调用、特定环境故障或显式会话选择分支；正常工作时无需运行后续修复操作。设备 CPN、工程路径和软件工程名须换成当前任务的实际值。未列出的启动选项查看 `cube lifecycle-mx --help` 及相应叶子命令帮助。

## 正常调用

```text
cube mx --version
cube mx finder device info --cpn STM32C532CCT6
```

正常情况下 CLI 自动连接或启动后端。成功输出应是 JSON，`status` 为 `success` 且含目标设备信息。命令选项帮助不需要后端，因此不能用 `--help` 代替这个检查。

后续 PowerShell 示例先在自己的工程目录执行：

```powershell
$p = (Resolve-Path -LiteralPath './C5App.ioc2').Path
cube mx project get-sw-project-list -p "$p"
$s = 'C5App'
```

`$p` 必须指向实际 `.ioc2`；`$s` 从上一步返回的软件工程名称选择，示例新建工程名称才是 `C5App`。工程可能有多个软件工程，不能盲目用文件名替代查询结果。

Bash/zsh 同样定义变量后使用相应的单行命令。软件工程选项按叶子帮助使用；`sw-config` 支持 `-s`，`peripherals enable` 需用 `--software-project`：

```bash
p="$(pwd)/C5App.ioc2"
s='C5App'
test -f "$p"
cube mx project get-sw-project-list -p "$p"
```

## IDE 中的 Electron 环境冲突

若后端被当成 Node 程序运行，检查 `ELECTRON_RUN_AS_NODE`。它可由 IDE 注入。只在本次子进程/终端中移除；不要永久改系统环境。

PowerShell 执行一个命令并恢复原值：

```powershell
$hadElectronFlag = Test-Path Env:ELECTRON_RUN_AS_NODE
$savedElectronFlag = $env:ELECTRON_RUN_AS_NODE
try {
    Remove-Item Env:ELECTRON_RUN_AS_NODE -ErrorAction SilentlyContinue
    cube mx finder device info --cpn STM32C532CCT6
} finally {
    if ($hadElectronFlag) { $env:ELECTRON_RUN_AS_NODE = $savedElectronFlag }
}
```

多条命令可放在同一个 `try` 内。Linux/macOS 可逐条使用 `env -u ELECTRON_RUN_AS_NODE cube mx ...`。不要为了消除这个错误改动固件工程。

## 显式持久会话

自动启动超时或需要连续执行很多查询时：

1. 在独立终端中清除该终端的 Electron 标志，运行 `cube lifecycle-mx start-only`。
2. 等待输出 `BACKEND_PORT 数字`；该进程保持运行。读取输出即可，不要等待后端退出才继续。
3. 在操作终端设置 `$mxPort` 为刚输出的数字，在每个**叶子命令**后追加 `--port "$mxPort"`。例如：

```powershell
$mxPort = Read-Host '输入刚输出的 BACKEND_PORT'
cube mx finder device info --cpn STM32C532CCT6 --port "$mxPort"
cube mx project get-sw-project-list -p "$p" --port "$mxPort"
```

4. 本次有修改时先保存；关闭本次打开的工程后，只终止本次启动并掌握 PID 的后端；不要结束所有 CubeMX2 进程，避免干扰用户打开的工程。通过系统进程工具核对 PID、命令行和启动时间再终止。

端口不是固定值，不使用旧日志中的端口。连接失败先核对进程和端口；新会话重新打开磁盘上的 `.ioc2`。直接运行 GUI 后端时缺 `CMSIS_PACK_ROOT` 等变量，应返回 cube 入口检查解析环境。

## 结果与失败判断

每一步检查退出码和 JSON `status`。错误响应可能包含参数、资源或约束原因；先读完整错误，再查当前命令 `--help`。

修改任务确需条件参数时，先启用必要父项再重新 `list-parameters`；纯查询不为取得子参数而修改现有工程。软件工程、Pack 版本或资源名称错误时，换一个猜测路径通常只会叠加错误。不要将旧命令失败后的空输出当作配置为空。
