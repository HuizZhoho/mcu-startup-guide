# MCU 以太网入门完全指南

> 从 MCU 启动到以太网组网 — 嵌入式工程师的完整学习路径

## 简介

本仓库收录了一套完整的嵌入式以太网学习文档，涵盖从微控制器上电复位到以太网协议栈组网的每一个关键环节。无论你是刚接触 MCU 开发的初学者，还是希望补齐网络知识短板的中级工程师，这套资料都能提供清晰的路径和足够深入的底层解析。

文档按学习路径分为四个部分：**MCU 启动流程**（复位 -> startup -> main）、**PHY/交换芯片驱动**（寄存器模型、状态机、驱动实现）、**网络协议分析**（ARP / IP / TCP / UDP / ICMP / VLAN / STP / IGMP / LLDP 逐字节拆解），以及 **STM32 + KSZ9897 交换芯片完整驱动实现**（含 CLI 监控接口）。

## 适用人群

- 想要深入理解 MCU 启动原理的嵌入式初学者
- 需要为 MCU 添加以太网功能但不知从何入手的开发者
- 希望理解 PHY 寄存器模型、自动协商、链路检测的固件工程师
- 需要驱动管理型交换芯片（如 KSZ9897）并实现 VLAN / MAC 过滤等功能的工程师
- 对网络协议栈底层实现感兴趣的嵌入式学习者

## 学习路径（推荐顺序）

### Step 1 — MCU 启动流程深度解读
**文档**：`MCU_Startup_Flow_Deep_Dive.md`
**难度**：Intermediate

从复位向量到 main() 函数的完整旅程：
- 链接脚本 (.ld) 逐行解析 — 堆栈、段、内存布局
- 启动文件 (.s) 逐行解析 — 中断向量表、复位处理、堆栈初始化
- SystemInit() 深度解读 — 时钟配置、FPU 使能、向量表重定位
- 内存布局完整可视化
- 启动流程图（指令级）
- 实际固件案例分析（STM32F407 / F767）
- 不同架构启动对比（ARM / RISC-V / AVR）
- 从启动到 RTOS 调度全链路时间线

### Step 2 — 以太网 PHY 与交换芯片深度解读
**文档**：`Ethernet_PHY_Switch_Deep_Dive.md`
**难度**：Intermediate

PHY 与交换芯片的完整驱动知识体系：
- MAC 与 PHY 的分工模型（OSI 分层视角）
- MII / RMII / GMII / RGMII 接口时序详解
- SMI / MDIO 协议与 IEEE 802.3 标准寄存器模型
- PHY 状态机与初始化全流程（复位 -> 配置 -> 自动协商 -> 链路检测）
- 自动协商（Auto-Negotiation）深度解析（FLP 脉冲、优先级解析、Next Page）
- 链路状态检测与中断管理（Link Up/Down、能量检测、中断 vs 轮询）
- 实战：STM32F407 + LAN8720 / DP83848 完整初始化代码
- 交换芯片架构基础：MAC 表、VLAN 表、端口转发逻辑
- 交换芯片驱动核心概念：SPI 配置、寄存器接口、MIB 统计
- 7 天 PHY 学习路线图
- RMII 时钟与 PCB 关键设计要点

### Step 3 — MCU 以太网协议深度解读
**文档**：`MCU_Ethernet_Protocols_Deep_Dive.md`
**难度**：Intermediate

从以太网帧到应用层 — 全栈协议逐字节拆解：
- 以太网帧结构（前导码、MAC 地址、类型/长度、FCS）
- ARP 协议 — 请求/响应、缓存表、免费 ARP
- IP 协议 — 头部每个字段（版本、IHL、TOS、总长度、标识、分片偏移、TTL、协议、校验和）
- ICMP 协议 — Echo / Echo Reply、差错报文、ping 实现原理
- UDP 协议 — 伪头部校验和、端口、无连接特性
- TCP 协议 — 三次握手、四次挥手、序列号/确认号、窗口、状态机
- VLAN (802.1Q) — Tag 结构、PVID、Trunk / Access 端口
- STP / RSTP — BPDU、根桥选举、端口状态、收敛过程
- IGMP 组播与 LLDP 链路发现
- 每个协议均附带 WireShark 抓包分析实验
- 每个协议均附带最小 C 语言实现示例

### Step 4 — STM32 + KSZ9897 交换芯片驱动实现
**文档**：`STM32_KSZ9897_Switch_Driver.md`
**难度**：Advanced

完整的工业级交换芯片驱动实现：
- KSZ9897 硬件架构总览（端口映射、内部交换机核心、SPI 寄存器映射）
- SPI 通信层实现（全双工 DMA、片选控制、超时与错误重试）
- 交换芯片初始化流程（复位、端口使能、MAC 地址设置）
- VLAN 配置与管理（802.1Q VLAN、端口 VLAN 成员、PVID）
- MAC 地址表操作（静态入口添加/删除、动态学习、老化时间）
- EEE / 节能以太网与电源管理
- MIB 统计计数器读取与实现（RMON 以太网统计）
- 数据处理：发送流程、接收流程、校验和卸载
- CLI 监控接口（AT 指令风格的完整交互式命令集）
- 异常处理与看门狗集成

## 学习时间估计

| 步骤 | 内容 | 建议时间 | 备注 |
|------|------|----------|------|
| Step 1 | MCU 启动流程 | 2–3 天 | 含实验操作 |
| Step 2 | PHY 与交换芯片 | 3–5 天 | 含 7 天 PHY 学习路线 |
| Step 3 | 以太网协议 | 5–7 天 | 含协议抓包实验 |
| Step 4 | KSZ9897 驱动 | 3–5 天 | 含交换芯片实验 |

总计约 **13–20 天**（每日 2–4 小时）。

## 所需硬件

- **MCU 开发板**：STM32F407 / F429 / F767 等自带以太网 MAC 的开发板
- **PHY 模块**：LAN8720（RMII 接口，用于 PHY 实验）
- **交换芯片模块**：KSZ9897 管理型交换芯片模块（用于 Step 4 实验）
- **RJ45 连接器**：带内部变压器的标准以太网插座
- **调试器**：ST-Link 或 J-Link
- **串口模块**：USB 转 TTL 串口模块（用于调试控制台输出）

## 所需软件

- **IDE**：STM32CubeIDE（推荐）或 Keil MDK
- **串口终端**：Putty、TeraTerm 或 sscom
- **抓包工具**：Wireshark（用于协议分析实验）
- **版本管理**：Git

## 快速开始

以下代码演示如何通过 GPIO 模拟 MDIO 协议读取 PHY ID 寄存器（寄存器 2–3），验证硬件连接是否正常 —— 这是所有以太网实验的第一步。

```c
// ============================================================
// GPIO 模拟 MDIO 读取 PHY ID（验证 PHY 硬件连接）
// 适用平台：STM32F407 + LAN8720 / DP83848
// ============================================================
#include "stm32f4xx.h"

// 请根据实际硬件修改以下引脚定义
#define MDIO_PIN       GPIO_Pin_1   // PH1
#define MDC_PIN        GPIO_Pin_2   // PH2
#define MDIO_PORT      GPIOH
#define MDC_PORT       GPIOH

#define PREAMBLE_LEN   32           // 32 个 1 的前导码
#define READ_OPCODE    0x02         // 10b
#define WRITE_OPCODE   0x01         // 01b

static void MDC_pulse(void) {
    MDC_PORT->BSRRL = MDC_PIN;  __NOP(); __NOP();
    MDC_PORT->BSRRH = MDC_PIN;  __NOP(); __NOP();
}

static void MDIO_dir_output(void) {
    MDIO_PORT->MODER |= (GPIO_MODE_OUTPUT_PP << (1 * 2));
}

static void MDIO_dir_input(void) {
    MDIO_PORT->MODER &= ~(GPIO_MODE_OUTPUT_PP << (1 * 2));
    MDIO_PORT->MODER |= (GPIO_MODE_INPUT << (1 * 2));
}

static void MDIO_write_bit(uint8_t bit) {
    MDIO_dir_output();
    if (bit) MDIO_PORT->BSRRL = MDIO_PIN;
    else     MDIO_PORT->BSRRH = MDIO_PIN;
    MDC_pulse();
}

static uint8_t MDIO_read_bit(void) {
    uint8_t bit;
    MDIO_dir_input();
    MDC_pulse();
    bit = (MDIO_PORT->IDR & MDIO_PIN) ? 1 : 0;
    return bit;
}

static void MDIO_preamble(void) {
    for (int i = 0; i < PREAMBLE_LEN; i++)
        MDIO_write_bit(1);
}

static void MDIO_turnaround(void) {
    MDIO_dir_input();                     // 高阻
    MDC_pulse();                          // 读端 Z
    MDIO_read_bit();                      // 读端 0
    MDIO_dir_output();                    // 恢复
}

uint16_t MDIO_read(uint8_t phy_addr, uint8_t reg_addr) {
    uint16_t data = 0;

    // 1. 前导码：32 个 1
    MDIO_preamble();

    // 2. 起始 + 操作码 + PHY 地址 + 寄存器地址
    MDIO_write_bit(0);   MDIO_write_bit(1);    // 起始码 01
    MDIO_write_bit(1);   MDIO_write_bit(0);    // 读操作码 10
    for (int i = 4; i >= 0; i--)               // PHY 地址 (5 bit)
        MDIO_write_bit((phy_addr >> i) & 1);
    for (int i = 4; i >= 0; i--)               // 寄存器地址 (5 bit)
        MDIO_write_bit((reg_addr >> i) & 1);

    // 3. 转向 (Turnaround)：Z + 0
    MDIO_turnaround();

    // 4. 读取 16 bit 数据
    for (int i = 15; i >= 0; i--)
        data = (data << 1) | MDIO_read_bit();

    return data;
}

int main(void) {
    // 初始化 MDIO/MDC GPIO（推挽输出、50 MHz、无上下拉）
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOHEN;
    MDC_PORT->MODER |= (GPIO_MODE_OUTPUT_PP << (2 * 2));
    MDIO_PORT->PUPDR &= ~(GPIO_PUPDR_PUPDR1 << (1 * 2));
    MDC_PORT->PUPDR &= ~(GPIO_PUPDR_PUPDR2 << (2 * 2));

    uint16_t phy_id1 = MDIO_read(0, 2);  // PHY ID1 寄存器
    uint16_t phy_id2 = MDIO_read(0, 3);  // PHY ID2 寄存器

    // LAN8720: ID1=0x0007, ID2=0xC0F1
    // DP83848: ID1=0x2000, ID2=0xA000
    // 如果读到 0xFFFF 或 0x0000，说明硬件连接或 PHY 地址有误

    while (1);
}
```

## 文档总览

| 文档 | 说明 | 行数 | 难度 |
|------|------|------|------|
| `MCU_Startup_Flow_Deep_Dive.md` | MCU 从复位到 main() 的完整启动流程，含链接脚本、启动文件、SystemInit、内存布局、RTOS 启动时间线及不同架构对比 | ~2600 | Intermediate |
| `Ethernet_PHY_Switch_Deep_Dive.md` | PHY/Switch 驱动知识体系，含 MII/RMII 时序、MDIO 协议、IEEE 802.3 寄存器、自动协商、链路检测、LAN8720/DP83848 实战代码及交换芯片基础 | ~2600 | Intermediate |
| `MCU_Ethernet_Protocols_Deep_Dive.md` | 网络协议全栈逐字节拆解，含以太网帧、ARP、IP、ICMP、UDP、TCP、VLAN、STP、IGMP、LLDP，每个协议均配有 WireShark 实验和 C 代码示例 | ~3800 | Intermediate |
| `STM32_KSZ9897_Switch_Driver.md` | STM32 + KSZ9897 交换芯片完整驱动实现，含 SPI 通信、初始化、VLAN/MAC 管理、MIB 统计、数据处理及 CLI 监控接口 | ~2500 | Advanced |

## 仓库结构

```
mcu-startup-guide/
├── MCU_Startup_Flow_Deep_Dive.md         # MCU 启动流程深度解读
├── Ethernet_PHY_Switch_Deep_Dive.md       # 以太网 PHY 与交换芯片深度解读
├── MCU_Ethernet_Protocols_Deep_Dive.md    # MCU 以太网协议深度解读
├── STM32_KSZ9897_Switch_Driver.md         # STM32 + KSZ9897 交换芯片驱动实现
└── README.md                              # 本文件
```

## License

MIT
