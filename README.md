# STM32C5DevSkills

面向 **STM32C5** 的七技能包，通过 `cube`、CubeMX2 和原生工具完成环境准备、工程创建、外设配置、固件开发、构建、烧录与调试。

内容中文为主，采用“简短入口 + 按任务读取的具体实例”。面向 Windows、Linux、macOS，覆盖裸机与 FreeRTOS；新建默认 CubeMX2、HAL2、CMake、Ninja、GCC。适合需要明确操作顺序、参数和检查点的编码助手。

本包整体安装；包内参考资料可相互引用，不依赖其他 STM32 技能包。它提供指令和资料，不包含工具二进制、SDK、运行包装脚本或开发板固件。

## 安装

两种方式选一种，避免重复发现相同技能。

```text
git clone https://github.com/keysking/STM32C5DevSkills.git
cd STM32C5DevSkills
```

### 标准技能目录

克隆本仓库，将七个 `stm32c5-*` 目录一起复制到目标工程的 `.agents/skills/`，或客户端支持的用户级技能目录。不能只复制 `SKILL.md` 或单个技能。

PowerShell，在克隆目录执行；目标路径输入实际工程根目录：

```powershell
$project = Read-Host '目标工程根目录的绝对路径'
if (-not (Test-Path -LiteralPath $project -PathType Container)) { throw '目标工程不存在' }
$dest = Join-Path $project '.agents/skills'
$skills = @(Get-ChildItem -LiteralPath './skills' -Directory -Filter 'stm32c5-*')
if ($skills.Count -ne 7) { throw '请在完整仓库根目录执行' }
foreach ($skill in $skills) {
    if (Test-Path -LiteralPath (Join-Path $dest $skill.Name)) { throw "已存在 $($skill.Name)，先比较已有版本" }
}
New-Item -ItemType Directory -Path $dest -Force | Out-Null
foreach ($skill in $skills) {
    Copy-Item -LiteralPath $skill.FullName -Destination $dest -Recurse
}
```

Linux/macOS Bash，在克隆目录执行：

```bash
read -r -p '目标工程根目录的绝对路径: ' project
test -d "$project" || exit 1
dest="$project/.agents/skills"
skills=(skills/stm32c5-*)
test "${#skills[@]}" -eq 7 || exit 1
for skill in "${skills[@]}"; do
  test ! -e "$dest/$(basename "$skill")" || { echo "已有同名技能，请先比较"; exit 1; }
done
mkdir -p "$dest"
for skill in "${skills[@]}"; do cp -R "$skill" "$dest/"; done
```

重新加载客户端或开始新会话，确认发现七个 `stm32c5-*` 技能。其他支持标准 SKILL.md 的客户端按其目录规则安装完整 `skills/` 内容。

### Codex 插件

插件名 `stm32c5-dev-skills`，显示名称 `STM32C5DevSkills`，版本 `0.1.0`，要求本地执行环境。根 `plugin.json` 和 `.codex-plugin/plugin.json` 使用同一份技能。

在克隆目录运行支持插件命令的 Codex CLI：

```text
codex plugin marketplace add .
codex plugin list
codex plugin add stm32c5-dev-skills@stm32c5-dev-skills
codex plugin list
```

重启会话，确认插件已启用且发现七技能。客户端显示的调用名称可能带 `stm32c5-dev-skills:` 命名空间，按实际发现名称调用。命令不支持时先查看 `codex plugin --help`，也可用标准目录安装。

此为仓库本地 marketplace 安装，不代表插件已上架公共目录。结构依据：[官方插件打包](https://developers.openai.com/plugins/build/plugins)、[本地执行声明](https://developers.openai.com/plugins/deploy/desktop-only)。

## 前置工具

- `cube` / STM32CubeCLI，已安装并可解析 CubeMX2。
- 目标 C5 DFP、HAL2/LL、CMSIS 及工程需要的其他 Pack。
- 新建默认 GCC、CMake、Ninja；已有工程保留其工具链。
- 烧录/调试需要支持目标器件的 STM32CubeProgrammer、ST-Link/GDB 工具及实际硬件。
- FreeRTOS 工程另外需要相应中间件与正确 port。

不要求所有工具预先位于 PATH。先调用 `stm32c5-env`，按[工具定位](skills/stm32c5-env/references/tools.md)和[后端启动](skills/stm32c5-env/references/mx-session.md)找到实际安装与版本。

## 七项能力

| 技能 | 用途 | 请求示例 |
| --- | --- | --- |
| [stm32c5-env](skills/stm32c5-env/SKILL.md) | 工具、版本、依赖、三平台环境 | “找出我的 CubeMX2，并检查 C5 编译环境。” |
| [stm32c5-project](skills/stm32c5-project/SKILL.md) | 新建、接手、例程、生成边界 | “新建 C532CCT6，USART1 用 PA9/PA10。” |
| [stm32c5-configure](skills/stm32c5-configure/SKILL.md) | 引脚、时钟、外设、DMA/IRQ、RTOS | “PA8 输出 1 kHz PWM，保留现有业务代码。” |
| [stm32c5-firmware](skills/stm32c5-firmware/SKILL.md) | HAL2/LL、移植、任务与同步 | “用两个 FreeRTOS 任务通过队列传递数据。” |
| [stm32c5-build](skills/stm32c5-build/SKILL.md) | 编译、链接、ELF/map/占用 | “修复 HSE API 编译错误并检查内存占用。” |
| [stm32c5-flash](skills/stm32c5-flash/SKILL.md) | 识别目标、下载、校验、启动 | “将这个 ELF 下载到指定 ST-Link 的 C5 板。” |
| [stm32c5-debug](skills/stm32c5-debug/SKILL.md) | 启动、Fault、外设、RTOS 诊断 | “DMA 收到第一帧后停止，帮我定位。” |

## 三条工作流

1. **CubeMX2 新建与维护**：`env → project → configure → firmware → build → flash/debug`。从[完整串口起步实例](skills/stm32c5-project/references/create.md)开始。
2. **官方 SDK / Example Library / Pack 工程**：识别版本和目录，先构建原例程，再调整板级配置与应用；见[例程入口](skills/stm32c5-project/references/existing-and-examples.md)。
3. **无生成器或已有手写工程**：保留现有构建与初始化，按 C5 HAL2/LL 接口维护；不要求强制转换成 `.ioc2`。

常用配置实例覆盖 GPIO/EXTI、时钟、UART、I2C、SPI、TIM/PWM、ADC、中断/DMA 和 FreeRTOS。选择当前任务对应参考，不必把全部资料加载进上下文。

## 适用条件

- 命令和参数实例基于 CubeMX2 CLI 1.1.2、C5 DFP/HAL2 2.1.0；代表型号为 `STM32C532CCT6`。FreeRTOS 示例使用 Pack 2.2.0。实际项目约束优先，升级后重新查询参数和头文件。
- C5 使用 CubeMX2；传统 CubeMX 的 `.ioc` 与 Java 脚本不用于此路线。`.ioc2` 不是纯 JSON。
- 重新生成默认导出到新的空目录，再比较和合并；不使用 `--force` 覆盖业务代码。
- 引脚、时钟、内存、DMA 和 RTOS 配置必须与实际型号及板卡相符；示例不能替代原理图和数据手册。
- 三平台都有操作说明，实际硬件功能仍需在对应板卡验证。构建、下载校验与功能测试是三个不同完成状态。

## 许可

本包使用 [MIT](LICENSE)，版权主体为 `STM32C5DevSkills contributors`。ST、Arm、FreeRTOS 和其他工具/SDK 的名称及许可归各自权利人；本包不重新分发它们的二进制或源代码包。
