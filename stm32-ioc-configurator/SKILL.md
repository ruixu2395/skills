---
name: stm32-ioc-configurator
description: 配置和修改 STM32CubeMX .ioc 工程文件，支持 STM32 外设、GPIO 引脚、复用功能、NVIC 中断、DMA、时钟、中间件以及生成文件元数据。用于 Codex 需要直接编辑 .ioc 文件并保持 STM32CubeMX 兼容性、推断启用外设和引脚所需键值记录，或验证 .ioc 修改后是否仍可被 STM32CubeMX 打开和重新生成代码的场景。
---

# STM32 IOC 配置器

## 工作流程

在手动或自动修改 STM32CubeMX `.ioc` 文件时使用本 skill。

1. 修改前先读取完整 `.ioc` 文件。如果用户希望生成的 C 文件或构建元数据也与 `.ioc` 匹配，同时检查 `Core/Src/main.c`、`Core/Src/*_hal_msp.c`、`Core/Src/*_it.c`、`Core/Inc/main.h`、`Core/Inc/*_hal_conf.h`、`CMakeLists.txt` 和 `.mxproject`。
2. 识别 MCU 信息行：`Mcu.CPN`、`Mcu.Name`、`Mcu.Package`、`PCC.PartNumber` 和 `Mcu.Family`。不要凭记忆假设某个引脚、复用功能、IRQ 名或外设实例有效，必须结合具体 STM32 系列/封装或现有生成头文件确认。
3. 把 `.ioc` 当作键值数据库修改，保留原有转义、CubeMX 风格名称和文件排序习惯。改动应保持最小且可预测。
4. 成组更新相关记录：`Mcu.IP*`、`Mcu.Pin*`、外设 `IPParameters`、引脚 `Signal`/`Mode`、`NVIC.*`、`ProjectManager.functionlistsort`、GPIO 标签/设置、DMA 记录、中间件记录和虚拟引脚。
5. 修改 `.ioc` 成功后，必须使用 STM32CubeMX 命令行脚本重新加载这个 `.ioc` 并生成代码。不能只做文本搜索后就声称完成；如果 CubeMX 不可用、命令失败或生成超时，必须停止并向用户报告命令、退出码和关键日志，不要声称 CubeMX 已验收。
6. CubeMX 生成完成后，必须校验配置对应的生成源文件中确实出现了相关内容，例如初始化函数、句柄、GPIO 复用、DMA、NVIC IRQ、HAL 模块开关和构建/工程元数据。只有 `.ioc`、CubeMX 生成日志和源文件内容三者一致时，才能说明修改已完成。

做非简单修改前，先阅读 `references/ioc-format.md`，其中包含更细的键值模式和示例。

## IOC 格式规则

- 将 `.ioc` 视为逐行 `key=value` 记录。尽量保留注释、转义、排序风格以及 CRLF/LF 换行习惯。
- `=` 两侧不要加空格。除非该字段在原文件中已经使用引号，否则不要给值加引号。
- 在 CubeMX 复合列表字段中，字面冒号要写成 `\:`，尤其是 `NVIC.*` 和 `PCC.Display`。
- 保持索引计数一致：
  - `Mcu.IPNb` 必须等于 `Mcu.IP0..N` 记录数量。
  - `Mcu.PinsNb` 必须等于 `Mcu.Pin0..N` 记录数量。
  - `*.IPParameters` 必须列出该 IP 块中显式使用的每个参数键。
  - `SH.<signal>.ConfNb` 必须匹配 `SH.<signal>.<index>` 编号记录数量。
- 优先在现有索引之后追加新的 `Mcu.IP*` 和 `Mcu.Pin*`。只有必要时才重排索引；一旦重排，必须同步更新所有索引和计数。
- 使用 CubeMX 引脚名，例如 `PC10`，不要使用板级丝印或其他生态的别名，例如 `PTC10`。只有确认用户的 `PTA`、`PTB`、`PTC` 指 STM32 端口时，才规范化为 `PA`、`PB`、`PC`。
- 已配置引脚通常写入 `<Pin>.Locked=true`，除非当前工程风格有意不锁定引脚。
- 系统功能和中间件信号使用虚拟引脚，例如 `VP_SYS_VS_Systick.Mode=SysTick`。

## 外设配置清单

启用某个外设实例时，检查并更新所有适用记录：

- 将 IP 添加到 `Mcu.IP*`，并递增 `Mcu.IPNb`。
- 将引脚添加到 `Mcu.Pin*`，并递增 `Mcu.PinsNb`。
- 设置每个引脚的信号和必要模式，例如 `PC10.Signal=UART4_TX`、`PC10.Mode=Asynchronous`。
- 添加外设模式和参数，例如 `UART4.IPParameters=VirtualMode`、`UART4.VirtualMode=VM_ASYNC`。
- 只有用户要求或外设确实需要中断时才启用 NVIC，并使用该 MCU 的准确 IRQ 键，例如 `NVIC.UART4_IRQn=...`。
- 当需要 CubeMX 生成初始化代码时，更新初始化函数顺序，例如 `ProjectManager.functionlistsort=...,2-MX_UART4_Init-UART4-false-HAL-true`。
- 如果用户要求同步生成代码文件，不要假设 CubeMX 已经重新生成；要同时更新 `Core` 文件和构建元数据，或明确说明还需要重新生成。

## 校验要求

修改后执行以下检查：

- 搜索每个请求的外设、引脚、IRQ、模式和初始化函数。
- 对照实际索引记录检查 `Mcu.IPNb` 和 `Mcu.PinsNb`。
- 检查每个 `*.IPParameters` 列表是否包含新增或删除的参数键。
- 确认没有把同一引脚分配给互斥信号。
- 对不确定的 IRQ 名和复用功能名，使用本地 CMSIS/HAL 头文件确认。
- 必须运行 STM32CubeMX 生成器验证本次 `.ioc` 修改。推荐脚本流程如下，其中 `<ioc>` 是 `.ioc` 绝对路径，`<project-root>` 是工程根目录：

  ```text
  config load "<ioc>"
  project generate "<project-root>"
  exit
  ```

  Windows 示例：

  ```powershell
  & "<STM32CubeMX>\jre\bin\java.exe" -jar "<STM32CubeMX>\STM32CubeMX.exe" -q "<script-file>"
  ```

  生成日志必须至少确认 `config load` 后有 `OK`、`project generate` 后有 `OK`，并最终出现正常退出信息。若输出包含 `ERROR`、超时、未执行到 `project generate`，或没有生成文件记录，必须按失败处理并报告原因。
- 生成完成后，根据本次变更校验 `Core/Src/main.c`、`Core/Src/*_hal_msp.c`、`Core/Src/*_it.c`、`Core/Inc/*_it.h`、`Core/Inc/main.h`、`Core/Inc/*_hal_conf.h`、`CMakeLists.txt`、`.mxproject` 等相关文件。典型检查包括：
  - 外设初始化函数和调用，例如 `MX_UART4_Init()`。
  - HAL 句柄和配置结构，例如 `UART_HandleTypeDef huart4`。
  - GPIO 复用、时钟使能、DMA stream/channel 和 `__HAL_LINKDMA`。
  - IRQ handler、`HAL_NVIC_SetPriority`、`HAL_NVIC_EnableIRQ` 和 HAL 模块宏。
  - 新增/删除源文件是否反映在构建元数据中。

除非 `.ioc` 已实际通过 STM32CubeMX 打开或重新生成成功，否则不要声称 CubeMX 已验收。仅做静态检查时，应说明“已按 CubeMX 风格做文本级校验”。
