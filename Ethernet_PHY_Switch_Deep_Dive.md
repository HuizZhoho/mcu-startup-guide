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
