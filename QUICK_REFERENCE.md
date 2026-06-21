# MCU 以太网 / PHY / 交换机快速参考卡 (Quick Reference Cheat Sheet)

> 打印机友好 | Bilingual: English register names, Chinese descriptions
> 从复位到 main()、PHY/Switch 驱动、网络协议一站速查

---

## 1. PHY Register Quick Reference (IEEE 802.3 Clause 22)

| Addr | Name | Bits | Description | Reset |
|:----:|------|:----:|-------------|:-----:|
| 0x00 | **BMCR** | 16 | Basic Mode Control Register | 0x3100 |
| | | bit15 | **Reset** — 软件复位 (自清除) | |
| | | bit14 | **Loopback** — 环回模式 | |
| | | bit13,6 | **Speed Select** — 00=10M, 01=100M, 10=1000M | |
| | | bit12 | **Auto-Neg Enable** — 使能自动协商 | |
| | | bit11 | **Power Down** — 省电模式 | |
| | | bit10 | **Isolate** — 隔离 MAC 接口 | |
| | | bit9 | **Restart Auto-Neg** — 重启协商 (自清除) | |
| | | bit8 | **Duplex Mode** — 1=Full, 0=Half | |
| 0x01 | **BMSR** | 16 | Basic Mode Status Register | 0x786D |
| | | bit15-11 | **Capabilities** — 100T4, 100TX-FD/HD, 10T-FD/HD | |
| | | bit5 | **Auto-Neg Complete** — 协商完成 | |
| | | bit4 | **Remote Fault** — 远端故障 | |
| | | bit3 | **Auto-Neg Ability** — 支持协商 (只读) | |
| | | bit2 | **Link Status** — **Latching Low** (须读两次!) | |
| 0x02 | **PHYIDR1** | 16 | PHY Identifier 1 — OUI 高 16 位 | 厂商 |
| 0x03 | **PHYIDR2** | 16 | PHY Identifier 2 — OUI低6位(15:10) + 型号(9:4) + 版本(3:0) | 厂商 |
| 0x04 | **ANAR** | 16 | Auto-Neg Advertisement Register — 本端能力发布 | 0x01E1 |
| | | bit9-5 | 100TX-FD(9), 100TX-HD(8), 10T-FD(7), 10T-HD(6) | |
| | | bit10-11 | **PAUSE** / **ASM_DIR** — 流控协商 | |
| | | bit4-0 | **Selector** — 00001 = IEEE 802.3 | |
| 0x05 | **ANLPAR** | 16 | Auto-Neg Link Partner Ability — 对端能力 (只读, 格式同 ANAR) | 0xXXXX |
| 0x06 | **ANER** | 16 | Auto-Neg Expansion Register | 0x0004 |
| | | bit0 | **LP_ANEG_ABLE** — 对端支持 Auto-Neg | |
| | | bit1 | **Page Received** — 收到新页 | |

### 芯片特有寄存器

| PHY | Addr | Name | Bits | Description |
|:---:|:----:|:----:|:----:|-------------|
| **DP83848** | 0x19 | **PHYSR** | 16 | PHY Status Register |
| | | | bit14 | **SPEED** — 1=100M, 0=10M |
| | | | bit13 | **DUPLEX** — 1=Full, 0=Half |
| | | | bit1 | **LINK** — 1=Up, 0=Down |
| **LAN8720A** | 0x1F | **MCSR** | 16 | Mode Control/Status Register |
| | | | bit14 | **SPEED** — 1=100M, 0=10M |
| | | | bit13 | **DUPLEX** — 1=Full, 0=Half |
| | | | bit12 | **ALE** — Auto-Neg 完成事件 |
| | | | bit10 | **LINK** — 1=Up, 0=Down |

---

## 2. Common PHY Control Values

```c
// BMCR (0x00) values
#define BMCR_RESET      0x8000   // Bit15: 软件复位
#define BMCR_AUTONEG    0x1200   // Bit12 + Bit9: Auto-Neg 使能 + 重启
#define BMCR_100M_FULL  0x2100   // Bit13 + Bit8: 100M Full (forced)
#define BMCR_100M_HALF  0x2000   // Bit13: 100M Half (forced)
#define BMCR_10M_FULL   0x0100   // Bit8: 10M Full (forced)
#define BMCR_10M_HALF   0x0000   // 10M Half (forced)

// BMSR (0x01) key flags
#define BMSR_LINK_UP    0x0004   // Bit2: Link status
#define BMSR_ANEG_DONE  0x0020   // Bit5: Auto-Neg complete

// Typical ANAR advertisement (all capabilities)
#define ANAR_ALL_10_100 0x01E1   // 100TX-FD|HD + 10T-FD|HD + selector
```

---

## 3. PHY ID Database

| PHY Chip | ID (32-bit) | OUI | Description |
|:---------|:-----------:|:---:|:------------|
| **LAN8720A** | 0x0007C0F1 | 00-0E-08 | Microchip 10/100 RMII, 内置 LDO |
| **DP83848** | 0x20005C90 | 08-00-28 | TI Industrial 10/100 MII/RMII |
| **KSZ8081** | 0x00221560 | 00-11-0A | Microchip 低功耗 10/100 |
| **KSZ9031** | 0x00221622 | 00-11-0A | Microchip Gigabit RGMII |
| **RTL8211F** | 0x001CC916 | 00-E0-0C | Realtek 千兆 RGMII/SGMII |

**Default PHY addresses**: LAN8720A=0x00, DP83848=0x10, KSZ8081=0x01, KSZ9031=0x01

---

## 4. Switch Register Quick Reference (KSZ9897)

| Addr | Name | Description |
|:----:|:----:|:------------|
| 0x0000-01 | **CHIP_ID** | Chip ID = 0x9897 (ID0=0x98, ID1=0x97) |
| 0x0002 | **GLOBAL_CTRL_0** | Global Control 0 — bit0=Start Switch |
| 0x0008 | **GLOBAL_CTRL_4** | MAC Learn(bit1), Aging(bit2), Tail Tag(bit0) |
| 0x00D0 | **RMII_CTRL** | Host port RMII config — bit0=RMII enable |
| 0x0100+N*0x20 | **PORT_N_CTRL** | Port N control registers (N=1-7) |
| 0x0104+N*0x20 | **PORT_N_STATUS** | Port N status — link(bit0), speed(bit1), duplex(bit2) |
| 0x0400 | **VLAN_TABLE** | VLAN table indirect access |
| 0x0500 | **MAC_TABLE** | MAC address table indirect access |
| 0x0600+(N-1)*0x40 | **MIB_COUNTERS** | MIB counters per port |

**Formula**: Port N base address = 0x0100 + (N - 1) * 0x20 (N=1-7)

**Port mapping**: Port 1-5 = user ports (integrated PHY), Port 7 = CPU host port (RMII)

**Port-based VLAN isolation registers** (KSZ9897):
| Addr | Port | Membership |
|:----:|:----:|:-----------|
| 0x000D | Port 1 | 0x42 (Port 1 + Port 7) |
| 0x000E | Port 2 | 0x44 (Port 2 + Port 7) |
| 0x000F | Port 3 | 0x48 (Port 3 + Port 7) |
| 0x0010 | Port 4 | 0x50 (Port 4 + Port 7) |
| 0x0011 | Port 5 | 0x60 (Port 5 + Port 7) |
| 0x0013 | Port 7 | 0xBF (all ports) |

---

## 5. Ethernet Frame Structure

**Software view** (Preamble + SFD stripped by PHY hardware):

```
Offset  Bytes  Field           Description
──────  ─────  ─────           ───────────
 0       6     Dst MAC         目的 MAC 地址 (e.g. FF-FF-FF-FF-FF-FF = broadcast)
 6       6     Src MAC         源 MAC 地址
12       2     EtherType       上层协议类型 (e.g. 0x0800 = IPv4, 0x0806 = ARP)
                                 (如有 VLAN Tag, 则此处为 0x8100, 实际 EtherType 在 offset 16)
14      46-1500 Payload         数据载荷 (最小 46B, 不足则填充 Pad)
        4     FCS              CRC32 校验和 (由硬件或软件计算)
```

**Full hardware view** (includes PHY layer overhead):

```
Offset  Bytes  Field
──────  ─────  ─────
 0       7     Preamble         前导码 (0xAA × 7, 用于时钟同步)
 7       1     SFD              帧起始定界符 (0xAB)
 8       6     Dst MAC
14       6     Src MAC
20       2     EtherType        (或 4B VLAN Tag + 2B EtherType)
22      46-1500 Payload + Pad
        4     FCS              CRC32
```

**Frame size limits**: Min = 64 bytes (incl. FCS), Max = 1518 bytes (standard) / 1522 bytes (with VLAN Tag)

**With 802.1Q VLAN Tag**: Dst(6) + Src(6) + TPID=0x8100(2) + TCI(2) + EtherType(2) + Payload(46-1500) + FCS(4)

```
VLAN Tag TCI: [PCP(3b) | DEI(1b) | VID(12b)]   VID range: 1-4094
```

---

## 6. Protocol EtherType Quick Reference

| Value | Protocol | Description |
|:-----:|:--------:|:------------|
| 0x0800 | **IPv4** | Internet Protocol v4 |
| 0x0806 | **ARP** | Address Resolution Protocol |
| 0x8100 | **VLAN** | 802.1Q VLAN Tag |
| 0x86DD | **IPv6** | Internet Protocol v6 |
| 0x8808 | **Pause** | 802.3x MAC Control Pause |
| 0x88CC | **LLDP** | Link Layer Discovery Protocol |
| 0x88A8 | **Q-in-Q** | 802.1ad Provider Bridging (S-Tag) |
| 0x0842 | **WoLAN** | Wake-on-LAN (old format) |

---

## 7. Common MAC Addresses

| MAC Address | Type | Description |
|:-----------|:----:|:------------|
| FF-FF-FF-FF-FF-FF | Broadcast | 广播 — 所有设备接收 |
| 01-80-C2-00-00-00 | Multicast | **STP BPDU** — 生成树协议 |
| 01-80-C2-00-00-01 | Multicast | **Pause Frame** — 802.3x 流控 |
| 01-80-C2-00-00-0E | Multicast | **LLDP** — 链路层发现 |
| 01-00-5E-xx-xx-xx | Multicast | **IPv4 Multicast** (01-00-5E = 0100.5Exx.xxxx) |
| 33-33-xx-xx-xx-xx | Multicast | **IPv6 Multicast** (33-33 prefix) |

---

## 8. IP Protocol Numbers

| Value | Protocol | Description |
|:-----:|:--------:|:------------|
| 1 | **ICMP** | Internet Control Message Protocol (ping) |
| 2 | **IGMP** | Internet Group Management Protocol |
| 6 | **TCP** | Transmission Control Protocol |
| 17 | **UDP** | User Datagram Protocol |
| 89 | **OSPF** | Open Shortest Path First (routing) |

---

## 9. Common TCP/UDP Ports

| Port | Protocol | Description |
|:----:|:--------:|:------------|
| 7 | TCP/UDP | **Echo** |
| 21 | TCP | **FTP** — File Transfer Protocol |
| 22 | TCP | **SSH** — Secure Shell |
| 23 | TCP | **Telnet** |
| 53 | TCP/UDP | **DNS** — Domain Name System |
| 67/68 | UDP | **DHCP** — DHCP server/client |
| 80 | TCP | **HTTP** |
| 161 | UDP | **SNMP** — Simple Network Management Protocol |
| 443 | TCP | **HTTPS** |
| 502 | TCP | **Modbus TCP** — 工业自动化 |

---

## 10. Auto-Negotiation Priority Table

```
Priority  Technology            Bit Position
────────  ──────────            ────────────
 1 (highest)  1000BASE-T Full Duplex     Extended reg 0x0F bit15
 2       1000BASE-T Half Duplex     Extended reg 0x0F bit14
 3       100BASE-T2 Full Duplex     BMSR bit9
 4       100BASE-TX Full Duplex     ANAR bit8   ← 最常见协商结果
 5       100BASE-T2 Half Duplex     BMSR bit8
 6       100BASE-TX Half Duplex     ANAR bit7
 7       100BASE-T4                 BMSR bit15  (deprecated)
 8       10BASE-T Full Duplex       ANAR bit6
 9 (lowest) 10BASE-T Half Duplex    ANAR bit5
```

**HCD Algorithm**: Local and partner advertise capabilities. The highest priority technology common to both sides is selected.

**Parallel Detection**: If partner does not support Auto-Neg (no FLP), PHY falls back to detecting signal type: NLP→10M-HD, MLT3→100M-HD.

**Flow Control Resolution** (PAUSE/ASM_DIR bits in ANAR):

| Local Pause | Local Asym | Peer Pause | Peer Asym | Result |
|:-----------:|:----------:|:----------:|:---------:|:-------|
| 1 | X | 1 | X | **Symmetric** (both can pause) |
| 1 | 1 | 0 | 0 | **ASM_DIR Local** (local can send pause) |
| 0 | 0 | 1 | 1 | **ASM_DIR Remote** (peer can send pause) |
| else | | | | **No pause** |

---

## 11. MDIO Frame Format (Clause 22)

64 clock cycles per transaction:

```
Read:  PRE(32) | ST(01) | OP(10) | PHYAD(5) | REGAD(5) | TA(Z0) | DATA(16) | IDLE(Z)
Write: PRE(32) | ST(01) | OP(01) | PHYAD(5) | REGAD(5) | TA(10) | DATA(16) | IDLE(Z)

Field   Bits  Description
─────   ────  ───────────
PRE     32    Preamble — 32 consecutive '1's (PHY sync)
ST       2    Start — "01" for Clause 22 ("00" for Clause 45)
OP       2    Operation — "01"=Write, "10"=Read
PHYAD    5    PHY Address (0x00-0x1F, up to 31 PHYs on bus)
REGAD    5    Register Address (0x00-0x1F)
TA       2    Turnaround — Read: MAC tri-states (Z), PHY drives "0"
                              Write: MAC drives "10"
DATA    16    Register data (MSB first)
IDLE     Z    Bus idle state (MDIO high-Z, pulled high by resistor)
```

**Clause 45** (for 10G/25G/40G PHY): ST="00", uses DEVAD(5bit) + PRTAD(5bit), supports 64K register space via MMD indirect access.

| DEVAD | MMD Type |
|:-----:|:---------|
| 0x00 | PCS |
| 0x01 | PMA/PMD |
| 0x07 | Auto-Negotiation |
| 0x1E | MMAP (mapped to C22) |

---

## 12. Common Debug Commands

### Linux

```bash
ethtool eth0                     # Show PHY negotiation, speed, duplex, link
ethtool -s eth0 speed 100 duplex full autoneg off  # Force mode
mii-tool eth0                    # Legacy PHY status (old systems)
ifconfig eth0                    # Interface config & stats
ifconfig eth0 hw ether 00:11:22:33:44:55  # Change MAC
ip link set eth0 up/down         # Enable/disable interface
tcpdump -i eth0 -XX -vv          # Capture packets (hex + ASCII)
tcpdump -i eth0 -e -n            # Capture with MAC addresses, no DNS
arp -a                           # Show ARP cache
```

### MCU (Embedded)

```c
// Register dump function
void phy_dump(uint8_t addr) {
    for (uint8_t r = 0; r <= 0x1F; r++) {
        uint16_t val = phy_read(addr, r);
        printf("Reg[0x%02X]=0x%04X\n", r, val);
    }
}

// Link status monitor (polling)
void link_monitor(uint8_t addr) {
    uint8_t last = 0;
    while (1) {
        uint16_t bmsr;
        phy_read(addr, 0x01, &bmsr);  // clear latch
        phy_read(addr, 0x01, &bmsr);  // real value
        uint8_t curr = (bmsr >> 2) & 1;
        if (curr != last) printf("Link %s\n", curr ? "UP" : "DOWN");
        last = curr;
        delay_ms(200);
    }
}
```

### Wireshark Display Filters

```
eth.addr == ff:ff:ff:ff:ff:ff      # Broadcast frames
eth.type == 0x0800                  # IPv4 only
vlan.id == 100                      # Specific VLAN
arp                                   # ARP only
tcp.port == 80 || udp.port == 53   # HTTP or DNS
!icmp                                # Exclude ICMP (no ping noise)
```

---

## 13. CRC32 Quick Reference (IEEE 802.3 Ethernet FCS)

```
Polynomial:     0x04C11DB7  (normal form)
Reflected poly: 0xEDB88320  (for lookup-table implementation)
Initial value:  0xFFFFFFFF
Final XOR:      0xFFFFFFFF
Test vector:    CRC32("123456789") = 0xCBF43926

Coverage:       Dst MAC(6) + Src MAC(6) + Type(2) + Payload (incl. VLAN Tag)
                Does NOT cover Preamble, SFD, or FCS itself.
```

```c
// Quick CRC32 table lookup implementation
uint32_t crc32_calculate(const uint8_t *data, uint32_t len) {
    static uint32_t table[256] = {0};
    if (!table[1]) /* init */
        for (int i = 0; i < 256; i++) {
            uint32_t crc = i;
            for (int j = 0; j < 8; j++)
                crc = (crc >> 1) ^ (crc & 1 ? 0xEDB88320 : 0);
            table[i] = crc;
        }
    uint32_t crc = 0xFFFFFFFF;
    for (uint32_t i = 0; i < len; i++)
        crc = table[(crc ^ data[i]) & 0xFF] ^ (crc >> 8);
    return crc ^ 0xFFFFFFFF;
}
```

---

## 14. Bit Manipulation Macros

```c
#define BIT(n)              (1UL << (n))

#define SET_BIT(reg, bit)   ((reg) |= (bit))
#define CLR_BIT(reg, bit)   ((reg) &= ~(bit))
#define GET_BIT(reg, bit)   (((reg) >> (bit)) & 1)

#define GET_FIELD(reg, mask, shift)  (((reg) & (mask)) >> (shift))
#define SET_FIELD(reg, mask, shift, val) \
    ((reg) = ((reg) & ~(mask)) | (((val) << (shift)) & (mask)))

// Convenience for PHY BMCR register
#define PHY_SOFT_RESET(addr) \
    phy_write(addr, 0x00, 0x8000); \
    while (phy_read(addr, 0x00) & 0x8000);
```

---

## 15. PHY Initialization Checklist (Quick Steps)

```
 1. Hardware Reset:  RST_N low ≥10ms → high, wait ~50ms (PLL lock)
 2. Verify PHY ID:   Read reg 0x02-0x03, match known IDs
 3. Software Reset:  BMCR.bit15=1, wait for self-clear
 4. Config Interface: RMII/MII mode via PHY-specific register
 5. Advertise:       Write ANAR with supported capabilities
 6. Start ANEG:      BMCR = 0x1200 (Auto-Neg + Restart)
 7. Wait Link Up:    Poll BMSR.bit5 (ANEG done) + BMSR.bit2 (Link), timeout ~4s
 8. Read Result:     Speed + Duplex from MCSR/PHYSR/ANLPAR
 9. Config MAC:      Set MAC speed/duplex to match PHY
10. Start DMA:       Enable MAC DMA Tx/Rx, begin frame transfer
```

### Common Reset Timing

| PHY | TRST (RST low min) | TINIT (post-reset wait) | Default Addr |
|:---|:------------------:|:-----------------------:|:------------:|
| LAN8720A | 10ms | 50ms | 0x00 |
| DP83848 | 10ms | 50ms | 0x10 |
| KSZ8081 | 10ms | 50ms | 0x01 |
| KSZ9031 | 10ms | 100ms | 0x01 |

---

## 16. BMSR Latching Low (Critical Warning!)

```
BMSR.bit2 (Link Status) is LATCHING LOW:
  - Link UP    → bit2 = 1 (normal)
  - Link DOWN  → bit2 = 0 and HOLDS until register is read
  - Link UP again → bit2 stays 0 until first read, then updates

  ALWAYS read BMSR TWICE:
    phy_read(addr, 0x01, &bmsr);  // #1: clear latch
    phy_read(addr, 0x01, &bmsr);  // #2: true value
    link_up = (bmsr >> 2) & 1;
```

---

## 17. TCP Header Quick Reference

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Src Port (16)         |        Dst Port (16)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Seq Number (32)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Ack Number (32)                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset(4) |Res(3)| Flags(9)    |         Window (16)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Checksum (16)          |       Urgent Ptr (16)         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**TCP Flags**: FIN(0x01), SYN(0x02), RST(0x04), PSH(0x08), ACK(0x10), URG(0x20)

**IP Header Protocol field**: 1=ICMP, 2=IGMP, 6=TCP, 17=UDP
