# Windows、Linux 与 macOS

只使用当前主机对应列；各平台命令是等价选择，不跨平台顺序执行。PATH、USB 规则和后端环境仅在相关问题存在时调整，变量中的目录须来自实际工具发现。

| 项目 | Windows | Linux | macOS |
| --- | --- | --- | --- |
| Shell | PowerShell，原生 exe 用 `&` 加引号路径 | Bash；检查可执行位 | Bash/zsh；核对工具支持的 CPU 架构 |
| 查程序 | `Get-Command cube -All` | `command -v cube` | `command -v cube` |
| 当前 PATH | `$env:PATH = "$bin;$env:PATH"` | `export PATH="$bin:$PATH"` | 同 Linux |
| 路径 | 尽量绝对路径，空格须引用 | 大小写敏感 | 不假设大小写不敏感 |
| ST-Link | 官方 USB 驱动 | 官方 udev 规则及用户组，重新登录/插拔 | 官方工具及 USB 访问状态 |
| 后端环境冲突时 | 移除子进程 `ELECTRON_RUN_AS_NODE` | `env -u ELECTRON_RUN_AS_NODE ...` | 同 Linux |

不要将 Windows 下生成的绝对 Pack 路径直接用于 Linux。换主机时恢复依赖，再重新生成对应构建目录。不要跨平台复用 CMakeCache。

如果 `cube`、CubeMX2 或某个编译器版本没有当前 OS/架构的官方包，应使用官方支持的主机环境；不能把 Windows exe 当作本机原生工具。Keil MDK 的构建一般需要 Windows；已有 IAR 工程需核对所用产品的主机支持。WSL 中的 USB 转发和 Windows 本机访问是两条路径，明确选择后再配置。

权限失败先查设备、规则、组和占用进程，不以长期管理员/root 运行代替修复。不自动解除系统下载隔离、安全策略或修改保护设置。

本包命令主体以 PowerShell 单行为主。Bash 使用同名 `$p`、`$s`、`$out` 字符串变量；把 `Resolve-Path` 换成经过检查的绝对路径、`$env:PATH` 换成 `PATH`。保留命令参数，不把 PowerShell 反引号当作 Bash 续行符。
