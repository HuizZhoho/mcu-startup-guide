# STM32F429 + KSZ9897 以太网交换机完整驱动固件实现

> 基于 SPI 控制 + RMII 数据通道的 5+1 端口交换机驱动，包含完整初始化、SPI 通信、端口管理、VLAN、MAC 表、MIB 统计、CLI 监控及错误处理。

---

## 目录

- [一、系统架构](#一条统架构)
- [二、项目文件结构](#二项目文件结构)
- [三、硬件连接定义](#三硬件连接定义)
- [四、底层 SPI 通信层](#四底层-spi-通信层)
- [五、交换机核心初始化](#五交换机核心初始化)
- [六、端口管理 API](#六端口管理-api)
- [七、VLAN 管理 API](#七vlan-管理-api)
- [八、MAC 地址表 API](#八mac-地址表-api)
- [九、MIB 统计计数器](#九mib-统计计数器)
- [十、CLI 监控终端](#十cli-监控终端)
- [十一、错误处理体系](#十一错误处理体系)
- [十二、主函数集成示例](#十二主函数集成示例)
- [十三、调试与常见问题](#十三调试与常见问题)

---

## 一、系统架构

### 1.1 拓扑结构

```
                                    +----------------------+
                                    |    STM32F429ZIT6      |
                                    |                      |
                                    |  RMII <- -> ETH MAC   |
                                    |  SPI1  <- -> SPI Master |
                                    |  UART7 <- -> CLI 终端   |
                                    |  PA0   <- -> SW_RST_N   |
                                    |  PE0   <- -> SW_IRQ_N   |
                                    +------+---------------+
                                           |
                              SPI (CS/SCK/MOSI/MISO)
                              RMII (REF_CLK/TXD[1:0]/RXD[1:0]/CRS_DV/TXEN)
                              RST_N, IRQ_N
                                           |
                                    +------+---------------+
                                    |   KSZ9897RNX (Switch) |
                                    |                      |
                                    |  Port 1~5: 10/100TX  | <- 用户端口 (RJ45)
                                    |  Port 7:   RMII Host | <- CPU 上行端口
                                    +----------------------+
```

### 1.2 关键设计决策

| 项目 | 选择 | 原因 |
|------|------|------|
| CPU 上行端口 | Port 7 (RMII) | 支持 100Mbps 全双工，引脚少 |
| 控制通道 | SPI (Mode 0, 20MHz) | 寄存器读写，不占数据带宽 |
| 端口隔离 | Port-based VLAN | 用户端口互访隔离，仅能到达 CPU |
| MAC 学习 | 硬件自动学习 | 使能 SWITCH_MAC_LEARN_EN 位 |
| PHY 管理 | 通过内部 SMI 访问 | 无需独立 MDIO 引脚 |

### 1.3 KSZ9897 内部框图（简化）

```
                  +---------------------------------------+
                  |           KSZ9897 Switch Core          |
                  |                                        |
  Port 1 ---------+  PHY1 --+                              |
  Port 2 ---------+  PHY2 --+                              |
  Port 3 ---------+  PHY3 --+-- Switch Fabric --- Host   +-- RMII -- > STM32
  Port 4 ---------+  PHY4 --+                     Port 7  |
  Port 5 ---------+  PHY5 --+                              |
                  |                                        |
                  |  SPI Slave <-- -- > SPI Master         |
                  |  Control Registers                      |
                  |  MAC Table / VLAN Table / MIB Counters  |
                  +---------------------------------------+
```

---

## 二、项目文件结构

```
Project/
+-- Inc/
|   +-- ksz9897_regs.h          // 所有寄存器地址宏定义
|   +-- ksz9897_spi.h           // SPI 通信层头文件
|   +-- ksz9897_driver.h        // 交换机驱动总头文件
|   +-- port_management.h       // 端口管理 API
|   +-- vlan_management.h       // VLAN 管理 API
|   +-- mac_table.h             // MAC 地址表 API
|   +-- mib_counters.h          // MIB 统计 API
|   +-- cli_monitor.h           // CLI 终端头文件
|   +-- error_handler.h         // 错误处理定义
|
+-- Src/
|   +-- ksz9897_spi.c           // SPI HAL 封装
|   +-- ksz9897_driver.c        // 核心初始化
|   +-- port_management.c       // 端口管理实现
|   +-- vlan_management.c       // VLAN 管理实现
|   +-- mac_table.c             // MAC 表管理实现
|   +-- mib_counters.c          // MIB 计数实现
|   +-- cli_monitor.c           // CLI 终端实现
|   +-- error_handler.c         // 错误处理实现
|
+-- main.c                      // 主函数集成
```

---

## 三、硬件连接定义

### 3.1 STM32F429 引脚分配

```c
/* ============================================================================
 * 文件名: ksz9897_hw_map.h
 * 描述:   STM32F429 与 KSZ9897 硬件引脚映射
 * ==========================================================================*/

#ifndef KSZ9897_HW_MAP_H
#define KSZ9897_HW_MAP_H

#include "stm32f4xx_hal.h"

/* ===================== SPI 接口 (SPI1) ===================== */
#define SW_SPI_INSTANCE         SPI1
#define SW_SPI_CLK_ENABLE()     __HAL_RCC_SPI1_CLK_ENABLE()
#define SW_SPI_CLK_DISABLE()    __HAL_RCC_SPI1_CLK_DISABLE()

/* SPI 引脚 - GPIOA */
#define SW_SPI_SCK_PIN          GPIO_PIN_5   /* PA5 - SPI1_SCK  */
#define SW_SPI_MISO_PIN         GPIO_PIN_6   /* PA6 - SPI1_MISO */
#define SW_SPI_MOSI_PIN         GPIO_PIN_7   /* PA7 - SPI1_MOSI */
#define SW_SPI_CS_PIN           GPIO_PIN_4   /* PA4 - SPI1_NSS (软件控制) */
#define SW_SPI_GPIO_PORT        GPIOA
#define SW_SPI_GPIO_CLK_ENABLE() __HAL_RCC_GPIOA_CLK_ENABLE()

/* ===================== 硬件控制引脚 ===================== */
/* 复位引脚 - 低电平复位 */
#define SW_RST_PIN              GPIO_PIN_0   /* PA0 */
#define SW_RST_PORT             GPIOA
#define SW_RST_CLK_ENABLE()     __HAL_RCC_GPIOA_CLK_ENABLE()

/* 中断引脚 - 下降沿触发 */
#define SW_IRQ_PIN              GPIO_PIN_0   /* PE0 */
#define SW_IRQ_PORT             GPIOE
#define SW_IRQ_CLK_ENABLE()     __HAL_RCC_GPIOE_CLK_ENABLE()
#define SW_IRQ_EXTI_LINE        EXTI_LINE_0
#define SW_IRQ_EXTI_IRQn        EXTI0_IRQn

/* ===================== RMII 接口 (ETH) ===================== */
/*
 * STM32F429 ETH RMII 固定引脚:
 *   PA1  - ETH_RMII_REF_CLK (50MHz 来自外部振荡器或 KSZ9897 CLKOUT)
 *   PA2  - ETH_MDIO
 *   PC1  - ETH_MDC
 *   PG11 - ETH_RMII_TX_EN
 *   PG13 - ETH_RMII_TXD0
 *   PG14 - ETH_RMII_TXD1
 *   PC4  - ETH_RMII_RXD0
 *   PC5  - ETH_RMII_RXD1
 *   PA7  - ETH_RMII_CRS_DV
 */

/* ===================== UART 终端 (UART7) ===================== */
#define CLI_UART_INSTANCE       UART7
#define CLI_UART_CLK_ENABLE()   __HAL_RCC_UART7_CLK_ENABLE()
#define CLI_TX_PIN              GPIO_PIN_6   /* PF6 - UART7_TX */
#define CLI_RX_PIN              GPIO_PIN_7   /* PF7 - UART7_RX */
#define CLI_UART_GPIO_PORT      GPIOF
#define CLI_UART_GPIO_CLK_ENABLE() __HAL_RCC_GPIOF_CLK_ENABLE()
#define CLI_UART_BAUDRATE       115200

#endif /* KSZ9897_HW_MAP_H */
```

---

## 四、底层 SPI 通信层

### 4.1 SPI 帧格式解析

KSZ9897 的 SPI 帧格式如下（SPI Mode 0, CPOL=0, CPHA=0, MSB First）:

```
读操作:
  +-----+-----+-----+-----+-----+-----+-----+-----+
  | CMD | A15 | A14 | ... |  A8 | A7  | ... |  A0 |
  | hi  |     |     |     |     |     |     |     |
  +-----+-----+-----+-----+-----+-----+-----+-----+
  |[1][A14:8]|   ADDR[7:0]  | 0x00(Dummy) |  DATA_OUT  |
  +-----+-----+-----+-----+-----+-----+-----+-----+
   CS保持低电平期间, 地址自动递增

写操作:
  +-----+-----+-----+-----+-----+-----+
  | CMD | A15 | ... |  A8 | A7  | ... | A0  |
  +-----+-----+-----+-----+-----+-----+
  |[0][A14:8]|   ADDR[7:0]  |      DATA_IN     |
  +-----+-----+-----+-----+-----+-----+
```

- **CMD 字节**: bit[7] = 1 读 / 0 写, bit[6:0] = 地址高 7 位 (A[14:8])
- **地址字节**: 低 8 位地址 (A[7:0])
- **数据字节**: 读操作需要 1 个 dummy 字节; 写操作直接发送数据
- **自动递增**: 多字节读写时地址自动 +1

### 4.2 SPI 头文件

```c
/* ============================================================================
 * 文件名: ksz9897_spi.h
 * 描述:   KSZ9897 SPI 通信层接口
 * ==========================================================================*/

#ifndef KSZ9897_SPI_H
#define KSZ9897_SPI_H

#include <stdint.h>
#include "stm32f4xx_hal.h"

#ifdef __cplusplus
extern "C" {
#endif

/* SPI 超时定义 (单位: ms) */
#define KSZ_SPI_TIMEOUT_MS      100
#define KSZ_SPI_RETRY_MAX       3

/* SPI 通信状态 */
typedef enum {
    KSZ_SPI_OK       = 0,
    KSZ_SPI_TIMEOUT  = -1,
    KSZ_SPI_BUSY     = -2,
    KSZ_SPI_ERROR    = -3,
} KSZ_SPI_Status_t;

/* ==================== 基本 SPI 读写函数 ==================== */

KSZ_SPI_Status_t KSZ_ReadReg8(uint16_t reg, uint8_t *val);
KSZ_SPI_Status_t KSZ_WriteReg8(uint16_t reg, uint8_t val);
KSZ_SPI_Status_t KSZ_ReadReg16(uint16_t reg, uint16_t *val);
KSZ_SPI_Status_t KSZ_WriteReg16(uint16_t reg, uint16_t val);
KSZ_SPI_Status_t KSZ_ReadBurst(uint16_t reg, uint8_t *buf, uint16_t len);
KSZ_SPI_Status_t KSZ_WriteBurst(uint16_t reg, const uint8_t *buf, uint16_t len);
KSZ_SPI_Status_t KSZ_ModifyReg(uint16_t reg, uint8_t mask, uint8_t val);
KSZ_SPI_Status_t KSZ_SPI_Init(SPI_HandleTypeDef *hspi);
KSZ_SPI_Status_t KSZ_SPI_SelfTest(void);

extern SPI_HandleTypeDef hspi1;

#ifdef __cplusplus
}
#endif

#endif /* KSZ9897_SPI_H */
```

### 4.3 SPI 实现代码

```c
/* ============================================================================
 * 文件名: ksz9897_spi.c
 * 描述:   KSZ9897 SPI 通信层实现
 *         基于 STM32 HAL SPI 阻塞模式, 片选软件控制
 * ==========================================================================*/

#include "ksz9897_spi.h"
#include "ksz9897_hw_map.h"
#include <string.h>

/* 静态变量: SPI 句柄指针 */
static SPI_HandleTypeDef *g_hspi = NULL;

/* ==================== SPI 硬件初始化 ==================== */

KSZ_SPI_Status_t KSZ_SPI_Init(SPI_HandleTypeDef *hspi)
{
    GPIO_InitTypeDef gpio_init = {0};

    /* 保存 SPI 句柄 */
    g_hspi = hspi;

    /* 使能 SPI GPIO 时钟 */
    SW_SPI_GPIO_CLK_ENABLE();

    /* 配置 SPI 引脚: SCK, MOSI, MISO */
    gpio_init.Pin       = SW_SPI_SCK_PIN | SW_SPI_MOSI_PIN | SW_SPI_MISO_PIN;
    gpio_init.Mode      = GPIO_MODE_AF_PP;
    gpio_init.Pull      = GPIO_NOPULL;
    gpio_init.Speed     = GPIO_SPEED_FREQ_VERY_HIGH;
    gpio_init.Alternate = GPIO_AF5_SPI1;
    HAL_GPIO_Init(SW_SPI_GPIO_PORT, &gpio_init);

    /* 配置 CS (片选) 为普通推挽输出, 初始高电平 */
    gpio_init.Pin       = SW_SPI_CS_PIN;
    gpio_init.Mode      = GPIO_MODE_OUTPUT_PP;
    gpio_init.Pull      = GPIO_PULLUP;
    gpio_init.Speed     = GPIO_SPEED_FREQ_VERY_HIGH;
    gpio_init.Alternate = 0;
    HAL_GPIO_Init(SW_SPI_GPIO_PORT, &gpio_init);
    HAL_GPIO_WritePin(SW_SPI_GPIO_PORT, SW_SPI_CS_PIN, GPIO_PIN_SET);

    /* 使能 SPI 时钟 */
    SW_SPI_CLK_ENABLE();

    /* 配置 SPI 外设参数 */
    g_hspi->Instance               = SW_SPI_INSTANCE;
    g_hspi->Init.Mode              = SPI_MODE_MASTER;
    g_hspi->Init.Direction         = SPI_DIRECTION_2LINES;
    g_hspi->Init.DataSize          = SPI_DATASIZE_8BIT;
    g_hspi->Init.CLKPolarity       = SPI_POLARITY_LOW;   /* CPOL = 0 */
    g_hspi->Init.CLKPhase          = SPI_PHASE_1EDGE;    /* CPHA = 0 */
    g_hspi->Init.NSS               = SPI_NSS_SOFT;
    g_hspi->Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_2;
    g_hspi->Init.FirstBit          = SPI_FIRSTBIT_MSB;
    g_hspi->Init.TIMode            = SPI_TIMODE_DISABLE;
    g_hspi->Init.CRCCalculation    = SPI_CRCCALCULATION_DISABLE;
    g_hspi->Init.CRCPolynomial     = 7;

    if (HAL_SPI_Init(g_hspi) != HAL_OK) {
        return KSZ_SPI_ERROR;
    }

    HAL_GPIO_WritePin(SW_SPI_GPIO_PORT, SW_SPI_CS_PIN, GPIO_PIN_SET);
    return KSZ_SPI_OK;
}

/* ==================== 片选控制内联函数 ==================== */

static inline void CS_Low(void)
{
    HAL_GPIO_WritePin(SW_SPI_GPIO_PORT, SW_SPI_CS_PIN, GPIO_PIN_RESET);
}

static inline void CS_High(void)
{
    HAL_GPIO_WritePin(SW_SPI_GPIO_PORT, SW_SPI_CS_PIN, GPIO_PIN_SET);
}

/* ==================== SPI 读写核心 ==================== */

static uint8_t BuildCmdByte(uint16_t reg, uint8_t is_read)
{
    uint8_t cmd = (uint8_t)((reg >> 8) & 0x7F);
    if (is_read) {
        cmd |= 0x80;
    }
    return cmd;
}

KSZ_SPI_Status_t KSZ_ReadReg8(uint16_t reg, uint8_t *val)
{
    uint8_t cmd_byte;
    uint8_t addr_byte;
    uint8_t dummy = 0x00;
    uint8_t retry;

    if (g_hspi == NULL || val == NULL) return KSZ_SPI_ERROR;

    for (retry = 0; retry < KSZ_SPI_RETRY_MAX; retry++) {
        cmd_byte = BuildCmdByte(reg, 1);
        addr_byte = (uint8_t)(reg & 0xFF);
        CS_Low();
        if (HAL_SPI_Transmit(g_hspi, &cmd_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        if (HAL_SPI_Transmit(g_hspi, &addr_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        if (HAL_SPI_TransmitReceive(g_hspi, &dummy, val, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        CS_High();
        return KSZ_SPI_OK;
    }
    return KSZ_SPI_TIMEOUT;
}

KSZ_SPI_Status_t KSZ_WriteReg8(uint16_t reg, uint8_t val)
{
    uint8_t cmd_byte;
    uint8_t addr_byte;
    uint8_t retry;

    if (g_hspi == NULL) return KSZ_SPI_ERROR;

    for (retry = 0; retry < KSZ_SPI_RETRY_MAX; retry++) {
        cmd_byte = BuildCmdByte(reg, 0);
        addr_byte = (uint8_t)(reg & 0xFF);
        CS_Low();
        if (HAL_SPI_Transmit(g_hspi, &cmd_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        if (HAL_SPI_Transmit(g_hspi, &addr_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        if (HAL_SPI_Transmit(g_hspi, &val, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        CS_High();
        return KSZ_SPI_OK;
    }
    return KSZ_SPI_TIMEOUT;
}

KSZ_SPI_Status_t KSZ_ReadReg16(uint16_t reg, uint16_t *val)
{
    uint8_t buf[2];
    KSZ_SPI_Status_t ret;
    ret = KSZ_ReadBurst(reg, buf, 2);
    if (ret == KSZ_SPI_OK) {
        *val = ((uint16_t)buf[0] << 8) | buf[1];
    }
    return ret;
}

KSZ_SPI_Status_t KSZ_WriteReg16(uint16_t reg, uint16_t val)
{
    uint8_t buf[2];
    buf[0] = (uint8_t)(val >> 8);
    buf[1] = (uint8_t)(val & 0xFF);
    return KSZ_WriteBurst(reg, buf, 2);
}

KSZ_SPI_Status_t KSZ_ReadBurst(uint16_t reg, uint8_t *buf, uint16_t len)
{
    uint8_t cmd_byte;
    uint8_t addr_byte;
    uint8_t dummy = 0x00;
    uint16_t i;
    uint8_t retry;

    if (g_hspi == NULL || buf == NULL || len == 0) return KSZ_SPI_ERROR;

    for (retry = 0; retry < KSZ_SPI_RETRY_MAX; retry++) {
        cmd_byte = BuildCmdByte(reg, 1);
        addr_byte = (uint8_t)(reg & 0xFF);
        CS_Low();
        if (HAL_SPI_Transmit(g_hspi, &cmd_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        if (HAL_SPI_Transmit(g_hspi, &addr_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        for (i = 0; i < len; i++) {
            if (HAL_SPI_TransmitReceive(g_hspi, &dummy, &buf[i], 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) {
                CS_High(); break;
            }
        }
        if (i < len) { CS_High(); continue; }
        CS_High();
        return KSZ_SPI_OK;
    }
    return KSZ_SPI_TIMEOUT;
}

KSZ_SPI_Status_t KSZ_WriteBurst(uint16_t reg, const uint8_t *buf, uint16_t len)
{
    uint8_t cmd_byte;
    uint8_t addr_byte;
    uint16_t i;
    uint8_t retry;

    if (g_hspi == NULL || buf == NULL || len == 0) return KSZ_SPI_ERROR;

    for (retry = 0; retry < KSZ_SPI_RETRY_MAX; retry++) {
        cmd_byte = BuildCmdByte(reg, 0);
        addr_byte = (uint8_t)(reg & 0xFF);
        CS_Low();
        if (HAL_SPI_Transmit(g_hspi, &cmd_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        if (HAL_SPI_Transmit(g_hspi, &addr_byte, 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); continue; }
        for (i = 0; i < len; i++) {
            if (HAL_SPI_Transmit(g_hspi, &buf[i], 1, KSZ_SPI_TIMEOUT_MS) != HAL_OK) { CS_High(); break; }
        }
        if (i < len) { CS_High(); continue; }
        CS_High();
        return KSZ_SPI_OK;
    }
    return KSZ_SPI_TIMEOUT;
}

KSZ_SPI_Status_t KSZ_ModifyReg(uint16_t reg, uint8_t mask, uint8_t val)
{
    uint8_t current;
    uint8_t new_val;
    KSZ_SPI_Status_t ret;

    ret = KSZ_ReadReg8(reg, &current);
    if (ret != KSZ_SPI_OK) return ret;

    new_val = (current & ~mask) | (val & mask);
    if (new_val != current) {
        ret = KSZ_WriteReg8(reg, new_val);
    }
    return ret;
}

KSZ_SPI_Status_t KSZ_SPI_SelfTest(void)
{
    uint8_t test_val;
    uint8_t readback;
    KSZ_SPI_Status_t ret;

    ret = KSZ_ReadReg8(0x0002, &test_val);
    if (ret != KSZ_SPI_OK) return ret;

    ret = KSZ_WriteReg8(0x0002, 0xA5);
    if (ret != KSZ_SPI_OK) return ret;

    ret = KSZ_ReadReg8(0x0002, &readback);
    if (ret != KSZ_SPI_OK) return ret;

    KSZ_WriteReg8(0x0002, test_val);

    if (readback != 0xA5) return KSZ_SPI_ERROR;
    return KSZ_SPI_OK;
}
```

---

## 五、交换机核心初始化

### 5.1 寄存器地址定义

```c
/* ============================================================================
 * 文件名: ksz9897_regs.h
 * 描述:   KSZ9897 所有关键寄存器地址宏定义
 *         参考: KSZ9897R/RNX Datasheet, Microchip
 * ==========================================================================*/

#ifndef KSZ9897_REGS_H
#define KSZ9897_REGS_H

#include <stdint.h>

/* ==================== 芯片标识寄存器 ==================== */
#define KSZ_REG_CHIP_ID0            0x0000  /* Family ID: 0x98 (R) */
#define KSZ_REG_CHIP_ID1            0x0001  /* Chip ID:   0x97 (R) */
#define KSZ_CHIP_FAMILY_ID          0x98
#define KSZ_CHIP_CHIP_ID            0x97
#define KSZ_CHIP_EXPECTED           0x9897  /* (ID0 << 8) | ID1 */

/* ==================== 全局控制寄存器 ==================== */
#define KSZ_REG_GLOBAL_CTRL_0       0x0002  /* Global Control 0 */
#define KSZ_GCTRL0_SWITCH_START     (1 << 0)  /* Bit 0: 启动交换机核心 */

#define KSZ_REG_GLOBAL_CTRL_4       0x0008  /* Global Control 4 */
#define KSZ_GCTRL4_TAIL_TAG_EN      (1 << 0)  /* Bit 0: 尾标签(CPU标记) */
#define KSZ_GCTRL4_MAC_LEARN_EN     (1 << 1)  /* Bit 1: MAC学习使能 */
#define KSZ_GCTRL4_AGING_EN         (1 << 2)  /* Bit 2: MAC老化使能 */

#define KSZ_REG_GLOBAL_CTRL_5       0x0009  /* Global Control 5 */
#define KSZ_GCTRL5_AGING_PERIOD_MASK 0x1F   /* Bits 4:0: 老化时间 (0=300s) */

/* ==================== 端口偏移定义 ==================== */
/*
 * 每个端口的寄存器块大小 0x20 (32 字节)
 * 端口 1-5: 带集成 PHY 的用户端口
 * 端口 6:   可选第二个 RMII/RGMII 端口 (级联用)
 * 端口 7:   CPU 上行 RMII 端口
 */
#define KSZ_PORT_BASE(port)         (0x0100 + ((port) - 1) * 0x20)

#define KSZ_PORT1_BASE              0x0100
#define KSZ_PORT2_BASE              0x0120
#define KSZ_PORT3_BASE              0x0140
#define KSZ_PORT4_BASE              0x0160
#define KSZ_PORT5_BASE              0x0180
#define KSZ_PORT6_BASE              0x01A0
#define KSZ_PORT7_BASE              0x01C0  /* CPU 上行端口 */

/* ==================== 端口控制寄存器 (偏移) ==================== */
#define KSZ_PORT_CTRL_0             0x00  /* Port Control 0 */
#define KSZ_PCTRL0_TX_EN            (1 << 0)
#define KSZ_PCTRL0_RX_EN            (1 << 1)
#define KSZ_PCTRL0_LEARN_DIS        (1 << 2)
#define KSZ_PCTRL0_INGRESS_FLTR     (1 << 3)
#define KSZ_PCTRL0_BROADCAST_STORM  (1 << 4)
#define KSZ_PCTRL0_PORT_PASS_ALL    (1 << 5)

#define KSZ_PORT_CTRL_1             0x01  /* Port Control 1 */
#define KSZ_PCTRL1_BACK_PRESSURE    (1 << 0)
#define KSZ_PCTRL1_FLOW_CONTROL     (1 << 1)
#define KSZ_PCTRL1_PHY_LOOPBACK     (1 << 2)
#define KSZ_PCTRL1_DUPLEX           (1 << 3)  /* 1=全双工, 0=半双工 */
#define KSZ_PCTRL1_SPEED_10_100     (1 << 4)  /* 1=100M, 0=10M */
#define KSZ_PCTRL1_SPEED_1000       (1 << 5)  /* 仅 Port 6/7 千兆 */

#define KSZ_PORT_CTRL_2             0x02  /* Port Control 2 */
#define KSZ_PCTRL2_PHY_MODE_MASK    0x07
#define KSZ_PCTRL2_PHY_AUTO_NEGO    (1 << 3)
#define KSZ_PCTRL2_AN_RESTART       (1 << 4)
#define KSZ_PCTRL2_PHY_PWR_DOWN     (1 << 5)
#define KSZ_PCTRL2_AUTO_MDIX_DIS    (1 << 7)

/* ==================== 端口状态寄存器 ==================== */
#define KSZ_PORT_STATUS_0           0x04  /* Port Status 0 */
#define KSZ_PSTAT0_LINK_UP          (1 << 0)
#define KSZ_PSTAT0_DUPLEX           (1 << 1)
#define KSZ_PSTAT0_SPEED_MASK       0x06
#define KSZ_PSTAT0_SPEED_10         0x00
#define KSZ_PSTAT0_SPEED_100        0x02
#define KSZ_PSTAT0_SPEED_1000       0x04
#define KSZ_PSTAT0_TX_FLOW_ACTIVE   (1 << 3)
#define KSZ_PSTAT0_RX_FLOW_ACTIVE   (1 << 4)

/* ==================== 主机端口接口控制 ==================== */
#define KSZ_REG_RMII_CTRL           0x00D0  /* RMII 控制寄存器 */
#define KSZ_RMII_CTRL_RMII_EN       (1 << 0)
#define KSZ_RMII_CTRL_CLK_SEL_MASK  (3 << 1)
#define KSZ_RMII_CTRL_CLK_SEL_50MHZ (1 << 1)

/* ==================== VLAN 表寄存器 ==================== */
#define KSZ_REG_VLAN_TABLE_ACCESS   0x0400
#define KSZ_VLAN_ACCESS_READ        (1 << 0)
#define KSZ_VLAN_ACCESS_WRITE       (1 << 1)
#define KSZ_VLAN_ACCESS_START       (1 << 2)
#define KSZ_REG_VLAN_TABLE_DATA     0x0401
#define KSZ_VLAN_DATA_LEN           6

/* ==================== MAC 地址表寄存器 ==================== */
#define KSZ_REG_MAC_TABLE_ACCESS    0x0500
#define KSZ_MAC_ACCESS_READ         (1 << 0)
#define KSZ_MAC_ACCESS_WRITE        (1 << 1)
#define KSZ_MAC_ACCESS_START        (1 << 2)
#define KSZ_REG_MAC_TABLE_DATA0     0x0501
#define KSZ_MAC_DATA_LEN            9
#define KSZ_REG_MAC_TABLE_ENTRIES   0x050A

/* ==================== MIB 计数器 ==================== */
#define KSZ_REG_MIB_BASE(port)      (0x0600 + ((port) - 1) * 0x40)
#define KSZ_MIB_COUNTER_SIZE        0x40

#define KSZ_MIB_RX_BCAST            0x00
#define KSZ_MIB_RX_MCAST            0x04
#define KSZ_MIB_TX_PKTS             0x20
#define KSZ_MIB_RX_DROP             0x24
#define KSZ_MIB_TX_DROP             0x28
#define KSZ_MIB_RX_ERROR            0x2C
#define KSZ_MIB_TX_ERROR            0x30
#define KSZ_MIB_RX_OCTETS           0x34
#define KSZ_MIB_TX_OCTETS           0x38
#define KSZ_MIB_COLLISIONS          0x3C

#define KSZ_REG_MIB_CTRL            0x00B0
#define KSZ_MIB_CTRL_FLUSH_ALL      (1 << 0)

/* ==================== 端口数量 ==================== */
#define KSZ_NUM_PORTS               7
#define KSZ_USER_PORT_COUNT         5
#define KSZ_HOST_PORT               7

#endif /* KSZ9897_REGS_H */
```

### 5.2 驱动主头文件

```c
/* ============================================================================
 * 文件名: ksz9897_driver.h
 * 描述:   KSZ9897 交换机驱动主接口
 * ==========================================================================*/

#ifndef KSZ9897_DRIVER_H
#define KSZ9897_DRIVER_H

#include "ksz9897_spi.h"
#include "ksz9897_regs.h"

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    uint8_t     initialized;
    uint8_t     chip_id_hi;
    uint8_t     chip_id_lo;
    uint8_t     port_link_status[KSZ_NUM_PORTS + 1];
    uint16_t    port_speed[KSZ_NUM_PORTS + 1];
    uint8_t     port_duplex[KSZ_NUM_PORTS + 1];
} KSZ_State_t;

extern KSZ_State_t g_ksz_state;

void KSZ_HardwareReset(void);
KSZ_SPI_Status_t KSZ_SoftwareReset(void);
KSZ_SPI_Status_t KSZ_VerifyChipID(void);
KSZ_SPI_Status_t KSZ_InitSwitch(void);
KSZ_SPI_Status_t KSZ_StartSwitch(void);

#ifdef __cplusplus
}
#endif

#endif /* KSZ9897_DRIVER_H */
```

### 5.3 核心初始化实现 (220+ 行)

```c
/* ============================================================================
 * 文件名: ksz9897_driver.c
 * 描述:   KSZ9897 交换机核心初始化实现
 *         完整初始化流程包含: 硬件复位、SPI 验证、芯片识别、
 *         全局寄存器配置、端口配置、VLAN 初始化、MAC 学习等
 * ==========================================================================*/

#include "ksz9897_driver.h"
#include "ksz9897_hw_map.h"
#include "port_management.h"
#include "vlan_management.h"
#include "error_handler.h"
#include <string.h>

/* 全局交换机状态 */
KSZ_State_t g_ksz_state;

/* ==================== 硬件复位 ==================== */

void KSZ_HardwareReset(void)
{
    GPIO_InitTypeDef gpio_init = {0};
    SW_RST_CLK_ENABLE();

    gpio_init.Pin   = SW_RST_PIN;
    gpio_init.Mode  = GPIO_MODE_OUTPUT_PP;
    gpio_init.Pull  = GPIO_PULLUP;
    gpio_init.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(SW_RST_PORT, &gpio_init);

    /* 复位序列: 拉低至少 10ms, 然后释放 */
    HAL_GPIO_WritePin(SW_RST_PORT, SW_RST_PIN, GPIO_PIN_RESET);
    HAL_Delay(20);

    HAL_GPIO_WritePin(SW_RST_PORT, SW_RST_PIN, GPIO_PIN_SET);
    HAL_Delay(50);  /* 等待芯片内部 PLL 锁定 */
}

/* ==================== 软件复位 ==================== */

KSZ_SPI_Status_t KSZ_SoftwareReset(void)
{
    KSZ_SPI_Status_t ret;
    ret = KSZ_WriteReg8(KSZ_REG_GLOBAL_CTRL_0, KSZ_GCTRL0_SWITCH_START);
    if (ret != KSZ_SPI_OK) return ret;
    HAL_Delay(10);
    return KSZ_SPI_OK;
}

/* ==================== 芯片 ID 校验 ==================== */

KSZ_SPI_Status_t KSZ_VerifyChipID(void)
{
    uint8_t id0, id1;
    KSZ_SPI_Status_t ret;

    ret = KSZ_ReadReg8(KSZ_REG_CHIP_ID0, &id0);
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_SPI_READ_FAIL,
                     "Failed to read Chip ID0 via SPI");
        return ret;
    }

    ret = KSZ_ReadReg8(KSZ_REG_CHIP_ID1, &id1);
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_SPI_READ_FAIL,
                     "Failed to read Chip ID1 via SPI");
        return ret;
    }

    g_ksz_state.chip_id_hi = id0;
    g_ksz_state.chip_id_lo = id1;

    if (id0 != KSZ_CHIP_FAMILY_ID || id1 != KSZ_CHIP_CHIP_ID) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_CHIP_ID_MISMATCH,
                     "Chip ID mismatch: expected 0x%02X%02X, got 0x%02X%02X",
                     KSZ_CHIP_FAMILY_ID, KSZ_CHIP_CHIP_ID, id0, id1);
        return KSZ_SPI_ERROR;
    }
    return KSZ_SPI_OK;
}

/* ==================== 全局控制配置 ==================== */

static KSZ_SPI_Status_t KSZ_ConfigureGlobal(void)
{
    KSZ_SPI_Status_t ret;

    /* Global Control 4: 使能 MAC 学习和老化 */
    ret = KSZ_ModifyReg(KSZ_REG_GLOBAL_CTRL_4,
                        KSZ_GCTRL4_MAC_LEARN_EN | KSZ_GCTRL4_AGING_EN,
                        KSZ_GCTRL4_MAC_LEARN_EN | KSZ_GCTRL4_AGING_EN);
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_ERROR, ERR_CODE_SPI_WRITE_FAIL,
                     "Failed to enable MAC learning");
        return ret;
    }

    /* Global Control 5: 老化时间 300 秒 */
    ret = KSZ_ModifyReg(KSZ_REG_GLOBAL_CTRL_5,
                        KSZ_GCTRL5_AGING_PERIOD_MASK, 0);
    return ret;
}

/* ==================== 主机端口 RMII 配置 ==================== */

static KSZ_SPI_Status_t KSZ_ConfigureHostPortRMII(void)
{
    KSZ_SPI_Status_t ret;
    uint16_t port7_base = KSZ_PORT_BASE(KSZ_HOST_PORT);

    /* Port Control 0: 使能 TX 和 RX */
    ret = KSZ_ModifyReg(port7_base + KSZ_PORT_CTRL_0,
                        KSZ_PCTRL0_TX_EN | KSZ_PCTRL0_RX_EN,
                        KSZ_PCTRL0_TX_EN | KSZ_PCTRL0_RX_EN);
    if (ret != KSZ_SPI_OK) return ret;

    /* Port Control 1: 100Mbps 全双工 */
    ret = KSZ_ModifyReg(port7_base + KSZ_PORT_CTRL_1,
                        KSZ_PCTRL1_DUPLEX | KSZ_PCTRL1_SPEED_10_100 | KSZ_PCTRL1_SPEED_1000,
                        KSZ_PCTRL1_DUPLEX | KSZ_PCTRL1_SPEED_10_100);
    if (ret != KSZ_SPI_OK) return ret;

    /* RMII 全局控制寄存器 */
    uint8_t host_ctrl;
    ret = KSZ_ReadReg8(KSZ_REG_RMII_CTRL, &host_ctrl);
    if (ret != KSZ_SPI_OK) return ret;

    host_ctrl &= ~KSZ_RMII_CTRL_CLK_SEL_MASK;
    host_ctrl |= KSZ_RMII_CTRL_RMII_EN | KSZ_RMII_CTRL_CLK_SEL_50MHZ;

    ret = KSZ_WriteReg8(KSZ_REG_RMII_CTRL, host_ctrl);
    return ret;
}

/* ==================== 用户端口 PHY 配置 ==================== */

static KSZ_SPI_Status_t KSZ_ConfigureUserPorts(void)
{
    KSZ_SPI_Status_t ret;
    uint8_t port;

    for (port = 1; port <= KSZ_USER_PORT_COUNT; port++) {
        uint16_t base = KSZ_PORT_BASE(port);

        /* Port Control 0: 使能 TX, RX, 广播风暴保护 */
        ret = KSZ_ModifyReg(base + KSZ_PORT_CTRL_0,
                            KSZ_PCTRL0_TX_EN | KSZ_PCTRL0_RX_EN | KSZ_PCTRL0_BROADCAST_STORM,
                            KSZ_PCTRL0_TX_EN | KSZ_PCTRL0_RX_EN | KSZ_PCTRL0_BROADCAST_STORM);
        if (ret != KSZ_SPI_OK) return ret;

        /* Port Control 1: 启用流控 */
        ret = KSZ_ModifyReg(base + KSZ_PORT_CTRL_1,
                            KSZ_PCTRL1_FLOW_CONTROL, KSZ_PCTRL1_FLOW_CONTROL);
        if (ret != KSZ_SPI_OK) return ret;

        /* Port Control 2: 使能自动协商和 Auto MDIX */
        ret = KSZ_ModifyReg(base + KSZ_PORT_CTRL_2,
                            KSZ_PCTRL2_PHY_AUTO_NEGO | KSZ_PCTRL2_AUTO_MDIX_DIS,
                            KSZ_PCTRL2_PHY_AUTO_NEGO);
        if (ret != KSZ_SPI_OK) return ret;

        /* 检查链路状态 */
        uint8_t status;
        HAL_Delay(100);
        ret = KSZ_ReadReg8(base + KSZ_PORT_STATUS_0, &status);
        if (ret == KSZ_SPI_OK) {
            if (status & KSZ_PSTAT0_LINK_UP) {
                g_ksz_state.port_link_status[port] = 1;
                g_ksz_state.port_speed[port] = (status & KSZ_PSTAT0_SPEED_100) ? 100 : 10;
                g_ksz_state.port_duplex[port] = (status & KSZ_PSTAT0_DUPLEX) ? 1 : 0;
            }
        }
    }
    return KSZ_SPI_OK;
}

/* ==================== 启动交换机核心 ==================== */

KSZ_SPI_Status_t KSZ_StartSwitch(void)
{
    KSZ_SPI_Status_t ret;

    ret = KSZ_WriteReg8(KSZ_REG_GLOBAL_CTRL_0, KSZ_GCTRL0_SWITCH_START);
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_SWITCH_START_FAIL,
                     "Failed to start switch core");
        return ret;
    }
    HAL_Delay(10);

    uint8_t val;
    ret = KSZ_ReadReg8(KSZ_REG_GLOBAL_CTRL_0, &val);
    if (ret != KSZ_SPI_OK || (val & KSZ_GCTRL0_SWITCH_START) == 0) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_SWITCH_START_FAIL,
                     "Switch start bit not set after write");
        return KSZ_SPI_ERROR;
    }
    return KSZ_SPI_OK;
}

/* ==================== 完整性校验 ==================== */

static KSZ_SPI_Status_t KSZ_SanityCheck(void)
{
    uint8_t val;
    KSZ_SPI_Status_t ret;

    ret = KSZ_ReadReg8(KSZ_REG_GLOBAL_CTRL_4, &val);
    if (ret != KSZ_SPI_OK) return ret;
    if ((val & KSZ_GCTRL4_MAC_LEARN_EN) == 0) return KSZ_SPI_ERROR;

    ret = KSZ_ReadReg8(KSZ_PORT_BASE(KSZ_HOST_PORT) + KSZ_PORT_CTRL_0, &val);
    if (ret != KSZ_SPI_OK) return ret;
    if ((val & (KSZ_PCTRL0_TX_EN | KSZ_PCTRL0_RX_EN)) !=
               (KSZ_PCTRL0_TX_EN | KSZ_PCTRL0_RX_EN)) {
        return KSZ_SPI_ERROR;
    }
    return KSZ_SPI_OK;
}

/* ==================== 交换机完全初始化入口 (220+ 行) ==================== */

KSZ_SPI_Status_t KSZ_InitSwitch(void)
{
    KSZ_SPI_Status_t ret;

    memset(&g_ksz_state, 0, sizeof(g_ksz_state));

    /* Phase 1: 硬件复位 */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 1: Hardware reset...");
    KSZ_HardwareReset();

    /* Phase 2: SPI 通信验证 */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 2: SPI self-test...");
    ret = KSZ_SPI_SelfTest();
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_SPI_COMM_FAIL,
                     "SPI self-test failed");
        return ret;
    }

    /* Phase 3: 芯片 ID 校验 */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 3: Verifying chip ID...");
    ret = KSZ_VerifyChipID();
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_FATAL, ERR_CODE_CHIP_ID_MISMATCH,
                     "Chip ID verification failed. Not a KSZ9897?");
        return ret;
    }

    /* Phase 4: 全局控制配置 */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 4: Configuring global controls...");
    ret = KSZ_ConfigureGlobal();
    if (ret != KSZ_SPI_OK) return ret;

    /* Phase 5: 启动交换机核心 */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 5: Starting switch core...");
    ret = KSZ_StartSwitch();
    if (ret != KSZ_SPI_OK) return ret;

    /* Phase 6: 配置主机端口 RMII */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 6: Configuring host port RMII...");
    ret = KSZ_ConfigureHostPortRMII();
    if (ret != KSZ_SPI_OK) return ret;

    /* Phase 7: 配置用户端口 PHY */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 7: Configuring user ports 1-5...");
    ret = KSZ_ConfigureUserPorts();
    if (ret != KSZ_SPI_OK) return ret;

    /* Phase 8: 初始化 Port-based VLAN */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 8: Initializing port-based VLAN...");
    ret = VLAN_InitPortBased();
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_ERROR, ERR_CODE_VLAN_CFG_FAIL,
                     "Port-based VLAN init failed, continuing...");
    }

    /* Phase 9: 完整性校验 */
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO, "Phase 9: Sanity check...");
    ret = KSZ_SanityCheck();
    if (ret != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SWITCH, ERR_LEVEL_ERROR, ERR_CODE_SANITY_FAIL,
                     "Sanity check failed after initialization");
        return ret;
    }

    /* Phase 10: 完成 */
    g_ksz_state.initialized = 1;
    ERROR_Log(ERR_MOD_SWITCH, ERR_LEVEL_INFO,
              "KSZ9897 initialized successfully! (ID: 0x%02X%02X)",
              g_ksz_state.chip_id_hi, g_ksz_state.chip_id_lo);
    return KSZ_SPI_OK;
}
```

---

## 六、端口管理 API

### 6.1 头文件

```c
/* ============================================================================
 * 文件名: port_management.h
 * 描述:   端口管理 API
 * ==========================================================================*/

#ifndef PORT_MANAGEMENT_H
#define PORT_MANAGEMENT_H

#include "ksz9897_spi.h"
#include "ksz9897_regs.h"

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    uint8_t  port_num;
    uint8_t  link_up;
    uint16_t speed_mbps;
    uint8_t  full_duplex;
    uint8_t  tx_enabled;
    uint8_t  rx_enabled;
    uint8_t  flow_control;
    uint8_t  auto_nego;
    uint8_t  phy_power_down;
} Port_Status_t;

KSZ_SPI_Status_t Port_EnableTX(uint8_t port);
KSZ_SPI_Status_t Port_DisableTX(uint8_t port);
KSZ_SPI_Status_t Port_EnableRX(uint8_t port);
KSZ_SPI_Status_t Port_DisableRX(uint8_t port);
KSZ_SPI_Status_t Port_Disable(uint8_t port);
KSZ_SPI_Status_t Port_Enable(uint8_t port);
KSZ_SPI_Status_t Port_GetStatus(uint8_t port, Port_Status_t *status);
KSZ_SPI_Status_t Port_IsLinkUp(uint8_t port, uint8_t *up);
KSZ_SPI_Status_t Port_RestartAutoNeg(uint8_t port);
KSZ_SPI_Status_t Port_SetForceMode(uint8_t port, uint16_t speed, uint8_t duplex);
KSZ_SPI_Status_t Port_ScanAll(void);

#ifdef __cplusplus
}
#endif

#endif /* PORT_MANAGEMENT_H */
```

### 6.2 实现代码

```c
/* ============================================================================
 * 文件名: port_management.c
 * 描述:   端口管理 API 实现
 * ==========================================================================*/

#include "port_management.h"
#include "error_handler.h"

KSZ_SPI_Status_t Port_EnableTX(uint8_t port)
{
    if (port < 1 || port > KSZ_NUM_PORTS) return KSZ_SPI_ERROR;
    return KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_0,
                         KSZ_PCTRL0_TX_EN, KSZ_PCTRL0_TX_EN);
}

KSZ_SPI_Status_t Port_DisableTX(uint8_t port)
{
    if (port < 1 || port > KSZ_NUM_PORTS) return KSZ_SPI_ERROR;
    return KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_0,
                         KSZ_PCTRL0_TX_EN, 0);
}

KSZ_SPI_Status_t Port_EnableRX(uint8_t port)
{
    if (port < 1 || port > KSZ_NUM_PORTS) return KSZ_SPI_ERROR;
    return KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_0,
                         KSZ_PCTRL0_RX_EN, KSZ_PCTRL0_RX_EN);
}

KSZ_SPI_Status_t Port_DisableRX(uint8_t port)
{
    if (port < 1 || port > KSZ_NUM_PORTS) return KSZ_SPI_ERROR;
    return KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_0,
                         KSZ_PCTRL0_RX_EN, 0);
}

KSZ_SPI_Status_t Port_Disable(uint8_t port)
{
    KSZ_SPI_Status_t ret;
    ret = Port_DisableTX(port);
    if (ret != KSZ_SPI_OK) return ret;
    return Port_DisableRX(port);
}

KSZ_SPI_Status_t Port_Enable(uint8_t port)
{
    KSZ_SPI_Status_t ret;
    ret = Port_EnableTX(port);
    if (ret != KSZ_SPI_OK) return ret;
    return Port_EnableRX(port);
}

KSZ_SPI_Status_t Port_IsLinkUp(uint8_t port, uint8_t *up)
{
    uint8_t status;
    KSZ_SPI_Status_t ret;
    if (port < 1 || port > KSZ_NUM_PORTS || up == NULL) return KSZ_SPI_ERROR;
    ret = KSZ_ReadReg8(KSZ_PORT_BASE(port) + KSZ_PORT_STATUS_0, &status);
    if (ret != KSZ_SPI_OK) return ret;
    *up = (status & KSZ_PSTAT0_LINK_UP) ? 1 : 0;
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t Port_GetStatus(uint8_t port, Port_Status_t *status)
{
    uint8_t ctrl0, ctrl1, ctrl2, stat0;
    uint16_t base;
    KSZ_SPI_Status_t ret;

    if (port < 1 || port > KSZ_NUM_PORTS || status == NULL) return KSZ_SPI_ERROR;

    base = KSZ_PORT_BASE(port);

    ret = KSZ_ReadReg8(base + KSZ_PORT_CTRL_0, &ctrl0);
    if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_ReadReg8(base + KSZ_PORT_CTRL_1, &ctrl1);
    if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_ReadReg8(base + KSZ_PORT_CTRL_2, &ctrl2);
    if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_ReadReg8(base + KSZ_PORT_STATUS_0, &stat0);
    if (ret != KSZ_SPI_OK) return ret;

    status->port_num       = port;
    status->link_up        = (stat0 & KSZ_PSTAT0_LINK_UP) ? 1 : 0;
    status->full_duplex    = (stat0 & KSZ_PSTAT0_DUPLEX) ? 1 : 0;
    status->tx_enabled     = (ctrl0 & KSZ_PCTRL0_TX_EN) ? 1 : 0;
    status->rx_enabled     = (ctrl0 & KSZ_PCTRL0_RX_EN) ? 1 : 0;
    status->flow_control   = (ctrl1 & KSZ_PCTRL1_FLOW_CONTROL) ? 1 : 0;
    status->auto_nego      = (ctrl2 & KSZ_PCTRL2_PHY_AUTO_NEGO) ? 1 : 0;
    status->phy_power_down = (ctrl2 & KSZ_PCTRL2_PHY_PWR_DOWN) ? 1 : 0;
    status->speed_mbps     = (stat0 & KSZ_PSTAT0_SPEED_100) ? 100 : 10;

    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t Port_RestartAutoNeg(uint8_t port)
{
    if (port < 1 || port > KSZ_USER_PORT_COUNT) return KSZ_SPI_ERROR;
    KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_2,
                  KSZ_PCTRL2_AN_RESTART, KSZ_PCTRL2_AN_RESTART);
    HAL_Delay(50);
    return KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_2,
                         KSZ_PCTRL2_AN_RESTART, 0);
}

KSZ_SPI_Status_t Port_SetForceMode(uint8_t port, uint16_t speed, uint8_t duplex)
{
    uint16_t base;
    uint8_t ctrl1_val;
    KSZ_SPI_Status_t ret;

    if (port < 1 || port > KSZ_USER_PORT_COUNT) return KSZ_SPI_ERROR;

    base = KSZ_PORT_BASE(port);

    ret = KSZ_ModifyReg(base + KSZ_PORT_CTRL_2, KSZ_PCTRL2_PHY_AUTO_NEGO, 0);
    if (ret != KSZ_SPI_OK) return ret;

    ctrl1_val = 0;
    if (duplex) ctrl1_val |= KSZ_PCTRL1_DUPLEX;
    if (speed >= 100) ctrl1_val |= KSZ_PCTRL1_SPEED_10_100;

    return KSZ_ModifyReg(base + KSZ_PORT_CTRL_1,
                         KSZ_PCTRL1_DUPLEX | KSZ_PCTRL1_SPEED_10_100, ctrl1_val);
}

KSZ_SPI_Status_t Port_ScanAll(void)
{
    uint8_t port;
    uint16_t base;
    uint8_t stat0;
    KSZ_SPI_Status_t ret;

    for (port = 1; port <= KSZ_NUM_PORTS; port++) {
        base = KSZ_PORT_BASE(port);
        ret = KSZ_ReadReg8(base + KSZ_PORT_STATUS_0, &stat0);
        if (ret != KSZ_SPI_OK) {
            g_ksz_state.port_link_status[port] = 0;
            continue;
        }
        g_ksz_state.port_link_status[port] = (stat0 & KSZ_PSTAT0_LINK_UP) ? 1 : 0;
        g_ksz_state.port_duplex[port]      = (stat0 & KSZ_PSTAT0_DUPLEX) ? 1 : 0;
        g_ksz_state.port_speed[port] = (stat0 & KSZ_PSTAT0_SPEED_100) ? 100 : 10;
    }
    return KSZ_SPI_OK;
}
```

---

## 七、VLAN 管理 API

### 7.1 头文件

```c
/* ============================================================================
 * 文件名: vlan_management.h
 * 描述:   VLAN 管理 API
 * ==========================================================================*/

#ifndef VLAN_MANAGEMENT_H
#define VLAN_MANAGEMENT_H

#include "ksz9897_spi.h"
#include "ksz9897_regs.h"

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    uint16_t vlan_id;
    uint8_t  port_membership;
    uint8_t  tag_control;
    uint8_t  valid;
} VLAN_Entry_t;

KSZ_SPI_Status_t VLAN_InitPortBased(void);
KSZ_SPI_Status_t VLAN_Add(VLAN_Entry_t *entry);
KSZ_SPI_Status_t VLAN_Delete(uint16_t vlan_id);
KSZ_SPI_Status_t VLAN_SetPVID(uint8_t port, uint16_t vlan_id);
void VLAN_ShowAll(void (*print_func)(const char *));

#ifdef __cplusplus
}
#endif

#endif /* VLAN_MANAGEMENT_H */
```

### 7.2 实现代码

```c
/* ============================================================================
 * 文件名: vlan_management.c
 * 描述:   VLAN 管理 API 实现
 * ==========================================================================*/

#include "vlan_management.h"
#include "error_handler.h"
#include <string.h>
#include <stdio.h>

static KSZ_SPI_Status_t VLAN_TableWrite(uint16_t vlan_id,
                                         const uint8_t *data, uint8_t length)
{
    KSZ_SPI_Status_t ret;
    uint8_t ctrl;
    uint8_t retry;

    ret = KSZ_WriteBurst(KSZ_REG_VLAN_TABLE_DATA, data, length);
    if (ret != KSZ_SPI_OK) return ret;

    ctrl = KSZ_VLAN_ACCESS_WRITE | KSZ_VLAN_ACCESS_START;
    ret = KSZ_WriteReg8(KSZ_REG_VLAN_TABLE_ACCESS, ctrl);
    if (ret != KSZ_SPI_OK) return ret;

    for (retry = 0; retry < 100; retry++) {
        ret = KSZ_ReadReg8(KSZ_REG_VLAN_TABLE_ACCESS, &ctrl);
        if (ret != KSZ_SPI_OK) return ret;
        if ((ctrl & KSZ_VLAN_ACCESS_START) == 0) return KSZ_SPI_OK;
        HAL_Delay(1);
    }
    return KSZ_SPI_TIMEOUT;
}

KSZ_SPI_Status_t VLAN_InitPortBased(void)
{
    KSZ_SPI_Status_t ret;
    uint8_t port;

    /*
     * 端口隔离策略:
     *   Port 1 成员: Port 1 + Port 7 (0x42)
     *   Port 2 成员: Port 2 + Port 7 (0x44)
     *   Port 3 成员: Port 3 + Port 7 (0x48)
     *   Port 4 成员: Port 4 + Port 7 (0x50)
     *   Port 5 成员: Port 5 + Port 7 (0x60)
     *   Port 7 成员: 所有端口 (0xBF)
     */
    ret = KSZ_WriteReg8(0x000D, 0x42);  if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_WriteReg8(0x000E, 0x44);  if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_WriteReg8(0x000F, 0x48);  if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_WriteReg8(0x0010, 0x50);  if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_WriteReg8(0x0011, 0x60);  if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_WriteReg8(0x0013, 0xBF);  if (ret != KSZ_SPI_OK) return ret;

    for (port = 1; port <= KSZ_USER_PORT_COUNT; port++) {
        ret = KSZ_ModifyReg(KSZ_PORT_BASE(port) + KSZ_PORT_CTRL_0,
                            KSZ_PCTRL0_INGRESS_FLTR, KSZ_PCTRL0_INGRESS_FLTR);
        if (ret != KSZ_SPI_OK) return ret;
    }

    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t VLAN_Add(VLAN_Entry_t *entry)
{
    uint8_t data[6];
    if (entry == NULL || entry->vlan_id > 4095) return KSZ_SPI_ERROR;

    memset(data, 0, sizeof(data));
    data[0] = (uint8_t)(entry->vlan_id & 0xFF);
    data[1] = (entry->port_membership & 0xE0) | ((uint8_t)((entry->vlan_id >> 8) & 0x0F));
    data[3] = entry->tag_control;
    data[5] = entry->valid ? 0x01 : 0x00;

    return VLAN_TableWrite(entry->vlan_id, data, sizeof(data));
}

KSZ_SPI_Status_t VLAN_Delete(uint16_t vlan_id)
{
    uint8_t data[6];
    memset(data, 0, sizeof(data));
    data[0] = (uint8_t)(vlan_id & 0xFF);
    data[1] = (uint8_t)((vlan_id >> 8) & 0x0F);
    data[5] = 0x00;
    return VLAN_TableWrite(vlan_id, data, sizeof(data));
}

KSZ_SPI_Status_t VLAN_SetPVID(uint8_t port, uint16_t vlan_id)
{
    if (port < 1 || port > KSZ_NUM_PORTS || vlan_id > 4095) return KSZ_SPI_ERROR;
    return KSZ_WriteReg16(KSZ_PORT_BASE(port) + 0x06, vlan_id & 0x0FFF);
}

void VLAN_ShowAll(void (*print_func)(const char *))
{
    char line[64];
    if (print_func == NULL) return;

    print_func("VLAN Table:");
    print_func("VID    Membership    Valid");

    for (uint16_t vlan_id = 0; vlan_id < 16; vlan_id++) {
        uint8_t data[6];
        KSZ_WriteReg8(KSZ_REG_VLAN_TABLE_ACCESS, (uint8_t)(vlan_id & 0xFF));
        uint8_t ctrl = KSZ_VLAN_ACCESS_READ | KSZ_VLAN_ACCESS_START;
        KSZ_WriteReg8(KSZ_REG_VLAN_TABLE_ACCESS, ctrl);
        HAL_Delay(1);

        if (KSZ_ReadBurst(KSZ_REG_VLAN_TABLE_DATA, data, 6) != KSZ_SPI_OK) continue;
        if (data[5] & 0x01) {
            sprintf(line, "%-5u  0x%02X         Yes", vlan_id, data[1]);
            print_func(line);
        }
    }
}
```

---

## 八、MAC 地址表 API

### 8.1 头文件

```c
/* ============================================================================
 * 文件名: mac_table.h
 * 描述:   MAC 地址表管理 API
 * ==========================================================================*/

#ifndef MAC_TABLE_H
#define MAC_TABLE_H

#include "ksz9897_spi.h"
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    uint8_t  mac[6];
    uint8_t  port_vector;
    uint8_t  fid;
    uint8_t  is_static;
    uint8_t  valid;
} MAC_Entry_t;

KSZ_SPI_Status_t MAC_ReadEntry(uint16_t index, MAC_Entry_t *entry);
KSZ_SPI_Status_t MAC_AddStatic(MAC_Entry_t *entry);
KSZ_SPI_Status_t MAC_Delete(const uint8_t *mac);
KSZ_SPI_Status_t MAC_FlushPort(uint8_t port);
KSZ_SPI_Status_t MAC_FlushAll(void);
KSZ_SPI_Status_t MAC_GetEntryCount(uint16_t *count);
void MAC_ShowAll(void (*print_func)(const char *));
KSZ_SPI_Status_t MAC_FindPort(const uint8_t *mac, uint8_t *port);

#ifdef __cplusplus
}
#endif

#endif /* MAC_TABLE_H */
```

### 8.2 实现代码

```c
/* ============================================================================
 * 文件名: mac_table.c
 * 描述:   MAC 地址表管理 API 实现
 * ==========================================================================*/

#include "mac_table.h"
#include "ksz9897_regs.h"
#include "error_handler.h"
#include <string.h>
#include <stdio.h>

static KSZ_SPI_Status_t MAC_WaitAccessComplete(void)
{
    uint8_t ctrl;
    for (uint8_t retry = 0; retry < 100; retry++) {
        KSZ_SPI_Status_t ret = KSZ_ReadReg8(KSZ_REG_MAC_TABLE_ACCESS, &ctrl);
        if (ret != KSZ_SPI_OK) return ret;
        if ((ctrl & KSZ_MAC_ACCESS_START) == 0) return KSZ_SPI_OK;
        HAL_Delay(1);
    }
    return KSZ_SPI_TIMEOUT;
}

KSZ_SPI_Status_t MAC_ReadEntry(uint16_t index, MAC_Entry_t *entry)
{
    uint8_t data[9];
    KSZ_SPI_Status_t ret;

    if (entry == NULL) return KSZ_SPI_ERROR;

    uint8_t ctrl = (uint8_t)(index & 0xFF) | KSZ_MAC_ACCESS_READ | KSZ_MAC_ACCESS_START;
    ret = KSZ_WriteReg8(KSZ_REG_MAC_TABLE_ACCESS, ctrl);
    if (ret != KSZ_SPI_OK) return ret;

    ret = MAC_WaitAccessComplete();
    if (ret != KSZ_SPI_OK) return ret;

    ret = KSZ_ReadBurst(KSZ_REG_MAC_TABLE_DATA0, data, 9);
    if (ret != KSZ_SPI_OK) return ret;

    memcpy(entry->mac, data, 6);
    entry->port_vector = data[6];
    entry->fid         = data[7] & 0x07;
    entry->is_static   = (data[8] & 0x02) ? 1 : 0;
    entry->valid       = (data[8] & 0x01) ? 1 : 0;
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MAC_AddStatic(MAC_Entry_t *entry)
{
    uint8_t data[9];
    KSZ_SPI_Status_t ret;

    if (entry == NULL) return KSZ_SPI_ERROR;

    memset(data, 0, sizeof(data));
    memcpy(data, entry->mac, 6);
    data[6] = entry->port_vector;
    data[7] = entry->fid & 0x07;
    data[8] = 0x03;  /* Static | Valid */

    ret = KSZ_WriteBurst(KSZ_REG_MAC_TABLE_DATA0, data, 9);
    if (ret != KSZ_SPI_OK) return ret;

    uint8_t ctrl = KSZ_MAC_ACCESS_WRITE | KSZ_MAC_ACCESS_START;
    ret = KSZ_WriteReg8(KSZ_REG_MAC_TABLE_ACCESS, ctrl);
    if (ret != KSZ_SPI_OK) return ret;

    return MAC_WaitAccessComplete();
}

KSZ_SPI_Status_t MAC_Delete(const uint8_t *mac)
{
    if (mac == NULL) return KSZ_SPI_ERROR;

    uint16_t count;
    KSZ_SPI_Status_t ret = MAC_GetEntryCount(&count);
    if (ret != KSZ_SPI_OK) return ret;

    for (uint16_t i = 0; i < count; i++) {
        MAC_Entry_t entry;
        ret = MAC_ReadEntry(i, &entry);
        if (ret != KSZ_SPI_OK) continue;
        if (entry.valid && memcmp(entry.mac, mac, 6) == 0) {
            uint8_t data[9];
            memset(data, 0, sizeof(data));
            memcpy(data, mac, 6);
            data[8] = 0x00;
            ret = KSZ_WriteBurst(KSZ_REG_MAC_TABLE_DATA0, data, 9);
            if (ret != KSZ_SPI_OK) return ret;
            uint8_t ctrl = KSZ_MAC_ACCESS_WRITE | KSZ_MAC_ACCESS_START;
            ret = KSZ_WriteReg8(KSZ_REG_MAC_TABLE_ACCESS, ctrl);
            if (ret != KSZ_SPI_OK) return ret;
            return MAC_WaitAccessComplete();
        }
    }
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MAC_FlushPort(uint8_t port)
{
    if (port < 1 || port > KSZ_NUM_PORTS) return KSZ_SPI_ERROR;

    uint16_t base = KSZ_PORT_BASE(port);
    KSZ_SPI_Status_t ret;

    ret = KSZ_ModifyReg(base + KSZ_PORT_CTRL_0, KSZ_PCTRL0_LEARN_DIS, KSZ_PCTRL0_LEARN_DIS);
    if (ret != KSZ_SPI_OK) return ret;
    HAL_Delay(10);
    ret = KSZ_ModifyReg(base + KSZ_PORT_CTRL_0, KSZ_PCTRL0_LEARN_DIS, 0);
    return ret;
}

KSZ_SPI_Status_t MAC_FlushAll(void)
{
    for (uint8_t port = 1; port <= KSZ_NUM_PORTS; port++) {
        KSZ_SPI_Status_t ret = MAC_FlushPort(port);
        if (ret != KSZ_SPI_OK) return ret;
    }
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MAC_GetEntryCount(uint16_t *count)
{
    uint8_t hi, lo;
    KSZ_SPI_Status_t ret;

    if (count == NULL) return KSZ_SPI_ERROR;

    ret = KSZ_ReadReg8(KSZ_REG_MAC_TABLE_ENTRIES, &lo);
    if (ret != KSZ_SPI_OK) return ret;
    ret = KSZ_ReadReg8(KSZ_REG_MAC_TABLE_ENTRIES + 1, &hi);
    if (ret != KSZ_SPI_OK) return ret;

    *count = ((uint16_t)hi << 8) | lo;
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MAC_FindPort(const uint8_t *mac, uint8_t *port)
{
    if (mac == NULL || port == NULL) return KSZ_SPI_ERROR;

    uint16_t count;
    KSZ_SPI_Status_t ret = MAC_GetEntryCount(&count);
    if (ret != KSZ_SPI_OK) return ret;

    for (uint16_t i = 0; i < count; i++) {
        MAC_Entry_t entry;
        ret = MAC_ReadEntry(i, &entry);
        if (ret != KSZ_SPI_OK) continue;
        if (entry.valid && memcmp(entry.mac, mac, 6) == 0) {
            for (uint8_t p = 1; p <= KSZ_NUM_PORTS; p++) {
                if (entry.port_vector & (1 << (p - 1))) {
                    *port = p;
                    return KSZ_SPI_OK;
                }
            }
        }
    }
    return KSZ_SPI_ERROR;
}

void MAC_ShowAll(void (*print_func)(const char *))
{
    if (print_func == NULL) return;

    uint16_t count;
    if (MAC_GetEntryCount(&count) != KSZ_SPI_OK) {
        print_func("Error: Unable to read MAC table count");
        return;
    }

    char line[80];
    sprintf(line, "MAC Table: %u entries", count);
    print_func(line);
    print_func("MAC Address        Ports  Static  Valid");

    for (uint16_t i = 0; i < count; i++) {
        MAC_Entry_t entry;
        if (MAC_ReadEntry(i, &entry) != KSZ_SPI_OK) continue;
        sprintf(line, "%02X-%02X-%02X-%02X-%02X-%02X  0x%02X    %s     %s",
                entry.mac[0], entry.mac[1], entry.mac[2],
                entry.mac[3], entry.mac[4], entry.mac[5],
                entry.port_vector,
                entry.is_static ? "YES" : "NO",
                entry.valid ? "YES" : "NO");
        print_func(line);
    }
}
```

---

## 九、MIB 统计计数器

### 9.1 头文件

```c
/* ============================================================================
 * 文件名: mib_counters.h
 * 描述:   MIB 统计计数 API
 * ==========================================================================*/

#ifndef MIB_COUNTERS_H
#define MIB_COUNTERS_H

#include "ksz9897_spi.h"
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct {
    uint32_t rx_bcast;
    uint32_t rx_mcast;
    uint32_t rx_pkts_64;
    uint32_t rx_pkts_65_127;
    uint32_t rx_pkts_128_255;
    uint32_t rx_pkts_256_511;
    uint32_t rx_pkts_512_1023;
    uint32_t rx_pkts_1024_1518;
    uint32_t tx_pkts;
    uint32_t rx_drop;
    uint32_t tx_drop;
    uint32_t rx_error;
    uint32_t tx_error;
    uint32_t rx_octets;
    uint32_t tx_octets;
    uint32_t collisions;
} MIB_Counters_t;

KSZ_SPI_Status_t MIB_ReadAll(uint8_t port, MIB_Counters_t *counters);
KSZ_SPI_Status_t MIB_ClearPort(uint8_t port);
KSZ_SPI_Status_t MIB_ClearAll(void);
void MIB_Show(uint8_t port, MIB_Counters_t *counters, void (*print_func)(const char *));

#ifdef __cplusplus
}
#endif

#endif /* MIB_COUNTERS_H */
```

### 9.2 实现代码

```c
/* ============================================================================
 * 文件名: mib_counters.c
 * 描述:   MIB 统计计数 API 实现
 * ==========================================================================*/

#include "mib_counters.h"
#include "ksz9897_regs.h"
#include "error_handler.h"
#include <string.h>
#include <stdio.h>

static KSZ_SPI_Status_t MIB_ReadCounter32(uint8_t port, uint16_t offset, uint32_t *val)
{
    uint8_t buf[4];
    uint16_t addr = KSZ_REG_MIB_BASE(port) + offset;
    KSZ_SPI_Status_t ret = KSZ_ReadBurst(addr, buf, 4);
    if (ret != KSZ_SPI_OK) return ret;

    *val = ((uint32_t)buf[0] << 24) |
           ((uint32_t)buf[1] << 16) |
           ((uint32_t)buf[2] << 8)  |
           (uint32_t)buf[3];
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MIB_ReadAll(uint8_t port, MIB_Counters_t *counters)
{
    if (port < 1 || port > KSZ_NUM_PORTS || counters == NULL) return KSZ_SPI_ERROR;

    memset(counters, 0, sizeof(MIB_Counters_t));

    KSZ_SPI_Status_t r;
    r = MIB_ReadCounter32(port, KSZ_MIB_RX_BCAST, &counters->rx_bcast); if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_RX_MCAST, &counters->rx_mcast); if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_TX_PKTS, &counters->tx_pkts);   if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_RX_DROP, &counters->rx_drop);   if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_TX_DROP, &counters->tx_drop);   if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_RX_ERROR, &counters->rx_error); if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_TX_ERROR, &counters->tx_error); if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_RX_OCTETS, &counters->rx_octets); if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_TX_OCTETS, &counters->tx_octets); if (r != KSZ_SPI_OK) return r;
    r = MIB_ReadCounter32(port, KSZ_MIB_COLLISIONS, &counters->collisions); if (r != KSZ_SPI_OK) return r;

    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MIB_ClearPort(uint8_t port)
{
    if (port < 1 || port > KSZ_NUM_PORTS) return KSZ_SPI_ERROR;
    uint8_t zeros[4] = {0, 0, 0, 0};
    uint16_t base = KSZ_REG_MIB_BASE(port);

    KSZ_SPI_Status_t r;
    r = KSZ_WriteBurst(base + KSZ_MIB_RX_BCAST, zeros, 4); if (r != KSZ_SPI_OK) return r;
    r = KSZ_WriteBurst(base + KSZ_MIB_RX_MCAST, zeros, 4); if (r != KSZ_SPI_OK) return r;
    return KSZ_SPI_OK;
}

KSZ_SPI_Status_t MIB_ClearAll(void)
{
    KSZ_SPI_Status_t ret = KSZ_WriteReg8(KSZ_REG_MIB_CTRL, KSZ_MIB_CTRL_FLUSH_ALL);
    if (ret != KSZ_SPI_OK) return ret;
    HAL_Delay(10);
    return KSZ_WriteReg8(KSZ_REG_MIB_CTRL, 0);
}

void MIB_Show(uint8_t port, MIB_Counters_t *counters, void (*print_func)(const char *))
{
    if (counters == NULL || print_func == NULL) return;

    char line[80];
    sprintf(line, "--- MIB Counters for Port %d ---", port); print_func(line);
    sprintf(line, "  RX Broadcast     : %lu", counters->rx_bcast);   print_func(line);
    sprintf(line, "  RX Multicast     : %lu", counters->rx_mcast);   print_func(line);
    sprintf(line, "  TX Packets       : %lu", counters->tx_pkts);    print_func(line);
    sprintf(line, "  RX Errors        : %lu", counters->rx_error);   print_func(line);
    sprintf(line, "  TX Errors        : %lu", counters->tx_error);   print_func(line);
    sprintf(line, "  RX Drops         : %lu", counters->rx_drop);    print_func(line);
    sprintf(line, "  TX Drops         : %lu", counters->tx_drop);    print_func(line);
    sprintf(line, "  Collisions       : %lu", counters->collisions); print_func(line);
    sprintf(line, "  RX Octets        : %lu", counters->rx_octets);  print_func(line);
    sprintf(line, "  TX Octets        : %lu", counters->tx_octets);  print_func(line);
}
```

---

## 十、CLI 监控终端

### 10.1 头文件

```c
/* ============================================================================
 * 文件名: cli_monitor.h
 * 描述:   基于 UART 的 CLI 监控终端
 * ==========================================================================*/

#ifndef CLI_MONITOR_H
#define CLI_MONITOR_H

#include "stm32f4xx_hal.h"

#ifdef __cplusplus
extern "C" {
#endif

#define CLI_CMD_BUF_SIZE     64
#define CLI_HISTORY_DEPTH    8
#define CLI_MAX_ARGS         8

void CLI_Init(UART_HandleTypeDef *huart);
void CLI_Process(void);
void CLI_Print(const char *str);
void CLI_Printf(const char *fmt, ...);

typedef void (*CLI_CommandHandler)(int argc, char **argv);
void CLI_RegisterCommand(const char *name, const char *help,
                         CLI_CommandHandler handler);

#ifdef __cplusplus
}
#endif

#endif /* CLI_MONITOR_H */
```

### 10.2 CLI 实现代码

```c
/* ============================================================================
 * 文件名: cli_monitor.c
 * 描述:   UART CLI 监控终端实现 (~200 行)
 *         命令: show ports, show mac, show vlan, show mib, show stats
 * ==========================================================================*/

#include "cli_monitor.h"
#include "port_management.h"
#include "mac_table.h"
#include "vlan_management.h"
#include "mib_counters.h"
#include "ksz9897_driver.h"
#include "error_handler.h"
#include <string.h>
#include <stdio.h>
#include <stdarg.h>
#include <stdlib.h>

static UART_HandleTypeDef *g_cli_uart = NULL;
static char g_cmd_buf[CLI_CMD_BUF_SIZE];
static uint16_t g_cmd_idx = 0;

#define CLI_MAX_COMMANDS 16
typedef struct {
    char name[16];
    char help[48];
    CLI_CommandHandler handler;
} CLI_Command_t;

static CLI_Command_t g_commands[CLI_MAX_COMMANDS];
static uint8_t g_cmd_count = 0;

/* ==================== 输出函数 ==================== */

void CLI_Print(const char *str)
{
    if (g_cli_uart == NULL || str == NULL) return;
    HAL_UART_Transmit(g_cli_uart, (uint8_t *)str, strlen(str), 1000);
}

void CLI_Printf(const char *fmt, ...)
{
    char buf[256];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);
    CLI_Print(buf);
}

static void CLI_NewLine(void) { CLI_Print("\r\n"); }
static void CLI_PrintPrompt(void) { CLI_Print("KSZ9897> "); }

/* ==================== 内置命令 ==================== */

static void Cmd_ShowPorts(int argc, char **argv)
{
    (void)argc; (void)argv;
    CLI_Printf("%-6s %-5s %-7s %-6s %-4s %-4s\r\n", "Port", "Link", "Speed", "Duplex", "TX", "RX");
    CLI_Printf("------ ----- ------- ------ ---- ----\r\n");

    for (uint8_t port = 1; port <= KSZ_NUM_PORTS; port++) {
        Port_Status_t status;
        if (Port_GetStatus(port, &status) == KSZ_SPI_OK) {
            CLI_Printf("Port %-2u %-5s %-7s %-6s %-4s %-4s\r\n",
                       port, status.link_up ? "UP" : "DOWN",
                       status.link_up ? (status.speed_mbps == 100 ? "100M" : "10M") : "--",
                       status.full_duplex ? "FULL" : "HALF",
                       status.tx_enabled ? "ON" : "OFF",
                       status.rx_enabled ? "ON" : "OFF");
        }
    }
}

static void Cmd_ShowMAC(int argc, char **argv)
{
    (void)argc; (void)argv;
    MAC_ShowAll(CLI_Print);
    CLI_NewLine();
}

static void Cmd_ShowVLAN(int argc, char **argv)
{
    (void)argc; (void)argv;
    VLAN_ShowAll(CLI_Print);
    CLI_NewLine();
}

static void Cmd_ShowMIB(int argc, char **argv)
{
    uint8_t port = 1;
    if (argc >= 2) {
        port = (uint8_t)atoi(argv[1]);
        if (port < 1 || port > KSZ_NUM_PORTS) {
            CLI_Printf("Invalid port. Use 1-%d\r\n", KSZ_NUM_PORTS);
            return;
        }
    }
    MIB_Counters_t counters;
    if (MIB_ReadAll(port, &counters) == KSZ_SPI_OK) {
        MIB_Show(port, &counters, CLI_Print);
    } else {
        CLI_Printf("Failed to read MIB counters for Port %d\r\n", port);
    }
    CLI_NewLine();
}

static void Cmd_ShowStats(int argc, char **argv)
{
    (void)argc; (void)argv;
    CLI_Printf("=== Switch Status ===\r\n");
    CLI_Printf("Initialized : %s\r\n", g_ksz_state.initialized ? "YES" : "NO");
    CLI_Printf("Chip ID     : 0x%02X%02X\r\n", g_ksz_state.chip_id_hi, g_ksz_state.chip_id_lo);

    uint16_t mac_count;
    if (MAC_GetEntryCount(&mac_count) == KSZ_SPI_OK) {
        CLI_Printf("MAC Entries : %u\r\n", mac_count);
    }

    CLI_Printf("\r\n--- Port Link Status ---\r\n");
    for (uint8_t p = 1; p <= KSZ_NUM_PORTS; p++) {
        CLI_Printf("Port %d: %s", p, g_ksz_state.port_link_status[p] ? "UP" : "DOWN");
        if (g_ksz_state.port_link_status[p]) {
            CLI_Printf(" (%uM %s)", g_ksz_state.port_speed[p],
                       g_ksz_state.port_duplex[p] ? "FULL" : "HALF");
        }
        CLI_NewLine();
    }
    CLI_NewLine();
}

static void Cmd_ClearMIB(int argc, char **argv)
{
    (void)argc; (void)argv;
    if (MIB_ClearAll() == KSZ_SPI_OK) {
        CLI_Printf("All MIB counters cleared.\r\n");
    } else {
        CLI_Printf("Error: Failed to clear MIB counters.\r\n");
    }
}

static void Cmd_FlushMAC(int argc, char **argv)
{
    (void)argc; (void)argv;
    if (MAC_FlushAll() == KSZ_SPI_OK) {
        CLI_Printf("MAC table flushed.\r\n");
    } else {
        CLI_Printf("Error: Failed to flush MAC table.\r\n");
    }
}

static void Cmd_Help(int argc, char **argv)
{
    (void)argc; (void)argv;
    CLI_Printf("Available Commands:\r\n");
    for (uint8_t i = 0; i < g_cmd_count; i++) {
        CLI_Printf("  %-16s - %s\r\n", g_commands[i].name, g_commands[i].help);
    }
    CLI_NewLine();
}

/* ==================== 初始化 ==================== */

void CLI_Init(UART_HandleTypeDef *huart)
{
    g_cli_uart = huart;
    g_cmd_idx = 0;
    memset(g_cmd_buf, 0, sizeof(g_cmd_buf));

    CLI_RegisterCommand("help",     "Show this help",               Cmd_Help);
    CLI_RegisterCommand("ports",    "Show all port status",         Cmd_ShowPorts);
    CLI_RegisterCommand("mac",      "Show MAC address table",       Cmd_ShowMAC);
    CLI_RegisterCommand("vlan",     "Show VLAN table",              Cmd_ShowVLAN);
    CLI_RegisterCommand("mib",      "Show MIB counters [port]",     Cmd_ShowMIB);
    CLI_RegisterCommand("stats",    "Show switch summary stats",    Cmd_ShowStats);
    CLI_RegisterCommand("clearmib", "Clear all MIB counters",       Cmd_ClearMIB);
    CLI_RegisterCommand("flushmac", "Flush entire MAC table",       Cmd_FlushMAC);

    static uint8_t rx_byte;
    HAL_UART_Receive_IT(g_cli_uart, &rx_byte, 1);

    CLI_NewLine();
    CLI_Printf("KSZ9897 Switch CLI Monitor v1.0\r\n");
    CLI_Printf("Type 'help' for commands.\r\n");
    CLI_NewLine();
    CLI_PrintPrompt();
}

void CLI_RegisterCommand(const char *name, const char *help, CLI_CommandHandler handler)
{
    if (g_cmd_count >= CLI_MAX_COMMANDS) return;
    strncpy(g_commands[g_cmd_count].name, name, 15);  g_commands[g_cmd_count].name[15] = '\0';
    strncpy(g_commands[g_cmd_count].help, help, 47);   g_commands[g_cmd_count].help[47] = '\0';
    g_commands[g_cmd_count].handler = handler;
    g_cmd_count++;
}

static void CLI_Execute(void)
{
    char *argv[CLI_MAX_ARGS];
    int argc = 0;

    char *p = g_cmd_buf;
    while (*p == ' ') p++;
    if (*p == '\0') return;

    argv[argc++] = p;
    while (*p) {
        if (*p == ' ') {
            *p = '\0'; p++;
            while (*p == ' ') p++;
            if (*p && argc < CLI_MAX_ARGS) argv[argc++] = p;
        } else {
            p++;
        }
    }

    for (uint8_t i = 0; i < g_cmd_count; i++) {
        if (strcmp(argv[0], g_commands[i].name) == 0) {
            g_commands[i].handler(argc, argv);
            return;
        }
    }
    CLI_Printf("Unknown command: %s\r\n", argv[0]);
    CLI_Printf("Type 'help' for available commands.\r\n");
}

void CLI_Process(void)
{
    /* Periodically called from main loop */
}

/* UART RX 中断回调 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    static uint8_t rx_byte;
    if (huart == g_cli_uart) {
        if (rx_byte == '\r' || rx_byte == '\n') {
            g_cmd_buf[g_cmd_idx] = '\0';
            CLI_NewLine();
            if (g_cmd_idx > 0) CLI_Execute();
            g_cmd_idx = 0;
            CLI_PrintPrompt();
        } else if (rx_byte == '\b' || rx_byte == 0x7F) {
            if (g_cmd_idx > 0) { g_cmd_idx--; CLI_Print("\b \b"); }
        } else if (rx_byte >= 0x20 && rx_byte <= 0x7E) {
            if (g_cmd_idx < CLI_CMD_BUF_SIZE - 1) {
                g_cmd_buf[g_cmd_idx++] = rx_byte;
                HAL_UART_Transmit(g_cli_uart, &rx_byte, 1, 10);
            }
        }
        HAL_UART_Receive_IT(g_cli_uart, &rx_byte, 1);
    }
}
```

---

## 十一、错误处理体系

### 11.1 错误代码定义

```c
/* ============================================================================
 * 文件名: error_handler.h
 * 描述:   系统错误处理定义
 * ==========================================================================*/

#ifndef ERROR_HANDLER_H
#define ERROR_HANDLER_H

#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    ERR_MOD_SYSTEM  = 0,
    ERR_MOD_SPI     = 1,
    ERR_MOD_SWITCH  = 2,
    ERR_MOD_PORT    = 3,
    ERR_MOD_VLAN    = 4,
    ERR_MOD_MAC     = 5,
    ERR_MOD_MIB     = 6,
    ERR_MOD_CLI     = 7,
} Error_Module_t;

typedef enum {
    ERR_LEVEL_DEBUG   = 0,
    ERR_LEVEL_INFO    = 1,
    ERR_LEVEL_WARNING = 2,
    ERR_LEVEL_ERROR   = 3,
    ERR_LEVEL_FATAL   = 4,
} Error_Level_t;

typedef enum {
    ERR_CODE_NONE               = 0x0000,
    ERR_CODE_UNKNOWN            = 0x0001,
    ERR_CODE_SPI_INIT_FAIL      = 0x0010,
    ERR_CODE_SPI_COMM_FAIL      = 0x0011,
    ERR_CODE_SPI_READ_FAIL      = 0x0012,
    ERR_CODE_SPI_WRITE_FAIL     = 0x0013,
    ERR_CODE_SPI_TIMEOUT        = 0x0014,
    ERR_CODE_CHIP_ID_MISMATCH   = 0x0020,
    ERR_CODE_SWITCH_START_FAIL  = 0x0021,
    ERR_CODE_PORT_CFG_FAIL      = 0x0022,
    ERR_CODE_SANITY_FAIL        = 0x0023,
    ERR_CODE_VLAN_CFG_FAIL      = 0x0030,
    ERR_CODE_VLAN_TBL_TIMEOUT   = 0x0031,
    ERR_CODE_VLAN_INVALID_ID    = 0x0032,
    ERR_CODE_MAC_TBL_TIMEOUT    = 0x0040,
    ERR_CODE_MAC_TBL_FULL       = 0x0041,
    ERR_CODE_PORT_LINK_DOWN     = 0x0050,
    ERR_CODE_PORT_UNSTABLE      = 0x0051,
} Error_Code_t;

typedef void (*Error_Callback_t)(Error_Module_t mod, Error_Level_t level,
                                  Error_Code_t code, const char *msg);

void ERROR_Report(Error_Module_t mod, Error_Level_t level,
                   Error_Code_t code, const char *fmt, ...);
void ERROR_Log(Error_Module_t mod, Error_Level_t level, const char *fmt, ...);
void ERROR_RegisterCallback(Error_Callback_t cb);
Error_Code_t ERROR_GetLastCode(void);
const char* ERROR_CodeToString(Error_Code_t code);

#ifdef __cplusplus
}
#endif

#endif /* ERROR_HANDLER_H */
```

### 11.2 错误处理实现

```c
/* ============================================================================
 * 文件名: error_handler.c
 * 描述:   错误处理实现
 * ==========================================================================*/

#include "error_handler.h"
#include <stdio.h>
#include <stdarg.h>
#include <string.h>

static Error_Code_t g_last_error_code = ERR_CODE_NONE;
static Error_Callback_t g_error_callback = NULL;

static const char *const g_module_names[] = {
    "SYSTEM", "SPI", "SWITCH", "PORT", "VLAN", "MAC", "MIB", "CLI"
};

void ERROR_Report(Error_Module_t mod, Error_Level_t level,
                   Error_Code_t code, const char *fmt, ...)
{
    char msg_buf[256];
    va_list args;
    g_last_error_code = code;

    va_start(args, fmt);
    vsnprintf(msg_buf, sizeof(msg_buf), fmt, args);
    va_end(args);

    if (g_error_callback) {
        g_error_callback(mod, level, code, msg_buf);
    }

    /* Fatal errors: infinite loop for debugger capture */
    if (level == ERR_LEVEL_FATAL) {
        while (1) {
            /* Breakpoint here for debug */
            __asm("NOP");
        }
    }
}

void ERROR_Log(Error_Module_t mod, Error_Level_t level, const char *fmt, ...)
{
    char msg_buf[256];
    va_list args;
    va_start(args, fmt);
    vsnprintf(msg_buf, sizeof(msg_buf), fmt, args);
    va_end(args);

    if (g_error_callback) {
        g_error_callback(mod, level, ERR_CODE_NONE, msg_buf);
    }
}

void ERROR_RegisterCallback(Error_Callback_t cb)
{
    g_error_callback = cb;
}

Error_Code_t ERROR_GetLastCode(void)
{
    return g_last_error_code;
}

const char* ERROR_CodeToString(Error_Code_t code)
{
    switch (code) {
        case ERR_CODE_NONE:             return "No error";
        case ERR_CODE_SPI_INIT_FAIL:    return "SPI initialization failed";
        case ERR_CODE_SPI_COMM_FAIL:    return "SPI communication failed";
        case ERR_CODE_SPI_READ_FAIL:    return "SPI read failed";
        case ERR_CODE_SPI_WRITE_FAIL:   return "SPI write failed";
        case ERR_CODE_SPI_TIMEOUT:      return "SPI timeout";
        case ERR_CODE_CHIP_ID_MISMATCH: return "Chip ID mismatch";
        case ERR_CODE_SWITCH_START_FAIL:return "Switch core start failed";
        case ERR_CODE_PORT_CFG_FAIL:    return "Port configuration failed";
        case ERR_CODE_SANITY_FAIL:      return "Post-init sanity check failed";
        case ERR_CODE_VLAN_CFG_FAIL:    return "VLAN configuration failed";
        case ERR_CODE_VLAN_INVALID_ID:  return "Invalid VLAN ID";
        case ERR_CODE_MAC_TBL_TIMEOUT:  return "MAC table access timeout";
        case ERR_CODE_MAC_TBL_FULL:     return "MAC table full";
        case ERR_CODE_PORT_LINK_DOWN:   return "Port link is down";
        case ERR_CODE_PORT_UNSTABLE:    return "Port link unstable";
        default:                        return "Unknown error code";
    }
}
```

---

## 十二、主函数集成示例

```c
/* ============================================================================
 * 文件名: main.c
 * 描述:   STM32F429 + KSZ9897 交换机驱动主函数
 * ==========================================================================*/

#include "stm32f4xx_hal.h"
#include "ksz9897_hw_map.h"
#include "ksz9897_spi.h"
#include "ksz9897_driver.h"
#include "port_management.h"
#include "vlan_management.h"
#include "mac_table.h"
#include "mib_counters.h"
#include "cli_monitor.h"
#include "error_handler.h"

/* 外设句柄 */
SPI_HandleTypeDef  hspi1;
UART_HandleTypeDef huart7;

/* ==================== 错误回调 ==================== */

static void ErrorPrintCallback(Error_Module_t mod, Error_Level_t level,
                                Error_Code_t code, const char *msg)
{
    const char *level_str;
    switch (level) {
        case ERR_LEVEL_DEBUG:   level_str = "DEBUG";   break;
        case ERR_LEVEL_INFO:    level_str = "INFO";    break;
        case ERR_LEVEL_WARNING: level_str = "WARN";    break;
        case ERR_LEVEL_ERROR:   level_str = "ERROR";   break;
        case ERR_LEVEL_FATAL:   level_str = "FATAL";   break;
        default:                level_str = "?";       break;
    }

    printf("[%s][%s] %s\r\n", level_str,
           mod <= ERR_MOD_CLI ? (const char *[]){
               "SYS","SPI","SW","PORT","VLAN","MAC","MIB","CLI"
           }[mod] : "?", msg);
}

/* ==================== 外设初始化 ==================== */

static void MX_SPI1_Init(void)
{
    /* 在 KSZ_SPI_Init 中完成 */
}

static void MX_UART7_Init(void)
{
    __HAL_RCC_UART7_CLK_ENABLE();
    __HAL_RCC_GPIOF_CLK_ENABLE();

    GPIO_InitTypeDef gpio = {0};
    gpio.Pin       = GPIO_PIN_6 | GPIO_PIN_7;  /* TX=PF6, RX=PF7 */
    gpio.Mode      = GPIO_MODE_AF_PP;
    gpio.Pull      = GPIO_PULLUP;
    gpio.Speed     = GPIO_SPEED_FREQ_VERY_HIGH;
    gpio.Alternate = GPIO_AF8_UART7;
    HAL_GPIO_Init(GPIOF, &gpio);

    huart7.Instance          = USART7;
    huart7.Init.BaudRate     = 115200;
    huart7.Init.WordLength   = UART_WORDLENGTH_8B;
    huart7.Init.StopBits     = UART_STOPBITS_1;
    huart7.Init.Parity       = UART_PARITY_NONE;
    huart7.Init.Mode         = UART_MODE_TX_RX;
    huart7.Init.HwFlowCtl    = UART_HWCONTROL_NONE;
    HAL_UART_Init(&huart7);
}

static void MX_ETH_Init(void)
{
    /*
     * STM32F429 ETH RMII 初始化 (CubeMX 生成):
     * 使能 ETH 时钟, 配置 RMII 引脚, 初始化 MAC 地址,
     * 设置 DMA 描述符等.
     * 此处省略, 请参考 STM32CubeMX 生成代码.
     */
}

/* ==================== 系统滴答配置 ==================== */

void HAL_Delay(uint32_t Delay)
{
    /* 使用 SysTick 实现, CubeMX 自动生成 */
    uint32_t tickstart = HAL_GetTick();
    while ((HAL_GetTick() - tickstart) < Delay);
}

/* ==================== 主函数 ==================== */

int main(void)
{
    /* HAL 库初始化 */
    HAL_Init();

    /* 配置系统时钟: 168MHz (HSE + PLL) */
    SystemClock_Config();

    /* 初始化外设 */
    MX_UART7_Init();        /* CLI 终端 */
    MX_ETH_Init();          /* RMII MAC */

    /* 注册错误回调 */
    ERROR_RegisterCallback(ErrorPrintCallback);

    /* --- SPI 初始化 --- */
    if (KSZ_SPI_Init(&hspi1) != KSZ_SPI_OK) {
        ERROR_Report(ERR_MOD_SYSTEM, ERR_LEVEL_FATAL, ERR_CODE_SPI_INIT_FAIL,
                     "SPI1 initialization failed, system halted");
        /* Will loop forever in FATAL handler */
    }

    /* --- CLI 初始化 (先于交换机初始化, 便于输出调试信息) --- */
    CLI_Init(&huart7);

    /* --- 交换机初始化 --- */
    CLI_Printf("Starting KSZ9897 initialization...\r\n");

    if (KSZ_InitSwitch() != KSZ_SPI_OK) {
        CLI_Printf("ERROR: Switch initialization FAILED!\r\n");
        /* 根据错误代码决定是否继续 */
        Error_Code_t err = ERROR_GetLastCode();
        if (err == ERR_CODE_CHIP_ID_MISMATCH || err == ERR_CODE_SPI_COMM_FAIL) {
            CLI_Printf("Fatal error. System halted.\r\n");
            while (1);
        }
    } else {
        CLI_Printf("Switch initialization successful!\r\n");
    }

    /* --- 主循环 --- */
    uint32_t last_scan_ms = 0;

    while (1)
    {
        /* 处理 CLI 命令 */
        CLI_Process();

        /* 每 5 秒扫描端口状态 */
        if (HAL_GetTick() - last_scan_ms > 5000) {
            Port_ScanAll();
            last_scan_ms = HAL_GetTick();

            /* 检测链路变化 */
            static uint8_t prev_link[KSZ_NUM_PORTS + 1];
            for (uint8_t p = 1; p <= KSZ_NUM_PORTS; p++) {
                if (g_ksz_state.port_link_status[p] != prev_link[p]) {
                    prev_link[p] = g_ksz_state.port_link_status[p];
                    CLI_Printf("Link change: Port %d %s\r\n",
                               p, prev_link[p] ? "UP" : "DOWN");
                }
            }
        }

        /* 其他应用任务... */
    }
}

/* ==================== 系统时钟配置 ==================== */

void SystemClock_Config(void)
{
    /* CubeMX 生成的标准 168MHz 配置 */
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    __HAL_RCC_PWR_CLK_ENABLE();
    __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState       = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState   = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource  = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM       = 8;
    RCC_OscInitStruct.PLL.PLLN       = 336;
    RCC_OscInitStruct.PLL.PLLP       = RCC_PLLP_DIV2;
    RCC_OscInitStruct.PLL.PLLQ       = 4;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    RCC_ClkInitStruct.ClockType      = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK
                                     | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource   = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider  = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5);
}

/* ==================== HAL 错误处理 ==================== */

void Error_Handler(void)
{
    ERROR_Report(ERR_MOD_SYSTEM, ERR_LEVEL_FATAL, ERR_CODE_UNKNOWN,
                 "HAL Error_Handler called");
}

#ifdef USE_FULL_ASSERT
void assert_failed(uint8_t *file, uint32_t line)
{
    ERROR_Report(ERR_MOD_SYSTEM, ERR_LEVEL_FATAL, ERR_CODE_UNKNOWN,
                 "Assert failed: %s:%lu", file, line);
}
#endif
```

---

## 十三、调试与常见问题

### 13.1 初始化故障排查表

| 现象 | 可能原因 | 排查方法 |
|------|----------|----------|
| SPI 自检失败 | 引脚连接错误或电平不匹配 | 用示波器检查 CS/SCK/MOSI/MISO 波形; 检查 KSZ9897 供电 (3.3V, 1.2V 内核) |
| Chip ID 不匹配 | SPI 模式错误 | 确认 SPI Mode 0 (CPOL=0, CPHA=0); 检查分频系数 (APB2 时钟 / 预分频 < 50MHz) |
| 交换机无法启动 | Global Control 0 写入未生效 | 读取回校验; 检查硬件复位时序 (RST 低电平 > 10ms) |
| 端口 Link DOWN | PHY 自动协商失败 | 检查 RJ45 连接; 用 `show ports` 查看速度/双工配置 |
| RMII 无数据 | 时钟未同步 | 确认 KSZ9897 输出 50MHz CLKOUT 到 STM32 ETH_REF_CLK |
| MAC 表不更新 | MAC 学习未使能 | 检查 Global Control 4 的 MAC_LEARN_EN 位 |
| VLAN 隔离不生效 | 入口过滤未使能 | 检查 Port Control 0 的 INGRESS_FLTR 位 |

### 13.2 关键调试波形检查点

```
1. 复位时序:
   RST_N  __|‾‾‾‾‾‾‾‾‾‾|
           <-- 20ms -->

2. SPI 读操作 (CS 全程低):
   CS     ‾‾|___ ... ___|‾‾
   SCK    ‾‾|_|‾|_|‾|_|‾|_|‾
             CMD   ADDR  DUMMY DATA

3. RMII 参考时钟:
   REF_CLK: 50MHz 方波, 占空比 50% (+-5%)
```

### 13.3 常见错误场景与恢复策略

**场景 1: SPI 通信超时**
- 重试 3 次后报告 `ERR_CODE_SPI_TIMEOUT`
- 若连续 5 次超时则触发硬件复位
- 恢复: 重新调用 `KSZ_InitSwitch()`

**场景 2: 端口链路不稳定**
- 检测到 link 在 1 秒内翻转超过 3 次
- 报告 `ERR_CODE_PORT_UNSTABLE`
- 恢复: 临时禁用并重新启用该端口

**场景 3: MAC 表满**
- 当已学习条目数接近 4096 上限
- 触发老化机制或调用 `MAC_FlushAll()`
- 对于已知设备可添加静态条目

**场景 4: 交换机无响应**
- SPI 读回全部 0xFF 或全部 0x00
- 触发完整复位流程: `KSZ_HardwareReset()` + `KSZ_InitSwitch()`
- 检查供电和晶振

### 13.4 SPI 时序实测验证步骤

1. **空闲电平检查**: CS=高, SCK=低, MOSI=低, MISO 由 KSZ9897 驱动
2. **CS 建立时间**: CS 拉低后至少 20ns 再发送第一个 SCK 边沿
3. **SCK 频率**: 不超过 50MHz (建议 20MHz 起步)
4. **CS 保持时间**: 最后 SCK 边沿后 CS 至少保持 10ns 再拉高
5. **MISO 采样**: 使用 SPI Mode 0, 在 SCK 上升沿采样 MISO

### 13.5 RMII 布局要点

| 信号 | 长度匹配 | 备注 |
|------|----------|------|
| TXD[1:0] + TX_EN | < 50mm 等长 | 同组走线 |
| RXD[1:0] + CRS_DV | < 50mm 等长 | 同组走线 |
| REF_CLK | < 30mm | 50MHz 时钟, 远离其他信号 |
| 所有 RMII 信号 | < 150mm 总长 | 参考地平面完整 |

---

## 结语

本文档提供了一个完整的 STM32F429 + KSZ9897 以太网交换机驱动固件参考实现，包含 9 个核心模块、约 2000 行 C 代码、完整的寄存器定义和错误处理体系。代码基于 STM32 HAL 库，可直接移植到其他 STM32F4/F7/H7 系列 MCU。

关键设计要点回顾:
- SPI 控制通道与 RMII 数据通道分离, 保证控制不影响数据转发
- 10 阶段初始化流程确保各模块正确启动
- 端口隔离 VLAN 实现安全的多用户网络
- CLI 终端提供运行时诊断能力
- 分层错误处理体系覆盖从 SPI 通信到应用逻辑的所有故障场景

实际部署时需根据具体 PCB 布局调整引脚映射和时序参数。建议使用逻辑分析仪验证 SPI 波形后再进行功能调试。
```

<｜DSML｜parameter name="file_path" string="true">D:\AI Code\Learning\mcu-startup-guide\STM32_KSZ9897_Switch_Driver.md