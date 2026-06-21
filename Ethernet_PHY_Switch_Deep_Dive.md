# Ethernet PHY / Switch 驱动固件深度解读

> 从寄存器级到驱动框架 -- 逐位解析 PHY 初始化、Link 状态机、Auto-MDI/X、中断处理、Switch 转发逻辑与驱动分层设计

---

## 目录

- [一、Ethernet PHY 基础概念](#一ethernet-phy-基础概念)
- [二、PHY 寄存器标准详解](#二phy-寄存器标准详解)
- [三、PHY 驱动分层架构](#三phy-驱动分层架构)
- [四、PHY 初始化流程详解](#四phy-初始化流程详解)
- [五、Link 检测与状态机](#五link-检测与状态机)
- [六、Auto-Negotiation 再探 -- 驱动侧实现](#六auto-negotiation-再探----驱动侧实现)
- [七、Auto-MDI/X 实现](#七auto-mdix-实现)
- [八、PHY 中断处理](#八phy-中断处理)
- [九、PHY 功耗管理](#九phy-功耗管理)
- [十、Switch PHY 与独立 PHY 的区别](#十switch-phy-与独立-phy-的区别)
- [十一、Switch 驱动核心 -- 转发逻辑](#十一switch-驱动核心----转发逻辑)
- [十二、Switch VLAN 硬件实现](#十二switch-vlan-硬件实现)
- [十三、Switch ACL / 过滤规则](#十三switch-acl--过滤规则)
- [十四、驱动性能优化与调试](#十四驱动性能优化与调试)
- [十五、常见 PHY 芯片寄存器速查](#十五常见-phy-芯片寄存器速查)

---

## 一、Ethernet PHY 基础概念

### 1.1 PHY 在协议栈中的位置

```
+----------------------------------------------------------+
|                       应用层                               |
+----------------------------------------------------------+
|                        TCP/IP 协议栈                       |
+----------------------------------------------------------+
|                     MAC (媒体访问控制器)                     |
|          ┌─────────── MII/RMII/RGMII ───────────┐          |
|          ↓                                        ↓          |
|                  PHY (物理层收发器)                           |
|          ┌─────── MDI (介质相关接口) ───────┐               |
|          ↓                                   ↓               |
|           网线 / 光纤 (物理介质)                             |
+----------------------------------------------------------+
```

**关键分界**：
- **MAC 侧接口**：MII / RMII / RGMII / SGMII -- 数据通道
- **MDIO 管理接口**：读取/写入 PHY 寄存器 -- 控制通道
- **MDI 侧接口**：网线上的差分对 -- 模拟信号

### 1.2 PHY 的核心职责

| 功能 | 说明 |
|------|------|
| **PCS (Physical Coding Sublayer)** | 数据编码 (4B/5B, 8B/10B, 64B/66B) |
| **PMA (Physical Medium Attachment)** | 串行化/解串行化, 时钟恢复 |
| **PMD (Physical Medium Dependent)** | 模拟信号驱动, 线路均衡 |
| **Auto-Negotiation** | 协商速度和双工模式 |
| **Auto-MDI/X** | 自动检测/切换交叉线直通线 |
| **Link 检测** | 检测链路是否建立 |
| **远程唤醒 (WoLAN)** | 接收魔术包唤醒主机 |

### 1.3 常见 PHY 芯片选型

| 芯片 | 速率 | 接口 | 特色 |
|------|------|------|------|
| LAN8720A | 10/100M | RMII | 最简方案, 25MHz 晶振直驱 |
| DP83848 | 10/100M | MII/RMII | 工业级, 宽温 |
| KSZ8081 | 10/100M | MII/RMII | Microchip, 低功耗 |
| KSZ9031 | 10/100/1000M | RGMII | 千兆, 内部端接 |
| RTL8211 | 10/100/1000M | RGMII/SGMII | Realtek, 性价比 |
| BCM89811 | 1000M | SGMII | Broadcom 汽车级 |
| AR8031 | 10/100/1000M | RGMII/SGMII | Qualcomm, 同步以太网 |

---

## 二、PHY 寄存器标准详解

### 2.1 IEEE 802.3 Clause 22 寄存器结构

```
 寄存器地址 (5-bit)    │      寄存器内容 (16-bit)
┌────────────────────┐┌────────────────────────────────────┐
│  0x00 - 0x1F       ││  bit15  bit14  ...  bit1  bit0    │
└────────────────────┘└────────────────────────────────────┘
```

Clause 22 定义了 32 个寄存器 (0x00-0x1F)，前 16 个 (0x00-0x0F) 是标准必须实现的，后 16 个 (0x10-0x1F) 是厂商自定义。

#### 寄存器 0x00 -- BMCR (Basic Mode Control Register)

```
位     名称              描述
15     Reset             1=软件复位, 自清除
14     Loopback          1=环回模式
13     Speed Select(LSB)  与 bit6 组合: 00=10M, 01=100M, 10=1000M
12     Auto-Neg Enable   1=使能自动协商
11     Power Down        1=省电模式
10     Isolate           1=隔离 MAC 接口
9      Restart Auto-Neg  1=重启协商, 自清除
8      Duplex Mode       1=全双工, 0=半双工
7      Collision Test(未用) 碰撞测试
6      Speed Select(MSB)  与 bit13 组合
5:0    预留
```

**典型初始化值**：
```c
// 使能 Auto-Neg, 不重启, 正常模式
uint16_t bmcr = BMCR_AUTONEG_ENABLE;  // 0x1000

// 强制 100M 全双工 (不推荐, 生产环境用 Auto-Neg)
uint16_t bmcr = BMCR_SPEED100 | BMCR_DUPLEX_FULL;  // 0x2100
```

#### 寄存器 0x01 -- BMSR (Basic Mode Status Register)

```
位     名称              描述
15     100Base-T4        支持 T4 (基本已废弃)
14     100Base-TX FD     支持 100M 全双工
13     100Base-TX HD     支持 100M 半双工
12     10Base-T FD       支持 10M 全双工
11     10Base-T HD       支持 10M 半双工
10     预留
9      100Base-T2 FD     支持 100M T2 全双工 (极少见)
8      100Base-T2 HD     支持 100M T2 半双工 (极少见)
7      扩展状态          有扩展寄存器 (需读 0x0F)
6      预留
5     Auto-Neg Complete  协商完成
4     远端故障            Remote Fault 检测到
3     Auto-Neg Ability   支持协商 (只读, 1=支持)
2     Link Status        链路状态 (1=up, 0=down, **latching low**)
1     Jabber Detect      Jabber 检测
0     扩展能力           有扩展寄存器
```

> **关键陷阱**: bit2 (Link Status) 是 **latching low** -- 一旦链路断开, 该位变 0 并保持, 直到读取寄存器才更新为当前值。所以读取 BMSR **必须读两次**, 第一次用于清除 latch, 第二次读取真实值。

```c
uint16_t read_link_status(PHY_Handle *phy) {
    uint16_t bmsr;
    phy->read_reg(phy->addr, 0x01, &bmsr);  // 第一次, 清除 latch
    phy->read_reg(phy->addr, 0x01, &bmsr);  // 第二次, 真实值
    return (bmsr >> 2) & 0x01;
}
```

#### 寄存器 0x02 -- PHYIDR1 (PHY Identifier Register 1)

```
位     描述
15:0   OUI 的 22-bit 的高 16 位
```

#### 寄存器 0x03 -- PHYIDR2 (PHY Identifier Register 2)

```
位     描述
15:10  OUI 的低 6 位 (OUI bit 22:17)
9:4    厂商型号 (Model Number)
3:0    版本号 (Revision Number)
```

常见 PHY ID：

| PHY | IDR1 | IDR2 | 完整 ID |
|-----|------|------|---------|
| LAN8720A | 0x0007 | 0xC0F1 | 0x0007C0F1 |
| DP83848 | 0x2000 | 0xA0B1 | 0x2000A0B1 |
| KSZ8081 | 0x0022 | 0x1560 | 0x00221560 |
| KSZ9031 | 0x0022 | 0x1620 | 0x00221620 |
| RTL8211F | 0x001C | 0xC916 | 0x001CC916 |
| BCM54210 | 0x0143 | 0xBCB0 | 0x0143BCB0 |

#### 寄存器 0x04 -- ANAR (Auto-Negotiation Advertisement Register)

```
位     名称              描述
15     下一页              Next Page 支持
14     预留
13     远端故障              Remote Fault
12     预留
11     不对称 PAUSE          ASM_DIR
10     PAUSE                  PAUSE (对称流控)
9      100Base-T4            100M T4
8      100Base-TX FD         100M 全双工
7      100Base-TX HD         100M 半双工
6      10Base-T FD           10M 全双工
5      10Base-T HD           10M 半双工
4:0   选择器字段         IEEE 802.3 编码 (00001)
```

#### 寄存器 0x05 -- ANLPAR (Auto-Negotiation Link Partner Ability Register)

格式与 ANAR 相同，但来自对端。读取此寄存器可知对方支持的能力。

#### 寄存器 0x06 -- ANER (Auto-Negotiation Expansion Register)

```
位     名称              描述
15:5   预留
4      PDF                 并行检测失败
3      LP_NP_ABLE          对端支持 Next Page
2      LP_NP_ACK           对端确认 Next Page
1      页接收               接收到新页
0      LP_AUTO_NEG_ABLE    对端支持 Auto-Neg
```

#### 寄存器 0x0F -- ESTATUS (Extended Status Register)

```
位     描述
15     1000Base-T FD 支持
14     1000Base-T HD 支持
13     预留
...
```

千兆能力存储在这里（如果 BMSR bit7=1）。

### 2.2 IEEE 802.3 Clause 45 寄存器结构

Clause 45 支持更多寄存器 (64K 地址空间)，用于 10G/25G/40G 等高速 PHY。

```
┌─────────────────────────────────────────────────────┐
│ MDIO 帧结构 (Clause 45):                            │
│                                                     │
│ PRE (32个1) | ST(00) | OP(00/01) | PRTAD(5bit) |    │
│ DEVAD(5bit) | TA(2bit) | ADDR/PRTDATA(16bit)        │
│                                                     │
│ 寄存器访问通过：                                      │
│   Device Type (5-bit) + Port Address (5-bit)        │
│   再通过独立的地址寄存器间接访问                         │
└─────────────────────────────────────────────────────┘
```

**MMD (MDIO Manageable Device)** 类型：

| DEVAD | 器件 |
|-------|------|
| 0x00 | PCS (Physical Coding Sublayer) |
| 0x01 | PMA/PMD |
| 0x02 | WIS (WAN Interface Sublayer) |
| 0x03 | PCS |
| 0x04 | PHY XS |
| 0x05 | DTE XS |
| 0x06 | TC (Transmission Convergence) |
| 0x07 | Auto-Negotiation |
| 0x1E | MMAP (MMD 映射到 C22 地址空间) |

### 2.3 MDIO 时序驱动实现

```c
// 软件 bit-bang MDIO 实现 (GPIO 模拟)
// 适用于没有硬件 MDIO 控制器的 MCU

#define MDIO_CLK_DELAY()    delay_us(1)  // 1MHz MDC

typedef struct {
    GPIO_TypeDef *mdc_port;
    uint32_t      mdc_pin;
    GPIO_TypeDef *mdio_port;
    uint32_t      mdio_pin;
} MDIO_Handle;

static void mdio_set_mdc(int level) {
    HAL_GPIO_WritePin(hMdio->mdc_port, hMdio->mdc_pin, level);
    MDIO_CLK_DELAY();
}

static void mdio_set_mdio(int level) {
    HAL_GPIO_WritePin(hMdio->mdio_port, hMdio->mdio_pin, level);
}

static int mdio_get_mdio(void) {
    return HAL_GPIO_ReadPin(hMdio->mdio_port, hMdio->mdio_pin);
}

static void mdio_dir_output(void) {
    GPIO_InitTypeDef init = {0};
    init.Pin = hMdio->mdio_pin;
    init.Mode = GPIO_MODE_OUTPUT_PP;
    init.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(hMdio->mdio_port, &init);
}

static void mdio_dir_input(void) {
    GPIO_InitTypeDef init = {0};
    init.Pin = hMdio->mdio_pin;
    init.Mode = GPIO_MODE_INPUT;
    init.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(hMdio->mdio_port, &init);
}

// Clause 22 读操作
uint16_t mdio_read_c22(uint8_t phy_addr, uint8_t reg_addr) {
    uint32_t frame;
    uint16_t data = 0;

    // PRE: 32个1
    mdio_dir_output();
    for (int i = 0; i < 32; i++) {
        mdio_set_mdio(1);
        mdio_set_mdc(1); mdio_set_mdc(0);
    }

    // ST(2bit) = 01, OP(2bit) = 10 (读), PHYAD(5bit), REGAD(5bit)
    // TA(2bit) = Z0 (输出高阻, 然后 PHY 驱动 0)
    frame = (0x1 << 28) | (0x2 << 26) | ((phy_addr & 0x1F) << 21)
          | ((reg_addr & 0x1F) << 16);

    for (int i = 15; i >= 0; i--) {
        mdio_set_mdio((frame >> i) & 1);
        mdio_set_mdc(1); mdio_set_mdc(0);
    }

    // TA: 第1个bit为高阻, 第2个bit PHY 驱动 0
    mdio_dir_input();
    mdio_set_mdc(1); mdio_set_mdc(0);  // Z
    mdio_get_mdio();                    // 采样, 应 = 0
    mdio_set_mdc(1); mdio_set_mdc(0);  // 第2个 TA bit

    // 读取 16-bit 数据
    for (int i = 15; i >= 0; i--) {
        mdio_set_mdc(1);
        data |= (mdio_get_mdio() << i);
        mdio_set_mdc(0);
    }

    return data;
}

// Clause 22 写操作
void mdio_write_c22(uint8_t phy_addr, uint8_t reg_addr, uint16_t data) {
    uint32_t frame;

    mdio_dir_output();

    // PRE: 32个1
    for (int i = 0; i < 32; i++) {
        mdio_set_mdio(1);
        mdio_set_mdc(1); mdio_set_mdc(0);
    }

    // ST=01, OP=01 (写), PHYAD, REGAD, TA=10, DATA
    frame = (0x1 << 28) | (0x1 << 26) | ((phy_addr & 0x1F) << 21)
          | ((reg_addr & 0x1F) << 16) | (0x2 << 14) | (data & 0xFFFF);

    for (int i = 31; i >= 0; i--) {
        mdio_set_mdio((frame >> i) & 1);
        mdio_set_mdc(1); mdio_set_mdc(0);
    }
}
```

---

## 三、PHY 驱动分层架构

### 3.1 经典分层模型

```
+------------------------------------------------------------+
|                     应用层 (App)                             |
|   TCP/IP 协议栈, LWIP / FreeRTOS+TCP / uC-TCP-IP          |
+------------------------------------------------------------+
|                 MAC 驱动层 (ETH MAC Driver)                   |
|   STM32 ETH / NXP ENET / TI CPSW ...                       |
+------------------------------------------------------------+
|              PHY 抽象层 (PHY Abstraction Layer)               |
|   统一接口: init / link_check / speed_duplex / loopback     |
+------------------------------------------------------------+
|          PHY 芯片驱动层 (PHY Chip Driver)                    |
|   LAN8720A / DP83848 / KSZ8081 / KSZ9031 各自实现           |
+------------------------------------------------------------+
|           MDIO 通信层 (MDIO Bus Layer)                      |
|   Clause 22 / Clause 45, 硬件 MDC/MDIO 或 GPIO 模拟         |
+------------------------------------------------------------+
```

### 3.2 Linux PHY 驱动框架 (phy_device)

Linux Kernel 的 PHY 驱动框架是当前最成熟的参考模型：

```c
// Linux include/linux/phy.h 核心结构

struct phy_device {
    struct phy_c45_device_ids c45_ids;

    // PHY 信息
    unsigned int phy_id;             // 32-bit PHY ID
    unsigned int phy_id_mask;        // ID 比较掩码
    int phy_uid;                     // 绑定匹配 UID

    // 接口类型
    phy_interface_t interface;       // MII/RMII/RGMII/RGMII-ID...

    // 协商结果
    int autoneg;                     // AUTONEG_ENABLE / AUTONEG_DISABLE
    int speed;                       // SPEED_10 / SPEED_100 / SPEED_1000
    int duplex;                      // DUPLEX_HALF / DUPLEX_FULL
    int pause;                       // 流控协商结果
    int asym_pause;                  // 不对称流控

    // 链路状态
    int link;                        // 0=down, 1=up

    // 状态机
    enum phy_state state;            // PHY_DOWN/READY/AN/RUNNING...

    // 操作函数
    const struct phy_driver *drv;    // 驱动操作方法
    void *priv;                      // 私有数据
};

struct phy_driver {
    u32 phy_id;                      // 设备 ID
    char *name;                      // 驱动名
    unsigned int phy_id_mask;        // 掩码
    u32 features;                    // 能力位图

    // 生命周期
    int (*probe)(struct phy_device *);
    int (*suspend)(struct phy_device *);
    int (*resume)(struct phy_device *);

    // 核心操作
    int (*config_init)(struct phy_device *);    // 配置初始化
    int (*config_aneg)(struct phy_device *);    // 配置协商
    int (*read_status)(struct phy_device *);    // 读取状态
    int (*config_intr)(struct phy_device *);    // 配置中断
    int (*did_interrupt)(struct phy_device *);  // 中断判断

    // 特定配置
    void (*link_change_notify)(struct phy_device *);
    int (*config_aneg_ext)(struct phy_device *);
    int (*aneg_done)(struct phy_device *);      // 协商完成?
    int (*soft_reset)(struct phy_device *);     // 软件复位
};
```

### 3.3 轻量级 MCU PHY 驱动抽象

对于 MCU 环境, 通常实现简化版分层:

```c
// phy_driver.h - 通用 PHY 驱动接口

// PHY 状态枚举
typedef enum {
    PHY_STATE_RESET,         // 复位中
    PHY_STATE_INIT,          // 初始化
    PHY_STATE_AUTONEG,       // 自动协商中
    PHY_STATE_LINK_UP,       // 链路已建立
    PHY_STATE_LINK_DOWN,     // 链路断开
    PHY_STATE_ERROR          // 错误
} PHY_State;

// 协商能力
typedef struct {
    uint8_t  speed_10m_hd  : 1;
    uint8_t  speed_10m_fd  : 1;
    uint8_t  speed_100m_hd : 1;
    uint8_t  speed_100m_fd : 1;
    uint8_t  speed_1000m_hd: 1;
    uint8_t  speed_1000m_fd: 1;
    uint8_t  pause_sym     : 1;
    uint8_t  pause_asym    : 1;
} PHY_Capabilities;

// 链接状态
typedef struct {
    uint8_t  link_up       : 1;     // 1=link up
    uint8_t  autoneg_done  : 1;     // 1=协商完成
    uint8_t  remote_fault  : 1;     // 1=对端故障
    uint8_t  speed_100m    : 1;     // 0=10M, 1=100M
    uint8_t  speed_1000m   : 1;     // 1=1000M (覆盖上面)
    uint8_t  full_duplex   : 1;     // 1=全双工
    uint8_t  pause_rx      : 1;     // 1=接收流控使能
    uint8_t  pause_tx      : 1;     // 1=发送流控使能
} PHY_LinkState;

// MDIO 总线操作 (由平台层提供)
typedef struct {
    void    *bus_priv;                              // 总线私有数据
    int     (*read_c22)(void *priv, uint8_t phy_addr, uint8_t reg, uint16_t *val);
    int     (*write_c22)(void *priv, uint8_t phy_addr, uint8_t reg, uint16_t val);
    int     (*read_c45)(void *priv, uint8_t phy_addr, uint8_t dev, uint16_t reg, uint16_t *val);
    int     (*write_c45)(void *priv, uint8_t phy_addr, uint8_t dev, uint16_t reg, uint16_t val);
} MDIO_Bus;

// PHY 设备
typedef struct {
    uint8_t      phy_addr;           // PHY 地址 (通常 0-31)
    uint16_t     phy_id_hi;          // PHY ID 高 16 位
    uint16_t     phy_id_lo;          // PHY ID 低 16 位
    uint32_t     phy_id;             // 32-bit 完整 ID
    MDIO_Bus    *bus;                // MDIO 总线
    PHY_State    state;              // 当前状态
    PHY_LinkState link;              // 当前链接状态
    PHY_Capabilities caps;           // PHY 能力
    void        *priv;               // PHY 私有数据 (芯片特有)

    // 驱动操作 (由具体 PHY 芯片驱动填充)
    int (*init)(struct phy_device *dev);
    int (*start_autoneg)(struct phy_device *dev);
    int (*read_link)(struct phy_device *dev, PHY_LinkState *link);
    int (*config_loopback)(struct phy_device *dev, int enable);
    int (*config_intr)(struct phy_device *dev, int enable);
    int (*power_save)(struct phy_device *dev, int enable);
    int (*soft_reset)(struct phy_device *dev);
} PHY_Device;
```

### 3.4 芯片驱动示例: LAN8720A 驱动

```c
// phy_lan8720a.c

#define LAN8720A_PHY_ID       0x0007C0F1

// LAN8720A 特有寄存器
#define LAN8720A_REG_SCSR     0x1F    // Special Control/Status Register
#define LAN8720A_SCSR_100M    (1 << 14)  // 100Mbps 模式
#define LAN8720A_SCSR_FD      (1 << 13)  // 全双工
#define LAN8720A_SCSR_AUTONEG (1 << 12)  // 协商完成

#define LAN8720A_REG_MSC      0x1B    // Mode Select Configuration
#define LAN8720A_MSC_RMII     (1 << 8)   // RMII 模式
#define LAN8720A_MSC_PHYAD    (0 << 4)   // PHY 地址

// LAN8720A 的模式选择 (通过 nINT/REGOFF 引脚)
#define LAN8720A_MODE_AUTO    0       // Auto-negotiation
#define LAN8720A_MODE_100_FD  1       // 100M 全双工
#define LAN8720A_MODE_100_HD  2       // 100M 半双工
#define LAN8720A_MODE_10_FD   3       // 10M 全双工
#define LAN8720A_MODE_10_HD   4       // 10M 半双工

// 检查 LAN8720A PHY ID
static int lan8720a_check_id(PHY_Device *dev) {
    uint16_t id1, id2;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x02, &id1) != 0)
        return -1;
    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x03, &id2) != 0)
        return -1;

    uint32_t phy_id = ((uint32_t)id1 << 16) | id2;
    dev->phy_id_hi = id1;
    dev->phy_id_lo = id2;
    dev->phy_id = phy_id;

    return (phy_id == LAN8720A_PHY_ID) ? 0 : -1;
}

// LAN8720A 初始化
static int lan8720a_init(PHY_Device *dev) {
    uint16_t val;

    // 1. 复位 PHY
    if (dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, 0x8000) != 0)
        return -1;
    delay_ms(100);  // 等待复位完成

    // 2. 读取 BMCR 确认复位完成
    for (int i = 0; i < 100; i++) {
        if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &val) != 0)
            return -1;
        if ((val & 0x8000) == 0)
            break;  // 复位自清除
        delay_ms(10);
    }

    // 3. 配置 MODE (特殊寄存器)
    //    LAN8720A 通过 nINT/REGOFF strapping 引脚选择模式
    //    也可以通过寄存器 0x1B 覆写
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, LAN8720A_REG_MSC, &val);
    val |= LAN8720A_MSC_RMII;  // RMII 模式
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, LAN8720A_REG_MSC, val);

    // 4. 使能 Auto-Negotiation
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, 0x1200);
    // 0x1200 = BMCR_AUTONEG_ENABLE | BMCR_RESTART_AUTONEG

    dev->state = PHY_STATE_AUTONEG;
    return 0;
}

// 读取链接状态
static int lan8720a_read_link(PHY_Device *dev, PHY_LinkState *link) {
    uint16_t bmsr, scsr;

    // 读两次 BMSR 以清除 latch
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);

    link->link_up = (bmsr >> 2) & 0x01;

    if (!link->link_up) {
        link->autoneg_done = 0;
        dev->state = PHY_STATE_LINK_DOWN;
        return 0;
    }

    // 读取 SCSR 获取速度和双工状态
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, LAN8720A_REG_SCSR, &scsr);

    link->autoneg_done = (scsr >> 12) & 0x01;
    link->speed_100m   = (scsr >> 14) & 0x01;
    link->full_duplex  = (scsr >> 13) & 0x01;
    link->speed_1000m  = 0;  // LAN8720A 不支持千兆

    dev->state = PHY_STATE_LINK_UP;
    return 0;
}

// PHY 驱动结构体
const PHY_Driver lan8720a_driver = {
    .phy_id      = LAN8720A_PHY_ID,
    .name        = "LAN8720A",
    .check_id    = lan8720a_check_id,
    .init        = lan8720a_init,
    .read_link   = lan8720a_read_link,
    .config_intr = lan8720a_config_intr,
    .soft_reset  = lan8720a_soft_reset,
    .power_save  = lan8720a_power_save,
};
```

---

## 四、PHY 初始化流程详解

### 4.1 完整初始化时序图

```
上电/复位
    │
    ├── 硬件复位 (HW RST) ──→ 等待 TRST (通常 50ms)
    │
    ├── 读取 PHY ID ──────→ 验证 PHY 型号
    │
    ├── 软件复位 ─────────→ BMCR.bit15=1, 等待自清除
    │
    ├── 配置介质接口 ──────→ MII/RMII/RGMII 选择
    │
    ├── 配置 LED 模式 ─────→ link/activity 指示
    │
    ├── 配置中断 ─────────→ 使能 link change 中断 (可选)
    │
    ├── 配置 Auto-Neg ────→ ASAR: 发布能力
    │                        BMCR: 使能 + 重启协商
    │
    ├── 等待协商完成 ─────→ 轮询 BMSR.bit5 或等待中断
    │
    └── 读取协商结果 ─────→ 速度/双工/流控
                             通知 MAC 层配置对应参数
```

### 4.2 各阶段详细代码

#### 阶段 1: 硬件复位

```c
void phy_hardware_reset(PHY_Device *dev) {
    // MCU 的 GPIO 控制 PHY 的 RST_N 引脚
    HAL_GPIO_WritePin(PHY_RST_PORT, PHY_RST_PIN, GPIO_PIN_RESET);  // 拉低
    delay_ms(50);      // 保持复位 50ms (TRST 通常 10-50ms)
    HAL_GPIO_WritePin(PHY_RST_PORT, PHY_RST_PIN, GPIO_PIN_SET);    // 释放
    delay_ms(100);     // 等待 PHY 完成内部初始化 (TINIT 通常 50-100ms)
}
```

**不同 PHY 复位时间要求**：

| PHY | TRST (最小) | TINIT (等待读取) | 备注 |
|-----|-------------|-------------------|------|
| LAN8720A | 10ms | 50ms | 复位后默认地址 0 |
| DP83848 | 10ms | 50ms | 默认地址 0x10 (AD0=1) |
| KSZ8081 | 10ms | 50ms | 默认地址 0x01 |
| KSZ9031 | 10ms | 100ms | 千兆需要更久 |
| RTL8211 | 10ms | 80ms | 默认地址 0x01 |

#### 阶段 2: 读取 PHY ID 并验证

```c
typedef struct {
    uint32_t phy_id;
    const char *name;
    uint8_t default_addr;
} PHY_ChipInfo;

static const PHY_ChipInfo known_phys[] = {
    { 0x0007C0F1, "LAN8720A",    0 },
    { 0x2000A0B1, "DP83848",     0x10 },
    { 0x00221560, "KSZ8081",     0x01 },
    { 0x00221620, "KSZ9031",     0x01 },
    { 0x001CC916, "RTL8211F",    0x01 },
    { 0x0143BCB0, "BCM54210",    0x00 },
    { 0,          NULL,          0 },
};

int phy_detect(MDIO_Bus *bus, uint8_t start_addr, uint8_t end_addr) {
    uint16_t id1, id2;

    for (uint8_t addr = start_addr; addr <= end_addr; addr++) {
        if (bus->read_c22(bus->bus_priv, addr, 0x02, &id1) != 0)
            continue;
        if (id1 == 0x0000 || id1 == 0xFFFF)  // 无设备
            continue;

        if (bus->read_c22(bus->bus_priv, addr, 0x03, &id2) != 0)
            continue;
        if (id2 == 0x0000 || id2 == 0xFFFF)
            continue;

        uint32_t phy_id = ((uint32_t)id1 << 16) | id2;

        for (int i = 0; known_phys[i].name != NULL; i++) {
            if (phy_id == known_phys[i].phy_id) {
                printf("PHY detected: %s at addr 0x%02X\n",
                       known_phys[i].name, addr);
                return addr;
            }
        }

        printf("Unknown PHY at addr 0x%02X, ID=0x%08X\n", addr, phy_id);
    }

    return -1;  // 未找到
}
```

#### 阶段 3: 软件复位

```c
int phy_soft_reset(PHY_Device *dev) {
    uint16_t val;
    uint32_t timeout;

    // 写 BMCR.bit15=1 触发复位
    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &val) != 0)
        return -1;

    val |= (1 << 15);  // BMCR_RESET
    if (dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, val) != 0)
        return -1;

    // 等待复位自清除 (最多等 500ms)
    timeout = 500;
    while (timeout--) {
        delay_ms(1);
        if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &val) != 0)
            return -1;
        if ((val & (1 << 15)) == 0)  // 复位完成
            return 0;
    }

    return -1;  // 超时
}
```

#### 阶段 4: 配置介质接口 (RMII 示例)

```c
// LAN8720A 通过特殊寄存器选择 RMII/MII
int lan8720a_config_interface(PHY_Device *dev, int mode) {
    uint16_t val;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x1B, &val) != 0)
        return -1;

    if (mode == PHY_IFACE_RMII) {
        val |= (1 << 8);   // RMII Mode Select
    } else {
        val &= ~(1 << 8);  // MII Mode
    }

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x1B, val);
}

// DP83848 通过 STRAP 引脚配置, 也可通过 LED/控制寄存器配置
int dp83848_config_interface(PHY_Device *dev, int mode) {
    uint16_t val;

    // DP83848 寄存器 0x12 (LED/General Purpose Control)
    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x12, &val) != 0)
        return -1;

    // bit5: RMII Enable (1=RMII, 0=MII)
    if (mode == PHY_IFACE_RMII) {
        val |= (1 << 5);
    } else {
        val &= ~(1 << 5);
    }

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x12, val);
}
```

#### 阶段 5: 配置 Auto-Negotiation

```c
int phy_config_autoneg(PHY_Device *dev) {
    uint16_t anar = 0;

    // ANAR 的默认能力 (发布给对端)
    anar |= (1 << 9);    // 100Base-TX 全双工
    anar |= (1 << 8);    // 100Base-TX 半双工
    anar |= (1 << 7);    // 10Base-T 全双工
    anar |= (1 << 6);    // 10Base-T 半双工
    anar |= (1 << 0);    // 协议选择器 (IEEE 802.3)

    // 流控能力 (可配置)
    // anar |= (1 << 10); // PAUSE 对称流控
    // anar |= (1 << 11); // ASM_DIR 不对称流控

    if (dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x04, anar) != 0)
        return -1;

    // 重启协商
    uint16_t bmcr;
    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &bmcr) != 0)
        return -1;

    bmcr |= (1 << 12);  // 使能 Auto-Neg
    bmcr |= (1 << 9);   // 重启 Auto-Neg

    if (dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr) != 0)
        return -1;

    dev->state = PHY_STATE_AUTONEG;
    return 0;
}
```

### 4.3 完整的 MCU PHY 初始化函数

```c
// 通用 PHY 初始化入口
int phy_init_generic(PHY_Device *dev) {
    int ret;

    printf("PHY init: addr=0x%02X\n", dev->phy_addr);

    // 1. 验证 PHY ID
    ret = dev->check_id(dev);
    if (ret != 0) {
        printf("ERROR: PHY ID check failed\n");
        return -1;
    }
    printf("  PHY ID: 0x%08X\n", dev->phy_id);

    // 2. 软件复位
    ret = dev->soft_reset(dev);
    if (ret != 0) {
        printf("ERROR: PHY soft reset failed\n");
        return -1;
    }
    printf("  Soft reset: OK\n");

    // 3. 芯片特定初始化 (配置接口模式等)
    ret = dev->init(dev);
    if (ret != 0) {
        printf("ERROR: PHY chip init failed\n");
        return -1;
    }
    printf("  Chip init: OK\n");

    // 4. 读取 PHY 能力
    phy_read_capabilities(dev);

    // 5. 配置并启动 Auto-Neg
    ret = dev->start_autoneg(dev);
    if (ret != 0) {
        printf("ERROR: PHY auto-neg start failed\n");
        return -1;
    }
    printf("  Auto-Neg started\n");

    dev->state = PHY_STATE_AUTONEG;
    return 0;
}
```

---

## 五、Link 检测与状态机

### 5.1 PHY 状态机

```
                 ┌─────────────────────────────────────────┐
                 │                                         │
                 ▼                                         │
          ┌───────────┐                                    │
          │   RESET   │                                    │
          └─────┬─────┘                                    │
                │ 复位完成                                 │
                ▼                                          │
          ┌───────────┐                                    │
          │   INIT    │ ◄─────── 配置失败                   │
          └─────┬─────┘                                    │
                │ init() OK                                │
                ▼                                          │
          ┌───────────┐                                    │
     ┌───►│AUTONEG    │                                    │
     │    └─────┬─────┘                                    │
     │          │ 协商超时/失败                             │
     │          ├────────────────────────────────────────┐  │
     │          │ 协商完成                               │  │
     │          ▼                                        │  │
     │    ┌───────────┐        ┌──────────────────┐      │  │
     │    │ LINK_UP   │◄──────►│   LINK_DOWN      │      │  │
     │    └─────┬─────┘  link  └──────────────────┘      │  │
     │          │         down                            │  │
     │          │ link up 重新协商                        │  │
     │          └─────────────────────────────────────────┘  │
     │                                                       │
     └───────────────────────────────────────────────────────┘
```

### 5.2 状态机实现

```c
// PHY 状态机
PHY_State phy_state_machine(PHY_Device *dev) {
    PHY_LinkState link = {0};

    switch (dev->state) {

    case PHY_STATE_RESET:
        // 等待复位完成 (由上层触发, 这里检查)
        if (phy_soft_reset(dev) == 0) {
            dev->state = PHY_STATE_INIT;
        }
        break;

    case PHY_STATE_INIT:
        if (dev->init(dev) == 0) {
            dev->start_autoneg(dev);
            dev->state = PHY_STATE_AUTONEG;
        }
        break;

    case PHY_STATE_AUTONEG:
        dev->read_link(dev, &link);
        if (link.link_up && link.autoneg_done) {
            // 通知 MAC 层配置
            mac_config_speed_duplex(link.speed_100m, link.full_duplex);
            dev->link = link;
            dev->state = PHY_STATE_LINK_UP;
            printf("LINK UP: %s %s Duplex\n",
                   link.speed_100m ? "100M" : "10M",
                   link.full_duplex ? "Full" : "Half");
        } else if (/* 超时 */) {
            // 协商超时, 可尝试 parallel detection
            dev->state = PHY_STATE_ERROR;
        }
        break;

    case PHY_STATE_LINK_UP:
        dev->read_link(dev, &link);
        if (!link.link_up) {
            dev->state = PHY_STATE_LINK_DOWN;
            dev->link.link_up = 0;
            printf("LINK DOWN\n");
        }
        // 更新速度和双工 (可能在协商后改变)
        dev->link = link;
        break;

    case PHY_STATE_LINK_DOWN:
        dev->read_link(dev, &link);
        if (link.link_up && link.autoneg_done) {
            mac_config_speed_duplex(link.speed_100m, link.full_duplex);
            dev->link = link;
            dev->state = PHY_STATE_LINK_UP;
            printf("LINK UP Again: %s %s Duplex\n",
                   link.speed_100m ? "100M" : "10M",
                   link.full_duplex ? "Full" : "Half");
        }
        break;

    case PHY_STATE_ERROR:
        // 尝试恢复: 重新复位
        dev->soft_reset(dev);
        dev->init(dev);
        dev->start_autoneg(dev);
        dev->state = PHY_STATE_AUTONEG;
        break;
    }

    return dev->state;
}
```

### 5.3 轮询 vs 中断模式

**轮询模式** (简单但浪费 CPU)：

```c
// 在主循环或 RTOS 任务中周期调用
void phy_poll_task(void *param) {
    PHY_Device *dev = (PHY_Device *)param;

    while (1) {
        phy_state_machine(dev);
        delay_ms(100);  // 100ms 轮询间隔
    }
}
```

**中断模式** (推荐, 响应更快)：

```c
// PHY 中断服务函数
// 在 MCU GPIO 中断中调用
void phy_irq_handler(PHY_Device *dev) {
    // 读取中断状态寄存器 (各 PHY 不同)
    uint16_t isr;
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                       PHY_REG_ISR, &isr);

    if (isr & PHY_ISR_LINK_DOWN) {
        dev->state = PHY_STATE_LINK_DOWN;
        notify_network_stack(NET_LINK_DOWN);
    }
    if (isr & PHY_ISR_LINK_UP) {
        // 延迟短暂时间等协商稳定
        vTaskDelay(pdMS_TO_TICKS(50));
        dev->read_link(dev, &dev->link);
        mac_config_speed_duplex(dev->link.speed_100m, dev->link.full_duplex);
        dev->state = PHY_STATE_LINK_UP;
        notify_network_stack(NET_LINK_UP);
    }
}
```

### 5.4 Link 抖动 (Bouncing) 处理

实际布线中, 网线插入/拔出瞬间会产生多次 link state 变化 (抖动)。

```c
// 去抖处理
typedef struct {
    uint8_t  last_stable_state;   // 上次稳定状态
    uint32_t last_change_tick;    // 上次变化时间戳
    uint8_t  debounce_count;      // 连续采样相同值次数
    uint8_t  debounce_threshold;  // 去抖阈值 (推荐 3-5)
} PHY_Debounce;

static PHY_Debounce phy_debounce;

uint8_t phy_debounce_link(PHY_Device *dev, uint8_t current_link) {
    uint32_t now = HAL_GetTick();

    if (current_link == phy_debounce.last_stable_state) {
        // 状态未变, 重置计数
        phy_debounce.debounce_count = 0;
        return current_link;
    }

    // 状态变化, 检查时间间隔
    if (now - phy_debounce.last_change_tick > 50) {
        // 超过 50ms, 可能是有效变化
        phy_debounce.debounce_count = 1;
    } else {
        // 短时间内反复变化
        phy_debounce.debounce_count++;
        if (phy_debounce.debounce_count >= phy_debounce.debounce_threshold) {
            // 连续采样确认, 接受新状态
            phy_debounce.last_stable_state = current_link;
            phy_debounce.debounce_count = 0;
            return current_link;
        }
    }

    phy_debounce.last_change_tick = now;
    return phy_debounce.last_stable_state;  // 返回旧状态
}
```

---

## 六、Auto-Negotiation 再探 -- 驱动侧实现

### 6.1 协商流程详细

```
本地 PHY                          对端 PHY
   │                                │
   ├─ FLP Burst (快速连接脉冲) ─────►│
   │   (内含本地能力 ANAR)          │
   │                                ├─ FLP Burst ────►
   │                                │   (内含对端能力 ANLPAR)
   │◄── FLP Burst ────────────────┤
   │                                │
   ├─ ACK 握手 ───────────────────►│
   │◄── ACK 握手 ─────────────────┤
   │                                │
   ├─ Next Page Exchange (可选) ───►│
   │◄── Next Page ─────────────────┤
   │                                │
   ├─ 决定 HCD (Highest Common      │
   │    Denominator)                │
   │  speed = min(本地速度, 对端速度)│
   │  duplex = min(本地双工, 对端)   │
   │                                │
   ├─ BMSR.bit5 = 1 (协商完成) ────►│
   │                                │
   └─ 读取协商结果                   │
```

### 6.2 HCD (Highest Common Denominator) 算法

```c
typedef enum {
    PHY_SPEED_10M_HD  = 0,
    PHY_SPEED_10M_FD  = 1,
    PHY_SPEED_100M_HD = 2,
    PHY_SPEED_100M_FD = 3,
    PHY_SPEED_1000M_HD = 4,
    PHY_SPEED_1000M_FD = 5,
} PHY_SpeedDuplex;

// 优先级: 1000M_FD > 1000M_HD > 100M_FD > 100M_HD > 10M_FD > 10M_HD
static const PHY_SpeedDuplex hcd_priority[] = {
    PHY_SPEED_1000M_FD,
    PHY_SPEED_1000M_HD,
    PHY_SPEED_100M_FD,
    PHY_SPEED_100M_HD,
    PHY_SPEED_10M_FD,
    PHY_SPEED_10M_HD,
};

// 计算最佳协商结果
PHY_SpeedDuplex phy_autoneg_resolve(uint16_t local_advert, uint16_t partner_advert) {
    // bit masks: T4=bit9, 100FD=bit8, 100HD=bit7, 10FD=bit6, 10HD=bit5
    // 为简化, 我们检查共有的最高优先级能力

    uint8_t common;  // bit 映射: bit5=10HD, bit6=10FD, bit7=100HD, bit8=100FD

    common = (local_advert >> 5) & partner_advert >> 5;

    for (int i = 0; i < 6; i++) {
        uint8_t bit;
        switch (hcd_priority[i]) {
            case PHY_SPEED_1000M_FD: bit = 0x80; break;  // 千兆需读扩展寄存器
            case PHY_SPEED_1000M_HD: bit = 0x40; break;
            case PHY_SPEED_100M_FD:  bit = 0x08; break;
            case PHY_SPEED_100M_HD:  bit = 0x04; break;
            case PHY_SPEED_10M_FD:   bit = 0x02; break;
            case PHY_SPEED_10M_HD:   bit = 0x01; break;
            default: continue;
        }
        if (common & bit) {
            return hcd_priority[i];
        }
    }

    return PHY_SPEED_10M_HD;  // 兜底
}
```

### 6.3 Parallel Detection (并行检测)

当对端不支持 Auto-Negotiation (不发送 FLP/NLP) 时, PHY 自动回退到并行检测:

```c
// 并行检测模式 (对端是 legacy hub 或强制模式)
int phy_parallel_detect(PHY_Device *dev, PHY_LinkState *link) {
    // 在未能完成协商时使用
    // 通过检测线缆上的信号判断速度
    //
    // - 检测到 10Base-T 的 Link Pulse → 10M
    // - 检测到 100Base-TX 的 4B/5B 空闲码 → 100M
    // - 检测到 1000Base-T 的 1000M 空闲 → 1000M
    //
    // 等待约 150-200ms (无 FLP 时回退)

    uint16_t bmsr;

    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);

    if ((bmsr & (1 << 5)) == 0) {
        // Auto-Neg 未完成, 尝试并行检测
        if (bmsr & (1 << 2)) {
            // Link 已建立, 但协商未完成
            // 读取特定 PHY 寄存器判断速度和双工
            uint16_t scsr;
            dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                               LAN8720A_REG_SCSR, &scsr);

            link->link_up = 1;
            link->autoneg_done = 0;    // 非协商模式
            link->speed_100m = (scsr >> 14) & 1;
            link->full_duplex = (scsr >> 13) & 1;
            return 0;
        }
    }

    return -1;
}
```

### 6.4 流控 (Flow Control/Pause) 协商

流控协商是 Auto-Neg 的一部分, 通过 ANAR 的 bit10-11:

```c
// 流控协商: 决定能否发送/接收 Pause 帧
// 根据 IEEE 802.3 表 28B-2

typedef enum {
    PAUSE_NONE = 0,     // 双方都不支持
    PAUSE_SYMMETRIC,    // 对称: 双方都能收发 Pause
    PAUSE_ASYM_LOCAL,   // 本地可发送, 对端不能
    PAUSE_ASYM_REMOTE,  // 对端可发送, 本地不能
} PauseResolution;

PauseResolution phy_resolve_pause(uint16_t local_advert, uint16_t partner_advert) {
    int local_pause  = (local_advert >> 10) & 1;
    int local_asym   = (local_advert >> 11) & 1;
    int peer_pause   = (partner_advert >> 10) & 1;
    int peer_asym    = (partner_advert >> 11) & 1;

    // IEEE 802.3 表 28B-2 真值表
    if (local_pause && peer_pause) {
        return PAUSE_SYMMETRIC;
    }
    if (local_pause && !peer_pause && local_asym) {
        return PAUSE_ASYM_LOCAL;   // 本地可发 Pause
    }
    if (!local_pause && peer_pause && peer_asym) {
        return PAUSE_ASYM_REMOTE;  // 对端可发 Pause
    }
    if (!local_pause && peer_pause && !peer_asym) {
        // 对端支持 Pause, 但本地不支持且无不对称标志
        // 实际中可能导致冲突, 最好回退到无流控
    }

    return PAUSE_NONE;
}
```

---

## 七、Auto-MDI/X 实现

### 7.1 MDI/MDI-X 概念

```
MDI (直连线):
  TX+ ──────────── TX+
  TX- ──────────── TX-
  RX+ ──────────── RX+
  RX- ──────────── RX-

MDI-X (交叉线):
  TX+ ──────────── RX+
  TX- ──────────── RX-
  RX+ ──────────── TX+
  RX- ──────────── TX-
```

Auto-MDI/X 自动检测线缆类型并切换引脚映射。

### 7.2 Auto-MDI/X 实现

```c
// 大多数现代 PHY 支持硬件 Auto-MDI/X
// 通过寄存器控制

// KSZ8081 MDI/MDI-X 控制 (寄存器 0x16)
#define KSZ8081_REG_PHYCTRL    0x16
#define KSZ8081_PHYCTRL_AMDIX  (1 << 7)   // Auto MDI-X 使能
#define KSZ8081_PHYCTRL_MDI    (1 << 6)   // 1=MDI, 0=MDI-X (手动模式)

int phy_enable_auto_mdix(PHY_Device *dev) {
    uint16_t val;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                           KSZ8081_REG_PHYCTRL, &val) != 0)
        return -1;

    val |= KSZ8081_PHYCTRL_AMDIX;   // 使能自动 MDI/X
    val &= ~KSZ8081_PHYCTRL_MDI;    // 清除手动模式选择

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                                KSZ8081_REG_PHYCTRL, val);
}

// 读取当前 MDI/MDI-X 状态
int phy_read_mdix_status(PHY_Device *dev) {
    uint16_t val;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                           KSZ8081_REG_PHYCTRL, &val) != 0)
        return -1;

    // bit5: MDI-X Status (1=MDI-X, 0=MDI)
    return (val >> 5) & 1;
}
```

### 7.3 手动 MDI/MDI-X 切换 (软件实现)

```c
// 当硬件不支持 Auto-MDI/X 时, 软件通过尝试两种模式实现

typedef enum {
    MDI_MODE_STRAIGHT,   // 直连 (MDI)
    MDI_MODE_CROSS,      // 交叉 (MDI-X)
} MDI_Mode;

int phy_set_mdix_mode(PHY_Device *dev, MDI_Mode mode) {
    uint16_t val;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                           KSZ8081_REG_PHYCTRL, &val) != 0)
        return -1;

    val &= ~(1 << 7);  // 关闭自动模式

    if (mode == MDI_MODE_STRAIGHT) {
        val |= KSZ8081_PHYCTRL_MDI;     // MDI
    } else {
        val &= ~KSZ8081_PHYCTRL_MDI;    // MDI-X
    }

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                                KSZ8081_REG_PHYCTRL, val);
}

// 软件 MDI/MDI-X 自动探测 (在 link 建立失败时尝试切换)
int phy_software_auto_mdix(PHY_Device *dev) {
    PHY_LinkState link;

    // 先尝试 MDI
    phy_set_mdix_mode(dev, MDI_MODE_STRAIGHT);
    delay_ms(300);  // 等待链路建立

    dev->read_link(dev, &link);
    if (link.link_up)
        return 0;  // MDI 通了

    // 再尝试 MDI-X
    phy_set_mdix_mode(dev, MDI_MODE_CROSS);
    delay_ms(300);

    dev->read_link(dev, &link);
    if (link.link_up)
        return 0;  // MDI-X 通了

    return -1;  // 都不通 (可能是线缆或对端问题)
}
```

---

## 八、PHY 中断处理

### 8.1 中断架构

不同的 PHY 芯片中断管理方式不同:

```c
// 通用 PHY 中断配置
int phy_generic_config_intr(PHY_Device *dev, int enable) {
    uint16_t val;

    // 1. 读取中断使能寄存器
    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                           PHY_REG_IER, &val) != 0)
        return -1;

    if (enable) {
        // 使能所需的中断源
        val |= PHY_IER_LINK_UP;
        val |= PHY_IER_LINK_DOWN;
        val |= PHY_IER_REMOTE_FAULT;
        // val |= PHY_IER_ANEG_DONE;  // 有些 PHY 需要单独使能
    } else {
        val = 0;
    }

    if (dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                            PHY_REG_IER, val) != 0)
        return -1;

    return 0;
}
```

### 8.2 LAN8720A 中断

LAN8720A 的中断通过 nINT 引脚输出, 复用了 nINT/REGOFF strapping 引脚:

```c
// LAN8720A 中断寄存器 (0x1D - Interrupt Source Register)
#define LAN8720A_REG_ISR      0x1D
#define LAN8720A_ISR_ANEG_DONE   (1 << 12)  // 自动协商完成
#define LAN8720A_ISR_LINK_DOWN   (1 << 11)  // 链路断开
#define LAN8720A_ISR_REMOTE_FAULT (1 << 10)
#define LAN8720A_ISR_LINK_UP     (1 << 9)
#define LAN8720A_ISR_WOL         (1 << 8)   // 魔术包唤醒

// LAN8720A 中断使能寄存器 (0x1E - Interrupt Mask Register)
#define LAN8720A_REG_IMR      0x1E
// 位定义与 ISR 相同, 1=使能

int lan8720a_config_intr(PHY_Device *dev, int enable) {
    uint16_t val = 0;

    if (enable) {
        // 使能 link up/down, aneg done 中断
        val = LAN8720A_ISR_LINK_UP |
              LAN8720A_ISR_LINK_DOWN |
              LAN8720A_ISR_ANEG_DONE;
    }

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                                LAN8720A_REG_IMR, val);
}

void lan8720a_irq_handler(PHY_Device *dev) {
    uint16_t isr;

    // 读取中断状态 (读取即清除)
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                       LAN8720A_REG_ISR, &isr);

    if (isr & LAN8720A_ISR_LINK_UP) {
        printf("LAN8720A IRQ: Link Up\n");
        dev->read_link(dev, &dev->link);
        dev->state = PHY_STATE_LINK_UP;
    }
    if (isr & LAN8720A_ISR_LINK_DOWN) {
        printf("LAN8720A IRQ: Link Down\n");
        dev->state = PHY_STATE_LINK_DOWN;
    }
    if (isr & LAN8720A_ISR_ANEG_DONE) {
        printf("LAN8720A IRQ: Auto-Neg Done\n");
    }
}
```

### 8.3 DP83848 中断

```c
// DP83848 中断寄存器
#define DP83848_REG_ISR      0x0E   // Interrupt Status Register
#define DP83848_ISR_LINK_CHG   (1 << 0)   // Link 状态变化
#define DP83848_ISR_ANEG_ERR   (1 << 1)   // 协商错误
#define DP83848_ISR_ANEG_DONE  (1 << 2)   // 协商完成
#define DP83848_ISR_REMOTE_FAULT (1 << 3)
#define DP83848_ISR_PAGE_RX    (1 << 4)

#define DP83848_REG_ICR      0x0D   // Interrupt Control Register
#define DP83848_ICR_INT_OE    (1 << 0)   // 输出使能
#define DP83848_ICR_INT_EN    (1 << 1)   // 全局中断使能

int dp83848_config_intr(PHY_Device *dev, int enable) {
    uint16_t val;

    if (enable) {
        // 使能中断输出
        val = DP83848_ICR_INT_OE | DP83848_ICR_INT_EN;
    } else {
        val = 0;
    }

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                                DP83848_REG_ICR, val);
}
```

### 8.4 中断 vs 轮询的权衡

| 方法 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 轮询 | 实现简单, 不占用引脚 | 浪费 CPU, 延迟大 | 低功耗 MCU (低速轮询) |
| 中断 | 响应快, 省 CPU | 需要 GPIO 引脚, 有抖动问题 | RTOS / 实时要求高 |
| 混合 | 中断唤醒 + 轮询确认 | 代码复杂 | 实际项目常用 |

---

## 九、PHY 功耗管理

### 9.1 电源管理能力

现代 PHY 支持多种功耗模式:

| 模式 | 功耗 | 功能 | 唤醒时间 |
|------|------|------|----------|
| Active | 100% | 全功能 | 0 |
| Low-Power Idle (LPI) | ~70% | 802.3az EEE | 几 us |
| Energy Detect | ~30% | 检测链路活动 | ~50ms |
| Software Power-Down | ~5% | BMCR.bit11, MDIO 可访问 | ~100ms |
| Hardware Power-Down | ~0.1% | PHY_RST_N 拉低, 完全关闭 | 需重新初始化 |

### 9.2 软件省电模式

```c
// 软件省电模式 (BMCR.bit11)
int phy_power_down(PHY_Device *dev) {
    uint16_t bmcr;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &bmcr) != 0)
        return -1;

    bmcr |= (1 << 11);  // Power Down

    return dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr);
}

int phy_power_up(PHY_Device *dev) {
    uint16_t bmcr;

    if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &bmcr) != 0)
        return -1;

    bmcr &= ~(1 << 11);  // 清除 Power Down

    if (dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr) != 0)
        return -1;

    // 退出省电模式后可能需要重新协商
    delay_ms(50);

    return phy_start_autoneg(dev);
}
```

### 9.3 Energy-Efficient Ethernet (EEE, 802.3az)

```c
// EEE 使能 (以 KSZ9031 为例)
#define KSZ9031_REG_EEE_CTRL     0x0D   // 通过 MMD 访问
#define KSZ9031_MMD_DEVAD        0x07   // Auto-Neg 器件
#define KSZ9031_EEE_ADVERT       0x3C   // EEE 能力通告

int phy_eee_enable(PHY_Device *dev) {
    uint16_t val;

    // 读取 EEE 能力 (通过 MMD 间接访问)
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                        0x0D, KSZ9031_MMD_DEVAD);
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr,
                        0x0E, KSZ9031_EEE_ADVERT);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x0D, &val);

    // 使能 100Base-TX EEE
    val |= (1 << 1);

    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x0D, val);

    return 0;
}
```

### 9.4 WoLAN (Wake-on-LAN)

```c
// 魔术包结构: 6字节 0xFF + 16次 MAC 地址
// 例: FF FF FF FF FF FF AA BB CC DD EE FF (重复16次)

int phy_wolan_enable(PHY_Device *dev, uint8_t *mac_addr) {
    // 1. 将 MAC 地址写入 PHY 的 WoL 寄存器 (各芯片不同)
    // 2. 使能魔术包检测
    // 3. 在省电模式下监测链路

    uint16_t val;

    // 以 LAN8720A 为例 (WOL 通过寄存器 0x10-0x1F)
    // 实际需参考各芯片手册

    printf("WoLAN enabled for MAC: "
           "%02X:%02X:%02X:%02X:%02X:%02X\n",
           mac_addr[0], mac_addr[1], mac_addr[2],
           mac_addr[3], mac_addr[4], mac_addr[5]);

    return 0;
}
```

---

## 十、Switch PHY 与独立 PHY 的区别

### 10.1 架构差异

```
独立 PHY:                         Switch PHY:
┌─────────┐                      ┌──────────────────────────────┐
│  MAC    │                      │  Switch Engine (MAC Table)   │
├─────────┤                      ├──────────┬──────────┬────────┤
│  PHY    │                      │ Port1 PHY│ Port2 PHY│ Port3  │
└─────────┘                      │  (PHY1)  │  (PHY2)  │  ...   │
                                 │          │          │        │
独立 PHY + MAC (传统架构)         │ MII/RMII  │          │        │
                                  │          │          │        │
                                  │   Switch Fabric (内部)      │
                                  └──────────────────────────────┘
                                              │
                                        CPU 上行端口
```

### 10.2 关键区别

| 特性 | 独立 PHY | Switch PHY |
|------|----------|------------|
| MDIO 地址 | 单 PHY 一个地址 | 端口 1-N 各一个地址 |
| 寄存器映射 | 标准 C22/C45 | 可能扩展或重新映射 |
| 管理方式 | MDIO 直接访问 | 通过 Switch 寄存器间接访问 |
| Link 中断 | 每个 PHY 独立 INT 引脚 | 共享 INT 引脚, 需查中断源 |
| LED 控制 | PHY 自带 LED 引脚 | 通过 Switch 寄存器控制 |
| SMI 级联 | 支持 (通过 MDO/MDI 级联) | 通常通过 Switch 内部桥接 |

### 10.3 Switch 中 PHY 的 MDIO 访问

以 KSZ9897 为例 (5+1 端口 Switch, 内置 5 个 PHY):

```c
// KSZ9897 PHY 寄存器访问
// PHY 地址映射:
//   Port 1: 0x01 (或通过全局寄存器设置)
//   Port 2: 0x02
//   Port 3: 0x03
//   Port 4: 0x04
//   Port 5: 0x05

// 方式 1: 直接通过 MDIO 访问 (如果 MDIO 连接到 Switch 的 SMI)
uint16_t ksz9897_phy_read(PHY_Device *dev, uint8_t phy_port, uint8_t reg) {
    uint16_t val;
    dev->bus->read_c22(dev->bus->bus_priv, phy_port, reg, &val);
    return val;
}

// 方式 2: 通过 Switch 内部寄存器间接访问 (通过 SPI/I2C)
// KSZ9897 通过 SPI 读写 PHY 寄存器
int ksz9897_phy_read_via_spi(uint8_t phy_port, uint8_t reg, uint16_t *val) {
    uint32_t addr;

    // KSZ9897 PHY 寄存器通过全局地址 0xN100 + phy_port 映射
    // reg 0x00 → 地址 0x0100 (Port1), 0x0200 (Port2), ...
    addr = (uint32_t)(phy_port) << 8 | reg;

    return ksz9897_spi_read(addr, 1, val);  // 1 = 16-bit
}
```

### 10.4 Switch PHY 中断处理

```c
// KSZ9897 中断系统 (通过 IRQ_N 引脚)
// 所有 PHY 和 Switch 事件共享一个中断引脚

#define KSZ9897_REG_INT_EN      0x0013   // 中断使能
#define KSZ9897_REG_INT_STAT    0x0014   // 中断状态
#define KSZ9897_INT_PHY1        (1 << 0)
#define KSZ9897_INT_PHY2        (1 << 1)
#define KSZ9897_INT_PHY3        (1 << 2)
#define KSZ9897_INT_PHY4        (1 << 3)
#define KSZ9897_INT_PHY5        (1 << 4)
#define KSZ9897_INT_SWITCH      (1 << 5)  // Switch 事件 (ACL/风暴等)

// 单个 PHY 的中断状态寄存器 (通过 PHY 地址访问)
// 不同 PHY 映射到不同偏移:
// KSZ9897_PHY1_REG_BASE = 0x0100
// KSZ9897_PHY2_REG_BASE = 0x0200
// ...

void ksz9897_irq_handler(PHY_Device *devices[], int num_phys) {
    uint16_t int_stat;

    ksz9897_spi_read(KSZ9897_REG_INT_STAT, 1, &int_stat);

    for (int i = 0; i < num_phys; i++) {
        if (int_stat & (1 << i)) {
            uint16_t phy_isr;
            // 读取对应 PHY 的中断状态
            uint32_t phy_base = (i + 1) << 8;
            ksz9897_spi_read(phy_base + 0x1E, 1, &phy_isr);  // PHY IMR
            ksz9897_spi_read(phy_base + 0x1D, 1, &phy_isr);  // PHY ISR

            if (phy_isr & (1 << 9)) {   // Link Up
                printf("Port %d: Link Up\n", i + 1);
                // 重新读取链接状态 ...
            }
            if (phy_isr & (1 << 11)) {  // Link Down
                printf("Port %d: Link Down\n", i + 1);
            }
        }
    }
}
```

---

## 十一、Switch 驱动核心 -- 转发逻辑

### 11.1 转发决策流程

```
             ┌──────────────────────┐
             │  从端口 X 收到帧      │
             └──────────┬───────────┘
                        ▼
              ┌───────────────────┐
              │  VLAN 过滤检查     │─── VLAN 不匹配 → 丢弃
              └──────────┬────────┘
                        ▼
              ┌───────────────────┐
              │ MAC 地址表查找     │
              └──────────┬────────┘
                     ┌───┴───┐
                     ▼       ▼
                 命中表项   未命中
                     │       │
                     ▼       ▼
              ┌──────────┐  ┌──────────────────┐
              │ 查端口     │  │ Flood (广播到     │
              │+ STP 检查  │  │  所有允许端口)    │
              └─────┬────┘  └────────┬─────────┘
                    ▼                ▼
              ┌───────────────────────────┐
              │ ACL 规则检查               │─── 匹配规则 → 允许/丢弃/镜像
              └─────────────┬─────────────┘
                           ▼
              ┌───────────────────────────┐
              │ 出端口队列 + 调度          │
              │ (优先级 / QOS)            │
              └─────────────┬─────────────┘
                           ▼
              ┌───────────────────────────┐
              │ 从端口 Y 发送              │
              └───────────────────────────┘
```

### 11.2 MAC 地址表 (Forwarding Database)

```c
// MAC 地址表是 Switch 的核心数据结构
// 硬件自动学习, 软件可读写 (用于静态配置)

typedef struct {
    uint8_t  mac[6];           // MAC 地址
    uint16_t fid;              // 过滤标识 (VLAN ID)
    uint8_t  port_map;         // 端口位图 (bit0=port1, ...)
    uint8_t  static_entry : 1; // 1=静态, 0=动态学习
    uint8_t  valid         : 1;
    uint8_t  age_timer     : 3; // 老化时间
} MAC_TableEntry;

// KSZ9897 MAC 地址表操作

// 读取动态学习的 MAC 地址
int ksz9897_mac_table_read(PHY_Device *dev, int entry_index, MAC_TableEntry *entry) {
    uint16_t val;

    // 1. 写索引到 MAC 表地址寄存器
    ksz9897_spi_write(KSZ9897_REG_MAC_ADDR_HI, 0x01, &entry_index);

    // 2. 读取 MAC 表
    uint8_t data[8];
    ksz9897_spi_read_buf(KSZ9897_REG_MAC_TABLE, 8, data);

    // 3. 解析条目
    for (int i = 0; i < 6; i++)
        entry->mac[i] = data[i];
    entry->port_map  = data[6] & 0x3F;  // port 1-5 + CPU
    entry->valid     = (data[7] >> 7) & 1;
    entry->static_entry = (data[7] >> 6) & 1;
    entry->fid       = (data[7] >> 0) & 0x0F;  // 假设 4-bit FID

    return 0;
}

// 添加静态 MAC 地址
int ksz9897_mac_table_add(PHY_Device *dev, MAC_TableEntry *entry) {
    uint8_t data[8];
    uint16_t val;

    for (int i = 0; i < 6; i++)
        data[i] = entry->mac[i];
    data[6] = entry->port_map & 0x3F;
    data[7] = (entry->valid << 7) | (entry->static_entry << 6) | (entry->fid & 0x0F);

    // 写入 MAC 表
    ksz9897_spi_write_buf(KSZ9897_REG_MAC_TABLE, 8, data);

    return 0;
}
```

### 11.3 老化机制

```c
// MAC 地址老化定时器
// 动态学习的地址在 TTL 到期后自动删除

typedef enum {
    MAC_AGE_5_SEC   = 0,
    MAC_AGE_10_SEC  = 1,
    MAC_AGE_30_SEC  = 2,
    MAC_AGE_60_SEC  = 3,
    MAC_AGE_300_SEC = 4,   // 5 分钟 (默认)
    MAC_AGE_DISABLE = 7,   // 禁止老化
} MAC_AgeTime;

// KSZ9897 老化时间设置
int ksz9897_set_mac_age(uint8_t seconds) {
    uint16_t age_val;
    uint16_t reg;

    // 将 seconds 转换为芯片支持的枚举值
    if (seconds <= 5)        age_val = MAC_AGE_5_SEC;
    else if (seconds <= 10)  age_val = MAC_AGE_10_SEC;
    else if (seconds <= 30)  age_val = MAC_AGE_30_SEC;
    else if (seconds <= 60)  age_val = MAC_AGE_60_SEC;
    else                     age_val = MAC_AGE_300_SEC;

    // 读取老化控制寄存器
    ksz9897_spi_read(KSZ9897_REG_AGE_CTRL, 1, &reg);
    reg &= ~0x0007;
    reg |= age_val;

    ksz9897_spi_write(KSZ9897_REG_AGE_CTRL, 1, &reg);

    return 0;
}

// 清除所有动态学习的 MAC 地址 (通常在链路变动时调用)
int ksz9897_mac_table_flush(void) {
    uint16_t reg;

    ksz9897_spi_read(KSZ9897_REG_SWITCH_CTRL, 1, &reg);
    reg |= (1 << 5);   // Flush MAC Table
    ksz9897_spi_write(KSZ9897_REG_SWITCH_CTRL, 1, &reg);

    // 等待清除完成
    delay_ms(10);

    return 0;
}
```

### 11.4 广播/未知单播/多播风暴控制

```c
// 风暴控制: 限制特定类型帧的速率

typedef struct {
    uint8_t  broadcast_limit;   // 广播帧限制 (packets/sec)
    uint8_t  unicast_limit;     // 未知单播限制
    uint8_t  multicast_limit;   // 多播限制
    uint16_t rate_limit;        // 速率限制 (kbps)
} StormControl;

// KSZ9897 风暴控制配置
int ksz9897_storm_control_set(PHY_Device *dev, StormControl *ctrl) {
    uint16_t reg;

    // 使能风暴控制
    ksz9897_spi_read(KSZ9897_REG_STORM_CTRL, 1, &reg);
    reg |= (1 << 15);   // 使能广播限制
    reg |= (1 << 14);   // 使能未知单播限制
    reg |= (1 << 13);   // 使能多播限制

    // 设置限速 (取决于芯片粒度)
    reg &= ~0x003F;
    reg |= (ctrl->rate_limit / 100) & 0x3F;  // 假设步进 100kbps

    ksz9897_spi_write(KSZ9897_REG_STORM_CTRL, 1, &reg);

    return 0;
}
```

### 11.5 端口镜像

```c
// 端口镜像: 将某端口流量复制到监控端口
// 用于网络抓包调试

typedef struct {
    uint8_t  monitor_port;    // 监控端口 (接收镜像流量)
    uint8_t  source_port;     // 被监控端口
    uint8_t  rx_mirror : 1;  // 镜像接收帧
    uint8_t  tx_mirror : 1;  // 镜像发送帧
} PortMirror;

int ksz9897_port_mirror_set(PHY_Device *dev, PortMirror *mirror) {
    uint16_t reg;

    // 读取镜像控制寄存器
    ksz9897_spi_read(KSZ9897_REG_MIRROR_CTRL, 1, &reg);
    reg &= ~0x007F;

    reg |= (mirror->monitor_port << 4);  // bit6:4 = 监控端口
    reg |= mirror->source_port;          // bit2:0 = 源端口

    if (mirror->rx_mirror) reg |= (1 << 8);   // RX 镜像
    if (mirror->tx_mirror) reg |= (1 << 9);   // TX 镜像

    ksz9897_spi_write(KSZ9897_REG_MIRROR_CTRL, 1, &reg);

    return 0;
}
```

---

## 十二、Switch VLAN 硬件实现

### 12.1 Port-based VLAN vs Tagged VLAN

```
Port-based VLAN:
  ┌────────┐      ┌────────┐
  │ Port 1 │      │ Port 2 │
  │ VLAN A │      │ VLAN B │
  └───┬────┘      └───┬────┘
      │                │
      └─── Switch ─────┘
      │                │
  ┌───┴────┐      ┌───┴────┐
  │ Port 3 │      │ Port 4 │
  │ VLAN A │      │ VLAN B │
  └────────┘      └────────┘
  Port 1 <-> Port 3
  Port 2 <-> Port 4
  Port 1 不能访问 Port 2

Tagged VLAN (802.1Q):
  帧中含有 VLAN Tag:
  DA | SA | 0x8100 | VID(12bit) | ... Data ... 

  同一端口可以属于多个 VLAN (trunk port)
```

### 12.2 VLAN 表结构 (以 KSZ9897 为例)

```c
// KSZ9897 VLAN 表 (支持 16 个 VLAN)

typedef struct {
    uint16_t vid;          // VLAN ID (0-4095)
    uint8_t  port_members; // 成员端口位图 (bit0-5)
    uint8_t  untag_ports;  // 去标签端口位图 (在这些端口去掉 VLAN Tag)
    uint8_t  fid;          // 过滤标识 (用于 MAC 地址表查询)
} VLAN_Entry;

// VLAN 表操作
int ksz9897_vlan_add(PHY_Device *dev, VLAN_Entry *vlan) {
    uint8_t data[4];

    data[0] = vlan->port_members & 0x3F;  // bit0-5
    data[1] = vlan->untag_ports & 0x3F;
    data[2] = vlan->fid & 0x0F;
    data[3] = (vlan->vid >> 8) & 0xFF;    // VID 高 8 位
    data[4] = vlan->vid & 0xFF;           // VID 低 8 位

    // 写入 VLAN 表 (KSZ9897 通过间接寻址)
    ksz9897_spi_write_buf(KSZ9897_REG_VLAN_TABLE, 5, data);

    return 0;
}

// 配置端口的 VLAN 成员关系
int ksz9897_port_vlan_config(PHY_Device *dev, uint8_t port, uint16_t pvid,
                              uint8_t member_ports, uint8_t untag_ports) {
    VLAN_Entry vlan;

    vlan.vid = pvid;
    vlan.port_members = member_ports;
    vlan.untag_ports = untag_ports;
    vlan.fid = 0;

    ksz9897_vlan_add(dev, &vlan);

    // 设置端口的 PVID (Port VLAN ID)
    ksz9897_port_pvid_set(dev, port, pvid);

    return 0;
}

// 设置端口的默认 VLAN ID (PVID)
int ksz9897_port_pvid_set(PHY_Device *dev, uint8_t port, uint16_t pvid) {
    uint16_t reg;
    uint32_t pvid_addr = KSZ9897_REG_PORT1_CTRL + ((port - 1) * 0x10);

    ksz9897_spi_read(pvid_addr + 0x00, 1, &reg);
    reg &= ~0x0FFF;
    reg |= pvid & 0x0FFF;
    ksz9897_spi_write(pvid_addr + 0x00, 1, &reg);

    return 0;
}
```

### 12.3 VLAN 过滤模式

```c
// VLAN 过滤: 决定如何处理不带 Tag 的帧

typedef enum {
    VLAN_FILTER_DISABLE = 0,    // 不过滤, 所有帧都可转发
    VLAN_FILTER_DOUBLE_CHECK,   // 双重检查 (端口 + VLAN 表)
    VLAN_FILTER_SINGLE_CHECK,   // 只检查 VLAN 表
} VLAN_FilterMode;

int ksz9897_vlan_filter_set(VLAN_FilterMode mode) {
    uint16_t reg;

    ksz9897_spi_read(KSZ9897_REG_VLAN_CTRL, 1, &reg);
    reg &= ~(3 << 12);
    reg |= (mode << 12);

    ksz9897_spi_write(KSZ9897_REG_VLAN_CTRL, 1, &reg);

    return 0;
}
```

---

## 十三、Switch ACL / 过滤规则

### 13.1 ACL (Access Control List) 原理

ACL 允许在硬件层面过滤特定流量的帧, 不占用 CPU。

```
ACL 匹配字段 (可配置):
  ┌─────┬──────┬──────┬──────┬─────┐
  │ MAC │ VLAN │ Type │ IP   │ Port│
  │ DA  │  VID │      │ Src  │     │
  └─────┴──────┴──────┴──────┴─────┘
           │
           ▼
     ┌───────────┐
     │ 规则 1     │── 不匹配 → 下一条
     └─────┬─────┘
           │ 匹配
           ▼
     ┌───────────┐
     │ 动作       │
     │ 允许/丢弃  │
     │ 镜像/重定向│
     │ 中断CPU    │
     └───────────┘
```

### 13.2 ACL 实现 (以 KSZ9897 为例)

```c
// KSZ9897 ACL 表条目

typedef struct {
    // 匹配条件
    uint8_t  smac[6];      // 源 MAC (0 = 不关心)
    uint8_t  dmac[6];      // 目的 MAC
    uint16_t vid;          // VLAN ID (0xFFFF = 不关心)
    uint16_t ether_type;   // EtherType (0 = 不关心)
    uint8_t  ingress_port; // 入端口 (0xFF = 任意)

    // 动作
    uint8_t  action;       // ACL_ACTION_DROP/ACL_ACTION_FORWARD/...
    uint8_t  prio;         // 新优先级
    uint8_t  redirect_port;// 重定向端口 (0 = 不重定向)
    uint8_t  cpu_interrupt;// 发送到 CPU 并中断

    // 控制
    uint8_t  enable;
    uint8_t  rule_id;
} ACL_Rule;

#define ACL_ACTION_FORWARD      0
#define ACL_ACTION_DROP         1
#define ACL_ACTION_REDIRECT     2
#define ACL_ACTION_MIRROR       3
#define ACL_ACTION_CPU_COPY     4  // 复制一份到 CPU
#define ACL_ACTION_CPU_TRAP     5  // 转发并中断 CPU

int ksz9897_acl_rule_add(ACL_Rule *rule) {
    uint8_t data[16];
    uint16_t reg;

    // 构建 ACL 条目数据
    // 前 12 字节: MAC 地址匹配
    // 后续字节: VLAN/Type/动作

    memset(data, 0, sizeof(data));
    memcpy(data, rule->smac, 6);
    memcpy(data + 6, rule->dmac, 6);

    // 写入 ACL 表
    ksz9897_spi_write_buf(KSZ9897_REG_ACL_TABLE(rule->rule_id), 16, data);

    // 使能规则
    ksz9897_spi_read(KSZ9897_REG_ACL_CTRL, 1, &reg);
    reg |= (1 << rule->rule_id);
    ksz9897_spi_write(KSZ9897_REG_ACL_CTRL, 1, &reg);

    return 0;
}

// 示例: 丢弃 LLDP 帧 (阻止 LLDP 穿越 Switch)
void acl_example_block_lldp(void) {
    ACL_Rule rule;

    memset(&rule, 0, sizeof(rule));
    rule.ether_type = 0x88CC;  // LLDP EtherType
    rule.action = ACL_ACTION_DROP;
    rule.enable = 1;
    rule.rule_id = 0;

    ksz9897_acl_rule_add(&rule);
    printf("ACL: LLDP packets will be dropped\n");
}

// 示例: 将特定 MAC 的流量镜像到 CPU
void acl_example_mirror_mac(uint8_t *target_mac, uint8_t monitor_port) {
    ACL_Rule rule;

    memset(&rule, 0, sizeof(rule));
    memcpy(rule.smac, target_mac, 6);  // 匹配源 MAC
    rule.action = ACL_ACTION_MIRROR;
    rule.redirect_port = monitor_port;
    rule.enable = 1;
    rule.rule_id = 1;

    ksz9897_acl_rule_add(&rule);
}
```

### 13.3 速率限制与 QoS

```c
// 端口速率限制 (Ingress / Egress)

typedef struct {
    uint8_t  port;
    uint16_t ingress_rate;    // 入方向限速 (kbps), 0=不限
    uint16_t egress_rate;     // 出方向限速 (kbps), 0=不限
    uint8_t  burst_size;      // 突发大小 (KB)
} PortRateLimit;

int ksz9897_port_rate_limit(PortRateLimit *rl) {
    uint16_t reg;
    uint32_t port_addr = KSZ9897_REG_PORT1_CTRL + ((rl->port - 1) * 0x10);

    // 入方向限速
    ksz9897_spi_read(port_addr + KSZ9897_REG_PORT_RATE_LIMIT_IN, 1, &reg);
    if (rl->ingress_rate > 0) {
        reg |= (1 << 15);  // 使能入方向限速
        reg &= ~0x3FFF;
        reg |= (rl->ingress_rate / 100) & 0x3FFF;  // 步进 100kbps
    } else {
        reg &= ~(1 << 15); // 关闭
    }
    ksz9897_spi_write(port_addr + KSZ9897_REG_PORT_RATE_LIMIT_IN, 1, &reg);

    // 出方向限速 (类似)
    ...

    return 0;
}

// 802.1p QoS (优先级队列)
// KSZ9897 每个端口有 4 个优先级队列

typedef enum {
    QOS_STRICT_PRIORITY = 0,  // 严格优先级
    QOS_WRR,                   // 加权轮询
    QOS_DWRR,                  // 差额加权轮询
} QOS_Schedule;

int ksz9897_qos_schedule_set(QOS_Schedule mode) {
    uint16_t reg;

    ksz9897_spi_read(KSZ9897_REG_QOS_CTRL, 1, &reg);
    reg &= ~(3 << 12);
    reg |= (mode << 12);

    ksz9897_spi_write(KSZ9897_REG_QOS_CTRL, 1, &reg);

    return 0;
}
```

---

## 十四、驱动性能优化与调试

### 14.1 MDIO 访问优化

```c
// 批量读取 PHY 寄存器 (比单次读取快得多)
void phy_burst_read(PHY_Device *dev, uint8_t start_reg, uint16_t *buf, int count) {
    // 有些 MDIO 控制器支持连续地址自动递增
    // Clause 45 支持更强的 burst 能力

    for (int i = 0; i < count; i++) {
        dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr,
                           start_reg + i, &buf[i]);
    }
}

// 缓存 BMSR (避免频繁读取导致 latching 问题)
typedef struct {
    uint16_t bmsr_cache;        // 上次读取的 BMSR
    uint32_t last_read_tick;    // 最后读取时间
    uint8_t  cache_valid;       // 缓存有效标志
} PHY_Cache;

int phy_read_bmsr_cached(PHY_Device *dev, PHY_Cache *cache, uint16_t *bmsr) {
    uint32_t now = HAL_GetTick();

    // 缓存 10ms 内有效
    if (cache->cache_valid && (now - cache->last_read_tick < 10)) {
        *bmsr = cache->bmsr_cache;
        return 0;
    }

    // 读两次以清除 latch
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, bmsr);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, bmsr);

    cache->bmsr_cache = *bmsr;
    cache->last_read_tick = now;
    cache->cache_valid = 1;

    return 0;
}
```

### 14.2 调试手段

```c
// PHY 寄存器 Dump (调试利器)

void phy_dump_registers(PHY_Device *dev) {
    uint16_t val;

    printf("\n===== PHY Register Dump (addr=0x%02X) =====\n", dev->phy_addr);

    for (uint8_t reg = 0x00; reg <= 0x1F; reg++) {
        if (dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, reg, &val) == 0) {
            printf("  Reg 0x%02X (0x%02d): 0x%04X", reg, reg, val);
            // 打印二进制
            printf("  [");
            for (int bit = 15; bit >= 0; bit--) {
                printf("%c", (val >> bit) & 1 ? '1' : '0');
                if (bit == 8 || bit == 4)
                    printf(" ");
            }
            printf("]\n");
        } else {
            printf("  Reg 0x%02X: <ERROR>\n", reg);
        }
    }

    printf("=========================================\n");
}

// PHY 环回 (Loopback) 测试
// 发送数据 → PHY TX → PHY 内部环回 → PHY RX → 验证接收数据

int phy_loopback_test(PHY_Device *dev) {
    uint16_t bmcr;
    int pass = 1;

    // 使能环回
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &bmcr);
    bmcr |= (1 << 14);  // Loopback
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr);

    delay_ms(10);

    // 发送测试数据 (通过 MAC 发送一个已知帧)
    // ...

    // 验证接收数据
    // ...

    // 关闭环回
    bmcr &= ~(1 << 14);
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr);

    printf("PHY Loopback Test: %s\n", pass ? "PASS" : "FAIL");
    return pass ? 0 : -1;
}
```

### 14.3 常见问题排查

| 问题 | 可能原因 | 排查方法 |
|------|----------|----------|
| Link up 但无法收发 | MDI/MDI-X 不匹配 | 检查 Auto-MDI/X 使能状态 |
| | MAC 侧速度/双工配置错 | 确认 MAC 与 PHY 配置一致 |
| | RMII REF_CLK 方向错 | RMII REF_CLK 必须由外部提供或由 PHY 提供 |
| Link 抖动 | 线缆质量差 | 检查 BMSR 的 latching 读数 |
| | PHY 电源噪声 | 加去耦电容, 检查电源纹波 |
| | 接地问题 | 确认 PCB 地线完整 |
| 协商失败 | 对端不支持协商 | 读取 ANLPAR, 如全 0 则尝试并行检测 |
| | PHY 默认配置不对 | 使用 strapping 引脚确认 |
| MDIO 通信失败 | MDIO 上拉电阻缺失 | 确认有 1.5k~10k 上拉到 3.3V |
| | MDC 频率太高 | C22 最大 2.5MHz (实际可更高但需验证) |
| | PHY 地址不对 | 读取 PHY ID 确认地址 |
| 中断不触发 | 中断极性不匹配 | 检查 PHY 是推挽/开漏, 高/低有效 |
| | 未清除中断状态 | ISR 寄存器读取即清除, 需先读取 |

### 14.4 Ring Buffer 与 DMA 的配合

```c
// 当 MAC 使用 DMA 进行收发包时, PHY 状态变化
// 需要同步通知 DMA 引擎重新配置

typedef struct {
    // MAC DMA 描述符
    void *rx_desc_ring;
    void *tx_desc_ring;
    uint32_t rx_desc_count;
    uint32_t tx_desc_count;

    // 当前配置
    uint8_t  active_speed;    // 0=10M, 1=100M, 2=1000M
    uint8_t  active_duplex;   // 0=half, 1=full
    uint8_t  hw_crc;          // 硬件 CRC 开关
    uint8_t  hw_padding;      // 硬件填充开关
} MAC_Config;

// Link 变化时的回调
void on_phy_link_change(PHY_Device *phy_dev, MAC_Config *mac_cfg) {
    // 1. 停止 DMA 收发
    mac_dma_stop(mac_cfg);

    // 2. 根据新速度重新配置 MAC
    uint16_t mac_cr;
    if (phy_dev->link.speed_100m) {
        mac_cr = MAC_CR_100M;
    } else {
        mac_cr = MAC_CR_10M;
    }

    if (phy_dev->link.full_duplex) {
        mac_cr |= MAC_CR_FULLDUPLEX;
    } else {
        mac_cr |= MAC_CR_HALFDUPLEX;
        // 半双工需要使能碰撞检测和背压
    }

    mac_configure(mac_cr);

    // 3. 重新配置 DMA 描述符 (清空缓冲区)
    mac_dma_reset_descriptors(mac_cfg);

    // 4. 重新启动 DMA
    mac_dma_start(mac_cfg);

    printf("MAC reconfigured: %s %s Duplex, DMA restarted\n",
           phy_dev->link.speed_100m ? "100M" : "10M",
           phy_dev->link.full_duplex ? "Full" : "Half");
}
```

---

## 十五、常见 PHY 芯片寄存器速查

### 15.1 LAN8720A

| 地址 | 名称 | 说明 |
|------|------|------|
| 0x00 | BMCR | 基本控制 |
| 0x01 | BMSR | 基本状态 |
| 0x02 | PHYIDR1 | PHY ID 高 |
| 0x03 | PHYIDR2 | PHY ID 低 (0xC0F1) |
| 0x04 | ANAR | 协商通告 |
| 0x05 | ANLPAR | 对端能力 |
| 0x06 | ANER | 协商扩展 |
| 0x08 | MCSR | 主/从配置 |
| 0x0B | MII Control | MII 模式控制 |
| 0x0C | MII Status | MII 状态 |
| 0x10 | ASR | 辅助状态 |
| 0x1B | MSC | 模式选择配置 (RMII 选择) |
| 0x1D | ISR | 中断状态 (读清除) |
| 0x1E | IMR | 中断屏蔽 |
| 0x1F | SCSR | 特殊控制/状态 (速度, 双工) |

### 15.2 DP83848

| 地址 | 名称 | 说明 |
|------|------|------|
| 0x00 | BMCR | 基本控制 |
| 0x01 | BMSR | 基本状态 |
| 0x02 | PHYIDR1 | PHY ID = 0x2000 |
| 0x03 | PHYIDR2 | PHY ID = 0xA0B1 |
| 0x04 | ANAR | 通告 |
| 0x05 | ANLPAR | 对端 |
| 0x06 | ANER | 扩展 |
| 0x07 | ANNPTR | Next Page |
| 0x08 | ANNPTR | Next Page |
| 0x09 | 100BASET2 | T2 控制 |
| 0x0A | 100BASET2 | T2 状态 |
| 0x0B | PCSR | PHY 控制/状态 |
| 0x0D | ICR | 中断控制 |
| 0x0E | ISR | 中断状态 |
| 0x0F | ESTATUS | 扩展状态 |
| 0x10-0x15 | 厂商自定义 | LED, 配置等 |
| 0x16 | RGMIICTL | RGMII 控制 |
| 0x17 | RGMIISR | RGMII 状态 |
| 0x18 | PHYCR | PHY 控制 |
| 0x19 | PHYSR | PHY 状态 |

### 15.3 KSZ8081

| 地址 | 名称 | 说明 |
|------|------|------|
| 0x00 | BMCR | 基本控制 |
| 0x01 | BMSR | 基本状态 |
| 0x02 | PHYIDR1 | 0x0022 |
| 0x03 | PHYIDR2 | 0x1560 |
| 0x04 | ANAR | 通告 |
| 0x05 | ANLPAR | 对端 |
| 0x06 | ANER | 扩展 |
| 0x0F | ESTATUS | 扩展状态 |
| 0x10 | PHYCTRL1 | PHY 控制 1 |
| 0x11 | PHYCTRL2 | PHY 控制 2 |
| 0x12 | PHYSTS | PHY 状态 |
| 0x13 | INT_CTRL | 中断控制 |
| 0x14 | INT_STATUS | 中断状态 |
| 0x15 | PCAPABILITY | 物理能力 |
| 0x16 | PHYCTRL | PHY 控制 (含 Auto-MDI/X) |
| 0x17 | LINKMD | LinkMD 诊断 |

### 15.4 KSZ9031 (千兆)

| 地址 | 名称 | 说明 |
|------|------|------|
| 0x00-0x0F | C22 标准 | 标准寄存器 |
| 0x0D | MMD_ACCESS | MMD 间接访问控制 |
| 0x0E | MMD_DATA | MMD 间接访问数据 |
| 0x10 | PCSCSR | 千兆 PCS 控制/状态 |
| 0x11 | 1000BASET_CTRL | 千兆基座控制 |
| 0x12 | 1000BASET_STAT | 千兆基座状态 |
| 0x13 | OMOTE | 其它模式 |
| 0x14-0x1A | 厂商自定义 | 各种配置 |
| 0x1B | RXER_CNT | RX 错误计数器 |
| 0x1C | INT_EN | 中断使能 |
| 0x1D | INT_STATUS | 中断状态 |
| 0x1E | PHY_CTRL | PHY 控制 |
| 0x1F | PHY_STAT | PHY 状态 |

注: KSZ9031 主要通过 MMD (0x0D/0x0E) 访问扩展寄存器。

---

## 附录 A: 跨芯片 PHY 驱动移植清单

从一颗 PHY 迁移到另一颗时需修改:

```c
// PHY 驱动移植检查表

// [ ] PHY_ID 宏定义
//     #define PHY_ID_LAN8720A     0x0007C0F1
//     #define PHY_ID_DP83848      0x2000A0B1

// [ ] 硬件复位时序
//     LAN8720A: TRST=10ms, TINIT=50ms
//     DP83848:  TRST=10ms, TINIT=50ms
//     KSZ9031:  TRST=10ms, TINIT=100ms

// [ ] 默认 PHY 地址
//     LAN8720A: 0x00 (AD0 引脚)
//     DP83848:  0x10 (AD0 引脚)
//     KSZ8081:  0x01 (PHYAD0-2)

// [ ] 介质接口选择方法
//     LAN8720A: MSC 寄存器 (0x1B) bit8
//     DP83848:  PHYCR 寄存器 (0x19) bit5
//     KSZ8081:  通过 strap 引脚

// [ ] Link 状态寄存器
//     LAN8720A: SCSR (0x1F) bit14=100M, bit13=FD
//     DP83848:  PHYSR (0x19) bit14=100M, bit13=FD
//     KSZ8081:  PHYSTS (0x12) bit1=100M, bit0=FD

// [ ] 中断寄存器
//     LAN8720A: ISR (0x1D), IMR (0x1E)
//     DP83848:  ISR (0x0E), ICR (0x0D)
//     KSZ8081:  INT_STATUS (0x14), INT_CTRL (0x13)

// [ ] Auto-MDI/X 支持
//     LAN8720A: 仅硬件 (strapping 引脚)
//     DP83848:  寄存器 PHYCR bit10 (MDI/MDI-X)
//     KSZ8081:  寄存器 PHYCTRL (0x16) bit7

// [ ] 千兆支持
//     KSZ9031:  需 MMD 访问 (0x0D/0x0E)
//               EEE 使能, 1000Base-T 协商
```

## 附录 B: PHY 驱动调试命令

```c
// CLI 调试命令示例 (用于串口终端)

void cli_phy_help(void) {
    printf("PHY Debug Commands:\n");
    printf("  phy id          - 读取 PHY ID\n");
    printf("  phy dump        - dump 所有寄存器\n");
    printf("  phy read <reg>  - 读取指定寄存器\n");
    printf("  phy write <reg> <val> - 写入寄存器\n");
    printf("  phy link        - 显示当前链接状态\n");
    printf("  phy loopback <0/1> - 环回测试\n");
    printf("  phy reset       - 软件复位 PHY\n");
    printf("  phy neg         - 重启自动协商\n");
}

void cli_phy_cmd(int argc, char **argv) {
    if (argc < 2) {
        cli_phy_help();
        return;
    }

    if (strcmp(argv[1], "dump") == 0) {
        phy_dump_registers(&g_phy_dev);
    }
    else if (strcmp(argv[1], "link") == 0) {
        PHY_LinkState link;
        g_phy_dev.read_link(&g_phy_dev, &link);
        printf("Link: %s\n", link.link_up ? "UP" : "DOWN");
        if (link.link_up) {
            printf("Speed: %s\n", link.speed_100m ? "100M" : "10M");
            printf("Duplex: %s\n", link.full_duplex ? "Full" : "Half");
        }
    }
    else if (strcmp(argv[1], "read") == 0 && argc == 3) {
        uint16_t val;
        uint8_t reg = strtol(argv[2], NULL, 0);
        if (g_phy_dev.bus->read_c22(g_phy_dev.bus->bus_priv,
                                     g_phy_dev.phy_addr, reg, &val) == 0) {
            printf("Reg 0x%02X = 0x%04X\n", reg, val);
        }
    }
    else if (strcmp(argv[1], "write") == 0 && argc == 4) {
        uint8_t reg = strtol(argv[2], NULL, 0);
        uint16_t val = strtol(argv[3], NULL, 0);
        g_phy_dev.bus->write_c22(g_phy_dev.bus->bus_priv,
                                  g_phy_dev.phy_addr, reg, val);
        printf("Written: Reg 0x%02X = 0x%04X\n", reg, val);
    }
    else if (strcmp(argv[1], "reset") == 0) {
        phy_soft_reset(&g_phy_dev);
        printf("PHY reset done\n");
    }
    else if (strcmp(argv[1], "neg") == 0) {
        phy_start_autoneg(&g_phy_dev);
        printf("Auto-Neg restart triggered\n");
    }
    else {
        cli_phy_help();
    }
}
```

---

## 附录 C: Beginner Learning Path for Ethernet PHY

### C.1 前置知识清单 (Prerequisites Checklist)

在开始学习 Ethernet PHY 驱动开发之前，请确保你已掌握以下基础知识：

| 编号 | 知识点 | 要求 | 自学建议 |
|------|--------|------|----------|
| 1 | **数字逻辑基础** | 理解寄存器、位操作、三态门、上拉/下拉电阻 | 复习"数字电路"前四章即可 |
| 2 | **C 语言位运算** | 熟练使用 `& \| ~ << >>`，能读写寄存器的特定位 | 练习: 写一个函数，将 uint16 变量的 bit3-bit0 设为 0b1010 |
| 3 | **MCU GPIO 编程** | 会配置 GPIO 输入/输出/推挽/开漏，会写 `HAL_GPIO_WritePin` | 在 STM32 上点个 LED 就算过关 |
| 4 | **MCU 定时器/延时** | 会使用 `HAL_Delay` 或自己实现 `delay_us` | 裸机编程基础，无难度 |
| 5 | **I2C/SPI 协议常识** | 理解时钟线、数据线、主从设备概念（MDIO 类似但更简单） | 不需精通，懂概念即可 |
| 6 | **串口 printf 调试** | 会用串口输出调试信息（非常重要！） | STM32CubeIDE 新建工程即可 |
| 7 | **Ethernet 基本概念** | 知道 MAC 地址、网线、交换机是什么 | 读一遍 Wikipedia "Ethernet" 条目 |

**自测题**：如果以下代码你能看懂每一行的作用，前置知识已经具备：

```c
uint16_t reg_val;
// 读取 PHY 寄存器 0x01
phy_read(phy_addr, 0x01, &reg_val);
// 检查 bit2 (Link Status)
if (reg_val & (1 << 2)) {
    printf("Link is UP\n");
} else {
    printf("Link is DOWN\n");
}
// 清除 bit2 以外的位，将结果写入寄存器 0x00
reg_val &= ~(1 << 15);  // 清除 bit15 (Reset)
reg_val |= (1 << 12);   // 设置 bit12 (Auto-Neg Enable)
phy_write(phy_addr, 0x00, reg_val);
```

如果以上有不明白的地方，请先补足基础知识再继续。

---

### C.2 7 天实战入门路线图 (7-Day Hands-On Learning Roadmap)

#### 第 1 天：Hello PHY -- 读取芯片 ID

**目标**：实现第一次 MDIO 通信，读取 PHY ID 寄存器，验证硬件连接是否正确。

**硬件准备**：
- STM32F407 开发板 (或任意带 GPIO 的 MCU)
- LAN8720A 模块
- 4 根杜邦线连接 MDIO/MDC/3.3V/GND

**接线**：
```
STM32F407          LAN8720A 模块
PA1  (MDC)   ----> MDC 引脚
PA2  (MDIO)  <---> MDIO 引脚
3.3V         ----> VCC
GND          ----> GND
```

**完整代码**：

```c
#include "stm32f4xx_hal.h"
#include <stdio.h>

// 引脚定义
#define MDC_PORT    GPIOA
#define MDC_PIN     GPIO_PIN_1
#define MDIO_PORT   GPIOA
#define MDIO_PIN    GPIO_PIN_2

// MDIO 时钟延迟 (1MHz)
#define MDIO_DELAY() delay_us(1)

// 延时函数 (粗略延时)
static void delay_us(uint32_t us) {
    // STM32F407 @168MHz, 约 168 个循环 = 1us
    for (uint32_t i = 0; i < us * 168; i++) {
        __NOP();
    }
}

static void delay_ms(uint32_t ms) {
    HAL_Delay(ms);
}

// 初始化 PHY 用的 GPIO
static void phy_gpio_init(void) {
    GPIO_InitTypeDef gpio = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();

    // MDC: 推挽输出
    gpio.Pin = MDC_PIN;
    gpio.Mode = GPIO_MODE_OUTPUT_PP;
    gpio.Pull = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(MDC_PORT, &gpio);
    HAL_GPIO_WritePin(MDC_PORT, MDC_PIN, GPIO_PIN_RESET);

    // MDIO: 开漏输出 (需要外部上拉电阻 1.5k-2.2k 到 3.3V)
    gpio.Pin = MDIO_PIN;
    gpio.Mode = GPIO_MODE_OUTPUT_OD;
    gpio.Pull = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(MDIO_PORT, &gpio);
    HAL_GPIO_WritePin(MDIO_PORT, MDIO_PIN, GPIO_PIN_SET);
}

// MDIO 为输出模式
static void mdio_set_output(void) {
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin = MDIO_PIN;
    gpio.Mode = GPIO_MODE_OUTPUT_OD;
    gpio.Pull = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(MDIO_PORT, &gpio);
}

// MDIO 为输入模式
static void mdio_set_input(void) {
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin = MDIO_PIN;
    gpio.Mode = GPIO_MODE_INPUT;
    gpio.Pull = GPIO_NOPULL;  // 外部已有上拉
    gpio.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(MDIO_PORT, &gpio);
}

// MDC 时钟脉冲
static void mdc_toggle(void) {
    HAL_GPIO_WritePin(MDC_PORT, MDC_PIN, GPIO_PIN_SET);
    MDIO_DELAY();
    HAL_GPIO_WritePin(MDC_PORT, MDC_PIN, GPIO_PIN_RESET);
    MDIO_DELAY();
}

// Clause 22 MDIO 读操作
static uint16_t mdio_read(uint8_t phy_addr, uint8_t reg_addr) {
    uint32_t frame;
    uint16_t data = 0;
    int i;

    // 1. PREAMBLE: 32 个 1
    mdio_set_output();
    for (i = 0; i < 32; i++) {
        HAL_GPIO_WritePin(MDIO_PORT, MDIO_PIN, GPIO_PIN_SET);
        mdc_toggle();
    }

    // 2. 构建读命令帧:
    //    ST(2)=01, OP(2)=10(read), PHYAD(5), REGAD(5), TA(2)=Z0
    //    高位在前发送
    frame = (0x01UL << 28)                // ST = 01
          | (0x02UL << 26)                // OP = 10 (读)
          | ((phy_addr & 0x1F) << 21)     // PHY 地址
          | ((reg_addr & 0x1F) << 16);    // 寄存器地址

    // 3. 发送 16 位 (ST + OP + PHYAD + REGAD = 14 位, 实际 frame 只用了高 14 位)
    //    实际发送: ST(2) + OP(2) + PHYAD(5) + REGAD(5) = 14 位
    for (i = 15; i >= 0; i--) {
        HAL_GPIO_WritePin(MDIO_PORT, MDIO_PIN,
            (frame & (1UL << i)) ? GPIO_PIN_SET : GPIO_PIN_RESET);
        mdc_toggle();
    }

    // 4. Turnaround: 第 1 位高阻 (Z), 第 2 位 PHY 驱动为 0
    mdio_set_input();               // 高阻
    mdc_toggle();                    // Z
    mdc_toggle();                    // PHY 驱动 0

    // 5. 读取 16 位数据
    for (i = 15; i >= 0; i--) {
        HAL_GPIO_WritePin(MDC_PORT, MDC_PIN, GPIO_PIN_SET);
        MDIO_DELAY();
        if (HAL_GPIO_ReadPin(MDIO_PORT, MDIO_PIN) == GPIO_PIN_SET) {
            data |= (1 << i);
        }
        HAL_GPIO_WritePin(MDC_PORT, MDC_PIN, GPIO_PIN_RESET);
        MDIO_DELAY();
    }

    mdio_set_output();
    return data;
}

// Clause 22 MDIO 写操作
static void mdio_write(uint8_t phy_addr, uint8_t reg_addr, uint16_t data) {
    uint32_t frame;
    int i;

    mdio_set_output();

    // PREAMBLE: 32 个 1
    for (i = 0; i < 32; i++) {
        HAL_GPIO_WritePin(MDIO_PORT, MDIO_PIN, GPIO_PIN_SET);
        mdc_toggle();
    }

    // ST=01, OP=01(write), PHYAD, REGAD, TA=10, DATA
    frame = (0x01UL << 28)
          | (0x01UL << 26)
          | ((phy_addr & 0x1F) << 21)
          | ((reg_addr & 0x1F) << 16)
          | (0x02UL << 14)             // TA = 10
          | (data & 0xFFFF);

    // 发送 32 位 (14 位命令 + 2 位 TA + 16 位数据)
    for (i = 31; i >= 0; i--) {
        HAL_GPIO_WritePin(MDIO_PORT, MDIO_PIN,
            (frame & (1UL << i)) ? GPIO_PIN_SET : GPIO_PIN_RESET);
        mdc_toggle();
    }
}

// 第 1 天主函数: 读取 PHY ID
int main(void) {
    HAL_Init();
    phy_gpio_init();

    printf("\r\n==================================\r\n");
    printf("   Day 1: Hello PHY\r\n");
    printf("==================================\r\n");

    uint8_t phy_addr = 0;  // LAN8720A 默认地址

    // 尝试读取 PHY ID 寄存器
    uint16_t id1 = mdio_read(phy_addr, 0x02);  // PHYIDR1
    uint16_t id2 = mdio_read(phy_addr, 0x03);  // PHYIDR2

    uint32_t phy_id = ((uint32_t)id1 << 16) | id2;

    printf("PHY ID High (Reg 0x02): 0x%04X\r\n", id1);
    printf("PHY ID Low  (Reg 0x03): 0x%04X\r\n", id2);
    printf("Full PHY ID: 0x%08X\r\n", phy_id);

    if (phy_id == 0x0007C0F1) {
        printf("*** LAN8720A 检测成功! ***\r\n");
    } else if (id1 == 0xFFFF && id2 == 0xFFFF) {
        printf("*** 错误: 读到 0xFFFF, 检查硬件连接和上拉电阻! ***\r\n");
    } else if (id1 == 0x0000 && id2 == 0x0000) {
        printf("*** 错误: 读到 0x0000, 检查 MDIO 引脚配置! ***\r\n");
    } else {
        printf("*** 未知 PHY, 检查 PHY 地址是否正确 (尝试 0x01-0x1F) ***\r\n");
    }

    while (1);
}
```

**预期输出**：
```
==================================
   Day 1: Hello PHY
==================================
PHY ID High (Reg 0x02): 0x0007
PHY ID Low  (Reg 0x03): 0xC0F1
Full PHY ID: 0x0007C0F1
*** LAN8720A 检测成功! ***
```

**故障排除 (如果输出不对)**：

| 现象 | 原因 | 解决方法 |
|------|------|----------|
| 读到 `0xFFFF` | MDIO 线上拉不足或 PHY 没上电 | 检查 MDIO 的 1.5k-2.2k 上拉到 3.3V，测量 PHY 电源 |
| 读到 `0x0000` | MDIO 引脚配置为推挽输出（不是开漏） | 修改 GPIO 模式为 `GPIO_MODE_OUTPUT_OD` |
| 读到其他值但不是 0x0007C0F1 | PHY 地址不对 | 尝试地址 1-31 (LAN8720A 的 RXER/PHYAD0 引脚决定地址) |
| 毫无输出 | 串口未初始化或 MCU 未运行 | 检查串口配置和时钟 |

**学到的东西**：
- MDIO 是类似 I2C 的同步串行总线，但更简单（单向时钟，主机控制全部时序）
- PHY ID 是识别芯片的唯一标识
- 读回 0xFFFF 通常表示总线通信失败（三态总线上的默认电平）

---

#### 第 2 天：PHY 复位序列

**目标**：掌握硬件复位和软件复位的正确时序。

**关键概念**：
- **硬件复位**：拉低 RST_N 引脚至少 10ms，释放后等待 50ms 让 PHY 内部初始化
- **软件复位**：写 BMCR(0x00) 的 bit15=1，然后轮询等待该位自清除

**完整代码**：

```c
// 第 2 天: PHY 复位
// 在前一天代码基础上添加以下函数

#define BMCR_REG        0x00
#define BMCR_RESET      (1 << 15)

// 硬件复位 (假设 PHY_RST 接 PC0)
#define RST_PORT    GPIOC
#define RST_PIN     GPIO_PIN_0

static void phy_hardware_reset(void) {
    GPIO_InitTypeDef gpio = {0};

    __HAL_RCC_GPIOC_CLK_ENABLE();
    gpio.Pin = RST_PIN;
    gpio.Mode = GPIO_MODE_OUTPUT_PP;
    gpio.Pull = GPIO_PULLUP;
    gpio.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(RST_PORT, &gpio);

    // 拉低复位引脚
    HAL_GPIO_WritePin(RST_PORT, RST_PIN, GPIO_PIN_RESET);
    delay_ms(50);   // 保持复位至少 10ms，给 50ms 留足余量
    printf("硬件复位: RST 拉低 50ms...\r\n");

    // 释放复位
    HAL_GPIO_WritePin(RST_PORT, RST_PIN, GPIO_PIN_SET);
    delay_ms(100);  // 等待 PHY 内部初始化完成
    printf("硬件复位释放，等待 100ms 初始化...\r\n");
}

// 软件复位
static int phy_soft_reset(uint8_t phy_addr) {
    uint16_t val;
    uint32_t timeout;

    printf("软件复位开始...\r\n");

    // 读 BMCR
    val = mdio_read(phy_addr, BMCR_REG);
    printf("  当前 BMCR = 0x%04X\r\n", val);

    // 设置 bit15 = 1 (触发复位)
    val |= BMCR_RESET;
    mdio_write(phy_addr, BMCR_REG, val);

    // 等待复位完成 (bit15 自清除)
    timeout = 500;  // 最多等 500ms
    while (timeout--) {
        delay_ms(1);
        val = mdio_read(phy_addr, BMCR_REG);
        if ((val & BMCR_RESET) == 0) {
            printf("  软件复位完成 (耗时 %lu ms)\r\n", 500 - timeout);
            return 0;  // 成功
        }
    }

    printf("  错误: 软件复位超时!\r\n");
    return -1;  // 失败
}

// 测试主函数
int main(void) {
    HAL_Init();
    phy_gpio_init();
    SystemClock_Config();  // 配置系统时钟

    printf("\r\n==================================\r\n");
    printf("   Day 2: PHY Reset Sequence\r\n");
    printf("==================================\r\n");

    uint8_t addr = 0;

    // 1. 硬件复位
    phy_hardware_reset();
    delay_ms(50);

    // 2. 确认 PHY 存在
    uint16_t id1 = mdio_read(addr, 0x02);
    uint16_t id2 = mdio_read(addr, 0x03);
    printf("复位后 PHY ID: 0x%04X%04X\r\n", id1, id2);

    // 3. 软件复位
    if (phy_soft_reset(addr) == 0) {
        printf("PHY 软复位成功!\r\n");
    } else {
        printf("PHY 软复位失败!\r\n");
    }

    // 4. 复位后验证 BMCR 应为默认值
    uint16_t bmcr = mdio_read(addr, BMCR_REG);
    printf("复位后 BMCR = 0x%04X (期望 0x1000 左右)\r\n", bmcr);

    while (1);
}
```

**预期输出**：
```
==================================
   Day 2: PHY Reset Sequence
==================================
硬件复位: RST 拉低 50ms...
硬件复位释放，等待 100ms 初始化...
复位后 PHY ID: 0x0007C0F1
软件复位开始...
  当前 BMCR = 0x1000
  软件复位完成 (耗时 2 ms)
PHY 软复位成功!
复位后 BMCR = 0x1000 (期望 0x1000 左右)
```

**调试技巧**：
- 如果软件复位后 BMCR 仍然是 0x8000（复位位没清除），说明要么 MDIO 写入没生效，要么 PHY 处于异常状态
- 如果在读 BMCR 时有时稳定有时读到 0xFFFF，检查 MDIO 线上拉和 MDC 频率（不要超过 2.5MHz）
- 不同 PHY 的复位时间不同：LAN8720A 最快（几 ms），KSZ9031 可能需 100ms

**学到的东西**：
- 硬件复位和软件复位是两种不同的复位方式
- BMCR 的 bit15 是"自清除"位（写 1 触发，复位完成后自动变回 0）
- 复位后 PHY 寄存器回到默认值

---

#### 第 3 天：链路状态检测

**目标**：实时监测网线插拔状态，理解 BMSR Latching 行为。

**关键知识点 -- Latching Low 机制**：
BMSR 寄存器的 bit2 (Link Status) 是一个"锁存低"位：
- 当链路正常时，bit2 = 1
- 当链路断开时，bit2 变 0 并**保持**，直到 CPU 读取该寄存器后才更新为当前值
- **必须连续读两次**：第一次清除 latch，第二次得到真实状态

```c
// 第 3 天: 链路状态检测

#define BMSR_REG        0x01
#define BMSR_LINK_STATUS  (1 << 2)
#define BMSR_AUTONEG_DONE (1 << 5)

// 读取链路状态 (正确做法: 读两次!)
static int phy_read_link(uint8_t phy_addr) {
    uint16_t val;

    // 第一次读取: 清除 latching
    mdio_read(phy_addr, BMSR_REG);

    // 第二次读取: 真实值
    val = mdio_read(phy_addr, BMSR_REG);

    return (val & BMSR_LINK_STATUS) ? 1 : 0;
}

// 读取协商完成状态
static int phy_is_autoneg_done(uint8_t phy_addr) {
    uint16_t val;

    mdio_read(phy_addr, BMSR_REG);  // 清 latch
    val = mdio_read(phy_addr, BMSR_REG);

    return (val & BMSR_AUTONEG_DONE) ? 1 : 0;
}

// 演示 Latching 现象
static void demo_latching(uint8_t phy_addr) {
    printf("\r\n--- Latching 行为演示 ---\r\n");

    uint16_t read1 = mdio_read(phy_addr, BMSR_REG);
    uint16_t read2 = mdio_read(phy_addr, BMSR_REG);

    printf("第一次读 BMSR = 0x%04X\r\n", read1);
    printf("第二次读 BMSR = 0x%04X\r\n", read2);
    printf("第一次 Link bit = %d\r\n", (read1 >> 2) & 1);
    printf("第二次 Link bit = %d (真实值)\r\n", (read2 >> 2) & 1);

    if (((read1 >> 2) & 1) == 0 && ((read2 >> 2) & 1) == 1) {
        printf("--> 看到 Latching 效果: 第一次读了 latch 的旧值 (0),\r\n");
        printf("    第二次才读到真实值 (1)。\r\n");
    }
}

// 链路监视器
static void link_monitor(uint8_t phy_addr) {
    int last_link = -1;  // 上次状态 (初始化为未定义)
    int link;
    uint32_t tick = HAL_GetTick();

    printf("\r\n链路监视器启动 (每 200ms 轮询)\r\n");
    printf("请插拔网线观察...\r\n");

    while (1) {
        link = phy_read_link(phy_addr);

        if (link != last_link) {
            uint32_t now = HAL_GetTick();
            printf("[%lu ms] 链路状态变化: %s -> %s\r\n",
                   now - tick,
                   last_link == -1 ? "?" : (last_link ? "UP" : "DOWN"),
                   link ? "UP" : "DOWN");

            if (link) {
                // 链路建立，看看协商完成了没有
                delay_ms(500);  // 等一会让协商完成
                if (phy_is_autoneg_done(phy_addr)) {
                    printf("  自动协商完成!\r\n");
                } else {
                    printf("  链路已建但协商未完成 (可能并行检测)\r\n");
                }
            }
            last_link = link;
        }

        delay_ms(200);
    }
}

int main(void) {
    HAL_Init();
    phy_gpio_init();
    SystemClock_Config();

    printf("\r\n==================================\r\n");
    printf("   Day 3: Link Status Detection\r\n");
    printf("==================================\r\n");

    uint8_t addr = 0;

    printf("PHY 地址: 0x%02X\r\n", addr);

    // 先做一次复位
    phy_soft_reset(addr);

    // 演示 Latching 行为
    demo_latching(addr);

    // 启动链路监视 (在这之前先把网线拔插几次)
    printf("\r\n按复位键重新开始链路监视...\r\n");
    delay_ms(3000);

    link_monitor(addr);

    return 0;
}
```

**预期输出**（当插拔网线时）：
```
==================================
   Day 3: Link Status Detection
==================================
PHY 地址: 0x00

--- Latching 行为演示 ---
第一次读 BMSR = 0x7824
第二次读 BMSR = 0x782D
第一次 Link bit = 1
第二次 Link bit = 1 (真实值)
--> 如果拔掉网线再读:
第一次读 BMSR = 0x7828 (Link bit = 0, 但其他位可能没变!)
第二次读 BMSR = 0x7820 (Link bit = 0)

链路监视器启动 (每 200ms 轮询)
请插拔网线观察...
[1234 ms] 链路状态变化: ? -> UP
  自动协商完成!
[5678 ms] 链路状态变化: UP -> DOWN
[7890 ms] 链路状态变化: DOWN -> UP
  自动协商完成!
```

**深入理解 Latching**：
Latching 机制是为"边沿检测"而设计的——硬件在 Link 从 Up 变 Down 时锁存 0，这样即使 Link 瞬间恢复，CPU 也能知道曾经断开过。代价是：必须读两次才能获得当前值。

**学到的东西**：
- BMSR Latching 行为是新手最常见的坑
- 链路检测是 PHY 驱动的核心功能
- 轮询间隔影响响应速度和 CPU 占用

---

#### 第 4 天：自动协商 (Auto-Negotiation)

**目标**：配置 ANAR 发布能力，启动协商，等待完成，读取协商结果。

```c
// 第 4 天: 自动协商

#define BMCR_REG            0x00
#define BMCR_AUTONEG_EN     (1 << 12)
#define BMCR_RESTART_ANEG   (1 << 9)

#define BMSR_REG            0x01
#define BMSR_AUTONEG_DONE   (1 << 5)

#define ANAR_REG            0x04
#define ANAR_100TX_FD       (1 << 8)
#define ANAR_100TX_HD       (1 << 7)
#define ANAR_10T_FD         (1 << 6)
#define ANAR_10T_HD         (1 << 5)
#define ANAR_SELECTOR       (1 << 0)  // IEEE 802.3

#define ANLPAR_REG          0x05

// 配置并启动自动协商
static int phy_start_autoneg(uint8_t phy_addr) {
    uint16_t anar, bmcr;

    // 1. 配置 ANAR: 发布本端能力
    anar = ANAR_SELECTOR       // IEEE 802.3 协议选择器
         | ANAR_100TX_FD       // 100M 全双工
         | ANAR_100TX_HD       // 100M 半双工
         | ANAR_10T_FD         // 10M 全双工
         | ANAR_10T_HD;        // 10M 半双工

    mdio_write(phy_addr, ANAR_REG, anar);
    printf("ANAR 配置: 0x%04X (已发布 10/100M FD/HD)\r\n", anar);

    // 2. 写 BMCR: 使能 Auto-Neg 并重启
    bmcr = mdio_read(phy_addr, BMCR_REG);
    bmcr |= BMCR_AUTONEG_EN | BMCR_RESTART_ANEG;
    mdio_write(phy_addr, BMCR_REG, bmcr);
    printf("BMCR 写入: 0x%04X (使能 ANEG + 重启)\r\n", bmcr);

    return 0;
}

// 等待自动协商完成 (带超时)
static int phy_wait_autoneg_done(uint8_t phy_addr, uint32_t timeout_ms) {
    uint32_t start = HAL_GetTick();

    printf("等待协商完成...\r\n");

    while ((HAL_GetTick() - start) < timeout_ms) {
        mdio_read(phy_addr, BMSR_REG);  // 清 latch
        uint16_t bmsr = mdio_read(phy_addr, BMSR_REG);

        if (bmsr & BMSR_AUTONEG_DONE) {
            printf("  协商完成! (耗时 %lu ms)\r\n", HAL_GetTick() - start);
            return 0;  // 成功
        }

        // 如果链路还没建，等一下
        if (bmsr & (1 << 2)) {  // Link up
            // 继续等协商完成
        } else {
            // 没链路的等待
        }

        delay_ms(50);
    }

    printf("  错误: 协商超时 (%lu ms)!\r\n", timeout_ms);
    return -1;
}

// 读取协商结果
static void phy_print_negotiation_result(uint8_t phy_addr) {
    uint16_t anar, anlpar;
    uint16_t common;

    // 读取本端发布的能力
    anar = mdio_read(phy_addr, ANAR_REG);

    // 读取对端能力 (Link Partner)
    mdio_read(phy_addr, BMSR_REG);  // 先清 latch
    anlpar = mdio_read(phy_addr, ANLPAR_REG);

    // 共同能力 (取交集)
    common = anar & anlpar;

    printf("\r\n--- 协商结果 ---\r\n");
    printf("本端 ANAR:  0x%04X\r\n", anar);
    printf("对端 ANLPAR: 0x%04X\r\n", anlpar);
    printf("共同能力:    0x%04X\r\n", common);

    printf("\r\n协商速度/双工:\r\n");
    if (common & ANAR_100TX_FD) printf("  [V] 100M 全双工 (首选)\r\n");
    if (common & ANAR_100TX_HD) printf("  [V] 100M 半双工\r\n");
    if (common & ANAR_10T_FD)   printf("  [V] 10M 全双工\r\n");
    if (common & ANAR_10T_HD)   printf("  [V] 10M 半双工\r\n");

    // 判断实际协商结果 (HCD 算法)
    if (common & ANAR_100TX_FD) {
        printf("\r\n实际协商: 100M 全双工\r\n");
    } else if (common & ANAR_100TX_HD) {
        printf("\r\n实际协商: 100M 半双工\r\n");
    } else if (common & ANAR_10T_FD) {
        printf("\r\n实际协商: 10M 全双工\r\n");
    } else if (common & ANAR_10T_HD) {
        printf("\r\n实际协商: 10M 半双工\r\n");
    } else {
        printf("\r\n协商失败 (无共同能力)\r\n");
    }
}

int main(void) {
    HAL_Init();
    phy_gpio_init();
    SystemClock_Config();

    printf("\r\n==================================\r\n");
    printf("   Day 4: Auto-Negotiation\r\n");
    printf("==================================\r\n");

    uint8_t addr = 0;

    // 1. 复位
    phy_soft_reset(addr);

    // 2. 启动协商
    phy_start_autoneg(addr);

    // 3. 等待完成 (最多等 5 秒)
    if (phy_wait_autoneg_done(addr, 5000) == 0) {
        // 4. 打印结果
        phy_print_negotiation_result(addr);
    } else {
        printf("协商未完成，请检查网线连接。\r\n");
    }

    while (1);
}
```

**预期输出**：
```
==================================
   Day 4: Auto-Negotiation
==================================
ANAR 配置: 0x01E1 (已发布 10/100M FD/HD)
BMCR 写入: 0x1200 (使能 ANEG + 重启)
等待协商完成...
  协商完成! (耗时 1523 ms)

--- 协商结果 ---
本端 ANAR:  0x01E1
对端 ANLPAR: 0x01E1
共同能力:    0x01E1

  [V] 100M 全双工 (首选)
  [V] 100M 半双工
  [V] 10M 全双工
  [V] 10M 半双工

实际协商: 100M 全双工
```

**故障排除**：

| 现象 | 原因 | 解决方法 |
|------|------|----------|
| 协商永远不完成 | 网线没插或对端没上电 | 插网线，确认对端设备亮了 |
| ANLPAR 全 0 | 对端不支持 Auto-Neg (老设备) | PHY 会自动回退到并行检测 |
| 协商结果总是 10M HD | 网线质量差或距离太长 | 换一根短网线试试 |
| 协商超时但 Link 是 up | 对端强制设置速度 | 读取 ANER 检查并行检测状态 |

**学到的东西**：
- ANAR 是"我说我能跑多快"，ANLPAR 是"对方说它能跑多快"
- HCD 算法取双方都支持的最高速度
- 协商需要 1-3 秒（FLP 脉冲交换需要时间）

---

#### 第 5 天：读取对端设备信息

**目标**：通过 ANLPAR 了解另一端网络设备的能力，连接到不同设备观察不同结果。

```c
// 第 5 天: 读取对端能力

// 解析 ANLPAR 并显示人类可读的信息
static void phy_print_partner_capabilities(uint8_t phy_addr) {
    uint16_t anlpar;
    int is_1000m = 0;
    uint16_t estatus;

    // 读取 ANLPAR
    mdio_read(phy_addr, BMSR_REG);  // 清 latch
    anlpar = mdio_read(phy_addr, ANLPAR_REG);

    printf("\r\n========== 对端设备能力 ==========\r\n");
    printf("ANLPAR 值: 0x%04X\r\n", anlpar);
    printf("二进制:    ");
    for (int i = 15; i >= 0; i--) {
        printf("%c", (anlpar >> i) & 1 ? '1' : '0');
        if (i == 8 || i == 4) printf(" ");
    }
    printf("\r\n\r\n");

    // 解析能力
    printf("支持的速率和双工:\r\n");

    if (anlpar & (1 << 9)) printf("  [ ] 100Base-T4 (已过时)\r\n");
    if (anlpar & (1 << 8)) printf("  [V] 100Base-TX 全双工\r\n");
    if (anlpar & (1 << 7)) printf("  [V] 100Base-TX 半双工\r\n");
    if (anlpar & (1 << 6)) printf("  [V] 10Base-T 全双工\r\n");
    if (anlpar & (1 << 5)) printf("  [V] 10Base-T 半双工\r\n");

    printf("\r\n流控能力:\r\n");
    if (anlpar & (1 << 10)) printf("  [V] PAUSE (对称流控)\r\n");
    if (anlpar & (1 << 11)) printf("  [V] ASM_DIR (不对称流控)\r\n");

    // 检查对端是否支持 Next Page
    if (anlpar & (1 << 15)) {
        printf("\r\n  [V] 支持 Next Page\r\n");

        // 检查是否有千兆能力
        estatus = mdio_read(phy_addr, 0x0F);
        if (estatus & (1 << 15)) {
            is_1000m = 1;
            printf("  [V] 1000Base-T 全双工 (千兆)\r\n");
        }
        if (estatus & (1 << 14)) {
            printf("  [V] 1000Base-T 半双工\r\n");
        }
    }

    printf("\r\n选择器字段: 0x%X (00001=IEEE 802.3)\r\n", anlpar & 0x1F);

    printf("\r\n设备类型推测:\r\n");
    if (anlpar & (1 << 10)) {
        printf("  -> 可能是交换机或路由器 (支持流控)\r\n");
    } else {
        printf("  -> 可能是 PC 或简单设备 (不支持流控)\r\n");
    }
    if (is_1000m) {
        printf("  -> 千兆设备\r\n");
    } else if (anlpar & (1 << 8)) {
        printf("  -> 百兆设备\r\n");
    } else {
        printf("  -> 十兆设备\r\n");
    }
    printf("==================================\r\n");
}

// 扫描所有 PHY 地址 (看总线上有哪些设备)
static void phy_scan_bus(void) {
    printf("\r\n--- 扫描 MDIO 总线 ---\r\n");

    for (uint8_t addr = 0; addr < 32; addr++) {
        uint16_t id1 = mdio_read(addr, 0x02);
        if (id1 == 0x0000 || id1 == 0xFFFF) {
            continue;  // 这个地址没有 PHY
        }
        uint16_t id2 = mdio_read(addr, 0x03);
        printf("地址 0x%02X: PHY ID = 0x%04X%04X\r\n", addr, id1, id2);
    }

    printf("--- 扫描结束 ---\r\n");
}

int main(void) {
    HAL_Init();
    phy_gpio_init();
    SystemClock_Config();

    printf("\r\n==================================\r\n");
    printf("   Day 5: Link Partner Info\r\n");
    printf("==================================\r\n");

    uint8_t addr = 0;

    // 先扫描总线看看有哪些 PHY
    phy_scan_bus();

    // 复位并等待链路
    phy_soft_reset(addr);

    printf("\r\n请将网线连接到不同设备观察:\r\n");
    printf("  - 交换机 (Switch)\r\n");
    printf("  - 路由器 (Router)\r\n");
    printf("  - 电脑 (PC)\r\n");
    printf("  - 另一块开发板\r\n");
    printf("每次连接后按复位键查看结果\r\n");

    delay_ms(3000);

    // 如果没连网线，等一会
    printf("\r\n等待链路建立...\r\n");
    for (int i = 0; i < 30; i++) {
        mdio_read(addr, BMSR_REG);
        uint16_t bmsr = mdio_read(addr, BMSR_REG);
        if (bmsr & (1 << 2)) {
            printf("链路已建立!\r\n");
            break;
        }
        printf(".");
        delay_ms(200);
    }
    printf("\r\n");

    // 显示对端能力
    phy_print_partner_capabilities(addr);

    while (1);
}
```

**实验建议**：
分别连接以下设备并观察 ANLPAR 的差异：

| 对端设备 | 预期 ANLPAR | 特点 |
|----------|-------------|------|
| 百兆交换机 | 0x01E1 | 支持 10/100 FD/HD |
| 千兆交换机 | 0x01E1 + ESTATUS 位 | 会额外有千兆能力位 |
| 电脑网卡 (1G) | 0x01E1 | 通常带流控标志 |
| 另一块开发板 (强制 100M FD) | 无 ANLPAR | 并行检测模式 |

**学到的东西**：
- ANLPAR 是了解对端设备最直接的窗口
- 不同设备类型有不同能力组合
- 千兆通过 Next Page 交换额外信息

---

#### 第 6 天：PHY 中断

**目标**：配置 PHY 中断，写中断服务函数，实现链路变化的事件驱动检测。

```c
// 第 6 天: PHY 中断

// LAN8720A 中断寄存器
#define LAN8720A_REG_ISR    0x1D  // 中断状态 (读即清除)
#define LAN8720A_REG_IMR    0x1E  // 中断掩码 (1=使能)

#define LAN8720A_ISR_ANEG_DONE   (1 << 12)
#define LAN8720A_ISR_LINK_DOWN   (1 << 11)
#define LAN8720A_ISR_REMOTE_FLT  (1 << 10)
#define LAN8720A_ISR_LINK_UP     (1 << 9)
#define LAN8720A_ISR_WOL         (1 << 8)

// 全局变量
static volatile int g_link_up = 0;
static volatile int g_int_occurred = 0;

// 配置 PHY 中断
static void phy_enable_interrupt(uint8_t phy_addr) {
    uint16_t mask;

    // 启用 Link Up + Link Down 中断
    mask = LAN8720A_ISR_LINK_UP | LAN8720A_ISR_LINK_DOWN;
    mdio_write(phy_addr, LAN8720A_REG_IMR, mask);

    // 读取一次 ISR 以清除任何挂起的中断
    mdio_read(phy_addr, LAN8720A_REG_ISR);

    printf("PHY 中断已使能 (监测 Link Up/Down)\r\n");
}

// 清除中断配置
static void phy_disable_interrupt(uint8_t phy_addr) {
    mdio_write(phy_addr, LAN8720A_REG_IMR, 0);
    printf("PHY 中断已禁用\r\n");
}

// PHY 中断服务函数 (在 GPIO 中断中调用)
void phy_irq_handler(uint8_t phy_addr) {
    uint16_t isr;

    // 读取中断状态 (读即清除)
    isr = mdio_read(phy_addr, LAN8720A_REG_ISR);

    g_int_occurred = 1;

    printf("\r\n[IRQ] PHY 中断触发! ISR = 0x%04X\r\n", isr);

    if (isr & LAN8720A_ISR_LINK_UP) {
        printf("[IRQ] Link UP!\r\n");

        // 等一会让协商稳定
        delay_ms(100);

        // 读取链接状态
        mdio_read(phy_addr, BMSR_REG);
        uint16_t bmsr = mdio_read(phy_addr, BMSR_REG);
        if (bmsr & (1 << 5)) {
            printf("[IRQ] 自动协商完成\r\n");
        }

        // 读取速度 (通过 LAN8720A 特殊寄存器 0x1F)
        uint16_t scsr = mdio_read(phy_addr, 0x1F);
        int speed_100 = (scsr >> 14) & 1;
        int full_duplex = (scsr >> 13) & 1;

        printf("[IRQ] 速度: %s, 双工: %s\r\n",
               speed_100 ? "100M" : "10M",
               full_duplex ? "Full" : "Half");

        g_link_up = 1;
    }

    if (isr & LAN8720A_ISR_LINK_DOWN) {
        printf("[IRQ] Link DOWN!\r\n");
        g_link_up = 0;
    }

    if (isr & LAN8720A_ISR_ANEG_DONE) {
        printf("[IRQ] 自动协商完成 (单独通知)\r\n");
    }

    if (isr & LAN8720A_ISR_REMOTE_FLT) {
        printf("[IRQ] 远端故障!\r\n");
    }
}

int main(void) {
    HAL_Init();
    phy_gpio_init();
    SystemClock_Config();

    printf("\r\n==================================\r\n");
    printf("   Day 6: PHY Interrupts\r\n");
    printf("==================================\r\n");

    uint8_t addr = 0;

    // 1. 复位并初始化
    phy_soft_reset(addr);

    // 2. 配置中断
    phy_enable_interrupt(addr);

    printf("\r\n现在插拔网线测试中断:\r\n");
    printf("(假设 PHY 的 nINT 引脚已连接到 MCU 的外部中断引脚)\r\n");
    printf("本示例中每 500ms 轮询模拟中断处理\r\n");

    // 由于本教程使用轮询方式访问 MDIO，
    // 我们模拟中断处理: 定期检查 ISR
    int last_link = -1;

    while (1) {
        // 模拟中断: 检查 ISR 是否有事件
        uint16_t isr = mdio_read(addr, LAN8720A_REG_ISR);

        if (isr != 0) {
            // 有中断事件
            printf("\r\n[轮询检测] ISR = 0x%04X\r\n", isr);

            if (isr & LAN8720A_ISR_LINK_UP) {
                printf("  -> Link UP\r\n");
                g_link_up = 1;
            }
            if (isr & LAN8720A_ISR_LINK_DOWN) {
                printf("  -> Link DOWN\r\n");
                g_link_up = 0;
            }
            if (isr & LAN8720A_ISR_ANEG_DONE) {
                printf("  -> Auto-Neg Done\r\n");
            }
        }

        // 打印状态变化
        int current_link = 0;
        mdio_read(addr, BMSR_REG);
        uint16_t bmsr = mdio_read(addr, BMSR_REG);
        current_link = (bmsr >> 2) & 1;

        if (current_link != last_link) {
            printf("链路状态: %s -> %s\r\n",
                   last_link == -1 ? "?" : (last_link ? "UP" : "DOWN"),
                   current_link ? "UP" : "DOWN");
            last_link = current_link;
        }

        delay_ms(500);
    }
}
```

**MCU 外部中断配置** (如果使用硬件中断引脚)：

```c
// 在 CubeMX 或代码中配置:
// nINT 引脚 (假设接 PB0):
//   GPIO mode: External Interrupt with Rising/Falling edge trigger
//   因为 LAN8720A nINT 是开漏输出, 低电平有效

void EXTI0_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET) {
        phy_irq_handler(0);  // 调用 PHY 中断处理
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);
    }
}
```

**学到的东西**：
- 中断比轮询更高效，但代码更复杂
- 不同 PHY 的中断寄存器布局不同
- LAN8720A 的 ISR 是"读即清除"

---

#### 第 7 天：完整 PHY 监视器

**目标**：综合前 6 天所学，做一个终端监视器，定期显示链路状态、速度、双工和对端能力。

```c
// 第 7 天: 完整 PHY 监视器

// BMSR 解析
static void parse_bmsr(uint16_t bmsr) {
    printf("  BMSR = 0x%04X\r\n", bmsr);
    printf("  Capabilities:");
    if (bmsr & (1 << 14)) printf(" 100FD");
    if (bmsr & (1 << 13)) printf(" 100HD");
    if (bmsr & (1 << 12)) printf(" 10FD");
    if (bmsr & (1 << 11)) printf(" 10HD");
    printf("\r\n");

    printf("  Link: %s  ", (bmsr >> 2) & 1 ? "UP  " : "DOWN");
    printf("ANEG: %s  ", (bmsr >> 5) & 1 ? "DONE" : "....");
    printf("Remote Fault: %s\r\n",
           (bmsr >> 4) & 1 ? "YES" : "no");
}

// 通过 LAN8720A 特殊寄存器读取速度和双工
static void read_speed_duplex(uint8_t phy_addr) {
    uint16_t scsr = mdio_read(phy_addr, 0x1F);
    int speed_100 = (scsr >> 14) & 1;
    int full_duplex = (scsr >> 13) & 1;
    int aneg_done = (scsr >> 12) & 1;

    printf("  Speed:   %s\r\n", speed_100 ? "100 Mbps" : "10 Mbps");
    printf("  Duplex:  %s\r\n", full_duplex ? "Full" : "Half");
    printf("  ANEG:    %s\r\n", aneg_done ? "Completed" : "In Progress");

    if (aneg_done) {
        uint16_t anar = mdio_read(phy_addr, ANAR_REG);
        uint16_t anlpar = mdio_read(phy_addr, ANLPAR_REG);
        printf("  ANAR:    0x%04X\r\n", anar);
        printf("  ANLPAR:  0x%04X\r\n", anlpar);
    }
}

// 清屏函数 (通过发送转义序列)
static void clear_screen(void) {
    printf("\033[2J\033[H");  // ANSI 清屏
}

// 打印标题栏
static void print_header(void) {
    printf("========================================\r\n");
    printf("   Ethernet PHY 监视器 v1.0\r\n");
    printf("   PHY: LAN8720A @ 0x%02X\r\n", 0);
    printf("   " __DATE__ " " __TIME__ "\r\n");
    printf("========================================\r\n");
}

int main(void) {
    HAL_Init();
    phy_gpio_init();
    SystemClock_Config();

    uint8_t addr = 0;
    uint32_t tick = 0;
    int prev_link = -1;
    uint32_t uptime = 0;

    // 初始化: 软复位 + 启动协商
    phy_soft_reset(addr);
    phy_start_autoneg(addr);

    printf("\r\nPHY 初始化完成，启动监视器...\r\n");
    delay_ms(1000);

    while (1) {
        uptime = HAL_GetTick() / 1000;

        // 每 1 秒刷新一次
        if (HAL_GetTick() - tick >= 1000) {
            clear_screen();
            print_header();

            printf("\r\n运行时间: %lu 秒\r\n", uptime);

            // 读两次 BMSR
            mdio_read(addr, BMSR_REG);
            uint16_t bmsr = mdio_read(addr, BMSR_REG);

            int link = (bmsr >> 2) & 1;
            parse_bmsr(bmsr);

            if (link) {
                read_speed_duplex(addr);

                // 每 10 秒打印一次对端能力
                if ((uptime % 10) == 0 && uptime > 0) {
                    printf("\r\n--- 对端能力 (每 10 秒刷新) ---\r\n");
                    uint16_t anlpar = mdio_read(addr, ANLPAR_REG);
                    printf("  100TX-FD: %s  100TX-HD: %s\r\n",
                           (anlpar >> 8) & 1 ? "Y" : "N",
                           (anlpar >> 7) & 1 ? "Y" : "N");
                    printf("  10T-FD:   %s  10T-HD:   %s\r\n",
                           (anlpar >> 6) & 1 ? "Y" : "N",
                           (anlpar >> 5) & 1 ? "Y" : "N");
                    printf("  PAUSE: %s  ASM_DIR: %s\r\n",
                           (anlpar >> 10) & 1 ? "Y" : "N",
                           (anlpar >> 11) & 1 ? "Y" : "N");
                }
            } else {
                printf("\r\n  请插入网线...\r\n");
            }

            // 链路变化时报警
            if (link != prev_link) {
                printf("\r\n  *** 链路状态变化! ***\r\n");
                prev_link = link;
            }

            printf("\r\n--- 按复位键重启 ---\r\n");

            tick = HAL_GetTick();
        }

        // 其他任务 (网络协议栈等) 可以放在这里
        // ...
    }
}
```

**预期输出**（终端监视器）：
```
========================================
   Ethernet PHY 监视器 v1.0
   PHY: LAN8720A @ 0x00
   Jun 18 2026 12:00:00
========================================

运行时间: 42 秒

  BMSR = 0x782D
  Capabilities: 100FD 100HD 10FD 10HD
  Link: UP    ANEG: DONE  Remote Fault: no
  Speed:   100 Mbps
  Duplex:  Full
  ANEG:    Completed
  ANAR:    0x01E1
  ANLPAR:  0x01E1

--- 对端能力 (每 10 秒刷新) ---
  100TX-FD: Y  100TX-HD: Y
  10T-FD:   Y  10T-HD:   Y
  PAUSE: N  ASM_DIR: N

--- 按复位键重启 ---
```

**扩展建议**：
- 增加链路抖动检测和去抖逻辑 (参考主线文档第 5.4 节)
- 保存历史状态用于诊断
- 通过串口命令交互 (参考附录 B)

---

### C.3 初学者常见错误 (Common Beginner Mistakes)

以下是从数十个 PHY 驱动开发实战中总结的高频问题：

#### 1. MDIO 缺少上拉电阻
- **错误现象**：所有 PHY 寄存器都读到 0xFFFF
- **根本原因**：MDIO 是开漏总线，必须外部上拉
- **正确做法**：在 MDIO 线上接 1.5k-2.2k 电阻到 3.3V
- **验证方法**：不接 PHY 时量 MDIO 电压，应为 3.3V

#### 2. PHY 地址搞错
- **错误现象**：读 ID 返回 0xFFFF 或 0x0000，但确认电路连接正确
- **根本原因**：PHY 地址由 strapping 引脚决定，不同模块可能不同
- **正确做法**：LAN8720A 默认地址是 **0**（RXER/PHYAD0 引脚拉低），但有些模块拉高到 **1**
- **扫描技巧**：写个循环扫 0-31 地址看哪个能返回有效 ID

#### 3. 没等 PLL 锁定就开始读
- **错误现象**：复位后立刻读寄存器，有时读到正确值，有时读 0xFFFF
- **根本原因**：PHY 内部的 PLL (锁相环) 上电后需要时间稳定
- **正确做法**：硬件复位释放后至少等 **100ms** 再开始 MDIO 通信

#### 4. BMSR Latching 导致误判
- **错误现象**：链路明明是通的，但软件判断为断开
- **根本原因**：只读了一次 BMSR，读到了 latch 的 0
- **正确做法**：BMSR 必须**连续读两次**，第二次才是当前状态

#### 5. ANLPAR 读取时机不对
- **错误现象**：ANLPAR 为 0 或不全
- **根本原因**：在协商完成前读取 ANLPAR
- **正确做法**：先确认 BMSR.bit5 (ANEG Complete) = 1，**再**读 ANLPAR

#### 6. ETH 时钟没使能
- **错误现象**：PHY 正常工作（Link 灯亮），但 MCU 的 MAC 收不到数据
- **根本原因**：RMII 模式下，REF_CLK (50MHz) 未正确配置
- **正确做法**：
  - LAN8720A：由外部 50MHz 晶振提供 REF_CLK，或由 MCU 输出
  - 确认 STM32 的 ETH 时钟在 RCC 中已使能

#### 7. 中断状态未清除
- **错误现象**：PHY 中断只触发一次，不再响应
- **根本原因**：中断服务函数中没有读取 ISR 寄存器
- **正确做法**：**每个中断处理都必须读取 ISR**（读即清除），否则 PHY 不会产生下一个中断

---

### C.4 最小硬件清单 (Minimal Hardware Required)

| 组件 | 型号 | 数量 | 预估价格 | 备注 |
|------|------|------|----------|------|
| **MCU 开发板** | STM32F407VET6 / STM32F407ZGT6 (淘宝"迷你版") | 1 块 | 30-60 元 | 自带 ETH 接口的型号最佳 |
| **PHY 模块** | LAN8720A 模块 (淘宝"LAN8720A 网络模块") | 1 块 | 5-10 元 | 带 RJ45 插座的一体模块 |
| **RJ45 插座** | 带网络变压器的 RJ45 (HR911105A 等) | 1 个 | 3-5 元 | 如模块已集成则不需 |
| **面包板** | 830 孔面包板 | 1 块 | 5-10 元 | 用于 GPIO 模拟 MDIO |
| **杜邦线** | 公对母 / 公对公 | 20 根 | 3-5 元 | 不同颜色方便区分 |
| **网线** | 超五类或六类直通线 | 1 根 | 5-10 元 | 至少 1 米 |
| **USB-TTL** | CH340G / CP2102 模块 | 1 块 | 5-10 元 | 串口输出调试信息 |
| **上拉电阻** | 2.2kΩ 电阻 | 1 个 | 0.1 元 | MDIO 上拉 (关键是必须有) |
| **可选: 交换机** | 百兆/千兆交换机 | 1 台 | 30-50 元 | 测试用对端设备 |
| **可选: USB 转网口** | USB 转 Ethernet 适配器 | 1 个 | 20-30 元 | 连电脑测试 |

**推荐购买链接**：淘宝搜索关键词 `STM32F407 最小系统板` + `LAN8720A 模块`，总成本约 50-80 元。

**接线图**：
```
STM32F407 最小板          LAN8720A 模块 + RJ45
┌─────────────┐          ┌──────────────┐
│ PA1 (MDC)   ├─────────►│ MDC          │
│ PA2 (MDIO)  ├◄────────►│ MDIO         │
│    3.3V     ├─────────►│ VCC          │
│    GND      ├─────────►│ GND          │
│ PC0 (RST)   ├─────────►│ nRST         │
└─────────────┘          │              │
                         │   RJ45 接网线│
                         └──────┬───────┘
                                │
                          [交换机/路由器/PC]
```

**注意**：LAN8720A 模块通常自带 RJ45 座和网络变压器，只需供电和 MDIO 四根线即可通信（但实际收发数据还需要 RMII 接口线）。

---

### C.5 快速参考卡 (Quick Reference Card)

#### IEEE 802.3 标准寄存器 (Clause 22)

| 地址 | 名称 | 读/写 | 关键位 |
|------|------|-------|--------|
| 0x00 | BMCR (基本控制) | R/W | bit15=Reset, bit12=ANEG_EN, bit9=Restart ANEG, bit13+6=Speed, bit8=Duplex |
| 0x01 | BMSR (基本状态) | R | bit2=Link, bit5=ANEG_DONE, bit14-11=能力 (**读两次!**) |
| 0x02 | PHYIDR1 | R | OUI 高 16 位 |
| 0x03 | PHYIDR2 | R | OUI 低 6 位 + 型号 + 版本 |
| 0x04 | ANAR | R/W | 本端发布的能力 (bit5-9, bit10-11=流控) |
| 0x05 | ANLPAR | R | 对端能力 (格式同 ANAR) |
| 0x06 | ANER | R | bit0=对端支持 ANEG, bit1=收到新页 |
| 0x0F | ESTATUS | R | bit15=1000T_FD, bit14=1000T_HD |

#### 常用 PHY 控制值速查

```c
// BMCR 典型值
#define BMCR_RESET             0x8000  // 软件复位
#define BMCR_LOOPBACK          0x4000  // 环回模式
#define BMCR_100M_FD           0x2100  // 强制 100M 全双工 (不推荐)
#define BMCR_10M_FD            0x0100  // 强制 10M 全双工
#define BMCR_AUTONEG           0x1000  // 使能自动协商
#define BMCR_AUTONEG_RESTART  0x1200  // 使能 + 重启协商

// ANAR 典型值
#define ANAR_ALL_10_100        0x01E1  // 发布所有 10/100M 能力
#define ANAR_100FD_ONLY        0x0181  // 只发布 100M 全双工
#define ANAR_10HD_ONLY         0x0021  // 只发布 10M 半双工

// BMSR 检查
#define BMSR_CHECK_LINK(bmsr)       ((bmsr >> 2) & 1)
#define BMSR_CHECK_ANEG_DONE(bmsr)  ((bmsr >> 5) & 1)
```

#### 快速 MDIO 读写函数 (复制即用)

```c
// === 最小化 MDIO 驱动 (Cortex-M, GPIO 模拟) ===
// 只需要实现以下两个函数就能开始调试 PHY:

#define MDIO_DELAY()   { int _d; for(_d=0;_d<10;_d++) __NOP(); }

static void mdc_tick(void) {
    MDC_GPIO->BSRR = MDC_PIN;     // 时钟高
    MDIO_DELAY();
    MDC_GPIO->BSRR = MDC_PIN << 16; // 时钟低
    MDIO_DELAY();
}

// 读 PHY 寄存器 (返回值: 16 位数据)
uint16_t phy_read(uint8_t phy_addr, uint8_t reg) {
    uint16_t data = 0;
    // PRE: 32 个 1
    MDIO_GPIO->MODER |= MDIO_PIN;  // 输出
    for (int i = 0; i < 32; i++) {
        MDIO_GPIO->BSRR = MDIO_PIN; mdc_tick();
    }
    // 帧: ST(01) + OP(10) + PHYAD(5) + REGAD(5)
    uint32_t f = (1<<28)|(2<<26)|((phy_addr&31)<<21)|((reg&31)<<16);
    for (int i = 15; i >= 0; i--) {
        if (f & (1<<i)) MDIO_GPIO->BSRR = MDIO_PIN;
        else MDIO_GPIO->BSRR = MDIO_PIN << 16;
        mdc_tick();
    }
    // TA: 高阻 + 采样
    MDIO_GPIO->MODER &= ~MDIO_PIN;  // 输入
    mdc_tick(); mdc_tick();
    // 数据
    for (int i = 15; i >= 0; i--) {
        MDIO_GPIO->BSRR = MDC_PIN; MDIO_DELAY();
        if (MDIO_GPIO->IDR & MDIO_PIN) data |= (1 << i);
        MDIO_GPIO->BSRR = MDC_PIN << 16; MDIO_DELAY();
    }
    return data;
}

// 写 PHY 寄存器
void phy_write(uint8_t phy_addr, uint8_t reg, uint16_t data) {
    MDIO_GPIO->MODER |= MDIO_PIN;  // 输出
    for (int i = 0; i < 32; i++) {
        MDIO_GPIO->BSRR = MDIO_PIN; mdc_tick();
    }
    uint32_t f = (1<<28)|(1<<26)|((phy_addr&31)<<21)|((reg&31)<<16)
                |(2<<14)|(data);
    for (int i = 31; i >= 0; i--) {
        if (f & (1<<i)) MDIO_GPIO->BSRR = MDIO_PIN;
        else MDIO_GPIO->BSRR = MDIO_PIN << 16;
        mdc_tick();
    }
}
```

#### 调试速查

```
读到 0xFFFF → 检查 MDIO 上拉电阻 (1.5k-2.2k)
读到 0x0000 → 检查 MDIO GPIO 模式 (应为开漏 OD)
读 BMSR 异常 → 记得读两次!
协商失败   → 检查 ANAR 配置 + 等 BMSR.bit5
中断不触发 → 检查 ISR 是否已读取清除
```

> **学习建议**：不要跳过任何一个 Day。每个 Day 的代码都是在前一天基础上增加的。把 7 天的代码顺序合并，就是一个小型 PHY 驱动框架。在此基础上再阅读主文档的深入内容，事半功倍。

---

## Appendix D: Real-World PHY Debugging Case Studies

> 本章收录了 7 个真实项目中遇到的 PHY 硬件/驱动故障案例。每个案例都按照"症状 -- 现象 -- 根因分析 -- 修复 -- 预防"的完整流程展开。目标是用实际踩过的坑让初学者少走弯路。

---

### Case 1: Link LED 亮但 Ping 不通

#### 症状

RJ45 插座上的绿色 Link LED 正常点亮（看起来链路已建立），但 `ping 192.168.1.1` 全部超时，`arp -a` 看不到对端 MAC 地址。用串口调试看到 PHY 的 BMSR 寄存器 bit2=1（Link UP）。

#### 看到的现场

串口输出：

```
--- PHY Register Dump (addr=0x00) ---
Reg 0x00 (BMCR):   0x1140   [0001 0001 0100 0000]
Reg 0x01 (BMSR):   0x782D   [0111 1000 0010 1101]
  Link Status: UP (bit2=1)
  Auto-Neg Done: YES (bit5=1)
Reg 0x04 (ANAR):   0x01E1   [0000 0001 1110 0001]
Reg 0x05 (ANLPAR): 0x01E1   [0000 0001 1110 0001]
```

BMSR 显示链路已建立，协商已完成，双方能力一致（0x01E1 表示 10/100M FD/HD 全部支持）。看起来一切正常，但就是 Ping 不通。

更令人困惑的是，Wireshark 在 PC 端抓包**能看到**开发板发出的 ARP 请求，但 PC 的 ARP 响应开发板收不到。

#### 根因分析

检查 MAC 侧的配置。在 STM32 中，MAC 控制器需要根据 PHY 的协商结果独立配置速度/双工。如果 MAC 和 PHY 的速度/双工配置不一致，数据帧会损坏。

关键寄存器对比：

```c
// 读取 PHY 的协商结果
uint16_t scsr = mdio_read(0x00, 0x1F);  // LAN8720A SCSR
// scsr = 0x5000
// bit14=1 → 100M
// bit13=0 → 半双工 !!! 问题在这里 !!!
// bit12=1 → 协商完成

// 检查 MAC 配置 (STM32 ETH_MACCR)
uint32_t maccr = ETH->MACCR;
// maccr = 0x00000004 (!!!)
// bit14=0 → 未设 DM (全双工)
// bit13=0 → FES=0 (10M 模式)
// MAC 配置为 10M 半双工，但 PHY 协商结果是 100M 半双工！
```

**问题所在**：PHY 通过 ANEG 协商到 100M 半双工，但 MAC 控制器配置代码中有一个 bug：

```c
// ===== 错误的 MAC 配置代码 =====
void mac_config_speed_duplex(uint8_t speed_100m, uint8_t full_duplex) {
    uint32_t reg = ETH->MACCR;

    if (speed_100m) {
        reg |= ETH_MACCR_FES;       // 100M
    } else {
        reg &= ~ETH_MACCR_FES;      // 10M
    }

    if (full_duplex) {
        reg |= ETH_MACCR_DM;        // 全双工
    }

    ETH->MACCR = reg;
    // 错误: 没有清除 DM 位就设置!
    // 如果 full_duplex=0, 本应清除 DM 位, 但代码没有清除!
}
```

正确做法：

```c
// ===== 正确的 MAC 配置代码 =====
void mac_config_speed_duplex(uint8_t speed_100m, uint8_t full_duplex) {
    uint32_t reg = ETH->MACCR;

    // 清除速度和双工位 (先清零再设置)
    reg &= ~(ETH_MACCR_FES | ETH_MACCR_DM);

    if (speed_100m) {
        reg |= ETH_MACCR_FES;       // 100M
    }
    // else: 10M (FES=0, 默认)

    if (full_duplex) {
        reg |= ETH_MACCR_DM;        // 全双工
    }
    // else: 半双工 (DM=0)

    ETH->MACCR = reg;
}
```

修复后，百兆全双工模式下的 MACCR 正确值应为 `0x00008014`（DM=1, FES=1）。

#### 验证 MAC 与 PHY 是否匹配的方法

手动对比以下配置：

```
MAC 侧:                              PHY 侧:
ETH->MACCR.DM (bit14)  = 1  ←→  SCSR.bit13 (Full Duplex) = 1
ETH->MACCR.FES (bit13) = 1  ←→  SCSR.bit14 (100M)       = 1
```

如果有一侧不匹配，就是问题。

#### 预防

1. **MAC 配置使用"先清零再设置"模式**，不要依赖寄存器的默认值。
2. **在驱动初始化时，增加一致性检查**：读取 PHY 协商结果后，打印 MAC 和 PHY 两边的配置做对比。
3. **使用环回测试验证**：先做 PHY 环回（BMCR bit14=1），确认 MAC→PHY→MAC 通路正常。

```c
// 一致性检查函数
int phy_mac_consistency_check(PHY_Device *dev) {
    uint16_t scsr = mdio_read(dev->phy_addr, 0x1F);
    uint32_t maccr = ETH->MACCR;

    int phy_100m  = (scsr >> 14) & 1;
    int phy_fd    = (scsr >> 13) & 1;
    int mac_100m  = (maccr >> 13) & 1;  // FES
    int mac_fd    = (maccr >> 14) & 1;  // DM

    printf("PHY: %s %s\n", phy_100m ? "100M" : "10M", phy_fd ? "FD" : "HD");
    printf("MAC: %s %s\n", mac_100m ? "100M" : "10M", mac_fd ? "FD" : "HD");

    if (phy_100m != mac_100m || phy_fd != mac_fd) {
        printf("错误: PHY 和 MAC 速度/双工不匹配!\n");
        return -1;
    }
    return 0;
}
```

---

### Case 2: 间歇性 Link 断开

#### 症状

设备运行中，每隔几秒到几十秒 Link 灯就会灭一下再亮。Ping 对端时出现间歇性 `Request timed out`，不稳定。

#### 看到的现场

串口监视器输出（每秒轮询一次）：

```
[T=0s]  Link UP   [BMSR=0x782D]
[T=1s]  Link UP   [BMSR=0x782D]
[T=2s]  Link UP   [BMSR=0x7829]   ← bit2=0! Link 刚掉又恢复
[T=3s]  Link UP   [BMSR=0x782D]
...
[T=17s] Link UP   [BMSR=0x7829]   ← 又掉一次
[T=18s] Link UP   [BMSR=0x782D]
```

注意 BMSR 值的变化：`0x782D`（Link UP） vs `0x7829`（Link 刚恢复，但 Latching 暴露了曾经断过）。因为 BMSR bit2 是 Latching Low，0x7829 表示这个寄存器的 bit2 在读取之前曾经为 0（链路断过）。

用示波器测量 PHY 的电源引脚（3.3V 和 1.2V 核心电压）：

```
示波器截图描述：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: PHY 3.3V 电源  (500mV/div, AC耦合)
CH2: Link LED 信号  (2V/div)

时间轴: 200ms/div

CH1 波形: 基准 3.3V 上叠加了约 200mVpp 的纹波
         在 Link 断开瞬间 (CH2 下降沿)，CH1 上
         可以看到一个约 400mV 的电压跌落 (droop)
         
         纹波频率约 100Hz (50Hz 全波整流的残留)
         在电压跌落时，3.3V 最低到约 2.85V
         
CH2 波形: 正常时高电平 (3.3V)
         断开时变为高阻 (被 LED 拉低)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 根因分析

PHY 对电源噪声非常敏感。LAN8720A 的 3.3V 电源纹波要求不超过 ±50mV。现场实测纹波达 200mVpp，远超规格。

进一步检查 PCB，发现 PHY 的电源去耦电容配置如下：

```
问题 PCB 的 PHY 电源布局：
  ┌─────────────────────────────────────┐
  │  3.3V 电源                            │
  │    │                                   │
  │    └─── 10μF 电解电容 (距离 PHY 5cm)    │
  │    └─── 0.1μF 瓷片电容 (距离 PHY 3cm)   │
  │         ↓ 电容离 PHY 太远，引线电感大    │
  │    PHY_VDD                              │
  └─────────────────────────────────────┘
```

去耦电容离 PHY 引脚太远（超过 1cm），导致引线电感过大，高频噪声无法有效滤波。当 PHY 发射数据时瞬间电流增大，电源电压跌落，PHY 内部电路复位或 PLL 失锁，导致 Link 断开。

#### 修复

在 PHY 电源引脚旁边紧贴放置合适的去耦电容：

```
修复后的 PCB 电源布局：
  ┌─────────────────────────────────────┐
  │  3.3V 电源                            │
  │    │                                   │
  │    ├─── 100nF (0402) ← 紧贴 PHY 引脚   │
  │    ├─── 10nF  (0402) ← 紧贴 PHY 引脚   │
  │    ├─── 4.7μF (0603) ← 距离 < 5mm     │
  │    └─── 10μF 电解    ← 在 PCB 背面     │
  │         ↓                               │
  │    PHY_VDD ← 电源引脚和电容之间走线 ≤2mm │
  └─────────────────────────────────────┘
```

修复后示波器测量：

```
示波器截图描述：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: PHY 3.3V 电源  (500mV/div, AC耦合)

时间轴: 200ms/div

波形: 纹波从 200mVpp 降至约 20mVpp
      Link 断开现象完全消失
      电压跌落从 400mV 降至 <50mV
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

量产验证：连续运行 72 小时，Link 无一次断开。

#### 预防

1. **去耦电容布局规则**：每个电源引脚配一个 100nF 陶瓷电容，距离引脚走线不超过 2mm。电容的接地端尽量短接到地平面。
2. **电源纹波要求**：PHY 供电纹波控制在 ±5%（3.3V 电源不超过 165mVpp，最好 <50mVpp）。
3. **如需长距离供电**：在 PHY 旁边加一个 LDO（低压差线性稳压器）二次稳压。
4. **软件去抖**：即使硬件解决了，软件也要做去抖处理（参考第 5.4 节），防止偶发误报。

---

### Case 3: MDIO 部分寄存器读到 0xFFFF

#### 症状

代码读取 PHY 寄存器时，有些寄存器（如 0x00-0x0F）能正常返回有效值，但另一些寄存器（如 0x10-0x1F）返回 `0xFFFF`。

#### 看到的现场

串口寄存器 Dump：

```
===== PHY Register Dump (addr=0x00) =====
Reg 0x00: 0x3100  [0011 0001 0000 0000]  ← OK
Reg 0x01: 0x782D  [0111 1000 0010 1101]  ← OK
Reg 0x02: 0x0007  [0000 0000 0000 0111]  ← OK
Reg 0x03: 0xC0F1  [1100 0000 1111 0001]  ← OK
Reg 0x04: 0x01E1  [0000 0001 1110 0001]  ← OK
Reg 0x05: 0x01E1  [0000 0001 1110 0001]  ← OK
...
Reg 0x0F: 0x0000  [0000 0000 0000 0000]  ← OK

Reg 0x10: 0xFFFF  [1111 1111 1111 1111]  ← 异常!
Reg 0x11: 0xFFFF  [1111 1111 1111 1111]  ← 异常!
Reg 0x12: 0xFFFF  [1111 1111 1111 1111]  ← 异常!
...
Reg 0x1F: 0xFFFF  [1111 1111 1111 1111]  ← 异常!
```

令人困惑的是，有些 `0xFFFF` 读取也会间歇性返回未知值，但 `0x00-0x0F` 始终正常。

#### 根因分析

**根本原因**：该 PHY 芯片（LAN8720A）的寄存器地址空间没有全部实现。根据数据手册，LAN8720A 只实现了以下寄存器：

```
LAN8720A 寄存器映射 (已实现):
┌────────┬──────────────────────┐
│ 0x00   │ BMCR (基本控制)       │ ← 标准
│ 0x01   │ BMSR (基本状态)       │ ← 标准
│ 0x02   │ PHYIDR1              │ ← 标准
│ 0x03   │ PHYIDR2              │ ← 标准
│ 0x04   │ ANAR                 │ ← 标准
│ 0x05   │ ANLPAR               │ ← 标准
│ 0x06   │ ANER                 │ ← 标准
│ 0x08   │ MCSR (主从配置)       │ ← 可选
│ 0x0B   │ MII Control          │ ← 可选
│ 0x0C   │ MII Status           │ ← 可选
│ 0x10   │ ASR (辅助状态)        │ ← 厂商特有
│ 0x1B   │ MSC (模式选择)        │ ← 厂商特有
│ 0x1D   │ ISR (中断状态)        │ ← 厂商特有
│ 0x1E   │ IMR (中断屏蔽)        │ ← 厂商特有
│ 0x1F   │ SCSR (特殊控制/状态)   │ ← 厂商特有
├────────┼──────────────────────┤
│ 其它   │ 未实现 → 读回 0xFFFF  │ ← 这是规范行为!
└────────┴──────────────────────┘
```

读取未实现的寄存器时，MDIO 总线上的 PHY 不会驱动数据线，MDIO 的上拉电阻将数据线拉到高电平，因此读到 `0xFFFF`。

文档中关于寄存器范围的说明：

```
IEEE 802.3 Clause 22:
- 地址 0x00-0x0F: 基本寄存器 (前 16 个)
  - 0x00-0x08: 必须实现
  - 0x09-0x0F: 可选的扩展寄存器
- 地址 0x10-0x1F: 厂商自定义 (每个芯片不同)
  - 即使地址存在，不同芯片实现的寄存器也不同
  - 读未实现寄存器 = 0xFFFF
```

#### 修复

修改寄存器 Dump 函数，只读取芯片支持的寄存器范围：

```c
// LAN8720A 支持的寄存器列表
static const uint8_t lan8720a_valid_regs[] = {
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06,
    0x08,
    0x0B, 0x0C,
    0x10,
    0x1B, 0x1D, 0x1E, 0x1F
};
#define LAN8720A_VALID_REGS_COUNT (sizeof(lan8720a_valid_regs) / sizeof(lan8720a_valid_regs[0]))

void phy_dump_registers_safe(PHY_Device *dev) {
    uint16_t val;
    uint16_t id1, id2;

    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x02, &id1);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x03, &id2);
    uint32_t phy_id = ((uint32_t)id1 << 16) | id2;

    const uint8_t *regs;
    int count;

    // 根据 PHY ID 选择对应的寄存器列表
    if (phy_id == 0x0007C0F1) {         // LAN8720A
        regs = lan8720a_valid_regs;
        count = LAN8720A_VALID_REGS_COUNT;
    } else if (phy_id == 0x2000A0B1) {  // DP83848
        regs = dp83848_valid_regs;
        count = DP83848_VALID_REGS_COUNT;
    } else {
        // 未知 PHY: 扫描全部 0x00-0x1F, 但跳过 0xFFFF
        printf("Unknown PHY, scanning all registers...\n");
        for (uint8_t r = 0; r <= 0x1F; r++) {
            dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, r, &val);
            if (val != 0xFFFF) {
                printf("  Reg 0x%02X: 0x%04X\n", r, val);
            }
        }
        return;
    }

    printf("Registers for PHY ID 0x%08X:\n", phy_id);
    for (int i = 0; i < count; i++) {
        dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, regs[i], &val);
        printf("  Reg 0x%02X: 0x%04X\n", regs[i], val);
    }
}
```

#### 预防

1. **永远不要假设所有 0x00-0x1F 寄存器都可用**。查阅数据手册的寄存器映射表，只读取已实现的寄存器。
2. **如果读到 0xFFFF**，先确认这个寄存器在数据手册中是否存在。不要盲目认为 PHY 坏了。
3. **写一个芯片特定的寄存器列表**，用 PHY ID 自动选择，方便移植。
4. **通用扫描工具应跳过 0xFFFF 寄存器**，但仍需列出地址以方便检查。

---

### Case 4: Auto-Negotiation 永不完成

#### 症状

PHY 初始化代码启动 Auto-Neg 后，BMSR.bit5（Auto-Neg Complete）永远不置位。等待 4 秒后超时，Link 始终建立不起来。

#### 看到的现场

串口输出：

```
=== PHY Auto-Neg Test ===
PHY ID: 0x0007C0F1 (LAN8720A)
ANAR 配置: 0x01E1 (发布 10/100M FD/HD)
BMCR 写入: 0x1200 (使能 ANEG + 重启)
等待协商完成...
[T=0ms]   BMSR=0x7828  ANEG_DONE=0  Link=UP    ← Link 已经 UP!
[T=200ms] BMSR=0x7828  ANEG_DONE=0  Link=UP    ← 但协商始终不完成
[T=400ms] BMSR=0x7828  ANEG_DONE=0  Link=UP
[T=600ms] BMSR=0x7828  ANEG_DONE=0  Link=UP
...
[T=4000ms] 错误: 协商超时!
```

注意：Link 已经建立（BMSR bit2=1），但 ANEG_DONE（bit5）始终为 0。这说明物理连接是通的，但自动协商协议没有完成。

读取 ANER（Auto-Neg Expansion Register, 0x06）：

```c
uint16_t aner = mdio_read(0x00, 0x06);
// aner = 0x0011
// bit4 (PDF, Parallel Detection Fault) = 1  ← 关键线索!
// bit0 (LP_AUTO_NEG_ABLE) = 1
```

PDF=1 表示并行检测失败 —— PHY 检测到线缆上有信号（Link 脉冲），但是无法通过并行检测确定速度。

进一步读取 ANLPAR 看看对端通告了什么能力：

```c
mdio_read(0x00, 0x01);  // 清 BMSR latch
uint16_t anlpar = mdio_read(0x00, 0x05);
// anlpar = 0x0000  (!!!)  对端没有发布任何能力
```

ANLPAR 全 0，说明对端根本不支持 Auto-Negotiation。对端是一个老旧设备（或配置为强制模式的设备），只发送 NLP（Normal Link Pulses）保持链路，但不发送 FLP（Fast Link Pulses）进行协商。

#### 根因分析

完整诊断流程：

```
检查 BMSR.bit5 (ANEG_DONE) = 0?
    │
    ├── 读取 ANER (0x06)
    │     │
    │     ├── bit4 (PDF)=1? → 并行检测失败
    │     │     │              对端可能不支持 Auto-Neg
    │     │     │
    │     │     └── 读取 ANLPAR (0x05)
    │     │           │
    │     │           ├── ANLPAR = 0x0000?
    │     │           │     → 确认: 对端不支持 Auto-Neg
    │     │           │
    │     │           └── ANLPAR ≠ 0?
    │     │                 → PDF 可能是其他原因 (线缆噪声)
    │     │
    │     ├── bit0=0? → 对端不支持 ANEG
    │     │     → 需要强制模式 (parallel detection fallback)
    │     │
    │     └── 正常 → 继续等待，或检查硬件
    │
    └── 链路是否建立 (BMSR.bit2=1)?
          │
          ├── Link=1, ANEG_DONE=0 → 典型"对端不支持 ANEG"场景
          │
          └── Link=0 → 物理层问题 (线缆/连接器/供电)
```

#### 修复

回退到强制模式（Parallel Detection Fallback）。当 Auto-Neg 超时且 Link 已建立时，强制设置速度和双工：

```c
// Auto-Neg 回退逻辑
#define AUTONEG_TIMEOUT_MS  4000   // 协商超时时间

int phy_autoneg_with_fallback(PHY_Device *dev) {
    uint32_t start = HAL_GetTick();
    uint16_t bmsr, aner;

    printf("启动 Auto-Neg...\n");

    // 1. 先尝试 Auto-Neg
    phy_start_autoneg(dev);

    // 2. 等待协商完成
    while ((HAL_GetTick() - start) < AUTONEG_TIMEOUT_MS) {
        dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);
        dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);

        if (bmsr & (1 << 5)) {  // ANEG_DONE
            printf("Auto-Neg 成功完成!\n");
            return 0;  // 正常完成
        }

        // 检查是否进入了并行检测模式
        if (bmsr & (1 << 2)) {  // Link UP
            dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x06, &aner);
            if (aner & (1 << 4)) {  // PDF = Parallel Detection Fault
                printf("并行检测失败 (PDF=1), 尝试强制模式...\n");
                break;
            }
        }

        delay_ms(50);
    }

    // 3. 回退到强制模式
    printf("Auto-Neg 超时，回退到强制模式...\n");
    return phy_force_mode(dev);
}

// 强制设置速度和双工
int phy_force_mode(PHY_Device *dev) {
    uint16_t bmcr;

    // 先读当前 BMCR
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, &bmcr);

    // 清除 Auto-Neg 相关位
    bmcr &= ~((1 << 12) | (1 << 9) | (1 << 13) | (1 << 6) | (1 << 8));

    // 强制设置为 100M 全双工 (最常见场景)
    // bit13=1, bit6=0 → 100M
    // bit8=1 → 全双工
    bmcr |= (1 << 13);   // Speed(LSB) = 1
    bmcr |= (1 << 8);    // Duplex = 1
    // bit6=0 → Speed(MSB) = 0
    // 组合: 01 → 100M

    // 关键: 关闭 Auto-Neg 使能位
    bmcr &= ~(1 << 12);  // Auto-Neg Enable = 0

    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr);

    printf("强制模式: BMCR=0x%04X (100M 全双工, Auto-Neg 关闭)\n", bmcr);

    // 等待链路稳定
    delay_ms(300);

    // 验证
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);

    if (bmsr & (1 << 2)) {
        printf("强制模式链路已建立!\n");
        dev->link.link_up = 1;
        dev->link.speed_100m = 1;
        dev->link.full_duplex = 1;
        dev->link.autoneg_done = 0;  // 标记为非协商模式
        return 0;
    }

    // 如果 100M FD 不行，尝试其他组合
    printf("100M FD 失败，尝试 10M HD...\n");
    bmcr = 0x0000;  // 10M 半双工 (复位后默认)
    dev->bus->write_c22(dev->bus->bus_priv, dev->phy_addr, 0x00, bmcr);
    delay_ms(300);

    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);
    dev->bus->read_c22(dev->bus->bus_priv, dev->phy_addr, 0x01, &bmsr);

    if (bmsr & (1 << 2)) {
        printf("10M HD 模式链路已建立!\n");
        dev->link.link_up = 1;
        dev->link.speed_100m = 0;
        dev->link.full_duplex = 0;
        dev->link.autoneg_done = 0;
        return 0;
    }

    printf("强制模式失败: 所有组合均无法建立链路\n");
    return -1;
}
```

#### 预防

1. **Auto-Neg 永远设超时**，不要无限等待。3-4 秒是合理的超时值。
2. **检查 ANER 的 PDF 位**：在等待协商过程中定期检查 PDF（bit4），如果检测到 PDF，立即回退到强制模式，不要等到超时。
3. **对端能力检查**：在启动 Auto-Neg 之前，可以读取 BMSR.bit3（Auto-Neg Ability），如果该位为 0，说明本 PHY 不支持协商，直接强制模式。
4. **产品配置策略**：对于已知连接固定设备的场景（如工业设备连接 PLC），可以直接强制模式，跳过协商过程，节省 2-3 秒的启动时间。

---

### Case 5: PHY 中断风暴导致 MCU 卡死

#### 症状

使能 PHY 中断后，MCU 不断进入中断服务函数，无法执行正常的主循环代码。程序看起来"卡死"了，串口输出停止，但用调试器暂停可以看到 CPU 总是在 `EXTI_IRQHandler` 中。

#### 看到的现场

调试器暂停后的 Call Stack（调用栈）：

```
HardFault_Handler
  └── EXTI0_IRQHandler          ← 反复进入
        └── phy_irq_handler
              └── mdio_read     ← 可能在这里异常
```

或者没有 HardFault，但：

```
Breakpoint at main_loop() line 42:
  从未命中! 程序一直在中断里循环
```

检查串口输出（如果能输出的话）：

```
[IRQ] PHY 中断触发! ISR = 0xFFFF
[IRQ] PHY 中断触发! ISR = 0xFFFF
[IRQ] PHY 中断触发! ISR = 0xFFFF
[IRQ] PHY 中断触发! ISR = 0xFFFF   ← 无限重复!
```

ISR 值为 0xFFFF，意味着所有中断源都 pending。

#### 根因分析

**问题代码**（错误的中断服务函数写法）：

```c
// ===== 错误的中断服务函数 =====
// 问题: 使用电平触发中断，但在 ISR 中未清除中断源

void EXTI0_IRQHandler(void) {
    // 检查中断标志
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET) {
        phy_irq_handler(0);           // 调用 PHY 中断处理
        // __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);   ← 忘记清除 MCU 侧中断!
    }
}

void phy_irq_handler(uint8_t phy_addr) {
    // 问题: 没有读取 ISR 寄存器清除 PHY 中断!
    // 只是打印了一下就返回了
    printf("[IRQ] PHY interrupt!\n");

    // 读取 BMSR 检查链路状态
    uint16_t bmsr = mdio_read(phy_addr, 0x01);  // 只读了一次!
    // 应该: 读两次 + 读取 ISR 清除中断
}
```

**为什么导致风暴**：

```
时间线:
1. PHY 检测到 Link 变化 → nINT 引脚拉低 (电平触发)
2. MCU 进入 EXTI0_IRQHandler
3. ISR 调用 phy_irq_handler()
4. phy_irq_handler() 没有读取 PHY 的 ISR 寄存器
   → PHY 认为中断未被处理，nINT 保持低电平
5. ISR 返回
6. 因为 nINT 仍然是低电平 → MCU 立即再次触发中断
7. 回到步骤 2，无限循环
```

电平触发中断的特性：中断引脚的电平保持有效状态时，CPU 会反复触发中断。只有读取 PHY 的 ISR 寄存器让 PHY 拉高 nINT 引脚，中断才会停止。

#### 修复

正确的中断服务函数：

```c
// ===== 正确的中断服务函数 =====
// 黄金法则: 读取 ISR 寄存器来清除 PHY 的中断状态

void EXTI0_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET) {
        phy_irq_handler(0);
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);  // 清除 MCU 侧中断标志
    }
}

void phy_irq_handler(uint8_t phy_addr) {
    uint16_t isr;

    // 关键: 读取 ISR 寄存器 (LAN8720A 读即清除)
    // 这一操作会让 PHY 拉高 nINT 引脚
    isr = mdio_read(phy_addr, LAN8720A_REG_ISR);  // 0x1D

    // 如果 ISR 读回 0xFFFF, 可能是 MDIO 通信异常
    if (isr == 0xFFFF) {
        printf("[IRQ] 警告: ISR=0xFFFF, MDIO 可能异常!\n");
        return;
    }

    printf("[IRQ] PHY 中断! ISR = 0x%04X\n", isr);

    if (isr & LAN8720A_ISR_LINK_UP) {
        // 延迟等待链路稳定
        delay_ms(50);

        // 读两次 BMSR 获取真实链路状态
        mdio_read(phy_addr, 0x01);  // 清 latch
        uint16_t bmsr = mdio_read(phy_addr, 0x01);

        if (bmsr & (1 << 2)) {
            printf("[IRQ] Link UP\n");
            g_link_state = 1;
        }
    }

    if (isr & LAN8720A_ISR_LINK_DOWN) {
        // 读两次 BMSR 确认
        mdio_read(phy_addr, 0x01);
        uint16_t bmsr = mdio_read(phy_addr, 0x01);

        if (!(bmsr & (1 << 2))) {
            printf("[IRQ] Link DOWN\n");
            g_link_state = 0;
        }
    }

    if (isr & LAN8720A_ISR_ANEG_DONE) {
        printf("[IRQ] Auto-Neg 完成\n");
    }

    if (isr & LAN8720A_ISR_REMOTE_FLT) {
        printf("[IRQ] 远端故障\n");
    }
}
```

#### 预防

1. **中断类型选择**：对于 PHY 中断，使用**边沿触发**（Falling Edge）而不是**电平触发**（Level Low）。边沿触发只会在引脚电平变化时触发一次，不会像电平触发那样持续触发。

```c
// 在 CubeMX 或代码中配置边沿触发
GPIO_InitTypeDef gpio = {0};
gpio.Pin = PHY_INT_PIN;
gpio.Mode = GPIO_MODE_IT_FALLING;  // 边沿触发，非电平触发!
gpio.Pull = GPIO_PULLUP;           // nINT 开漏，外部上拉，默认高电平
HAL_GPIO_Init(PHY_INT_PORT, &gpio);
```

2. **ISR 黄金法则**：**每个 PHY 中断服务函数必须读取 ISR 寄存器**。如果 ISR 是"读即清除"类型，读取 ISR 是清除中断的唯一方式。

3. **中断标志位去抖**：在 ISR 中设置一个标志，在主循环或任务中处理具体业务逻辑，减少 ISR 执行时间。

```c
// 推荐架构: ISR 只做最轻量的事
volatile uint8_t g_phy_int_pending = 0;

void EXTI0_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET) {
        g_phy_int_pending = 1;                     // 设置标志
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);      // 清 MCU 标志
    }
}

// 在主循环中处理
void main_loop(void) {
    while (1) {
        if (g_phy_int_pending) {
            g_phy_int_pending = 0;
            phy_irq_handler(0);  // 在主循环中调用，不在 ISR 中
        }
        // ... 其他任务
    }
}
```

4. **中断超时保护**：如果连续 N 次中断 ISR 值相同，可能是 PHY 故障，此时可以暂时禁用中断。

---

### Case 6: Wireshark 抓到 CRC 错误包

#### 症状

设备正常通信，Ping 大部分成功，但偶尔出现超时。在 PC 上用 Wireshark 抓包发现有一些数据包的 Ethernet FCS（Frame Check Sequence）错误：

```
Wireshark 截图描述:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
No.  Time      Source          Destination      Info
 1   0.000     PC              DevBoard         Echo (ping) request
 2   0.001     DevBoard        PC               Echo (ping) reply
 3   0.002     PC              DevBoard         Echo (ping) request
 4   0.003     DevBoard        PC               Echo (ping) reply
 5   0.005     DevBoard        PC               [Malformed Packet] ← !
 6   0.006     PC              DevBoard         Echo (ping) request
 7   0.009     DevBoard        PC               Echo (ping) reply

帧 5: 帧长 1518 字节，Ethernet FCS 校验错误
      (Wireshark 标注: [Expert Info (Error/Malformed): FCS Error])
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

错误不是持续发生，而是大约每 100-200 个包出现一次。错误包的帧长都是 1518 字节（最大帧长），且错误随机分布。

在 MCU 端检查 MAC 的错误计数器：

```c
// 以 STM32 ETH MAC 为例
uint32_t alignd_err = ETH->MACRCR;  // 接收错误计数器
// MACRCR bit3: CRC Error
// MACRCR bit2: Alignment Error
// 这些计数器在增加!
```

#### 根因分析

这是典型的 **RMII 时钟抖动/偏斜** 问题。

RMII 接口的信号时序：

```
RMII 时钟拓扑:

方案 A (正确):
  50MHz 晶振 ────┬─── PHY  ──── REF_CLK ──── MCU
                 │
                 └─── (PHY 内部生成 50MHz REF_CLK 输出给 MCU)

方案 B (错误):
  MCU ──── REF_CLK ──── PHY
  (MCU 的 MCO 引脚输出 50MHz 给 PHY，但走线过长)

方案 C (另一种错误):
  两个独立的 50MHz 晶振:
  晶振1 ──── PHY
  晶振2 ──── MCU
  (两个晶振频率有微小差异，导致时钟漂移)
```

在这个案例中，PCB 使用了方案 C —— 两个独立的 50MHz 有源晶振分别给 PHY 和 MCU 提供 RMII REF_CLK。用示波器测量两个时钟信号：

```
示波器截图描述 (CH1=PHY REF_CLK, CH2=MCU REF_CLK):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
时间轴: 10ns/div (看偏斜), 或 1us/div (看漂移)

CH1: 50.000MHz, 占空比 50%, Vpp=3.3V
CH2: 50.001MHz, 占空比 48%, Vpp=3.1V
     ↑ 两个晶振频率差了 1kHz!
     
长时间观察 (1ms/div):
CH1 和 CH2 的相位差从 0° 逐渐漂移到 180°，然后回到 0°
周期性地对齐和错开

在相位差接近 180° 时: CRC 错误大量出现
在相位差接近 0° 时: 通信正常
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

RMII 要求 REF_CLK 的频率精度为 ±50ppm（百万分之五十），即 50MHz 时偏差不超过 ±2.5kHz。50.000MHz 和 50.001MHz 相差 1kHz（20ppm），看似在规格内。但问题在于**两个晶振的相对偏差**和**占空比差异**：

1. 两个晶振的频率差导致时钟相位持续漂移
2. 在相位差最大时，MCU 在时钟边沿采样数据时可能采到数据线上的毛刺或过渡状态
3. 占空比 48% 意味着高电平时间稍短，对某些芯片可能接近时序裕量边界

#### 修复

改用方案 A：由 PHY 提供 REF_CLK 给 MCU，确保 PHY 和 MCU 使用**同一个时钟源**。

```
修复后的时钟拓扑:

  25MHz 晶振 ──── LAN8720A ──── REF_CLK(50MHz) ──── MCU
                          ↑
                   PHY 内部 PLL 将 25MHz 倍频到 50MHz
                   50MHz REF_CLK 从 PHY 的 REF_CLKO 引脚输出
```

或者在只有一个 50MHz 晶振的情况下：

```
  50MHz 晶振 ────┬─── PHY (REF_CLK 输入)
                 │
                 └─── MCU (ETH_RMII_REF_CLK)
```

改动后，用示波器验证：

```
示波器截图描述 (修复后):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: PHY REF_CLK (50MHz)
CH2: MCU REF_CLK (50MHz) — 同源

两个时钟完全同频同相，无漂移
占空比 50%，Vpp=3.3V
长时间观察 (10ms/div): 相位固定不变
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

修复后运行 24 小时，Wireshark 未抓到任何 CRC 错误包。

#### 预防

1. **RMII 必须使用单一时钟源**：PHY 和 MAC 必须共享同一个 REF_CLK。推荐由 PHY 输出 REF_CLK 给 MCU。
2. **时钟走线**：REF_CLK 是高速信号（50MHz），走线要短，远离其他高速信号，避免串扰。
3. **PCB 设计时考虑时钟偏斜**：REF_CLK 到 PHY 和到 MCU 的走线长度差控制在 ±5mm 以内。
4. **如果只能用两个晶振**：选择频率精度 ±25ppm 或更好的晶振，并在固件中增加 CRC 错误计数监控。
5. **调试时先用 Wireshark 确认**：CRC 错误一般是物理层问题。如果 Wireshark 报告 FCS 错误，排查顺序：时钟 → 电源 → 信号完整性 → 网线质量。

---

### Case 7: 首板 Bring-Up —— 什么都读不到

#### 症状

新做的 PCB 板第一次上电，MDIO 总线读任何 PHY 地址都返回 `0x0000` 或 `0xFFFF`。PHY ID 读不到，Link LED 不亮。

#### 看到的现场

串口输出：

```
PHY Detection: Scanning addresses 0x00-0x1F...
  Addr 0x00: ID=0x00000000 ← 无响应
  Addr 0x01: ID=0x00000000 ← 无响应
  Addr 0x02: ID=0x00000000 ← 无响应
  ...
  Addr 0x1F: ID=0x00000000 ← 无响应
错误: 未检测到 PHY!
```

或：

```
PHY Detection: Scanning addresses 0x00-0x1F...
  Addr 0x00: ID=0xFFFFFFFF ← 全 F
  Addr 0x01: ID=0xFFFFFFFF ← 全 F
  ...
错误: 未检测到 PHY!
```

新板，第一次上电，没有任何进展。

#### 根因分析 —— 系统故障排查流程

这是最考验耐心和系统化思维的问题。不要猜，按以下流程一步一步排查：

```
新板 PHY 调试流程
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                           ┌─────────────────────────────┐
                           │  Step A: 检查电源            │
                           │  PHY_VDD = 3.3V?             │
                           │  VDD_CORE = 1.2V?            │
                           └─────────────┬───────────────┘
                                         │ 不正常
                                         ▼
                           ┌─────────────────────────────┐
                           │  Step B: 检查复位引脚         │
                           │  nRST 上电后是否变为高电平?   │
                           └─────────────┬───────────────┘
                                         │ 不正常
                                         ▼
                           ┌─────────────────────────────┐
                           │  Step C: 检查时钟             │
                           │  晶振 25MHz 起振?            │
                           │  REF_CLK 有 50MHz 输出?      │
                           └─────────────┬───────────────┘
                                         │ 不正常
                                         ▼
                           ┌─────────────────────────────┐
                           │  Step D: 检查 MDIO 上拉      │
                           │  MDIO 电压 = 3.3V?           │
                           │  上拉电阻 1.5k~2.2k?         │
                           └─────────────┬───────────────┘
                                         │ 不正常
                                         ▼
                           ┌─────────────────────────────┐
                           │  Step E: 尝试不同 PHY 地址   │
                           │  Strapping 引脚决定了地址    │
                           └─────────────┬───────────────┘
                                         │ 不正常
                                         ▼
                           ┌─────────────────────────────┐
                           │  Step F: 示波器看 MDIO       │
                           │  确认 MDIO/MDC 信号时序     │
                           └─────────────────────────────┘
```

#### Step A: 检查电源

**万用表测量**：

```
引脚               | 期望值  | 实测值  | 判断
───────────────────|─────────|────────|──────
PHY 3.3V (VDD)    | 3.3V    | 3.28V  | OK (±5% 范围内)
PHY 1.2V (VDD_CORE)| 1.2V   | 0.00V  | 异常! ←
GND               | 0V      | 0V     | OK
```

LAN8720A 的 1.2V 核心电压可以由内部 LDO 从 3.3V 产生，但需要外部去耦电容。检查数据手册：

```
LAN8720A 内部 LDO 要求:
  VDD_CORE (引脚 11): 内部 1.2V LDO 输出
  外部必须接 10μF + 0.1μF 去耦电容到 GND
  如果没有去耦电容, LDO 可能振荡或无输出
```

发现原理图中 VDD_CORE 引脚只画了一个 0.1μF 电容，但数据手册要求至少 10μF + 0.1μF。10μF 电容被遗漏了。

**补充 10μF 电容后**，测量 VDD_CORE = 1.18V，恢复正常。

#### Step B: 检查复位引脚

PHY 的 nRST 引脚（有些芯片叫 RST_N）必须在上电后由低变高，释放复位。

**万用表测量**：

```
引脚   | 上电后期望 | 实测     | 判断
───────|────────────|──────────|───────
nRST   | 3.3V       | 0.15V   | 异常!
```

nRST 一直是低电平，PHY 一直处于复位状态，不会响应 MDIO。

**检查原理图**：

```
问题电路:
                    3.3V
                      │
                      ├─── 10kΩ
                      │
  MCU_PA0 ────────────┴─── nRST (PHY 引脚)

问题: 如果 MCU PA0 在上电时是 Open-Drain 或未初始化，
      nRST 可能被 PA0 内部下拉到低电平!
```

**修复方法**：给 nRST 加一个外部上拉电阻（到 3.3V），或者在 MCU 初始化代码中先拉高 PA0，再使能 PHY 的复位时序。

```c
// 正确的复位初始化
void phy_hardware_reset_init(void) {
    GPIO_InitTypeDef gpio = {0};

    // 先配置 RST 引脚为推挽输出，初始为高
    gpio.Pin = PHY_RST_PIN;
    gpio.Mode = GPIO_MODE_OUTPUT_PP;
    gpio.Pull = GPIO_PULLUP;     // 内部上拉确保不是低电平
    gpio.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(PHY_RST_PORT, &gpio);

    // 确保复位释放
    HAL_GPIO_WritePin(PHY_RST_PORT, PHY_RST_PIN, GPIO_PIN_SET);
    delay_ms(100);

    // 现在可以做 PHY 复位序列了
    // (如果需要，再拉低-拉高)
}
```

#### Step C: 检查时钟

PHY 需要时钟才能工作。LAN8720A 需要外部 25MHz 晶振或 50MHz 时钟输入。

**示波器测量晶振引脚**：

```
示波器截图描述 (无时钟):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: XI (晶振输入), AC 耦合, 1V/div
时间轴: 100ns/div

波形: 一条平直的线，约 0.5V DC
      → 晶振没有起振!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

晶振不起振的常见原因：

| 原因 | 检查方法 | 修复 |
|------|----------|------|
| 晶振负载电容不匹配 | 计算 CL = (C1×C2)/(C1+C2) + 寄生电容 | 一般选 18pF~22pF |
| 晶振焊错或损坏 | 换一个晶振 | 替换 |
| PHY 工作模式设置错 | 检查 strapping 引脚 | 确认 nINT/REGOFF 模式 |
| 电源未稳定 | 先检查 Step A | 供电正常后再看时钟 |

**正确的 25MHz 晶振电路**：

```
LAN8720A 晶振电路:
                    25MHz 晶振
                  ┌──────────┐
                  │          │
  XI (Pin 5) ─────┤1        3├────── XO (Pin 4)
                  │          │
                  │    4    2├────── GND
                  └──────────┘
                     │    │
                    ┌┴┐  ┌┴┐
                    │ │  │ │  18pF (负载电容)
                    └┬┘  └┬┘
                     │    │
                    GND  GND
```

修复晶振电路后（更换匹配的负载电容）：

```
示波器截图描述 (有时钟):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: XI (晶振输入), AC 耦合, 1V/div
时间轴: 20ns/div

波形: 25.000MHz 正弦波, Vpp≈1.0V
      稳定振荡!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### Step D: 检查 MDIO 上拉

即使 PHY 已经正常工作，MDIO 通信仍然可能失败。

**测量 MDIO 引脚电压**：

```
MDIO 引脚           | MCU 初始化前 | MCU 初始化后 (输入模式)
───────────────────|─────────────|─────────────────────
电压               | 0.1V        | 0.1V
                   |             ↑ 应该是 3.3V!
```

MDIO 电压不对，说明上拉电阻有问题：

- 原理图上画了 2.2k 上拉到 3.3V
- 但检查 PCB，发现上拉电阻 R12 是**空贴**（Not Fitted/不焊接）

在 PCB 上补焊一颗 2.2kΩ 电阻后：

```
MDIO 引脚           | MCU 初始化前 | MCU 初始化后
───────────────────|─────────────|─────────────
电压               | 3.3V        | 3.3V  ← OK
```

#### Step E: 尝试不同 PHY 地址

PHY 地址由 strapping 引脚决定。LAN8720A 的 PHYAD0 引脚（复用 RXER 引脚）的状态决定了地址：

```
PHYAD0/RXER 引脚状态 | PHY 地址
──────────────────────|─────────
拉低 (10k 到 GND)    | 0x00  (最常见)
拉高 (10k 到 3.3V)  | 0x01
浮空                | 可能是 0x00 或 0x01 (不稳定)
```

商品模块上这个引脚的电平可能不同，所以不要假定地址是 0。扫描所有 32 个地址：

```c
// 暴力扫描所有 PHY 地址
int phy_scan_all_addresses(MDIO_Bus *bus) {
    uint16_t id1, id2;

    for (uint8_t addr = 0; addr <= 31; addr++) {
        if (bus->read_c22(bus->bus_priv, addr, 0x02, &id1) != 0)
            continue;
        if (id1 == 0x0000 || id1 == 0xFFFF)
            continue;

        bus->read_c22(bus->bus_priv, addr, 0x03, &id2);
        if (id2 == 0x0000 || id2 == 0xFFFF)
            continue;

        uint32_t phy_id = ((uint32_t)id1 << 16) | id2;
        printf("发现 PHY! 地址=0x%02X, ID=0x%08X\n", addr, phy_id);
        return addr;
    }

    return -1;  // 没找到
}
```

在这个案例中，PHY 地址是 0x01（模块上的 PHYAD0 被拉高了），而代码中默认用了地址 0x00。

#### Step F: 示波器看 MDIO 波形

如果以上所有步骤都检查了仍然不行，用示波器看 MDIO/MDC 的实际时序：

```
示波器截图描述 (正确的 MDIO 读操作):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: MDC (时钟), 2V/div
CH2: MDIO (数据), 2V/div
时间轴: 2us/div

波形分析:
  [CH1] MDC: 约 1MHz 方波, 占空比 50%
  [CH2] 前 32 位: 高电平 (PREAMBLE 32个1)
        接着 14 位: 命令帧 (ST+OP+PHYAD+REGAD)
        TA: 高阻 → 0
        后 16 位: PHY 驱动的数据 (MDIO 从输出切换到输入)

用光标测量:
  MDC 频率: 1.04MHz (OK, ≤2.5MHz)
  PRE 长度: 32个脉冲 (OK)
  帧格式: ST=01, OP=10 (OK)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

与正确的波形对比：

```
示波器截图描述 (错误的 MDIO 读操作 ─ 缺少 PREAMBLE):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CH1: MDC
CH2: MDIO

波形: ST 和 OP 字段之前没有 32 个 1
      直接在 MDIO 上发送了命令
      
问题: 有些 MDIO 控制器或软件实现忘记发送 PREAMBLE
      大多数 PHY 可以容忍缺少 PREAMBLE, 但不是所有
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**修复 PREAMBLE**：确保 MDIO 驱动在每次读/写前发送 32 个 1。

#### 综合诊断决策表

```
问题: PHY ID 返回 0x0000 或 0xFFFF

读到 0x0000?
  ├── MDIO 引脚是推挽输出? → 改为开漏输出
  ├── MDIO 上拉电阻缺失?  → 加 1.5k~2.2k 上拉
  └── MDC 没有时钟?       → 检查 GPIO 配置和时钟

读到 0xFFFF?
  ├── PHY 没上电?         → 测量 VDD 3.3V 和 VDD_CORE 1.2V
  ├── PHY 在复位状态?     → 测量 nRST 引脚
  ├── PHY 时钟没起振?     → 示波器看晶振
  ├── PHY 地址不对?       → 扫描 0-31 地址
  └── 示波器看 MDIO 没信号→ 检查 GPIO 初始化代码

MDIO 波形看起来正常但读回 0xFFFF?
  ├── PREAMBLE 不够?      → 发送完整 32 个 1
  ├── MDC 频率太高?       → 降到 ≤2.5MHz
  └── PHY 地址/reg 地址错位? → 用示波器 decode 确认
```

#### 综合修复后的 Bring-Up Checklist

```c
// 新板 PHY Bring-Up 完整检查程序
void phy_bring_up_checklist(void) {
    printf("\n========== PHY Bring-Up 检查清单 ==========\n");

    // [√] Step A: 电源
    printf("Step A: 电源\n");
    printf("  3.3V:  %s\n", check_voltage(3.3f, 3.3f, 0.15f) ? "OK" : "FAIL");
    printf("  1.2V:  %s\n", check_voltage(1.2f, 1.2f, 0.10f) ? "OK" : "FAIL");

    // [√] Step B: 复位
    printf("Step B: nRST 引脚\n");
    GPIO_PinState rst = HAL_GPIO_ReadPin(PHY_RST_PORT, PHY_RST_PIN);
    printf("  nRST = %s\n", rst == GPIO_PIN_SET ? "HIGH (OK)" : "LOW (FAIL)");

    // [√] Step C: 时钟 (通过读取 PHY 寄存器间接确认)
    printf("Step C: 时钟\n");
    uint16_t id1 = mdio_read(0x00, 0x02);
    printf("  PHY ID1 = 0x%04X %s\n", id1, id1 != 0xFFFF ? "(OK)" : "(FAIL)");

    // [√] Step D: MDIO 上拉
    printf("Step D: MDIO 电平\n");
    // 在输入模式下读取 MDIO 引脚
    mdio_set_input();
    GPIO_PinState mdio_level = HAL_GPIO_ReadPin(MDIO_PORT, MDIO_PIN);
    printf("  MDIO 电平 = %s (应为 HIGH)\n",
           mdio_level == GPIO_PIN_SET ? "HIGH" : "LOW");

    // [√] Step E: 扫描地址
    printf("Step E: 扫描 PHY 地址\n");
    int found_addr = phy_scan_all_addresses(&g_mdio_bus);
    if (found_addr >= 0) {
        printf("  PHY 在地址 0x%02X 发现!\n", found_addr);
    } else {
        printf("  PHY 未找到! 检查以上步骤\n");
    }
}
```

---

> **学习建议**：把这 7 个案例的调试思路记在心里。当你的 PHY 不工作时，不要只盯着代码看——用万用表测电压、示波器看时钟和 MDIO 时序，往往比看代码更快找到问题。**"先硬件，后软件"** 是调试物理层设备的铁律。

---

> **参考文档**
>
> - IEEE 802.3-2022 Clause 22 (MDIO/MDC Management)
> - IEEE 802.3-2022 Clause 28 (Auto-Negotiation)
> - IEEE 802.3-2022 Clause 40 (1000Base-T PHY)
> - LAN8720A Datasheet (Microchip)
> - DP83848 Datasheet (TI)
> - KSZ8081/KSZ9031 Datasheet (Microchip)
> - KSZ9897 Datasheet (Microchip) — Switch with integrated PHYs
> - Linux Kernel: `drivers/net/phy/` — PHY 驱动参考实现
