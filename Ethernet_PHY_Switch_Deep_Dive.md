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
