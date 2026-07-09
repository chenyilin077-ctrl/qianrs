# ADC — 模数转换器

> 芯片：STM32F4 | APB2 | 12 位 | 0~3.3V → 0~4095

## 原理

### 逐次逼近型 (SAR)
输入电压经采样保持 → 与 DAC 逐次比较 → 12 次比较 → 12bit 数字量

```
数字量 = Vin / VREF × 4095
Vin    = 数字量 × VREF / 4095          (VREF 通常 = 3.3V)
```

### 关键参数
- 分辨率：12bit（0~4095）
- VREF+：通常接 3.3V
- 采样时间：3/15/28/56/84/112/144/480 周期
- 转换模式：单次 / 连续 / 扫描（多通道）

---

## CubeMX 配置

### 单通道轮询（PA1 = ADC1_IN1）

| 步骤 | 参数 | 值 |
|------|------|-----|
| **Pinout** | PA1 | 选 **ADC1_IN1** |
| **Analog** → **ADC1** | Mode | 勾选 **IN1** |
| **Configuration → ADC1** | | |
| | Clock Prescaler | **PCLK2 divided by 4**（84/4=21MHz < 36MHz） |
| | Resolution | **12 bits** |
| | Scan Conversion Mode | **Disabled**（单通道） |
| | Continuous Conversion Mode | **Enabled**（连续转换） |
| **Configuration → ADC1 → IN1** | Sampling Time | **480 Cycles**（高阻抗源用长采样） |

### 多通道 + DMA

| 步骤 | 参数 | 值 |
|------|------|-----|
| **Analog → ADC1** | Number of Conversion | **2**（两个通道） |
| | Scan Conversion Mode | **Enabled** |
| | Continuous Conversion Mode | **Enabled** |
| **DMA Settings** 选项卡 | Add → ADC1 | |
| | Mode | **Circular** |
| | Data Width | **Half Word**（12bit 用 16bit） |

---

## CubeMX 自动生成代码

```c
// adc.c — MX_ADC1_Init()
static void MX_ADC1_Init(void) {
    ADC_ChannelConfTypeDef sConfig = {0};

    hadc1.Instance = ADC1;
    hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
    hadc1.Init.Resolution = ADC_RESOLUTION_12B;
    hadc1.Init.ScanConvMode = DISABLE;
    hadc1.Init.ContinuousConvMode = ENABLE;
    hadc1.Init.NbrOfConversion = 1;
    HAL_ADC_Init(&hadc1);

    sConfig.Channel = ADC_CHANNEL_1;
    sConfig.Rank = 1;
    sConfig.SamplingTime = ADC_SAMPLETIME_480CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);
}
// 选 DMA 后外加 MX_DMA_Init() 并自动绑定
```

---

## 手写业务代码

### 单通道轮询 + 电压显示

```c
int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_ADC1_Init();     // CubeMX 生成
    MX_USART1_UART_Init();

    while (1) {
        HAL_ADC_Start(&hadc1);
        if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK) {
            uint32_t val = HAL_ADC_GetValue(&hadc1);
            float voltage = val * 3.3f / 4095.0f;
            printf("ADC=%4lu, Voltage=%.3fV\r\n", val, voltage);
        }
        HAL_Delay(500);
    }
}
```

### DMA 循环采集 + 回调

```c
#define ADC_BUF_SIZE 100
uint16_t adc_buf[ADC_BUF_SIZE];

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc) {
    if (hadc->Instance == ADC1) {
        uint32_t sum = 0;
        for (int i = 0; i < ADC_BUF_SIZE; i++) sum += adc_buf[i];
        uint16_t avg = sum / ADC_BUF_SIZE;
        printf("Avg ADC=%u, Voltage=%.3fV\r\n", avg, avg * 3.3f / 4095.0f);
    }
}

int main(void) {
    // ... Init ...
    HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adc_buf, ADC_BUF_SIZE);
    while (1);  // 回调自动打印
}
```

---

## 注意事项
- ⚠️ ADC 时钟不能超 36MHz，CubeMX 里 PCLK2 是 84MHz 就选 /4
- ⚠️ PA1 引脚设 ADC1_IN1 后 CubeMX 自动配为 Analog 模式，不需要手动改成 Input
- ⚠️ 采样时间选长一点（480 cycles）避免读数偏低，CubeMX 里直接下拉选
- ⚠️ VREF+ 加 0.1μF+10μF 电容滤波，不稳定 = 所有读数不准确
- ⚠️ 输入电压不能超 VREF+（3.3V）
- ⚠️ DMA 模式：`Continuous DMA Requests` 选 `Enabled` 才持续采
