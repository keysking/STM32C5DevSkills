# FreeRTOS 生成器配置

按所需操作选择章节，不将“启用 RTOS、换时间基准、改堆、加任务”视为一套必做步骤。路径依据 C532CCT6、CubeMX2 1.1.2、FreeRTOS Pack 2.2.0；该 Pack 使用原生 API，已有 CMSIS-RTOS 工程保留其适配层。先设置[工程变量](operations.md)。

## 查询当前中间件与参数

```powershell
cube mx middleware get-available-middleware -p "$p" -s "$s"
cube mx sw-config list-configurations -p "$p" -s "$s"
cube mx sw-config list-parameters -p "$p" -s "$s" -r FreeRTOS
cube mx sw-config get-parameter-value --help
cube mx sw-config add-array-element --help
```

先看资源是否已存在；未启用时不能依赖 FreeRTOS 资源查询成功。本文未列出的选项从实际参数列表及可选值查询，不能凭旧版本路径补写。

## 为裸机工程新增 FreeRTOS

仅在需求选择 FreeRTOS 且当前未启用时操作。先查项目约束及已安装 Pack；缺少所需版本才安装。若已确定需求版本为 2.2.0：

```powershell
cube pack install STMicroelectronics.freertos.2.2.0
```

确认当前模型提供 FreeRTOS 后启用：

```powershell
cube mx middleware enable-middleware -p "$p" -s "$s" --name FreeRTOS
cube mx sw-config list-parameters -p "$p" -s "$s" -r FreeRTOS
```

已有 RTOS 工程增加任务时跳过安装和启用，保持现有 API 层、内核及对象创建所有者。

## HAL tick 与 RTOS tick 共存

先检查实际 SysTick、HAL tick 和 port 调用链。只有新接入 RTOS 或现有时间基准冲突时才调整；已正确使用定时器的工程保持现状。此 Pack 的常用安排为 SysTick 供 RTOS、空闲定时器供 HAL。[ST 时间基准说明](https://dev.st.com/stm32cube-docs/mw-freertos/2.2.0/en/docs/markup/how_to_use.html)

假定已确认 TIM6 可用、未被其他功能占用，并选择它作为 HAL tick：

```powershell
# TIM6 尚未启用时才执行 enable
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral TIM6
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r HALCore --parameter-path common/timebase_feature/hal_timebase_selection --value TIM
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r HALCore --parameter-path common/timebase_feature/timer_selection
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r HALCore --parameter-path common/timebase_feature/timer_selection --value TIM6
```

先启用候选定时器，它才会出现在选择列表。其他定时器以当前设备支持和占用为准，不能因 TIM6 被占用就抢占。检查生成 IRQ、HAL tick 更新及定时器所有权。

## 按需求调整内核参数

增加一个任务不要求同时改变 tick、堆或 FPU。先回读相关项，只有需求或测量结果要求改变时才设置。

| 路径（资源 `FreeRTOS`） | 决策依据 |
| --- | --- |
| `core/basic/system/rtos_tick_frequency` | 调度精度、功耗和已有超时换算；1000 Hz 不是固定要求。 |
| `core/basic/memory_management/dynamic_memory_allocation` | 使用的对象创建 API 和 HAL OS port 是否需要动态内存。 |
| `core/basic/memory_management/total_heap_size` | 任务、队列、内核对象及实测余量；不统一设为 8 KiB。 |
| `core/basic/kernel/fpu_mode` | port 配置、编译器 FPU ABI 和任务浮点使用；不能因芯片有 FPU 就无条件开启。 |

以查询堆容量为例：

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/basic/memory_management/total_heap_size --use-internal-value
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r FreeRTOS --parameter-path core/basic/memory_management/total_heap_size
```

需要修改时按[参数设置语法](operations.md)传入预算所得值，再回读。生成器默认可能关闭 FPU 支持，不能据芯片能力推断上下文已保存。C532 使用 `ARM_CM33_NTZ/non_secure` port、`configENABLE_TRUSTZONE=0`；不能替换为 CM4 port 或仅据 `non_secure` 目录名要求安全镜像。

## 新增或修改生成器管理的任务

先确认现有对象由生成器管理，再读取任务数组：

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks
```

修改已有任务时定位实际元素索引，直接修改目标字段。**仅需新增且尚无该任务**时添加一个元素，再回读确定新索引：

```powershell
cube mx sw-config add-array-element -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks -e '{}'
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks
```

默认字段包括 `task_name`、`stack_size`、`task_priority`、`entry_function`、`parameter`、`memory_allocation`。假定查询确认目标索引为 0，且实际入口需要命名为 `worker_entry`，单字段写法为：

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks/0/entry_function --value worker_entry
```

其他字段同样按需求逐项设定、回读；不能把索引 0、函数名、优先级或栈大小原样搬到已有任务。原生 `xTaskCreate` 栈单位为 `StackType_t` 个数，不是字节。静态分配还要检查内存回调和缓冲区。

任务循环须有适当阻塞等待或延时，避免高优先级空转；保留代码按[重新生成流程](../../stm32c5-project/references/regenerate.md)处理。

## 应用管理的任务与调度器接入

若现有项目由应用创建对象，继续在应用源文件维护；不为新增任务同时添加生成器任务。新项目可按维护方式选择所有者，不重复调用两套创建函数。

仅在尚未接入启动链时增加对象创建与调度器启动，见[RTOS API 参考](../../stm32c5-firmware/references/freertos.md)。启用中间件不保证调度器已启动，模板 `main.c` 可能仍只有初始化和 `while (1)`。已有调度器不得重复启动。

## 变更后的检查

保存重载本次变更；需更新代码时暂存生成、比较并构建：

- `FreeRTOSConfig.h` 与需求一致，未改变无关 tick、堆和 FPU 设置。
- heap 实现只编译一份，port 匹配编译器和内核。
- 对象创建返回值受检查；确认生成函数的实际名称，可能为 `app_synctasks_init()`。
- 启动链只有一个调度器入口，返回时进入失败处理。
- SysTick/SVC/PendSV 绑定正确，HAL tick IRQ 唯一；已分配 HAL tick 的定时器不被应用重初始化。

链接结果不能证明任务运行；运行期栈、堆与调度检查见[固件 RTOS 资料](../../stm32c5-firmware/references/freertos.md)。
