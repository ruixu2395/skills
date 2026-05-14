# STM32CubeMX IOC 格式参考

本文件记录 `.ioc` 的常用键值模式，用于让 AI 在修改 STM32CubeMX 工程时快速判断要改哪些记录。`.ioc` 不是稳定公开 API，字段会随 MCU 系列、CubeMX 版本和中间件版本变化；优先遵循当前工程已有风格，再参考本文件。

## 基本规则

`.ioc` 是逐行键值数据库：

```text
Key=Value
Section.Subsection.Field=value
Indexed.Record0=value
Indexed.Record1=value
Indexed.RecordNb=2
```

规则：

- 键名大小写敏感，`=` 两侧不要加空格。
- 保留已有注释、排序、换行和反斜杠转义。
- `NVIC.*`、`PCC.Display`、ADC channel key 等复合字段里的冒号或 `#` 要保持 CubeMX 转义风格，例如 `true\:0\:0...`、`Rank-0\#ChannelRegularConversion`。
- 修改前先确认 `Mcu.CPN`、`Mcu.Family`、`Mcu.Package`、`MxCube.Version`、`MxDb.Version`。
- 不确定外设实例、IRQ、DMA stream/channel、引脚复用时，必须查本地 CMSIS/HAL 头文件、CubeMX 生成结果或同版本 `.ioc` 样例。

## IP、引脚和函数顺序

启用外设通常至少涉及三组记录：

```text
Mcu.IP0=NVIC
Mcu.IP1=RCC
Mcu.IP2=SYS
Mcu.IP3=UART4
Mcu.IPNb=4

Mcu.Pin0=PC10
Mcu.Pin1=PC11
Mcu.Pin2=VP_SYS_VS_Systick
Mcu.PinsNb=3

ProjectManager.functionlistsort=1-SystemClock_Config-RCC-false-HAL-false,2-MX_UART4_Init-UART4-false-HAL-true
```

检查：

- `Mcu.IPNb` 等于 `Mcu.IP[0-9]+=` 数量。
- `Mcu.PinsNb` 等于 `Mcu.Pin[0-9]+=` 数量。
- 需要生成初始化函数时加入 `MX_<IP>_Init`，DMA 一般放在使用 DMA 的外设前。
- 系统、RTOS、中间件常使用虚拟引脚 `VP_*`，并计入 `Mcu.Pin*`。

## GPIO 和 EXTI

普通 GPIO：

```text
PA5.GPIOParameters=GPIO_Label,GPIO_ModeDefaultOutputPP,PinState,GPIO_Speed
PA5.GPIO_Label=LED
PA5.GPIO_ModeDefaultOutputPP=GPIO_MODE_OUTPUT_PP
PA5.PinState=GPIO_PIN_SET
PA5.GPIO_Speed=GPIO_SPEED_FREQ_HIGH
PA5.Locked=true
PA5.Signal=GPIO_Output
```

输入和外部中断：

```text
PC13.Signal=GPXTI13
SH.GPXTI13.0=GPIO_EXTI13
SH.GPXTI13.ConfNb=1
NVIC.EXTI15_10_IRQn=true\:0\:0\:false\:false\:true\:true\:true\:true
```

注意：

- `GPXTI<n>` 需要匹配 `SH.GPXTI<n>.*`。
- EXTI IRQ 名按线号分组，常见为 `EXTI0_IRQn`、`EXTI1_IRQn`、`EXTI2_IRQn`、`EXTI3_IRQn`、`EXTI4_IRQn`、`EXTI9_5_IRQn`、`EXTI15_10_IRQn`；低端系列可能是 `EXTI4_15_IRQn`。

## NVIC

格式依 CubeMX 版本变化，但都使用：

```text
NVIC.<IRQn>=true\:0\:0\:false\:false\:true\:true\:true\:true
NVIC.PriorityGroup=NVIC_PRIORITYGROUP_4
NVIC.ForceEnableDMAVector=true
```

规则：

- 第一个字段通常表示启用状态；后续字段含抢占优先级、子优先级、生成选项等。
- 新增 IRQ 时，复制同一文件中相近外设的字段长度和布尔位风格。
- IRQ 键必须来自目标 MCU 头文件，例如 `UART4_IRQn`、`DMA1_Stream2_IRQn`、`DMA1_Channel1_IRQn`、`OTG_FS_IRQn`、`SDIO_IRQn`、`FDCAN1_IT0_IRQn`。

## DMA 和 DMAMUX

CubeMX 使用 `Dma.Request*` 总表加每个请求的详细参数：

```text
Mcu.IP4=DMA
Dma.Request0=UART4_RX
Dma.Request1=UART4_TX
Dma.RequestsNb=2
Dma.UART4_RX.0.Instance=DMA1_Stream2
Dma.UART4_RX.0.Direction=DMA_PERIPH_TO_MEMORY
Dma.UART4_RX.0.PeriphInc=DMA_PINC_DISABLE
Dma.UART4_RX.0.MemInc=DMA_MINC_ENABLE
Dma.UART4_RX.0.PeriphDataAlignment=DMA_PDATAALIGN_BYTE
Dma.UART4_RX.0.MemDataAlignment=DMA_MDATAALIGN_BYTE
Dma.UART4_RX.0.Mode=DMA_NORMAL
Dma.UART4_RX.0.Priority=DMA_PRIORITY_LOW
Dma.UART4_RX.0.FIFOMode=DMA_FIFOMODE_DISABLE
Dma.UART4_RX.0.RequestParameters=Instance,Direction,PeriphInc,MemInc,PeriphDataAlignment,MemDataAlignment,Mode,Priority,FIFOMode
NVIC.DMA1_Stream2_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:true
```

F4/F7 stream DMA 常见还需要 `Channel`：

```text
Dma.SPI1_RX.0.Channel=DMA_CHANNEL_3
Dma.SPI1_RX.0.RequestParameters=Instance,Channel,Direction,PeriphInc,MemInc,PeriphDataAlignment,MemDataAlignment,Mode,Priority,FIFOMode
```

带 DMAMUX 的 H7/G4/U5 等系列常见附加字段：

```text
Dma.SPI1_RX.1.EventEnable=DISABLE
Dma.SPI1_RX.1.SignalID=NONE
Dma.SPI1_RX.1.Polarity=HAL_DMAMUX_REQ_GEN_RISING
Dma.SPI1_RX.1.RequestNumber=1
Dma.SPI1_RX.1.SyncSignalID=NONE
Dma.SPI1_RX.1.SyncPolarity=HAL_DMAMUX_SYNC_NO_EVENT
Dma.SPI1_RX.1.SyncEnable=DISABLE
Dma.SPI1_RX.1.SyncRequestNumber=1
```

方向：

- RX、ADC、SPI RX：`DMA_PERIPH_TO_MEMORY`
- TX、DAC、SPI TX、UART TX：`DMA_MEMORY_TO_PERIPH`
- ADC 连续采样常用 `DMA_CIRCULAR`
- UART/SPI 单次收发常用 `DMA_NORMAL`

必须查目标 MCU 的 DMA 映射表；不要跨系列复用 stream/channel。

## UART、USART、LPUART

引脚：

```text
PC10.Mode=Asynchronous
PC10.Signal=UART4_TX
PC11.Mode=Asynchronous
PC11.Signal=UART4_RX
```

外设参数有两种常见写法：

```text
UART4.IPParameters=VirtualMode
UART4.VirtualMode=VM_ASYNC
```

```text
USART3.IPParameters=VirtualMode-Asynchronous,Mode
USART3.VirtualMode-Asynchronous=VM_ASYNC
USART3.Mode=MODE_TX_RX
```

显式串口参数可追加：

```text
USART1.IPParameters=VirtualMode,BaudRate,WordLength,Parity,StopBits,OverSampling
USART1.BaudRate=115200
USART1.WordLength=WORDLENGTH_8B
USART1.Parity=PARITY_NONE
USART1.StopBits=STOPBITS_1
USART1.OverSampling=UART_OVERSAMPLING_16
```

中断和 DMA：

- 串口中断：`NVIC.USART1_IRQn=...`、`NVIC.UART4_IRQn=...`
- DMA 请求：`Dma.USART1_RX.*`、`Dma.USART1_TX.*`、`Dma.UART4_RX.*`、`Dma.UART4_TX.*`

## I2C

新系列常用 `Timing`：

```text
I2C4.IPParameters=Timing
I2C4.Timing=0x00707CBB
PF5.Mode=I2C
PF5.Signal=I2C4_SCL
PG6.Mode=I2C
PG6.Signal=I2C4_SDA
```

旧系列可能使用速度模式：

```text
I2C1.I2C_Mode=I2C_Standard
I2C1.IPParameters=I2C_Mode
```

或：

```text
I2C1.I2C_Speed_Mode=I2C_Fast
I2C1.IPParameters=I2C_Speed_Mode,Timing
I2C1.Timing=0x00000000
```

I2C 中断通常分事件/错误：`I2C1_EV_IRQn`、`I2C1_ER_IRQn`。DMA 请求通常为 `I2C1_RX`、`I2C1_TX`。

## SPI 和 I2S

SPI 主机常见：

```text
SPI3.IPParameters=VirtualType,Mode,Direction,CalculateBaudRate,BaudRatePrescaler
SPI3.VirtualType=VM_MASTER
SPI3.Mode=SPI_MODE_MASTER
SPI3.Direction=SPI_DIRECTION_2LINES
SPI3.BaudRatePrescaler=SPI_BAUDRATEPRESCALER_32
SPI3.CalculateBaudRate=1.3125 MBits/s
```

更多参数：

```text
SPI1.IPParameters=VirtualType,Mode,Direction,DataSize,FirstBit,CLKPhase,CLKPolarity,BaudRatePrescaler,CalculateBaudRate
SPI1.DataSize=SPI_DATASIZE_16BIT
SPI1.FirstBit=SPI_FIRSTBIT_LSB
SPI1.CLKPhase=SPI_PHASE_2EDGE
SPI1.CLKPolarity=SPI_POLARITY_LOW
SPI1.Direction=SPI_DIRECTION_2LINES_RXONLY
```

引脚信号：

```text
PA5.Signal=SPI1_SCK
PA6.Signal=SPI1_MISO
PA7.Signal=SPI1_MOSI
PA4.Signal=SPI1_NSS
```

DMA 请求：

```text
Dma.Request0=SPI1_RX
Dma.Request1=SPI1_TX
Dma.SPI1_RX.0.Direction=DMA_PERIPH_TO_MEMORY
Dma.SPI1_TX.1.Direction=DMA_MEMORY_TO_PERIPH
```

I2S 常作为 SPI 扩展出现，键名可能仍以 `SPIx.` 或 `I2Sx.` 开头；优先参考同系列 CubeMX 生成样例。

## TIM、PWM、输入捕获、编码器

内部时钟虚拟引脚：

```text
VP_TIM2_VS_ClockSourceINT.Mode=Internal
VP_TIM2_VS_ClockSourceINT.Signal=TIM2_VS_ClockSourceINT
```

PWM：

```text
TIM3.Channel-PWM\ Generation1\ CH1=TIM_CHANNEL_1
TIM3.Channel-PWM\ Generation2\ CH2=TIM_CHANNEL_2
TIM3.IPParameters=Channel-PWM Generation1 CH1,Channel-PWM Generation2 CH2,Prescaler,Period,Pulse-PWM Generation1 CH1,Pulse-PWM Generation2 CH2
TIM3.Prescaler=72
TIM3.Period=1000
TIM3.Pulse-PWM\ Generation1\ CH1=700
TIM3.Pulse-PWM\ Generation2\ CH2=700
PA6.Signal=S_TIM3_CH1
SH.S_TIM3_CH1.0=TIM3_CH1,PWM Generation1 CH1
SH.S_TIM3_CH1.ConfNb=1
```

输出比较：

```text
TIM1.Channel-Output\ Compare1\ No\ Output=TIM_CHANNEL_1
TIM1.IPParameters=Channel-Output Compare1 No Output,Prescaler,Period
VP_TIM1_VS_no_output1.Mode=Output Compare1 No Output
VP_TIM1_VS_no_output1.Signal=TIM1_VS_no_output1
```

编码器：

```text
TIM4.EncoderMode=TIM_ENCODERMODE_TI12
TIM4.IPParameters=Period,EncoderMode
SH.S_TIM4_CH1.0=TIM4_CH1,Encoder_Interface
SH.S_TIM4_CH2.0=TIM4_CH2,Encoder_Interface
```

定时器中断：`NVIC.TIMx_IRQn`、高级定时器常拆成 `TIM1_UP_IRQn`、`TIM1_CC_IRQn` 等。

## ADC

规则通式：

```text
ADC1.IPParameters=Rank-0\#ChannelRegularConversion,Channel-0\#ChannelRegularConversion,SamplingTime-0\#ChannelRegularConversion,NbrOfConversionFlag,master
ADC1.Channel-0\#ChannelRegularConversion=ADC_CHANNEL_2
ADC1.Rank-0\#ChannelRegularConversion=1
ADC1.SamplingTime-0\#ChannelRegularConversion=ADC_SAMPLETIME_2CYCLES_5
ADC1.NbrOfConversionFlag=1
ADC1.master=1
PA0.Signal=ADCx_IN0
```

多通道规则转换：

```text
ADC1.NbrOfConversion=6
ADC1.Rank-1\#ChannelRegularConversion=2
ADC1.Channel-1\#ChannelRegularConversion=ADC_CHANNEL_3
ADC1.SamplingTime-1\#ChannelRegularConversion=ADC_SAMPLETIME_239CYCLES_5
```

带触发、连续转换、DMA：

```text
ADC1.EnableRegularConversion=ENABLE
ADC1.ExternalTrigConv=ADC_EXTERNALTRIG_HRTIM_TRG8
ADC1.ContinuousConvMode=ENABLE
ADC1.DMAContinuousRequests=DISABLE
ADC1.DMAAccessModeView=ENABLE
Dma.Request0=ADC1
Dma.ADC1.0.Direction=DMA_PERIPH_TO_MEMORY
Dma.ADC1.0.Mode=DMA_CIRCULAR
Dma.ADC1.0.PeriphDataAlignment=DMA_PDATAALIGN_HALFWORD
Dma.ADC1.0.MemDataAlignment=DMA_MDATAALIGN_WORD
```

内部通道使用虚拟引脚：

```text
VP_ADC1_TempSens_Input.Mode=IN-TempSens
VP_ADC1_TempSens_Input.Signal=ADC1_TempSens_Input
VP_ADC1_Vref_Input.Mode=IN-Vrefint
VP_ADC1_Vref_Input.Signal=ADC1_Vref_Input
```

## DAC

DAC 通常有物理输出、触发源和可选 DMA：

```text
PA4.Signal=COMP_DAC12_group
SH.COMP_DAC12_group.0=DAC1_OUT1,DAC_OUT1
SH.COMP_DAC12_group.ConfNb=1
DAC1.IPParameters=Trigger
DAC1.Trigger=DAC_TRIGGER_T6_TRGO
Dma.Request0=DAC1_CH1
Dma.DAC1_CH1.0.Direction=DMA_MEMORY_TO_PERIPH
```

不同系列键名差异较大，尤其是 `DAC1_OUTx`、`DAC_OUTx`、trigger 名称和 DMA request 名称，必须用同系列样例或 CubeMX 确认。

## CAN 和 FDCAN

经典 bxCAN 常见信号：

```text
PA11.Signal=CAN1_RX
PA12.Signal=CAN1_TX
CAN1.IPParameters=Prescaler,Mode,SyncJumpWidth,TimeSeg1,TimeSeg2
CAN1.Mode=CAN_MODE_NORMAL
```

FDCAN 常见：

```text
FDCAN1.IPParameters=CalculateTimeQuantumNominal,CalculateTimeBitNominal,CalculateBaudRateNominal,Mode,AutoRetransmission,FrameFormat,NominalSyncJumpWidth,DataSyncJumpWidth,DataTimeSeg1,DataTimeSeg2,StdFiltersNbr,ExtFiltersNbr,NominalPrescaler,NominalTimeSeg1,NominalTimeSeg2,DataPrescaler
FDCAN1.Mode=FDCAN_MODE_NORMAL
FDCAN1.FrameFormat=FDCAN_FRAME_CLASSIC
FDCAN1.AutoRetransmission=ENABLE
FDCAN1.NominalPrescaler=8
FDCAN1.NominalSyncJumpWidth=1
FDCAN1.NominalTimeSeg1=6
FDCAN1.NominalTimeSeg2=5
FDCAN1.StdFiltersNbr=0
FDCAN1.ExtFiltersNbr=0
```

FDCAN IRQ 按系列可能是 `FDCAN1_IT0_IRQn`、`FDCAN1_IT1_IRQn`、`FDCAN_CAL_IRQn` 等。信号常为 `FDCAN1_RX`、`FDCAN1_TX`。

## USB OTG 和 USB_DEVICE

底层 USB 外设：

```text
USB_OTG_FS.IPParameters=VirtualMode
USB_OTG_FS.VirtualMode=Device_Only
PA11.Signal=USB_OTG_FS_DM
PA12.Signal=USB_OTG_FS_DP
PA9.Signal=USB_OTG_FS_VBUS
PA10.Signal=USB_OTG_FS_ID
NVIC.OTG_FS_IRQn=true\:0\:0\:false\:false\:true\:true\:true\:true
```

USB Device CDC：

```text
Mcu.IPx=USB_DEVICE
USB_DEVICE.CLASS_NAME_FS=CDC
USB_DEVICE.IPParameters=VirtualMode,VirtualModeFS,CLASS_NAME_FS
USB_DEVICE.VirtualMode=Cdc
USB_DEVICE.VirtualModeFS=Cdc_FS
VP_USB_DEVICE_VS_USB_DEVICE_CDC_FS.Mode=CDC_FS
VP_USB_DEVICE_VS_USB_DEVICE_CDC_FS.Signal=USB_DEVICE_VS_USB_DEVICE_CDC_FS
```

CubeMX 6.x 中也常见：

```text
USB_DEVICE.IPParameters=VirtualMode-CDC_FS,VirtualModeFS,CLASS_NAME_FS,PRODUCT_STRING_CDC_FS
USB_DEVICE.VirtualMode-CDC_FS=Cdc
USB_DEVICE.VirtualModeFS=Cdc_FS
USB_DEVICE.PRODUCT_STRING_CDC_FS=STM32 Virtual Port
```

USB MSC：

```text
USB_DEVICE.CLASS_NAME_FS=MSC
USB_DEVICE.VirtualMode=Msc
USB_DEVICE.VirtualModeFS=Msc_FS
VP_USB_DEVICE_VS_USB_DEVICE_MSC_FS.Mode=MSC_FS
VP_USB_DEVICE_VS_USB_DEVICE_MSC_FS.Signal=USB_DEVICE_VS_USB_DEVICE_MSC_FS
```

## SDIO、SDMMC 和 FATFS

F1/F4 SDIO：

```text
Mcu.IPx=SDIO
SDIO.ClockDiv=4
SDIO.IPParameters=ClockDiv
PC8.Signal=SDIO_D0
PC9.Signal=SDIO_D1
PC10.Signal=SDIO_D2
PC11.Signal=SDIO_D3
PC12.Signal=SDIO_CK
PD2.Signal=SDIO_CMD
NVIC.SDIO_IRQn=true\:0\:0\:false\:false\:true\:true\:true\:true
```

新系列 SDMMC：

```text
Mcu.IPx=SDMMC1
SDMMC1.ClockDiv=0x02
SDMMC1.IPParameters=ClockDiv
PC12.Signal=SDMMC1_CK
```

FATFS 绑定：

```text
Mcu.IPx=FATFS
VP_FATFS_VS_SDIO.Mode=SDIO
VP_FATFS_VS_SDIO.Signal=FATFS_VS_SDIO
```

自定义块设备：

```text
VP_FATFS_VS_Generic.Mode=User_defined
VP_FATFS_VS_Generic.Signal=FATFS_VS_Generic
```

## ETH 和 LwIP

ETH RMII：

```text
Mcu.IPx=ETH
ETH.IPParameters=MediaInterface,PHY_Name,PHY_Value,PhyAddress
ETH.MediaInterface=ETH_MEDIA_INTERFACE_RMII
ETH.PHY_Name=LAN8742A_PHY_ADDRESS
ETH.PHY_Value=0
ETH.PhyAddress=0
PA1.Signal=ETH_REF_CLK
PA2.Signal=ETH_MDIO
PC1.Signal=ETH_MDC
PC4.Signal=ETH_RXD0
PC5.Signal=ETH_RXD1
PA7.Signal=ETH_CRS_DV
PG11.Signal=ETH_TX_EN
PG13.Signal=ETH_TXD0
PB13.Signal=ETH_TXD1
```

H5 等系列可能用更明确的 RMII 信号名：

```text
ETH.MediaInterface=HAL_ETH_RMII_MODE
PA1.Signal=ETH_RMII_REF_CLK
PD1.Signal=ETH_RMII_CRS_DV
PG12.Signal=ETH_RMII_TXD1
```

LwIP 是中间件，通常有 `LWIP.*` 或 `LwIP.*` 参数、虚拟引脚和 `ProjectManager.functionlistsort` 条目。缺少同版本样例时不要手写复杂网络参数。

## RTC、CRC、RNG、看门狗

RTC：

```text
Mcu.IPx=RTC
VP_RTC_VS_RTC_Activate.Mode=RTC_Enabled
VP_RTC_VS_RTC_Activate.Signal=RTC_VS_RTC_Activate
```

CRC：

```text
Mcu.IPx=CRC
CRC.IPParameters=DefaultPolynomialUse,DefaultInitValueUse,CRCLength
CRC.DefaultPolynomialUse=DEFAULT_POLYNOMIAL_DISABLE
CRC.DefaultInitValueUse=DEFAULT_INIT_VALUE_ENABLE
CRC.CRCLength=CRC_POLYLENGTH_16B
VP_CRC_VS_CRC.Mode=CRC_Activate
VP_CRC_VS_CRC.Signal=CRC_VS_CRC
```

RNG：

```text
Mcu.IPx=RNG
VP_RNG_VS_RNG.Mode=RNG_Activate
VP_RNG_VS_RNG.Signal=RNG_VS_RNG
```

IWDG/WWDG 通常只有 IP 参数和生成函数，没有物理引脚；具体超时字段随系列变化，优先参考 CubeMX 生成。

## FreeRTOS

基础启用：

```text
Mcu.IPx=FREERTOS
VP_FREERTOS_VS_CMSIS_V1.Mode=CMSIS_V1
VP_FREERTOS_VS_CMSIS_V1.Signal=FREERTOS_VS_CMSIS_V1
```

默认任务常见：

```text
FREERTOS.Tasks01=defaultTask,0,128,StartDefaultTask,Default,NULL
```

CubeMX 6.x 动态分配样例：

```text
FREERTOS.Tasks01=defaultTask,0,4096,StartDefaultTask,Default,NULL,Dynamic,NULL,NULL
```

规则：

- RTOS 会影响 `NVIC.TimeBase`，有些工程把 HAL tick 改成 TIM。
- 不要只删 `FREERTOS.Tasks01`；CubeMX 可能重新生成默认任务。
- 新增任务、队列、信号量时必须匹配当前 CubeMX 版本的 `FREERTOS.*` 字段格式。

## RCC 和 SYS

RCC 参数必须出现在 `RCC.IPParameters`：

```text
RCC.IPParameters=AHBFreq_Value,APB1Freq_Value,APB2Freq_Value,SYSCLKFreq_VALUE,SYSCLKSource,HSE_VALUE,HSI_VALUE,PLLCLKFreq_Value
RCC.SYSCLKSource=RCC_SYSCLKSOURCE_PLLCLK
RCC.SYSCLKFreq_VALUE=72000000
RCC.USBFreq_Value=48000000
```

SYS 调试和时基：

```text
VP_SYS_VS_Systick.Mode=SysTick
VP_SYS_VS_Systick.Signal=SYS_VS_Systick
VP_SYS_VS_ND.Mode=No_Debug
VP_SYS_VS_ND.Signal=SYS_VS_ND
NVIC.TimeBase=TIM1_UP_IRQn
NVIC.TimeBaseIP=TIM1
VP_SYS_VS_tim1.Mode=TIM1
VP_SYS_VS_tim1.Signal=SYS_VS_tim1
```

修改时钟是高风险操作：必须同步频率值、PLL 参数、Flash latency、USB/SDMMC/ETH/ADC 外设时钟。

## 中间件和第三方包

第三方包通常使用：

```text
Mcu.ThirdParty0=STMicroelectronics.X-CUBE-MEMS1.5.2.1
Mcu.ThirdPartyNb=1
VP_STMicroelectronics.X-CUBE-MEMS1_VS_BoardOoExtensionJjMEMS_5.2.1.Mode=...
VP_STMicroelectronics.X-CUBE-MEMS1_VS_BoardOoExtensionJjMEMS_5.2.1.Signal=...
```

规则：

- `Mcu.ThirdPartyNb` 必须匹配 `Mcu.ThirdParty*` 数量。
- 第三方虚拟引脚名常包含版本号，不要手写猜测；优先复制同版本包生成结果。

## 修改检查清单

每次修改后：

- `rg -n "外设名|引脚名|IRQ名|Dma\\.|Mcu\\.IP|Mcu\\.Pin|functionlistsort" project.ioc`
- 校验 `Mcu.IPNb`、`Mcu.PinsNb`、`Dma.RequestsNb`、`Mcu.ThirdPartyNb`。
- 校验每个 `*.IPParameters` 包含新增参数键。
- 校验每个 `S_` 或 `GPXTI` 信号有对应 `SH.*.ConfNb`。
- 校验每个虚拟引脚 `VP_*` 同时在 `Mcu.Pin*` 中出现。
- 校验 DMA IRQ、外设 IRQ、stream/channel/request 名来自目标 MCU。
- 仅文本检查时说明“按 CubeMX 风格文本级校验”；只有实际打开或重新生成成功后，才能说 CubeMX 已验收。

## 参考样例来源

- STMicroelectronics `STM32_open_pin_data`，官方 CubeMX 6.17.0 board all-config `.ioc` 集合，覆盖 GPIO、USART、I2C、ADC、TIM、USB、ETH、SDMMC 等基础格式：https://github.com/STMicroelectronics/STM32_open_pin_data/tree/master/boards
- STM32F407 UART4 DMA `.ioc` 样例，用于确认 `Dma.UART4_RX/TX`、`DMA1_Stream2/4` 风格：https://git.pcmuhely.hu/playground-stm32/f407ve_hs_uart/blame/commit/bc01b1f0e8a117ec8b319aa287e2f04438bbefee/f407ve_hs_uart.ioc
- SPI + DMA + DMAMUX `.ioc` 片段，包含 `Dma.SPI1_RX.*`、`EventEnable`、`SignalID`、`Sync*` 字段：https://electronics.stackexchange.com/questions/635117/external-adc-with-spi-nucleo-h743zi2
- ADC + DMA + I2C + RTC `.ioc` 样例，包含 `Dma.ADC1.*`、多通道 ADC 和 F1 风格 I2C：https://git.lantian.pub/zjui/zjui-ece445/src/branch/master/stm32/stm32.ioc?display=source
- ST Community FreeRTOS 任务字段讨论，包含 `FREERTOS.Tasks01` 格式：https://community.st.com/t5/stm32cubemx-mcus/is-there-a-way-to-disable-the-automatic-default-task-creation-in/td-p/393332
- ST Community USB_DEVICE CDC 和 TIM PWM `.ioc` 片段：https://community.st.com/t5/stm32cubemx-mcus/error-after-choosing-usb-communication-with-stm32cubemx/td-p/152651
- FDCAN `.ioc` 参数片段，包含 nominal/data phase timing 和 filter 数量字段：https://community.st.com/t5/stm32-mcus-products/stm32h7-can-bus-communication-issue/td-p/684904
- CRC、SDIO、FATFS、USB MSC `.ioc` 样例：https://gitee.com/yes-stm32-f103-rc-v101/ytc130-dlaa-06-100-nhb/blob/main/TEST_STM32.ioc
