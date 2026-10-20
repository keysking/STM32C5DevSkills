# 编译与链接错误定位

保留失败命令、第一条有效错误及其上下文。日志很长时先检索 `error:`、`undefined reference`、`region ... overflowed`，再回到首次报错。不能仅看最后一行 `ninja stopped`。

| 现象 | 核查顺序 |
| --- | --- |
| 找不到 arm-none-eabi-gcc | Bundle 实际版本与 bin → 当前 PATH → CMake 工具链/缓存 |
| 找不到 C5/HAL 头文件 | Pack 真正安装 → include 目录 → 导出依赖策略 → 文件大小写 |
| HAL1 handle/函数不存在 | 确认 HAL2 头文件签名，按接口迁移，不添加假声明 |
| DMA API 不可见 | `USE_HAL_*_DMA` → DMA 源文件/绑定 → 正确 HAL2 名称 |
| 回调注册不存在 | 对应 `USE_HAL_*_REGISTER_CALLBACKS` 与版本条件 |
| HSE API 或宏缺失 | `USE_EXTERNAL_ENV` → `stm32_external_env.h` → `HSE_VALUE` |
| undefined reference | 源文件是否进 target → 符号拼写/条件宏 → C/C++ linkage → 静态库顺序 |
| 多重 IRQ/tick 定义 | 生成器、用户代码、HAL timebase、RTOS port 各自定义位置 |
| 浮点 ABI 冲突 | 所有对象/库的 `mcpu/mfpu/mfloat-abi` 一致，含 RTOS port |
| Flash/RAM 溢出 | 链接区域是否正确 → map 最大对象 → 堆栈/缓冲/库占用 |

定位命令示例：

```text
cube cmake --build --preset debug_GCC_STM32C532CCT6 --verbose
rg -n "USE_EXTERNAL_ENV|HSE_VALUE|stm32_external_env" .
rg -n "SysTick_Handler|PendSV_Handler|SVC_Handler|HAL_InitTick" .
rg -n "add_executable|target_sources|target_include_directories|target_compile_definitions" CMakeLists.txt cmake generated
```

若要用 `compile_commands.json`，在该工程支持的配置方式中打开 `CMAKE_EXPORT_COMPILE_COMMANDS=ON`，检查实际源文件命令；不要只凭 IDE 索引报错判断编译失败。

HSE 缺失尤其要恢复设备环境包含链，见[时钟资料](../../stm32c5-configure/references/clocks.md)。手动加一个随机频率宏可能让工程编译通过但导致真实时钟错误。

每次修复只针对已找到的原因，重新构建受影响目标。工具链/关键宏变化后用新的构建目录验证，防止旧对象混用。成功时确认产物更新而非复用旧 ELF。
