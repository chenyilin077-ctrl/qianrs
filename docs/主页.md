# 🎛️ 嵌入式开发笔记

## 快速入口
- 📥 [[00_Inbox/|收件箱]] — 快速记录
- 🔲 [[10_芯片手册/|芯片手册]]
  - [[10_芯片手册/STM32芯片手册索引|📋 手册索引]] — STM32F1/F4/N6
- ⚙️ [[20_外设笔记/|外设笔记]]
  - [[20_外设笔记/外设笔记索引|📋 外设索引]]
  - [[20_外设笔记/RCC|RCC 时钟]] · [[20_外设笔记/GPIO|GPIO]] · [[20_外设笔记/UART|UART]] · [[20_外设笔记/SPI|SPI]]
  - [[20_外设笔记/I2C|I2C]] · [[20_外设笔记/TIM|TIM/PWM]] · [[20_外设笔记/ADC|ADC]] · [[20_外设笔记/DMA|DMA]]
  - [[20_外设笔记/NVIC_EXTI|NVIC/EXTI 中断]]
- 📁 [[30_项目/|项目]] — 课程设计、比赛
- 🔌 [[40_电路设计/|电路设计]]
  - [[40_电路设计/电路原理图索引|📋 原理图索引]]
- 📋 [[50_代码片段/|代码片段]] — 复用代码
- 🐛 [[90_踩坑记录/|踩坑记录]] — bug 日记

## 常用芯片
- [[10_芯片手册/STM32芯片手册索引|STM32F103]] — 经典入门
- [[10_芯片手册/STM32芯片手册索引|STM32F429]] — 阿波罗 V2
- [[10_芯片手册/STM32芯片手册索引|STM32N647]] — 正点原子 AI 开发板

## 常用外设速查
| 外设 | 关键 API |
|------|----------|
| [[GPIO]] | `HAL_GPIO_WritePin` / `TogglePin` / `ReadPin` |
| [[UART]] | `HAL_UART_Transmit` / `Receive_IT` / `printf` 重定向 |
| [[SPI]] | `HAL_SPI_TransmitReceive` / W25Q64 读写 |
| [[I2C]] | `HAL_I2C_Mem_Write` / `Mem_Read` / 器件地址表 |
| [[TIM]] | `HAL_TIM_PWM_Start` / `__SET_COMPARE` / 舵机 50Hz |
| [[ADC]] | `HAL_ADC_PollForConversion` / `ADC+DMA` 循环采集 |
| [[DMA]] | `HAL_UART_Transmit_DMA` / `Receive_DMA` / 通道映射 |
| [[NVIC_EXTI]] | 按键中断 / IDLE+DMA 不定长接收 |
| [[RCC]] | `SystemClock_Config` / MCO 输出 / 时钟树 |

## 最近笔记
```dataview
LIST FROM "20_外设笔记" OR "10_芯片手册"
SORT file.mtime DESC
LIMIT 10
```
