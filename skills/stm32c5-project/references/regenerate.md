# 修改配置后保留应用代码

适用于 CubeMX2 CMake 导出工程，尤其含手写 `main.c`、自定义源文件或修改过构建文件的工程。

## 可靠默认流程

1. 读取 `git status --short`；保存已有修改或制作本次可恢复副本。区分用户原有工作与本次改动。
2. 修改 `.ioc2` 后查询回读、保存、关闭并重载。确认磁盘状态正确，再生成。
3. 选择一个从未使用的兄弟暂存目录，例如 `export-stage-01`。若已存在，换新名称；不清空不明目录。
4. 使用与原工程相同的 `--build-target`、`--format CMake` 和依赖导出策略生成到该目录，**不使用 `--force` 覆盖正在开发的目录**。
5. 比较整个工程树，排除构建产物。包括 `main.c`、`generated/`、`user_modifiable/`、设备配置、依赖源文件、`CMakeLists.txt`、预设和链接脚本。
6. 查看 `.settings/exported-files.json` 等导出元数据的文件分组；它只是帮助判断文件归属，不能代替实际 diff。
7. 合并本次配置导致的变化，保留应用实现和用户修改。新增/删除的组件必须同步源文件、include、宏和链接项。
8. 在实际开发目录构建；检查修改的生成代码对应目标参数，再检查应用代码仍存在。

可用原生 Git 比较两个目录：

```text
git diff --no-index --stat ./export-current ./export-stage-01
git diff --no-index -- ./export-current/generated ./export-stage-01/generated
```

`git diff --no-index` 的退出码 1 表示存在差异，不是比较失败；路径不存在等错误才需修复。在大型带 build 的目录中先用编辑器目录比较或有界文件清单，避免把二进制输出送入模型上下文。

## 为何不能依赖强制覆盖

CubeMX2 1.1.2 中，`ide-project generate --force` 会覆盖导出目标文件；即使同时传入 `--diff keep_user`，也不能据此保证修改过的 `main.c` 得到保留。`USER CODE` 风格注释本身不提供保留保证。

因此默认采用暂存导出。只有实际版本的机制和具体文件归属已确认，才使用相应的原位更新方式。不得为了自动化方便无条件删除导出目录。

## 小改动示例

USART1 波特率从 57600 改为 115200：

1. 在 `.ioc2` 中设置并回读 `USART1/basic/baud_rate`。
2. 导出暂存目录，检查 `mx_usart1.c` 的 `hal_uart_config_t`。
3. 若差异仅为串口初始化与初始模板 `main.c`，合并串口初始化，保留已有应用 `main.c`。
4. 构建后用相同波特率观察串口输出。

如果新增 DMA、RTOS 或驱动，则差异可能跨越配置头、IRQ、依赖和构建文件，不能只复制 `generated/`。
