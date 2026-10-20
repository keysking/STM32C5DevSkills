# CubeMX2 配置 FreeRTOS

示例 C532CCT6、CubeMX2 1.1.2、FreeRTOS Pack 2.2.0。该 Pack 使用原生 FreeRTOS API；已有 CMSIS-RTOS 工程不因此移除其适配层。先设置[工程变量](operations.md)。

## 启用与 HAL 时间基准

```powershell
cube pack install STMicroelectronics.freertos.2.2.0
cube mx middleware get-available-middleware -p "$p" -s "$s"
cube mx middleware enable-middleware -p "$p" -s "$s" --name FreeRTOS
cube mx sw-config list-configurations -p "$p" -s "$s"
cube mx sw-config list-parameters -p "$p" -s "$s" -r FreeRTOS
cube mx peripherals enable -p "$p" --software-project "$s" --peripheral TIM6
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r HALCore --parameter-path common/timebase_feature/hal_timebase_selection --value TIM
cube mx sw-config get-parameter-possible-values -p "$p" -s "$s" -r HALCore --parameter-path common/timebase_feature/timer_selection
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r HALCore --parameter-path common/timebase_feature/timer_selection --value TIM6
```

先启用 TIM6，选择列表里才有 TIM6。已有工程若 TIM6 被其他用途占用，选择设备支持的其他空闲定时器并核对生成结果。SysTick 留给 RTOS；HAL tick 使用选定的定时器。[ST 时间基准说明](https://dev.st.com/stm32cube-docs/mw-freertos/2.2.0/en/docs/markup/how_to_use.html)

## 明确内核配置

```powershell
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/basic/system/rtos_tick_frequency --value 1000
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/basic/memory_management/dynamic_memory_allocation --value true
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/basic/memory_management/total_heap_size --value 8192
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/basic/kernel/fpu_mode --value true
```

1000 Hz、8 KiB 堆只是小型实例起点，不是所有项目默认容量。检查 `configENABLE_FPU` 与实际编译器 FPU ABI、任务浮点使用一致；生成器默认关闭 FPU 支持，不能因芯片有 FPU 就认为内核已保存其上下文。

C532 示例使用 `ARM_CM33_NTZ/non_secure` port，`configENABLE_TRUSTZONE=0`。不要换成 CM4 port 或凭目录 `non_secure` 推断需要另一份安全镜像。

## 任务由生成器管理时

先回读数组，避免重复添加：

```powershell
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks
```

新工程为空时：

```powershell
cube mx sw-config add-array-element -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks -e '{}'
cube mx sw-config get-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks/0/task_name --value Worker
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks/0/stack_size --value 256
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks/0/task_priority --value 1
cube mx sw-config set-parameter-value -p "$p" -s "$s" -r FreeRTOS --parameter-path core/applicative/tasks/0/entry_function --value worker_entry
```

默认元素包含 `task_name`、`stack_size`、`task_priority`、`entry_function`、`parameter`、`memory_allocation`。这里原生 `xTaskCreate` 的 stack size 单位为 `StackType_t`，不是字节。启用静态分配时同时检查生成的内存回调和缓冲区。

生成器创建的空任务循环需要加入阻塞等待或延时，避免高优先级空转。任务函数和对象初始化要遵循[重新生成保留流程](../../stm32c5-project/references/regenerate.md)。

## 任务由应用维护时

也可让 `core/applicative/tasks` 保持空数组，在自有 `app.c` 中创建任务、队列和同步对象。这能减少业务代码与生成文件的混合。不要同时调用生成对象创建函数和手写版本而重复创建同一任务。

按[原生 RTOS 应用](../../stm32c5-firmware/references/freertos.md)接入应用启动。**启用中间件、生成文件和构建通过，不代表调度器已启动。** 此导出模板的 `main.c` 仍可能只有初始化和 `while (1)`，需要检查实际调用链。

## 生成后检查

保存重载，暂存生成并合并。检查：

- `FreeRTOSConfig.h` 的 tick、堆、优先级位数、FPU 和分配配置。
- `heap_4.c` 等 heap 实现只编译一份；port 与编译器/内核匹配。
- `mx_freertos_app.c` 的对象创建返回值；示例函数为 `app_synctasks_init()`。
- `main` 调用对象创建和 `vTaskStartScheduler()`；调度器返回进入故障处理。
- SysTick/SVC/PendSV 绑定到 port；HAL 定时器 IRQ 与时间基准只各有一份。
- TIM6 被时间基准使用时，不再作为普通应用计时器初始化。

最后检查链接与运行期栈/堆。ISR 调用规则及同步 API 见[固件 RTOS 资料](../../stm32c5-firmware/references/freertos.md)。
