# I2C — 集成电路总线

> 芯片：STM32F4 | APB1 | 两线制：SCL + SDA

## 原理

### 两线制开漏总线
- **SCL**：时钟线，主机驱动。**SDA**：数据线，双向
- **开漏输出 + 外部上拉电阻**（典型 4.7kΩ @ 3.3V）
- 标准 100kHz / 快速 **400kHz**
- 7 位地址（128 个设备），一条总线挂多个器件

### 时序
```
START: SCL=高 时 SDA↓          STOP: SCL=高 时 SDA↑
发送: [START][7bit地址+W][ACK][寄存器地址][ACK][数据][ACK]...[STOP]
```

---

## CubeMX 配置

以 **I2C1：PB6=SCL, PB7=SDA, 400kHz** 为例。

### Pinout
| 步骤 | 操作 |
|------|------|
| **Connectivity** → **I2C1** | Mode 选 **I2C** |
| PB6 自动设为 I2C1_SCL, PB7 自动设为 I2C1_SDA | — |

### Configuration → I2C1
| 参数 | 值 |
|------|-----|
| I2C Speed Mode | **Fast Mode** |
| I2C Speed Frequency | **400 KHz** |
| Rise Time (Fm mode) | 保持默认 |
| Fall Time (Fm mode) | 保持默认 |

> CubeMX 自动算出时序参数，不需要手动算

---

## CubeMX 自动生成的代码

```c
// i2c.c — MX_I2C1_Init()
static void MX_I2C1_Init(void) {
    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 400000;
    hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
    hi2c1.Init.OwnAddress1 = 0;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    HAL_I2C_Init(&hi2c1);
}
```

---

## 常用 API

| 函数 | 说明 |
|------|------|
| `HAL_I2C_Mem_Write(&hi2c1, DevAddr, MemAddr, AddrSize, buf, len, timeout)` | 写器件寄存器 |
| `HAL_I2C_Mem_Read(&hi2c1, DevAddr, MemAddr, AddrSize, buf, len, timeout)` | 读器件寄存器 |
| `HAL_I2C_IsDeviceReady(&hi2c1, DevAddr, trials, timeout)` | 检测设备在线 |

> ⚠️ 器件地址要**左移 1 位**：手册写 `0x50` → 传 `0xA0`（器件地址 = 0x50 << 1）

---

## 完整示例：读写 AT24C02 EEPROM

```c
#define AT24C02_ADDR  0xA0   // 0x50 << 1

void AT24C02_WriteByte(uint8_t addr, uint8_t data) {
    HAL_I2C_Mem_Write(&hi2c1, AT24C02_ADDR, addr,
                      I2C_MEMADD_SIZE_8BIT, &data, 1, 10);
    HAL_Delay(5);  // EEPROM 写入需要 5ms！
}

uint8_t AT24C02_ReadByte(uint8_t addr) {
    uint8_t data;
    HAL_I2C_Mem_Read(&hi2c1, AT24C02_ADDR, addr,
                     I2C_MEMADD_SIZE_8BIT, &data, 1, 10);
    return data;
}

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();     // CubeMX 生成
    MX_USART1_UART_Init();

    // 写 → 读 → 打印
    AT24C02_WriteByte(0x00, 'A');
    AT24C02_WriteByte(0x01, 'B');
    printf("Addr 0x00 = %c, 0x01 = %c\r\n",
           AT24C02_ReadByte(0x00), AT24C02_ReadByte(0x01));
    while (1);
}
```

---

## 常见 I2C 器件地址

| 器件 | 功能 | 7bit 地址 | HAL 传参（<<1） |
|------|------|-----------|----------------|
| AT24C02 | EEPROM | 0x50 | 0xA0 |
| MPU6050 | 6 轴陀螺仪 | 0x68 | 0xD0 |
| OLED SSD1306 | 128×64 | 0x3C | 0x78 |
| BH1750 | 光照 | 0x23 | 0x46 |
| SHT30 | 温湿度 | 0x44 | 0x88 |

---

## 注意事项
- ⚠️ 外部必须接 **4.7kΩ 上拉电阻**到 3.3V，开漏没上拉 SDA/SCL 永远低
- ⚠️ 7bit 地址**左移 1 位**，HAL 用 8bit，忘移了不响应
- ⚠️ EEPROM 写后 **Delay 5ms**，写入周期完成前读不到新数据
- ⚠️ CubeMX 里 Frequency 往上调可能红字警告——降一点频率或调 Rise/Fall Time
