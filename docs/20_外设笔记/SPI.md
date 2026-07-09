# SPI — 串行外设接口

> 芯片：STM32F4 | APB2（SPI1）/ APB1（SPI2-3）| 4 线全双工

## 原理

### 信号线
| 信号 | 方向 | 作用 |
|------|------|------|
| **SCK** | Master→Slave | 时钟，主机产生 |
| **MOSI** | Master→Slave | 主机发数据 |
| **MISO** | Slave→Master | 从机发数据 |
| **NSS/CS** | Master→Slave | 片选，低电平选中 |

### CPOL + CPHA 四种模式
| 模式 | CPOL | CPHA | SCK 空闲 | 采样沿 |
|------|------|------|----------|--------|
| 0 | 0 | 0 | 低 | 第一边沿 |
| 1 | 0 | 1 | 低 | 第二边沿 |
| 2 | 1 | 0 | 高 | 第一边沿 |
| 3 | 1 | 1 | 高 | 第二边沿 |

> 📌 查器件手册的 SPI 时序图匹配 CPOL/CPHA 是调通的关键

---

## CubeMX 配置

以 **SPI1：PA5=SCK, PA6=MISO, PA7=MOSI, PA4=软件 CS** 为例。

### Pinout
| 步骤 | 操作 |
|------|------|
| **Connectivity** → **SPI1** | Mode 选 **Full-Duplex Master** |
| PA5/PA6/PA7 自动分配 | SCK / MISO / MOSI |
| PA4 手动设为 GPIO_Output | 右键 → `Enter User Label` → `SPI_CS` |
| **NSS Signal Type** | 选 **Software**（硬件 NSS 有坑） |

### Configuration → SPI1
| 参数 | W25Q64 示例 | 说明 |
|------|------------|------|
| Frame Format | **Motorola** | 标准 SPI |
| Data Size | **8 Bits** | — |
| First Bit | **MSB First** | — |
| **CPOL** | **Low** | W25Q64 模式 0 |
| **CPHA** | **1 Edge** | W25Q64 模式 0 |
| Baud Rate Prescaler | 视需求而定 | 84MHz / 2 = 42MHz（接近 W25Q64 上限） |

---

## CubeMX 自动生成的代码

```c
// spi.c — MX_SPI1_Init()
static void MX_SPI1_Init(void) {
    hspi1.Instance = SPI1;
    hspi1.Init.Mode = SPI_MODE_MASTER;
    hspi1.Init.Direction = SPI_DIRECTION_2LINES;
    hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
    hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;
    hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;
    hspi1.Init.NSS = SPI_NSS_SOFT;
    hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_2;
    hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
    HAL_SPI_Init(&hspi1);
}
```

---

## 常用 API

| 函数 | 说明 |
|------|------|
| `HAL_SPI_Transmit(&hspi1, buf, len, timeout)` | 只发 |
| `HAL_SPI_Receive(&hspi1, buf, len, timeout)` | 只收 |
| `HAL_SPI_TransmitReceive(&hspi1, tx, rx, len, timeout)` | 全双工收发 |

---

## 完整示例：读 W25Q64 JEDEC ID

CubeMX 已生成 `MX_SPI1_Init()` 和 CS 引脚的 `MX_GPIO_Init()`。只需写：

```c
#define CS_LOW()  HAL_GPIO_WritePin(SPI_CS_GPIO_Port, SPI_CS_Pin, GPIO_PIN_RESET)
#define CS_HIGH() HAL_GPIO_WritePin(SPI_CS_GPIO_Port, SPI_CS_Pin, GPIO_PIN_SET)

uint8_t SPI_ReadWriteByte(uint8_t tx_data) {
    uint8_t rx_data;
    HAL_SPI_TransmitReceive(&hspi1, &tx_data, &rx_data, 1, 10);
    return rx_data;
}

void Read_JEDEC_ID(void) {
    uint8_t id[3];
    CS_LOW();
    SPI_ReadWriteByte(0x9F);           // 命令
    id[0] = SPI_ReadWriteByte(0xFF);   // Manufacturer ID
    id[1] = SPI_ReadWriteByte(0xFF);   // Memory Type
    id[2] = SPI_ReadWriteByte(0xFF);   // Capacity
    CS_HIGH();
    printf("JEDEC ID: %02X %02X %02X\r\n", id[0], id[1], id[2]);
    // 预期：EF 40 17 = Winbond W25Q64
}

// 页读取
void Read_Page(uint32_t addr, uint8_t *buf, uint16_t len) {
    CS_LOW();
    SPI_ReadWriteByte(0x03);
    SPI_ReadWriteByte((addr >> 16) & 0xFF);
    SPI_ReadWriteByte((addr >> 8) & 0xFF);
    SPI_ReadWriteByte(addr & 0xFF);
    for (uint16_t i = 0; i < len; i++)
        buf[i] = SPI_ReadWriteByte(0xFF);
    CS_HIGH();
}

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();      // CubeMX 生成
    MX_SPI1_Init();      // CubeMX 生成
    MX_USART1_UART_Init();

    Read_JEDEC_ID();
    while (1);
}
```

---

## 注意事项
- ⚠️ CPOL/CPHA 必须匹配器件，CubeMX 里改这两个参数就可以适配所有 SPI 设备
- ⚠️ SPI 收发同时进行，`HAL_SPI_TransmitReceive` 一次搞定
- ⚠️ SCK 频率不要超过器件上限，CubeMX 的 Prescaler 控制
- ⚠️ 软件 NSS 自己拉 CS 引脚比硬件 NSS 靠谱，CubeMX 里选 `Software`
- ⚠️ MOSI↔DI、MISO↔DO 别接反
