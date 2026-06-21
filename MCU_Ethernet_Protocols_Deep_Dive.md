# MCU 以太网高级协议深度解读

> 从字节流到状态机 -- 逐字节解析 TCP、VLAN、STP/RSTP、IGMP Snooping、LLDP、流控、Auto-Neg 与 MDIO

---

## 目录

- [一、TCP 协议深度解析](#一tcp-协议深度解析)
- [二、VLAN (802.1Q) 协议深度解析](#二vlan-8021q-协议深度解析)
- [三、STP/RSTP 生成树协议深度解析](#三stprstp-生成树协议深度解析)
- [四、IGMP Snooping 深度解析](#四igmp-snooping-深度解析)
- [五、LLDP 链路层发现协议深度解析](#五lldp-链路层发现协议深度解析)
- [六、802.3x 流控 Pause 帧深度解析](#六8023x-流控-pause-帧深度解析)
- [七、Auto-Negotiation 自动协商协议深度解析](#七auto-negotiation-自动协商协议深度解析)
- [八、MDIO Clause 22 / 45 深度解析](#八mdio-clause-22--45-深度解析)

---

## 一、TCP 协议深度解析

### 1.1 TCP 头部 -- 逐字节结构

TCP 头部最小 20 字节，最大 60 字节（含选项）。Ethernet 帧中 EtherType = `0x0800`（IPv4），IP 协议字段 Protocol = `0x06`（TCP）。

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         源端口 (Src Port)       |       目的端口 (Dst Port)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       序列号 (Sequence Number)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     确认号 (Acknowledgment Number)              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| 偏移 | 保留  |N|C|E|U|A|P|R|S|F|       窗口大小 (Window)        |
|(4bit)|(3bit)  |S|W|C|R|C|S|S|Y|I|                            |
|               | |R|R|G|K|H|T|N|N|                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         校验和 (Checksum)       |       紧急指针 (Urgent Ptr)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    选项 (Options, 可变长度)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**实际抓包示例**（SYN 报文，从 192.168.1.100:12345 到 192.168.1.1:80）：

```
以太网帧: 00-11-22-33-44-55 → AA-BB-CC-DD-EE-FF  EtherType=0x0800
IP头部:   45 00 00 28 ... 06 ... C0 A8 01 64 C0 A8 01 01
TCP头部:  30 39 00 50   <-- 源端口=0x3039=12345, 目的端口=0x0050=80
          AA BB CC DD   <-- 序列号=0xAABBCCDD
          00 00 00 00   <-- 确认号=0 (SYN 不需要 ACK)
          50 02 20 00   <-- offset=5(20B), flags=SYN, window=8192
          XX XX 00 00   <-- 校验和, 紧急指针=0
```

### 1.2 Flags 字段详细

Offset（高 4 字节的第 1 字节高 4 位）：数据偏移量，单位 4 字节。`0x50` = 20 字节头部。

```
Flags (1 字节, 6 个标志位 + 2 个保留位 + 3 个 ECN 位):
Bit 7,6: CWR, ECE (ECN 拥塞通知)
Bit 5:   URG (紧急指针有效)
Bit 4:   ACK (确认号有效)
Bit 3:   PSH (推送，接收方应立即交付)
Bit 2:   RST (重置连接)
Bit 1:   SYN (同步序列号，建立连接)
Bit 0:   FIN (结束发送)
```

### 1.3 三次握手状态机

```
CLOSED                              CLOSED
   |                                    |
   | (send SYN, seq=x)                  |
   +------> SYN_SENT                    |
              |                         | (listen)
              |                         +------> LISTEN
              | (recv SYN+ACK)                   |
              | (send ACK, seq=x+1)             | (recv SYN)
              +--------> ESTABLISHED <----------+
                                        |
                                        | (recv ACK)
                                        |
                                   ESTABLISHED

四次挥手:
ESTABLISHED
   | (send FIN, seq=u)
   +------> FIN_WAIT_1
              | (recv ACK)
              +------> FIN_WAIT_2
                         | (recv FIN)
                         +------> TIME_WAIT (2MSL timeout) --> CLOSED

另一方:
ESTABLISHED
   | (recv FIN)
   +------> CLOSE_WAIT
              | (send FIN)
              +------> LAST_ACK
                         | (recv ACK)
                         +------> CLOSED
```

### 1.4 序列号与滑动窗口

- 序列号（32 位无符号）：标识从 TCP 发送端到接收端的数据字节流，初始值 ISN 为随机值。
- 确认号：接收方期望收到的下一个字节的序列号。
- 窗口大小（16 位）：接收方愿意接收的字节数，实现流量控制。

### 1.5 代码：TCP 段解析（C，裸机环境）

```c
#include <stdint.h>
#include <string.h>

/* TCP 头部结构（20 字节） */
typedef struct __attribute__((packed)) {
    uint16_t src_port;      /* 源端口 */
    uint16_t dst_port;      /* 目的端口 */
    uint32_t seq_num;       /* 序列号 */
    uint32_t ack_num;       /* 确认号 */
    uint8_t  offset_reserved; /* 高4位: 数据偏移(×4), 低4位: 保留 */
    uint8_t  flags;         /* 标志位 */
    uint16_t window;        /* 窗口大小 */
    uint16_t checksum;      /* 校验和 */
    uint16_t urgent_ptr;    /* 紧急指针 */
} tcp_header_t;

/* TCP 标志位定义 */
#define TCP_FLAG_FIN  0x01
#define TCP_FLAG_SYN  0x02
#define TCP_FLAG_RST  0x04
#define TCP_FLAG_PSH  0x08
#define TCP_FLAG_ACK  0x10
#define TCP_FLAG_URG  0x20

/* 获取 TCP 头部长度（字节） */
static inline uint8_t tcp_header_len(uint8_t offset_reserved) {
    return ((offset_reserved >> 4) & 0x0F) * 4;
}

/* TCP 伪头部校验和计算（RFC 793） */
/* 伪头部：源IP(4B) + 目的IP(4B) + 0(1B) + protocol(1B) + TCP长度(2B) */
typedef struct __attribute__((packed)) {
    uint32_t src_ip;
    uint32_t dst_ip;
    uint8_t  zero;
    uint8_t  protocol;  /* 0x06 for TCP */
    uint16_t tcp_len;
} tcp_pseudo_header_t;

uint16_t tcp_checksum(uint32_t src_ip, uint32_t dst_ip,
                      uint8_t *tcp_seg, uint16_t tcp_len) {
    tcp_pseudo_header_t ph;
    uint32_t sum = 0;
    uint16_t *p;
    int i, len;

    ph.src_ip   = src_ip;
    ph.dst_ip   = dst_ip;
    ph.zero     = 0;
    ph.protocol = 0x06;
    ph.tcp_len  = __builtin_bswap16(tcp_len);

    /* 累加伪头部 */
    p = (uint16_t *)&ph;
    for (i = 0; i < (int)sizeof(tcp_pseudo_header_t) / 2; i++) {
        sum += __builtin_bswap16(p[i]);
    }

    /* 累加 TCP 段 */
    p = (uint16_t *)tcp_seg;
    len = tcp_len;
    while (len > 1) {
        sum += __builtin_bswap16(*p++);
        len -= 2;
    }
    if (len == 1) {
        sum += __builtin_bswap16((*p) & 0xFF00);
    }

    /* 进位折叠 */
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }

    return (uint16_t)(~sum & 0xFFFF);
}

/* 解析 TCP 段的示例 */
int tcp_parse_segment(uint8_t *frame, uint16_t frame_len) {
    /* 假设 Ethernet + IP 头部已处理, frame 指向 TCP 头部 */
    tcp_header_t *tcp = (tcp_header_t *)frame;

    uint16_t src_port = __builtin_bswap16(tcp->src_port);
    uint16_t dst_port = __builtin_bswap16(tcp->dst_port);
    uint32_t seq      = __builtin_bswap32(tcp->seq_num);
    uint32_t ack      = __builtin_bswap32(tcp->ack_num);
    uint8_t  hdr_len  = tcp_header_len(tcp->offset_reserved);
    uint8_t  flg      = tcp->flags;

    /* 检查标志位 */
    if (flg & TCP_FLAG_SYN) {
        /* SYN 报文处理 */
    }
    if ((flg & (TCP_FLAG_SYN | TCP_FLAG_ACK)) == (TCP_FLAG_SYN | TCP_FLAG_ACK)) {
        /* SYN-ACK 报文处理 */
    }

    return 0;
}
```

### 1.6 代码：LwIP raw API 回调示例

```c
#include "lwip/pbuf.h"
#include "lwip/tcp.h"

/* 连接建立回调 */
static err_t my_connected_cb(void *arg, struct tcp_pcb *tpcb, err_t err) {
    if (err == ERR_OK) {
        /* 发送 HTTP GET 请求 */
        const char *req = "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
        tcp_write(tpcb, req, strlen(req), TCP_WRITE_FLAG_COPY);
        tcp_output(tpcb);
    }
    return ERR_OK;
}

/* 接收数据回调 */
static err_t my_recv_cb(void *arg, struct tcp_pcb *tpcb,
                        struct pbuf *p, err_t err) {
    if (p == NULL) {
        /* 连接关闭 */
        tcp_close(tpcb);
        return ERR_OK;
    }

    /* 处理接收到的数据 */
    for (struct pbuf *q = p; q != NULL; q = q->next) {
        uint8_t *data = (uint8_t *)q->payload;
        uint16_t len  = q->len;
        /* 处理 data[0..len-1] */
    }

    tcp_recved(tpcb, p->tot_len); /* 通知 LwIP 已处理 */
    pbuf_free(p);
    return ERR_OK;
}

/* 错误回调 */
static void my_err_cb(void *arg, err_t err) {
    /* 连接异常处理 */
}

/* 启动 TCP 连接 */
void tcp_client_connect_example(void) {
    struct tcp_pcb *pcb = tcp_new();

    if (pcb != NULL) {
        tcp_recv(pcb, my_recv_cb);
        tcp_err(pcb, my_err_cb);

        ip_addr_t server_ip;
        IP4_ADDR(&server_ip, 192, 168, 1, 1); /* 192.168.1.1 */

        tcp_connect(pcb, &server_ip, 80, my_connected_cb); /* 端口 80 */
    }
}

/* TCP 服务器回调 */
static err_t my_accept_cb(void *arg, struct tcp_pcb *newpcb, err_t err) {
    /* 新连接到达, 注册接收回调 */
    tcp_recv(newpcb, my_recv_cb);
    tcp_err(newpcb, my_err_cb);
    return ERR_OK;
}

void tcp_server_init_example(void) {
    struct tcp_pcb *pcb = tcp_new();

    if (pcb != NULL) {
        /* 绑定端口 8080 */
        tcp_bind(pcb, IP_ADDR_ANY, 8080);

        pcb = tcp_listen(pcb);
        tcp_accept(pcb, my_accept_cb);
    }
}
```

---

## 二、VLAN (802.1Q) 协议深度解析

### 2.1 VLAN Tag -- 逐字节结构

802.1Q Tag 插入在源 MAC 地址之后、EtherType 之前，共 4 字节。

```
原始以太网帧:
+--------+--------+--------+--------+--------+
| Dst MAC| Src MAC| EtherType | Payload       |
| (6B)   | (6B)   | (2B=0800)| (46-1500B)    |
+--------+--------+--------+--------+--------+

带 802.1Q Tag:
+--------+--------+----------+--------+--------+
| Dst MAC| Src MAC| 802.1Q   | EtherType|Payload|
| (6B)   | (6B)   | Tag (4B) | (2B)    |       |
+--------+--------+----------+--------+--------+
```

**802.1Q Tag 内部结构**（4 字节）：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     TPID (Tag Protocol ID)       |PCP| DEI |     VID          |
|         0x8100                   | 3b| 1b  |     12b          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **TPID (16 bit)** = `0x8100` -- 标识这是一个 802.1Q 帧
- **PCP (3 bit)** = Priority Code Point, 0-7, 优先级映射到 802.1p
  - `0xE000` >> 13 = 7 (最高优先级, 网络控制)
  - `0x0000` >> 13 = 0 (尽力而为)
- **DEI (1 bit)** = Drop Eligible Indicator, 拥塞时是否可丢弃
- **VID (12 bit)** = VLAN Identifier, 范围 1-4094 (0=不指定, 4095=保留)

**实际 Tag 字节**:
```
TPID=0x8100, PCP=5, DEI=0, VID=100
= 0x81 0x00 | (5<<5)<<8 | (100 & 0xFF)  = 0x81 0x00 0xA0 0x64

逐位:  1000 0001 0000 0000 | 1010 0000 0110 0100
       ^---TPID=0x8100---^   ^PCP^D^^--VID=100---^
                               101  0  000001100100
```

Q-in-Q (802.1ad, 服务提供商 VLAN): 两层 Tag，外层 TPID=`0x88A8`（S-Tag），内层 TPID=`0x8100`（C-Tag）。

```
+--------+--------+----------+----------+--------+
| Dst MAC| Src MAC| S-Tag    | C-Tag    |EtherType|
| (6B)   | (6B)   | 0x88A8   | 0x8100   | (2B)   |
+--------+--------+----------+----------+--------+
```

### 2.2 标记 (Tagged) vs 未标记 (Untagged)

| 帧类型 | 是否含 Tag | 适用场景 |
|--------|-----------|----------|
| Untagged | 不含 | 终端设备（PC、服务器） |
| Tagged | 含 802.1Q | 交换机间链路 (Trunk) |

- **PVID (Port VLAN ID)**：当端口收到 **Untagged** 帧时，交换机为其打上 PVID 对应的 Tag。
- **Access 端口**：发往终端的帧剥离 Tag，收到的帧打上 PVID Tag。
- **Trunk 端口**：帧保持 Tag 不变，仅允许特定 VID 列表通过。

### 2.3 Port-based vs Tag-based VLAN

| 类型 | 原理 | 特点 |
|------|------|------|
| Port-based | 端口划分到固定 VLAN | 简单，终端无需感知 VLAN |
| Tag-based | 根据 Tag 中的 VID 划分 | 灵活，支持多 VLAN 混合 |

### 2.4 代码：VLAN Tag 插入与移除（C）

```c
#include <stdint.h>
#include <string.h>

/* VLAN Tag 结构 */
typedef struct __attribute__((packed)) {
    uint16_t tpid;  /* 0x8100 */
    uint16_t tci;   /* PCP(3) + DEI(1) + VID(12) */
} vlan_tag_t;

#define VLAN_TPID        0x8100
#define VLAN_TPID_S_TAG  0x88A8  /* Q-in-Q S-Tag */

/* 构建 TCI 值 */
static inline uint16_t vlan_build_tci(uint8_t pcp, uint8_t dei, uint16_t vid) {
    return ((uint16_t)(pcp & 0x07) << 13) |
           ((uint16_t)(dei & 0x01) << 12) |
           (vid & 0x0FFF);
}

/* 提取字段 */
static inline uint8_t  vlan_get_pcp(uint16_t tci) { return (tci >> 13) & 0x07; }
static inline uint8_t  vlan_get_dei(uint16_t tci) { return (tci >> 12) & 0x01; }
static inline uint16_t vlan_get_vid(uint16_t tci) { return tci & 0x0FFF; }

/**
 * 插入 VLAN Tag
 * buffer: 以太网帧缓冲区 (包含 dstMAC + srcMAC)
 * len:    dstMAC + srcMAC 之后的位置（即在 buffer + 12 处插入）
 * tci:    TCI 值
 * 返回新总长度
 */
int vlan_tag_insert(uint8_t *buffer, int len, uint16_t tci) {
    /* buffer 布局: [dstMAC(6)] [srcMAC(6)] [EtherType(2)] [Payload...] */
    /* 插入后:      [dstMAC(6)] [srcMAC(6)] [Tag(4)] [EtherType(2)] [Payload...] */

    if (len < 14) return -1; /* 以太网帧头至少 14 字节 */

    /* 记下原 EtherType (buffer + 12, 13) */
    uint16_t ethertype = (buffer[12] << 8) | buffer[13];

    /* 将 payload 向后移动 4 字节 */
    for (int i = len - 1; i >= 14; i--) {
        buffer[i + 4] = buffer[i];
    }

    /* 写入 Tag */
    buffer[12] = (VLAN_TPID >> 8) & 0xFF;     /* 0x81 */
    buffer[13] = VLAN_TPID & 0xFF;             /* 0x00 */
    buffer[14] = (tci >> 8) & 0xFF;
    buffer[15] = tci & 0xFF;

    /* 写入原 EtherType (在 Tag 之后) */
    buffer[16] = (ethertype >> 8) & 0xFF;
    buffer[17] = ethertype & 0xFF;

    return len + 4;
}

/**
 * 移除 VLAN Tag
 * 返回: 0=无 Tag, 1=移除成功, -1=错误
 */
int vlan_tag_remove(uint8_t *buffer, int *len) {
    if (*len < 18) return -1;

    uint16_t tpid = (buffer[12] << 8) | buffer[13];

    if (tpid != VLAN_TPID && tpid != VLAN_TPID_S_TAG) {
        return 0; /* 不是 VLAN 帧 */
    }

    uint16_t tci = (buffer[14] << 8) | buffer[15];
    uint16_t ethertype = (buffer[16] << 8) | buffer[17];

    /* 将 payload 向前移动 4 字节 */
    int payload_len = *len - 18; /* 从 buffer[18] 开始 */
    for (int i = 0; i < payload_len; i++) {
        buffer[14 + i] = buffer[18 + i];
    }

    /* 恢复 EtherType */
    buffer[12] = (ethertype >> 8) & 0xFF;
    buffer[13] = ethertype & 0xFF;

    *len -= 4;
    return 1; /* VID = vlan_get_vid(tci) */
}

/* 判断是否是 VLAN 帧（检查 TPID） */
static inline int is_vlan_frame(const uint8_t *buffer) {
    uint16_t tpid = (buffer[12] << 8) | buffer[13];
    return (tpid == VLAN_TPID) || (tpid == VLAN_TPID_S_TAG);
}
```

### 2.5 代码：交换机 VLAN 表配置

```c
#include <stdint.h>
#include <stdbool.h>

/* 交换机端口 VLAN 配置结构 */
typedef struct {
    uint8_t  port_count;          /* 端口数量 */
    uint16_t pvid[16];            /* 每个端口的 PVID */
    uint32_t vlan_port_map[16];   /* VLAN 到端口位图, 索引=VID */
    bool     tagged_members[16][16]; /* [vlan][port] = true 表示 tagged */
} switch_vlan_config_t;

/* 初始化 VLAN 表 */
void vlan_table_init(switch_vlan_config_t *cfg, uint8_t port_count) {
    cfg->port_count = port_count;
    for (int i = 0; i < port_count; i++) {
        cfg->pvid[i] = 1; /* 默认 PVID = 1 */
    }
    for (int i = 0; i < 16; i++) {
        cfg->vlan_port_map[i] = 0;
        for (int j = 0; j < 16; j++) {
            cfg->tagged_members[i][j] = false;
        }
    }
    /* 默认所有端口在 VLAN 1, untagged */
    cfg->vlan_port_map[1] = (port_count < 32) ? ((1 << port_count) - 1) : 0xFFFFFFFF;
}

/* 将端口添加到 VLAN（tagged 或 untagged） */
void vlan_add_port(switch_vlan_config_t *cfg, uint16_t vid,
                   uint8_t port, bool tagged) {
    cfg->vlan_port_map[vid] |= (1 << port);
    cfg->tagged_members[vid][port] = tagged;
}

/* 从 VLAN 移除端口 */
void vlan_remove_port(switch_vlan_config_t *cfg, uint16_t vid, uint8_t port) {
    cfg->vlan_port_map[vid] &= ~(1 << port);
}

/* 转发决策：根据 VID 和目的端口查找出口端口 */
int vlan_lookup_egress(switch_vlan_config_t *cfg, uint16_t vid, uint8_t ingress_port) {
    if (vid > 15) return -1; /* 简化: 只支持 VID 0-15 */
    uint32_t members = cfg->vlan_port_map[vid];

    /* 入端口不在该 VLAN 中？丢弃 */
    if (!(members & (1 << ingress_port))) {
        return -1; /* 丢弃 */
    }

    return members; /* 返回成员位图，由调用方决定具体端口 */
}

/* 出口时判断是否需要保留 Tag */
bool vlan_egress_tagged(switch_vlan_config_t *cfg, uint16_t vid, uint8_t port) {
    return cfg->tagged_members[vid][port];
}
```

---

## 三、STP/RSTP 生成树协议深度解析

### 3.1 BPDU -- 逐字节结构

BPDU (Bridge Protocol Data Unit) 封装在以太网帧中，目的 MAC 地址为组播地址 `01-80-C2-00-00-00`，EtherType=`0x42` (LLC，长度编码)。

**Configuration BPDU 结构**（35 字节）：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Protocol Identifier (2B)      |  Protocol Version ID (1B)     |
|         0x0000                 |  STP=0x00, RSTP=0x02          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  BPDU Type (1B)                |  Flags (1B)                   |
|  Config=0x00, TCN=0x80,        |  TC|TCA|...                   |
|  RSTP=0x02                     |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Root Identifier (8B)                     |
|               Bridge Priority (2B) + MAC (6B)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   Root Path Cost (4B)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Bridge Identifier (8B)                    |
|               Bridge Priority (2B) + MAC (6B)                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Port Identifier (2B)          |  Message Age (2B, /256s)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Max Age (2B, /256s)           |  Hello Time (2B, /256s)       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Forward Delay (2B, /256s)     |  (Version 1 Length=0 for STP)|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**实际 BPDU 抓包示例** (hex)：
```
01 80 C2 00 00 00  -- Dst MAC (STP 组播)
00 11 22 33 44 55  -- Src MAC (本机)
00 26              -- LLC: DSAP=0x42, SSAP=0x42, Ctrl=0x03

00 00              -- Protocol ID = 0x0000 (802.1D)
00                 -- Version = 0 (STP)
00                 -- Type = 0x00 (Config BPDU)
00                 -- Flags = 0
80 00 00 11 22 33 44 55  -- Root ID: Pri=0x8000, MAC=00-11-22-33-44-55
00 00 00 00         -- Root Path Cost = 0
80 00 00 11 22 33 44 55  -- Bridge ID: Pri=0x8000, MAC=00-11-22-33-44-55
80 01              -- Port ID: Pri=0x80, Port=1
00 00              -- Message Age = 0
00 14              -- Max Age = 20s (0x14 × 256 = 5120 ticks?)
00 02              -- Hello Time = 2s
00 0F              -- Forward Delay = 15s
```

**RSTP 的 Flags 字段** (802.1w)：

```
Bit 7: TCA (Topology Change Acknowledgment)
Bit 6: Agreement
Bit 5: Forwarding
Bit 4: Learning
Bit 3: Port Role (2bit, upper)
Bit 2: Port Role (2bit, lower)
       00 = Unknown / Deprecated
       01 = Alternate / Backup
       10 = Root
       11 = Designated
Bit 1: Proposal
Bit 0: TC (Topology Change)
```

### 3.2 STP 端口状态机

```
  +-----------+
  | Disabled  |  (管理关闭或链路断开)
  +-----------+
       |
       | (链路 UP)
       v
  +-----------+
  | Blocking  |  (不转发数据，不学习 MAC，仅收 BPDU)
  +-----------+
       |
       | (Max Age 超时，成为 Designated Port 或 Root Port)
       v
  +-----------+
  | Listening |  (不转发数据，不学习 MAC，发送/接收 BPDU)
  +-----------+
       |
       | (Forward Delay 超时，通常 15s)
       v
  +-----------+
  | Learning  |  (不转发数据，学习 MAC 地址)
  +-----------+
       |
       | (Forward Delay 超时)
       v
  +-----------+
  |Forwarding |  (正常转发数据)
  +-----------+
```

**STP 状态转换条件**：
- `Blocking -> Listening`：端口被选为 Root Port 或 Designated Port
- `Listening -> Learning`：Forward Delay 超时（默认 15s）
- `Learning -> Forwarding`：Forward Delay 超时（默认 15s）
- 任何状态 -> `Blocking`：收到更优 BPDU (端口不再是最优路径)

### 3.3 RSTP 改进：端口角色与状态

RSTP 将 STP 的 5 种状态简化为 3 种状态，引入 4 种端口角色：

**端口角色**：
- **Root Port (RP)**：到达根桥路径最优的端口
- **Designated Port (DP)**：每个网段中到达根桥最优的端口
- **Alternate Port (AP)**：Root Port 的备份（阻塞状态，接收更优 BPDU）
- **Backup Port (BP)**：同一网段中 Designated Port 的备份

**RSTP 状态**：
```
  +-----------+-----------+----------+
  | STP 状态  | RSTP 状态 | 转发数据 |
  +-----------+-----------+----------+
  | Disabled  | Discarding|  否      |
  | Blocking  | Discarding|  否      |
  | Listening | Discarding|  否      |
  | Learning  | Learning  |  否(学习)|
  | Forwarding| Forwarding|  是      |
  +-----------+-----------+----------+
```

**RSTP 核心改进**：
1. **Edge Port**：连接终端的端口，进入 Forwarding 不需延迟
2. **Proposal/Agreement**：快速协商，替代 STP 的计时器
3. **PA 机制**：DP 发送 Proposal (flags=0x02)，下游 RP 回 Agreement (flags=0x40) ，立即进入 Forwarding

### 3.4 代码：BPDU 解析（C）

```c
#include <stdint.h>
#include <string.h>

/* MAC 地址长度 */
#define ETH_ALEN 6

/* BPDU 头部 */
typedef struct __attribute__((packed)) {
    uint16_t protocol_id;       /* 0x0000 */
    uint8_t  version;           /* STP=0x00, RSTP=0x02, MSTP=0x03 */
    uint8_t  type;              /* Config=0x00, TCN=0x80, RSTP=0x02 */
    uint8_t  flags;             /* TC, TCA, etc. */
    uint8_t  root_id[8];        /* Priority(2) + MAC(6) */
    uint32_t root_path_cost;    /* Root Path Cost */
    uint8_t  bridge_id[8];      /* Priority(2) + MAC(6) */
    uint16_t port_id;           /* Port Priority(1) + Port Number(1) */
    uint16_t message_age;       /* 单位 1/256 秒 */
    uint16_t max_age;           /* 单位 1/256 秒 */
    uint16_t hello_time;        /* 单位 1/256 秒 */
    uint16_t forward_delay;     /* 单位 1/256 秒 */
} bpdu_config_t;

/* BPDU 类型 */
#define BPDU_TYPE_CONFIG  0x00
#define BPDU_TYPE_TCN     0x80
#define BPDU_TYPE_RSTP    0x02

/* 版本 */
#define STP_VERSION  0
#define RSTP_VERSION 2
#define MSTP_VERSION 3

/* BPDU Flags */
#define BPDU_FLAG_TC       0x01  /* Topology Change */
#define BPDU_FLAG_PROPOSAL 0x02  /* RSTP Proposal */
#define BPDU_FLAG_TCA      0x80  /* Topology Change Ack */
#define BPDU_FLAG_AGREEMENT 0x40 /* RSTP Agreement */
#define BPDU_FLAG_FORWARDING 0x20
#define BPDU_FLAG_LEARNING  0x10

/* 从 Bridge ID / Root ID 中提取优先级 */
static inline uint16_t bridge_id_priority(const uint8_t id[8]) {
    return ((uint16_t)id[0] << 8) | id[1];
}

/* 比较两个 Bridge ID (优先级 + MAC) */
/* 返回: -1=a<b, 0=a==b, 1=a>b */
int bridge_id_compare(const uint8_t a[8], const uint8_t b[8]) {
    uint16_t pa = bridge_id_priority(a);
    uint16_t pb = bridge_id_priority(b);
    if (pa != pb) return (pa < pb) ? -1 : 1;
    for (int i = 2; i < 8; i++) {
        if (a[i] != b[i]) return (a[i] < b[i]) ? -1 : 1;
    }
    return 0;
}

/* 端口 ID */
static inline uint8_t  port_id_priority(uint16_t port_id) { return (port_id >> 8) & 0xFF; }
static inline uint8_t  port_id_number(uint16_t port_id)   { return port_id & 0xFF; }

/* 提取 MAC 地址 (从 ID[2..7]) */
void bridge_id_mac(const uint8_t id[8], uint8_t mac[ETH_ALEN]) {
    memcpy(mac, id + 2, ETH_ALEN);
}

/* BPDU 处理函数 */
/* 返回值: 0=丢弃, 1=更新信息, 2=拓扑变更 */
int stp_process_bpdu(const uint8_t *frame, int len, uint8_t *best_root_id,
                     uint32_t *best_path_cost, uint8_t *best_bridge_id) {
    if (len < (int)sizeof(bpdu_config_t)) return 0;

    const bpdu_config_t *bpdu = (const bpdu_config_t *)frame;

    if (bpdu->type == BPDU_TYPE_CONFIG) {
        /* 比较 BPDU 根桥信息与本地最优信息 */
        int cmp = bridge_id_compare(bpdu->root_id, best_root_id);
        if (cmp < 0) {
            /* 收到更优的根桥信息 */
            memcpy(best_root_id, bpdu->root_id, 8);
            *best_path_cost = __builtin_bswap32(bpdu->root_path_cost)
                + 1; /* 增加本端路径开销 */
            memcpy(best_bridge_id, bpdu->bridge_id, 8);
            return 1;
        } else if (cmp == 0) {
            /* 相同根桥，比较路径开销 */
            uint32_t rcvd_cost = __builtin_bswap32(bpdu->root_path_cost);
            if (rcvd_cost < *best_path_cost) {
                *best_path_cost = rcvd_cost + 1;
                memcpy(best_bridge_id, bpdu->bridge_id, 8);
                return 1;
            }
        }

        /* 检查 TC 标志 */
        if (bpdu->flags & BPDU_FLAG_TC) {
            return 2; /* 拓扑变更 */
        }
    }

    if (bpdu->type == BPDU_TYPE_TCN) {
        /* Topology Change Notification -- 需要向上级桥发送 TCA */
        return 2;
    }

    return 0;
}

/* RSTP Proposal / Agreement 处理 */
int rstp_process_flags(uint8_t flags, uint8_t port_role) {
    int action = 0;

    if (flags & BPDU_FLAG_PROPOSAL) {
        /* 上游发出 Proposal, 如果本端口是 Alternate/Backup, 则同意 */
        action |= 0x01; /* 需要回复 Agreement */
    }

    if (flags & BPDU_FLAG_AGREEMENT) {
        /* 下游同意, 可以立即进入 Forwarding */
        action |= 0x02;
    }

    if (flags & BPDU_FLAG_TC) {
        /* 拓扑变更, 刷新 MAC 表 */
        action |= 0x04;
    }

    return action;
}
```

---

## 四、IGMP Snooping 深度解析

### 4.1 IGMP 报文结构

IGMP (Internet Group Management Protocol) 用于管理多播组成员关系，封装在 IP 包中，IP 协议号 = `0x02`。

**IGMP v2 Membership Query** (目的 IP: `224.0.0.1`, TTL=1)：
```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type=0x11    | Max Resp Time |         Checksum              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Group Address (0.0.0.0 for General Query) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**IGMP v2 Membership Report** (目的 IP = 组播地址)：
```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type=0x16    | Max Resp Time |         Checksum              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Group Address (组播地址)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**IGMP v2 Leave Group** (目的 IP: `224.0.0.2`, TTL=1)：
```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type=0x17    | Max Resp Time |         Checksum              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Group Address (组播地址)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**实际报文 hex 示例**（Report for 239.1.2.3）：
```
46 00 00 20 02 00 00 00 02 11  ...  -- IP 头部
C0 A8 01 64 EF 01 02 03             -- 192.168.1.100 -> 239.1.2.3

16 00 FA 00                         -- Type=0x16(Report), 校验和字节序...
EF 01 02 03                         -- 组播地址 239.1.2.3
```

### 4.2 IGMP Snooping 原理

交换机侦听 IGMP 报文，建立多播组到端口的映射表，实现多播帧的精确转发（而不是向所有端口广播）。

**IGMP Snooping 状态机（每个多播组，每个端口）**：

```
    +------------------+
    |    Non-Member    |  (初始状态)
    +------------------+
         |         ^
         | (收到   | (Group Timer 超时
         |  Report) |  或 Leave Done)
         v         |
    +------------------+
    |     Idle       |  (端口是该组的成员)
    +------------------+
         |         ^
         | (Query  | (Group Timer 超时)
         |  应答)  |
         v         |
    +------------------+
    |  Active Member  |
    +------------------+
```

### 4.3 代码：IGMP Snooping 表管理

```c
#include <stdint.h>
#include <string.h>

/* IGMP 表项 */
typedef struct {
    uint32_t    group_addr;     /* 组播地址 (网络字节序) */
    uint32_t    port_bitmap;    /* 哪些端口有成员 */
    uint32_t    timer_ticks;    /* 剩余 ticks, 超时移除 */
    uint8_t     valid;
} igmp_group_entry_t;

#define IGMP_MAX_GROUPS 32

/* IGMP Snooping 表 */
typedef struct {
    igmp_group_entry_t entries[IGMP_MAX_GROUPS];
    int                entry_count;
} igmp_snooping_table_t;

/* 初始化 */
void igmp_table_init(igmp_snooping_table_t *tbl) {
    memset(tbl, 0, sizeof(*tbl));
}

/* 查找组播地址 */
static igmp_group_entry_t *igmp_table_lookup(igmp_snooping_table_t *tbl,
                                              uint32_t group_addr) {
    for (int i = 0; i < IGMP_MAX_GROUPS; i++) {
        if (tbl->entries[i].valid &&
            tbl->entries[i].group_addr == group_addr) {
            return &tbl->entries[i];
        }
    }
    return NULL;
}

/* 分配新表项 */
static igmp_group_entry_t *igmp_table_alloc(igmp_snooping_table_t *tbl,
                                             uint32_t group_addr) {
    /* 先尝试重用无效表项 */
    for (int i = 0; i < IGMP_MAX_GROUPS; i++) {
        if (!tbl->entries[i].valid) {
            tbl->entries[i].group_addr   = group_addr;
            tbl->entries[i].port_bitmap  = 0;
            tbl->entries[i].timer_ticks  = 260; /* ~260s 超时 */
            tbl->entries[i].valid        = 1;
            return &tbl->entries[i];
        }
    }
    return NULL; /* 表满 */
}

/**
 * 处理 IGMP Report
 * 将端口加入组播组
 */
void igmp_handle_report(igmp_snooping_table_t *tbl,
                        uint32_t group_addr, uint8_t ingress_port) {
    igmp_group_entry_t *entry = igmp_table_lookup(tbl, group_addr);

    if (entry == NULL) {
        entry = igmp_table_alloc(tbl, group_addr);
        if (entry == NULL) return; /* 表满，忽略 */
    }

    /* 将端口加入成员位图 */
    entry->port_bitmap |= (1 << ingress_port);
    entry->timer_ticks  = 260; /* 刷新计时器 */
}

/**
 * 处理 IGMP Leave
 * 在特定端口上移除组播组
 */
void igmp_handle_leave(igmp_snooping_table_t *tbl,
                       uint32_t group_addr, uint8_t ingress_port) {
    igmp_group_entry_t *entry = igmp_table_lookup(tbl, group_addr);
    if (entry == NULL) return;

    /* 移除端口 */
    entry->port_bitmap &= ~(1 << ingress_port);

    /* 如果所有端口都离开了，移除表项 */
    if (entry->port_bitmap == 0) {
        entry->valid = 0;
    } else {
        /* 向组播组发送 Group-Specific Query */
        /* 等待响应，如无响应则移除 */
        entry->timer_ticks = 10; /* Last Member Query Interval */
    }
}

/**
 * 周期性调用（每 1 秒），处理超时
 */
void igmp_table_tick(igmp_snooping_table_t *tbl) {
    for (int i = 0; i < IGMP_MAX_GROUPS; i++) {
        if (!tbl->entries[i].valid) continue;

        if (tbl->entries[i].timer_ticks > 0) {
            tbl->entries[i].timer_ticks--;
        }

        if (tbl->entries[i].timer_ticks == 0) {
            tbl->entries[i].valid = 0; /* 超时移除 */
        }
    }
}

/**
 * 多播转发决策
 * 根据组播目标 IP 查找出口端口位图
 * 返回: 端口位图 (bit n = 端口 n 需要转发)
 */
uint32_t igmp_lookup_egress(igmp_snooping_table_t *tbl,
                            uint32_t dst_ip) {
    igmp_group_entry_t *entry = igmp_table_lookup(tbl, dst_ip);
    if (entry == NULL) {
        /* 未找到 => 向所有端口广播 (除了入端口) */
        return 0;
    }
    return entry->port_bitmap;
}

/* 处理 IGMP Query */
void igmp_handle_query(igmp_snooping_table_t *tbl,
                       uint32_t group_addr, uint8_t ingress_port) {
    if (group_addr == 0) {
        /* General Query: 查询所有组, 刷新所有条目的计时器 */
        for (int i = 0; i < IGMP_MAX_GROUPS; i++) {
            if (tbl->entries[i].valid) {
                tbl->entries[i].timer_ticks = 100; /* 随机 0-100 */
            }
        }
    } else {
        /* Group-Specific Query */
        igmp_group_entry_t *entry = igmp_table_lookup(tbl, group_addr);
        if (entry) {
            entry->timer_ticks = 10;
        }
    }
}
```

---

## 五、LLDP 链路层发现协议深度解析

### 5.1 LLDP 帧结构

LLDP 帧使用组播目的 MAC `01-80-C2-00-00-0E`，EtherType=`0x88CC`，基于 TLV (Type-Length-Value) 格式。

**LLDPDU 结构** (每个 TLV 最少 2 字节，最大包含多个 TLV)：

```
+--------+--------+--------+--------+---------+----------+
| Dst MAC| Src MAC| 0x88CC | LLDPDU (TLVs)  | 结束TLV  |
| (6B)   | (6B)   | (2B)   | (可变)          | 00 00    |
+--------+--------+--------+--------+---------+----------+
```

**TLV 格式**（每个 TLV 头部 2 字节）：

```
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-----+
|   Type (7 bit)   |   Length (9 bit)   |Value|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-----+
```

**标准 LLDP TLV 类型**：

| Type | 名称 | 长度 | 说明 |
|------|------|------|------|
| 0 | End of LLDPDU | 2B | 00 00，标记结束 |
| 1 | Chassis ID | 可变 | 设备标识 |
| 2 | Port ID | 可变 | 端口标识 |
| 3 | TTL | 2B | 生存时间（秒） |
| 4 | Port Description | 可变 | 端口描述 |
| 5 | System Name | 可变 | 系统名称 |
| 6 | System Description | 可变 | 系统描述 |
| 7 | System Capabilities | 4B | 系统能力 |
| 8 | Management Address | 可变 | 管理地址 |

**实际 LLDP 帧示例**：
```
01 80 C2 00 00 0E  -- Dst MAC (LLDP 组播 MAC)
00 11 22 33 44 55  -- Src MAC
88 CC              -- EtherType (0x88CC)

02 07 04 00 11 22 33 44 55  -- Chassis ID TLV
                              Type=1(001), Len=7
                              SubType=4(MAC), MAC=00-11-22-33-44-55

04 05 05 01 00 01          -- Port ID TLV
                              Type=2(010), Len=5
                              SubType=5(ifName), "eth0"=65 74 68 30...?

06 02 00 78                -- TTL TLV
                              Type=3(011), Len=2
                              Value=120(秒)

00 00                      -- End of LLDPDU TLV
```

**Chassis ID SubType**：
```
1 = Chassis component
2 = Interface alias
3 = Port component
4 = MAC address
5 = Network address
6 = Interface name
7 = Locally assigned
```

**System Capabilities 位图** (4 字节)：
```
Bit 0: Other
Bit 1: Repeater
Bit 2: Bridge (交换机)
Bit 3: WLAN Access Point
Bit 4: Router
Bit 5: Telephone
Bit 6: DOCSIS cable device
Bit 7: Station (终端)
Bit 8-15: 保留
```

### 5.2 代码：LLDP 报文生成

```c
#include <stdint.h>
#include <string.h>

#define ETH_ALEN 6

/* LLDP 组播 MAC */
static const uint8_t lldp_mcast_mac[ETH_ALEN] = {0x01, 0x80, 0xC2, 0x00, 0x00, 0x0E};

/* TLV 结构 */
typedef struct __attribute__((packed)) {
    uint16_t type_len;  /* Type(7) | Length(9) */
    uint8_t  value[];   /* 柔性数组 */
} lldp_tlv_t;

#define LLDP_TYPE_SHIFT 9
#define LLDP_TYPE_MASK  0xFE00
#define LLDP_LEN_MASK   0x01FF

#define LLDP_TYPE_END        0
#define LLDP_TYPE_CHASSIS    1
#define LLDP_TYPE_PORT       2
#define LLDP_TYPE_TTL        3
#define LLDP_TYPE_PORT_DESC  4
#define LLDP_TYPE_SYS_NAME   5
#define LLDP_TYPE_SYS_DESC   6
#define LLDP_TYPE_CAP        7

/* 构建 TLV 头部 */
static inline uint16_t lldp_make_tlv_hdr(uint8_t type, uint16_t len) {
    return ((uint16_t)(type & 0x7F) << 9) | (len & LLDP_LEN_MASK);
}

/* 填充 TLV */
static int lldp_add_tlv(uint8_t *buf, int offset, uint8_t type,
                        const uint8_t *value, uint16_t len) {
    uint16_t *hdr = (uint16_t *)(buf + offset);
    *hdr = __builtin_bswap16(lldp_make_tlv_hdr(type, len));
    memcpy(buf + offset + 2, value, len);
    return offset + 2 + len;
}

/* Chassis ID SubType: 4 = MAC Address */
#define LLDP_CHASSIS_SUBTYPE_MAC 4
/* Port ID SubType: 5 = Interface Name */
#define LLDP_PORT_SUBTYPE_IFNAME 5

/**
 * 生成 LLDP 报文
 * buf:      输出缓冲区 (至少 1514 字节)
 * src_mac:  源 MAC 地址 (6 字节)
 * sys_name: 系统名称字符串
 * port_name:端口名称字符串
 * ttl:      TTL 值 (秒)
 * 返回: 报文总长度
 */
int lldp_build_frame(uint8_t *buf, const uint8_t src_mac[ETH_ALEN],
                     const char *sys_name, const char *port_name,
                     uint16_t ttl) {
    int offset = 0;

    /* Ethernet 头部 (14B) */
    memcpy(buf, lldp_mcast_mac, ETH_ALEN);
    offset += ETH_ALEN;

    memcpy(buf + offset, src_mac, ETH_ALEN);
    offset += ETH_ALEN;

    buf[offset++] = 0x88;  /* EtherType = 0x88CC */
    buf[offset++] = 0xCC;

    /* 1. Chassis ID TLV (SubType = 4, MAC Address) */
    {
        uint8_t chassis_value[7];
        chassis_value[0] = LLDP_CHASSIS_SUBTYPE_MAC;  /* SubType */
        memcpy(chassis_value + 1, src_mac, ETH_ALEN); /* MAC */
        offset = lldp_add_tlv(buf, offset, LLDP_TYPE_CHASSIS,
                              chassis_value, 7);
    }

    /* 2. Port ID TLV (SubType = 5, Interface Name) */
    {
        uint8_t port_value[32];
        uint8_t name_len = (uint8_t)strlen(port_name);
        if (name_len > 31) name_len = 31;
        port_value[0] = LLDP_PORT_SUBTYPE_IFNAME;
        memcpy(port_value + 1, port_name, name_len);
        offset = lldp_add_tlv(buf, offset, LLDP_TYPE_PORT,
                              port_value, 1 + name_len);
    }

    /* 3. TTL TLV */
    {
        uint16_t ttl_be = __builtin_bswap16(ttl);
        offset = lldp_add_tlv(buf, offset, LLDP_TYPE_TTL,
                              (uint8_t *)&ttl_be, 2);
    }

    /* 4. System Name TLV */
    {
        uint8_t name_len = (uint8_t)strlen(sys_name);
        if (name_len > 255) name_len = 255;
        offset = lldp_add_tlv(buf, offset, LLDP_TYPE_SYS_NAME,
                              (uint8_t *)sys_name, name_len);
    }

    /* 5. System Capabilities TLV */
    {
        uint8_t caps[4] = {0x00, 0x00, 0x00, 0x00};
        caps[1] = 0x04; /* Bit 2 = Bridge (交换机) */
        caps[0] = 0x04; /* Bit 10 = Bridge enabled (shift?) */
        /* 实际: Capabilities(2B) + Enabled Capabilities(2B) */
        /* 简化: 设置为支持 Bridge + Station */
        caps[1] = 0x14; /* Bridge(2) + Station(7) */
        caps[0] = 0x14; /* 启用相同的 */
        offset = lldp_add_tlv(buf, offset, LLDP_TYPE_CAP,
                              caps, 4);
    }

    /* 6. End of LLDPDU TLV (00 00) */
    {
        uint16_t end = 0;
        offset = lldp_add_tlv(buf, offset, LLDP_TYPE_END,
                              (uint8_t *)&end, 0);
    }

    return offset;
}

/**
 * LLDP 接收解析
 * 提取 Chassis ID 和 System Name
 */
void lldp_parse_frame(const uint8_t *buf, int len) {
    const uint8_t *lldpdu = buf + 14; /* 跳过 Ethernet 头部 */

    int offset = 0;
    int llpdulen = len - 14;

    while (offset + 2 <= llpdulen) {
        uint16_t hdr = __builtin_bswap16(*(uint16_t *)(lldpdu + offset));
        uint8_t  type = (hdr >> 9) & 0x7F;
        uint16_t tlv_len = hdr & LLDP_LEN_MASK;

        if (type == LLDP_TYPE_END) break; /* 结束 */

        if (offset + 2 + tlv_len > llpdulen) break; /* 越界检查 */

        const uint8_t *value = lldpdu + offset + 2;

        switch (type) {
        case LLDP_TYPE_CHASSIS: {
            uint8_t subtype = value[0];
            /* Chassis ID 是 value[1..1+tlv_len-1] */
            break;
        }
        case LLDP_TYPE_SYS_NAME: {
            /* System Name 是 value[0..tlv_len-1] (字符串) */
            break;
        }
        case LLDP_TYPE_CAP: {
            uint16_t caps = (value[0] << 8) | value[1];
            uint16_t enabled = (value[2] << 8) | value[3];
            /* caps bit 2 = Bridge */
            if (caps & 0x0004) {
                /* 这是一个交换机 */
            }
            break;
        }
        }

        offset += 2 + tlv_len;
    }
}
```

---

## 六、802.3x 流控 Pause 帧深度解析

### 6.1 Pause 帧结构

Pause 帧是 MAC Control 帧的一种，EtherType=`0x8808`，Opcode=`0x0001`。

```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Dst MAC (6B)                          |
|                  01-80-C2-00-00-01                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Src MAC (6B)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       MAC Control EtherType    |     Opcode                   |
|           0x8808               |     0x0001 (PAUSE)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Pause Time (2B)       |         Reserved (42B)       |
|        单位: 512 bit-times     |       (全 0x00)               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             FCS (4B)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Pause 时间计算**：
- Pause Time 的单位是 **512 bit-times**
- 对于 100Mbps 以太网，1 bit-time = 10ns，512 bit-times = 5.12us
- 例如 Pause Time = 0xFFFF (65535)，暂停持续 = 65535 x 5.12us ≈ 335.5ms
- 对于 1Gbps 以太网，1 bit-time = 1ns，512 bit-times = 0.512us
- Pause Time = 0xFFFF 时，暂停持续 ≈ 33.55ms

**完整 Pause 帧 hex 示例**：
```
01 80 C2 00 00 01  -- 组播 MAC (MAC Control)
00 11 22 33 44 55  -- Src MAC
88 08              -- EtherType = 0x8808
00 01              -- Opcode = 0x0001 (PAUSE)
FF FF              -- Pause Time = 65535 (尽力暂停)
00 00 00 ...       -- Reserved (42 字节全零)
XX XX XX XX        -- FCS
```

### 6.2 代码：Pause 帧生成

```c
#include <stdint.h>
#include <string.h>

#define ETH_ALEN 6

/* MAC Control 组播目的地址 */
static const uint8_t pause_dst_mac[ETH_ALEN] = {0x01, 0x80, 0xC2, 0x00, 0x00, 0x01};

/* Pause 帧结构 */
typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[ETH_ALEN];     /* 01-80-C2-00-00-01 */
    uint8_t  src_mac[ETH_ALEN];     /* 源 MAC */
    uint16_t ethertype;             /* 0x8808 */
    uint16_t opcode;                /* 0x0001 */
    uint16_t pause_time;            /* 单位: 512 bit-times */
    uint8_t  reserved[42];          /* 全 0 */
} pause_frame_t;

#define PAUSE_ETHERTYPE   0x8808
#define PAUSE_OPCODE      0x0001

/**
 * 生成 Pause 帧
 * buf:      输出缓冲区 (至少 64 字节)
 * src_mac:  源 MAC
 * pause_time: 暂停时间 (0-65535, 单位 512 bit-times)
 * 返回帧长度 (固定 64 字节，包含 FCS 则 64)
 *
 * 注意: pause_time=0 表示取消暂停 (XON)
 */
int pause_frame_build(uint8_t *buf, const uint8_t src_mac[ETH_ALEN],
                      uint16_t pause_time) {
    pause_frame_t *pf = (pause_frame_t *)buf;

    memcpy(pf->dst_mac, pause_dst_mac, ETH_ALEN);
    memcpy(pf->src_mac, src_mac, ETH_ALEN);
    pf->ethertype  = __builtin_bswap16(PAUSE_ETHERTYPE);  /* 0x8808 */
    pf->opcode     = __builtin_bswap16(PAUSE_OPCODE);     /* 0x0001 */
    pf->pause_time = __builtin_bswap16(pause_time);
    memset(pf->reserved, 0, 42);

    return sizeof(pause_frame_t);
}

/* XON 帧: pause_time = 0 (恢复发送) */
#define pause_frame_send_xon(buf, src_mac) \
    pause_frame_build(buf, src_mac, 0)

/* XOFF 帧: pause_time = 最大值 (完全暂停) */
#define pause_frame_send_xoff(buf, src_mac) \
    pause_frame_build(buf, src_mac, 0xFFFF)

/**
 * 基于接收缓冲区水位线自动发送 Pause 帧
 *
 * rx_buf_used: 当前接收缓冲区使用量
 * rx_buf_size: 接收缓冲区总大小
 * threshold_high:  触发 XOFF 的水位线 (如 75%)
 * threshold_low:   触发 XON 的水位线 (如 25%)
 */
static int pause_last_state = 0; /* 0=normal, 1=paused */

void pause_flow_control_update(uint8_t *buf, const uint8_t src_mac[ETH_ALEN],
                                uint32_t rx_buf_used, uint32_t rx_buf_size,
                                uint32_t threshold_high, uint32_t threshold_low) {
    uint32_t usage_pct = (rx_buf_used * 100) / rx_buf_size;

    if (usage_pct >= threshold_high && !pause_last_state) {
        /* 缓冲区水位过高，发送 XOFF */
        pause_frame_build(buf, src_mac, 0xFFFF);
        pause_last_state = 1;
        /* EMAC 发送该帧 */
        // emac_send_frame(buf, 64);
    }
    else if (usage_pct <= threshold_low && pause_last_state) {
        /* 缓冲区水位恢复，发送 XON */
        pause_frame_build(buf, src_mac, 0);
        pause_last_state = 0;
        // emac_send_frame(buf, 64);
    }
}

/**
 * 解析收到的 Pause 帧
 * 返回: 0=非 Pause 帧, 其他=pause_time
 */
uint16_t pause_frame_parse(const uint8_t *buf, int len) {
    if (len < (int)sizeof(pause_frame_t)) return 0;

    const pause_frame_t *pf = (const pause_frame_t *)buf;

    /* 检查目的 MAC */
    if (memcmp(pf->dst_mac, pause_dst_mac, ETH_ALEN) != 0) return 0;
    /* 检查 EtherType */
    if (__builtin_bswap16(pf->ethertype) != PAUSE_ETHERTYPE) return 0;
    /* 检查 Opcode */
    if (__builtin_bswap16(pf->opcode) != PAUSE_OPCODE) return 0;

    return __builtin_bswap16(pf->pause_time);
}
```

---

## 七、Auto-Negotiation 自动协商协议深度解析

### 7.1 FLP Burst 结构

Auto-Negotiation 通过 FLP (Fast Link Pulse) Burst 在 PHY 之间交换能力信息。

**FLP Burst 时序** (100Base-TX)：
```
              +--+  +--+  +--+  +--+  +--+
Normal Link   |  |  |  |  |  |  |  |  |  |
Pulse (NLP)   +--+  +--+  +--+  +--+  +--+
               ^     ^     ^     ^     ^
               |100us|100us|100us|100us|
              间隔   间隔   间隔   间隔

实际上: FLP Burst = 17 个脉冲 (33 个时钟位置, 每 62.5us 一个脉冲)
总持续时间约 2ms, 每 16ms 重复一次
```

**时钟脉冲 (Clock Pulse)** 位置固定，**数据脉冲 (Data Pulse)** 在交替位置，编码 16 位能力码。

### 7.2 Base Page (基础页)

Base Page 是 16 位的链接码字 (Link Code Word, LCW)：

```
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|S0|S1|S2|S3|S4| A0| A1| A2| A3| A4| A5| A6| A7| RF|ACK|NP|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  选择器字段     |   技术能力字段      |  扩展 |RF|ACK|NP|
|  (5 bit)       |   (8 bit)          |       |   |   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**选择器 (Selector, 5 bit)**：
- `00001` = IEEE 802.3

**技术能力 (Technology Ability, 8 bit)**：
```
Bit 15 (A0): 10BASE-T (半双工)
Bit 14 (A1): 10BASE-T (全双工)
Bit 13 (A2): 100BASE-TX (半双工)
Bit 12 (A3): 100BASE-TX (全双工)
Bit 11 (A4): 100BASE-T4
Bit 10 (A5): Pause (对称流控)
Bit  9 (A6): Asymmetric Pause (非对称流控)
Bit  8 (A7): 保留 (1000BASE-T 用)
```

**其他位**：
- `Bit 7`: RF (Remote Fault)
- `Bit 6`: ACK (确认，收到 3 次一致 FLP 后置 1)
- `Bit 5`: NP (Next Page 存在)

**实际 Base Page 示例**（100BASE-TX 全双工 + Pause）：
```
二进制: 0000 0001 0001 1100
十六进制: 0x011C

位分解:
S4-S0 = 00001 = IEEE 802.3
A0    = 1 = 10BASE-T 半双工
A1    = 1 = 10BASE-T 全双工
A2    = 0 = 100BASE-TX 半双工关闭
A3    = 1 = 100BASE-TX 全双工开启
A4    = 1 = 100BASE-T4 (某些旧设备)
A5    = 0 = 不支持 Pause
RF=0, ACK=0, NP=0
```

### 7.3 Next Page (下一页面)

当 Base Page 中 NP=1 时，发送 Next Page。1000BASE-T 使用 Next Page 交换额外能力。

**1000BASE-T 的 Next Page** (格式不同)：
```
本文略，详细参考 IEEE 802.3ab Clause 40
```

### 7.4 并行检测 (Parallel Detection)

如果 Link Partner 不支持 Auto-Negotiation，PHY 通过检测 Link Pulses 来判断速度：
- **10BASE-T**: 检测到 Normal Link Pulses (NLP, 每 16ms)
- **100BASE-TX**: 检测到 100BASE-TX 的闲置信号 (IDLE, 每 8ns)

**并行检测优先级**：1000BASE-T > 100BASE-TX > 10BASE-T

### 7.5 自动协商状态机

```
          +-------------+
          |  Power On   |
          +-------------+
                |
                v
    +-----------------------+
    |  Ability Detect       |  <-- 发送 FLP Burst, 监听对方 FLP
    |   (FLP Burst 传输中)   |
    +-----------------------+
                |
                | (3 次一致 FLP Burst 接收完成)
                v
    +-----------------------+
    |  Acknowledge Detect   |  <-- 设置 ACK 位
    +-----------------------+
                |
                | (接收方也 ACK)
                v
    +-----------------------+
    |  Complete             |  <-- 协商完成，进入 HCD (最高公共分母)
    |  Acknowledge          |
    +-----------------------+
                |
                v
    +-----------------------+
    |  FLP Link Good Check  |  <-- 等待链路建立
    +-----------------------+
                |
                v
    +-----------------------+
    |  Link Up              |  <-- 建立链路，按协商结果配置
    +-----------------------+
```

### 7.6 代码：Auto-Negotiation 状态机

```c
#include <stdint.h>
#include <stdbool.h>

/* PHY 寄存器定义 (见第八节 MDIO) */
#define PHY_BMCR          0x00
#define PHY_BMSR          0x01
#define PHY_ANAR          0x04  /* Auto-Neg Advertisement */
#define PHY_ANLPAR        0x05  /* Link Partner Ability */
#define PHY_ANER          0x06  /* Auto-Neg Expansion */

/* BMCR 位 */
#define BMCR_RESET      0x8000
#define BMCR_AN_EN      0x1000  /* Auto-Neg 使能 */
#define BMCR_AN_RESTART 0x0200  /* 重新协商 */

/* BMSR 位 */
#define BMSR_AN_CAP     0x0008  /* Auto-Neg 能力 */
#define BMSR_LINK_STATUS 0x0004
#define BMSR_AN_COMP    0x0020  /* Auto-Neg 完成 */

/* ANAR / ANLPAR 位 */
#define AN_10BT_HD      0x0020  /* 10BASE-T 半双工 */
#define AN_10BT_FD      0x0040  /* 10BASE-T 全双工 */
#define AN_100TX_HD     0x0080  /* 100BASE-TX 半双工 */
#define AN_100TX_FD     0x0100  /* 100BASE-TX 全双工 */
#define AN_PAUSE        0x0400  /* Pause 流控 */
#define AN_NP           0x8000  /* Next Page */

/* 自动协商状态机状态 */
typedef enum {
    AN_STATE_DISABLED,
    AN_STATE_ABILITY_DETECT,
    AN_STATE_ACKNOWLEDGE_DETECT,
    AN_STATE_COMPLETE_ACK,
    AN_STATE_LINK_GOOD_CHECK,
    AN_STATE_LINK_UP,
    AN_STATE_PARALLEL_DETECTION,
    AN_STATE_AN_GOOD
} an_state_t;

typedef struct {
    an_state_t state;
    uint16_t   anar;      /* 本端能力 */
    uint16_t   anlpar;    /* 远端能力 */
    uint16_t   advertise; /* 公告的能力 */
    uint8_t    flp_count; /* FLP Burst 计数 */
    uint16_t   speed;     /* 协商结果: 10/100/1000 */
    uint8_t    duplex;    /* 0=half, 1=full */
    bool       pause;     /* 流控使能 */
} an_context_t;

/* 读 PHY 寄存器 (MDIO 接口) */
extern uint16_t phy_read_reg(uint8_t phy_addr, uint8_t reg_addr);
extern void     phy_write_reg(uint8_t phy_addr, uint8_t reg_addr, uint16_t val);

/* 初始化 Auto-Neg */
void an_init(an_context_t *ctx, uint16_t advertise) {
    ctx->state   = AN_STATE_DISABLED;
    ctx->anar    = advertise;
    ctx->flp_count = 0;

    ctx->advertise = advertise;
    ctx->speed     = 0;
    ctx->duplex    = 0;
    ctx->pause     = false;

    /* 写入 ANAR 寄存器 */
    phy_write_reg(0, PHY_ANAR, advertise | AN_NP); /* 示例 */
}

/* 优先级解析: 从 ANLPAR 提取最佳公共能力 */
static uint16_t an_resolve_hcd(uint16_t local, uint16_t partner) {
    uint16_t common = local & partner;

    /* 按优先级从高到低选择 */
    if (common & AN_100TX_FD) return AN_100TX_FD;
    if (common & AN_100TX_HD) return AN_100TX_HD;
    if (common & AN_10BT_FD)  return AN_10BT_FD;
    if (common & AN_10BT_HD)  return AN_10BT_HD;

    return 0; /* 无公共能力 */
}

/* Auto-Neg 状态机主函数 (周期性调用) */
void an_state_machine_tick(an_context_t *ctx, bool link_up) {
    uint16_t bmsr = phy_read_reg(0, PHY_BMSR);
    uint16_t lpar = phy_read_reg(0, PHY_ANLPAR);

    switch (ctx->state) {
    case AN_STATE_DISABLED:
        /* 使能 Auto-Neg */
        phy_write_reg(0, PHY_BMCR, BMCR_AN_EN | BMCR_AN_RESTART);
        ctx->state = AN_STATE_ABILITY_DETECT;
        break;

    case AN_STATE_ABILITY_DETECT:
        /* 检查是否已建立 FLP 交换 (BMSR.AN_COMP) */
        if (bmsr & BMSR_AN_COMP) {
            ctx->anlpar = lpar;
            ctx->state = AN_STATE_ACKNOWLEDGE_DETECT;
        }
        /* 超时处理: 如果 FLP 超时, 进入并行检测 */
        break;

    case AN_STATE_ACKNOWLEDGE_DETECT:
        /* 验证 ACK 位 */
        if (bmsr & BMSR_AN_COMP) {
            ctx->state = AN_STATE_COMPLETE_ACK;
        }
        break;

    case AN_STATE_COMPLETE_ACK:
    case AN_STATE_LINK_GOOD_CHECK:
        /* 解析协商结果 */
        {
            uint16_t hcd = an_resolve_hcd(ctx->anar, ctx->anlpar);
            switch (hcd) {
            case AN_100TX_FD: ctx->speed = 100; ctx->duplex = 1; break;
            case AN_100TX_HD: ctx->speed = 100; ctx->duplex = 0; break;
            case AN_10BT_FD:  ctx->speed = 10;  ctx->duplex = 1; break;
            case AN_10BT_HD:  ctx->speed = 10;  ctx->duplex = 0; break;
            default:
                /* 无法协商, 重新开始 */
                ctx->state = AN_STATE_DISABLED;
                return;
            }
            ctx->state = AN_STATE_LINK_UP;
        }

        if (bmsr & BMSR_LINK_STATUS) {
            ctx->state = AN_STATE_LINK_UP;
        }
        break;

    case AN_STATE_LINK_UP:
        if (!(bmsr & BMSR_LINK_STATUS)) {
            /* 链路断开, 重新协商 */
            ctx->state = AN_STATE_DISABLED;
        }
        break;

    default:
        break;
    }
}
```

---

## 八、MDIO Clause 22 / 45 深度解析

### 8.1 MDIO 帧结构 (Clause 22)

MDIO (Management Data Input/Output) 是 IEEE 802.3 Clause 22 定义的 PHY 管理接口。

**Clause 22 帧格式**（64 个时钟周期）：

```
  +-------+---------+------+-------+-------+-------+--------+--------+
  | PRE   | ST      | OP   | PHYAD | REGAD | TA    | DATA   | IDLE   |
  | 32个1 | 01 (Cl22)| 10(R) | 5 bit | 5 bit | Z0(R) | 16 bit | Z      |
  |       |         |    01(W) |       |       | 10(W) |        |        |
  +-------+---------+------+-------+-------+-------+--------+--------+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| PRE | 32 bit | 32 个 '1'，用于同步 |
| ST | 2 bit | `01` = Clause 22, `00` = Clause 45 |
| OP | 2 bit | `10` = Read, `01` = Write |
| PHYAD | 5 bit | PHY 地址 (0-31) |
| REGAD | 5 bit | 寄存器地址 (0-31, Clause 22 约束) |
| TA | 2 bit | 转向时间: 读=Z0, 写=10 |
| DATA | 16 bit | 读/写数据 |
| IDLE | 可变 | 高阻态 Z |

**读操作时序**：
```
MDC:  _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-...
MDIO: 1111...1111 01 10 AAAAARRRRR Z0 DDDDDDDDDDDDDDDD ZZZZ...
      ^-- PRE(32)--^  ^^ ^^^^^^^^^^ ^^ ^^^^^^^^^^^^^^^^ ^^^^^
                       ST OP PHY/REG TA DATA  (16位, PHY驱动)
                              |    |
                           主机驱动   PHY驱动
```

**写操作时序**：
```
MDC:  _-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-_-...
MDIO: 1111...1111 01 01 AAAAARRRRR 10 DDDDDDDDDDDDDDDD ZZ...
      ^-- PRE(32)--^  ^^ ^^^^^^^^^^ ^^ ^^^^^^^^^^^^^^^^ ^^^^
                       ST  OP  PHY/REG TA   DATA (全部由主机驱动)
```

### 8.2 Clause 45 帧格式

Clause 45 支持更大的设备地址空间（DEVAD 5 bit）和 32 位寄存器。

**Clause 45 帧格式**：

```
+-------+---------+------+-------+-------+--------+---------+
| PRE   | ST      | OP   | DEVAD | PRTAD | TA     | DATA    |
| 32个1 | 00 (Cl45)| 见下  | 5 bit | 5 bit | 2 bit  | 16 bit  |
+-------+---------+------+-------+-------+--------+---------+
```

**Clause 45 操作码**：

| OP | 操作 | 说明 |
|----|------|------|
| 00 | Address | 设置内部地址 (后续 Read/Write 使用) |
| 01 | Write | 写入寄存器数据 |
| 10 | Read inc | 读取并自动递增地址 |
| 11 | Read | 读取寄存器数据 |

**Clause 45 读操作 (两阶段)**：
```
Phase 1 (Address): PRE(32) 00 00 DEVAD PRTAD 10 ADDR_DATA(16)
Phase 2 (Read):    PRE(32) 00 11 DEVAD PRTAD Z0 READ_DATA(16) ...
```

### 8.3 MDC 时序要求 (Clause 22 / 45)

| 参数 | 最小值 | 最大值 | 单位 |
|------|--------|--------|------|
| MDC 频率 (Clause 22) | - | 2.5 | MHz |
| MDC 频率 (Clause 45) | - | 40 | MHz |
| MDC 高电平时间 | 160 | - | ns |
| MDC 低电平时间 | 160 | - | ns |
| MDIO 数据建立时间 (写) | 10 | - | ns |
| MDIO 数据保持时间 (写) | 10 | - | ns |
| MDIO 采样时间 (读, 从 MDC 上升沿算) | 0 | 300 | ns |

### 8.4 代码：MDIO 位操作实现

```c
#include <stdint.h>
#include <stdbool.h>

/* 硬件抽象: 需要实现下面两个函数 */
extern void mdio_set_mdc(int level);  /* 0 或 1 */
extern void mdio_set_mdio(int level); /* 0, 1, 或高阻 */
extern int  mdio_get_mdio(void);      /* 读 MDIO 引脚电平 */

/* 微秒延时 */
extern void delay_us(uint32_t us);

/* MDC 半周期延迟 (400ns = 1.25MHz, 适用于 Clause 22) */
#define MDC_HALF_PERIOD_US 0  /* 如果使用 nanosecond 精度, 用 400ns */
/* 实际中通常用 GPIO 翻转 + nop 实现 */

/* MDIO 引脚模式 */
typedef enum {
    MDIO_OUTPUT,
    MDIO_INPUT
} mdio_dir_t;

extern void mdio_set_dir(mdio_dir_t dir);

/* 本地函数声明 */
static void mdio_cycle(void);
static void mdio_write_bits(uint32_t value, int bit_count);
static uint16_t mdio_read_bits(int bit_count);

/**
 * MDIO 时钟周期
 */
static void mdio_cycle(void) {
    mdio_set_mdc(1);
    delay_us(1);  /* 或短延时 */
    mdio_set_mdc(0);
    delay_us(1);
}

/**
 * 向 MDIO 写入比特
 * value:    要写入的数据
 * bit_count: 位数 (MSB first)
 */
static void mdio_write_bits(uint32_t value, int bit_count) {
    mdio_set_dir(MDIO_OUTPUT);

    for (int i = bit_count - 1; i >= 0; i--) {
        mdio_set_mdio((value >> i) & 1);
        mdio_cycle();  /* MDC 上升沿采样 */
    }
}

/**
 * 从 MDIO 读取比特 (MSB first)
 */
static uint16_t mdio_read_bits(int bit_count) {
    uint16_t value = 0;

    mdio_set_dir(MDIO_INPUT);

    for (int i = bit_count - 1; i >= 0; i--) {
        mdio_set_mdc(1);
        delay_us(1);

        value |= (mdio_get_mdio() << i);

        mdio_set_mdc(0);
        delay_us(1);
    }

    return value;
}

/* ============================================================ */
/* Clause 22 API                                                */
/* ============================================================ */

/**
 * Clause 22 读操作
 * phy_addr: 0-31
 * reg_addr: 0-31
 * 返回: 16 位寄存器值
 */
uint16_t mdio_cl22_read(uint8_t phy_addr, uint8_t reg_addr) {
    uint16_t data;

    /* 1. PRE: 32 个 1 */
    mdio_write_bits(0xFFFFFFFF, 32);

    /* 2. ST = 01 (Clause 22) */
    mdio_write_bits(0x01, 2);

    /* 3. OP = 10 (Read) */
    mdio_write_bits(0x02, 2);

    /* 4. PHYAD = 5 bit */
    mdio_write_bits(phy_addr & 0x1F, 5);

    /* 5. REGAD = 5 bit */
    mdio_write_bits(reg_addr & 0x1F, 5);

    /* 6. TA: 主机释放总线 (Z), PHY 拉低第 2 个 MDC 周期 */
    mdio_set_dir(MDIO_INPUT);  /* 高阻 Z */
    mdio_cycle();              /* 第一个 TA 周期 */
    mdio_set_mdc(1);           /* 第二个 TA 周期 */
    delay_us(1);
    /* PHY 会在上升沿后不久拉低 MDIO */
    mdio_set_mdc(0);
    delay_us(1);

    /* 7. DATA: PHY 驱动, 16 bit */
    data = mdio_read_bits(16);

    /* 8. IDLE: 高阻 */
    mdio_set_dir(MDIO_INPUT);

    return data;
}

/**
 * Clause 22 写操作
 */
void mdio_cl22_write(uint8_t phy_addr, uint8_t reg_addr, uint16_t data) {
    /* 1. PRE: 32 个 1 */
    mdio_write_bits(0xFFFFFFFF, 32);

    /* 2. ST = 01 (Clause 22) */
    mdio_write_bits(0x01, 2);

    /* 3. OP = 01 (Write) */
    mdio_write_bits(0x01, 2);

    /* 4. PHYAD = 5 bit */
    mdio_write_bits(phy_addr & 0x1F, 5);

    /* 5. REGAD = 5 bit */
    mdio_write_bits(reg_addr & 0x1F, 5);

    /* 6. TA = 10 (主机驱动) */
    mdio_write_bits(0x02, 2);

    /* 7. DATA = 16 bit */
    mdio_write_bits(data, 16);

    /* 8. IDLE: 高阻 */
    mdio_set_dir(MDIO_INPUT);
}

/* ============================================================ */
/* Clause 45 API                                                */
/* ============================================================ */

/* Clause 45 OP 码 */
#define CL45_OP_ADDR 0x00  /* 设置内部地址 */
#define CL45_OP_WRITE 0x01 /* 写入 */
#define CL45_OP_READ_INC 0x02 /* 读取并递增 */
#define CL45_OP_READ 0x03  /* 读取 */

/**
 * Clause 45 读操作 (两阶段)
 *
 * Phase 1: Address (设置内部寄存器地址)
 * Phase 2: Read (读取数据)
 */
uint16_t mdio_cl45_read(uint8_t devad, uint8_t port_addr,
                        uint16_t reg_addr) {
    /* Phase 1: Address 操作 */
    {
        /* PRE: 32 个 1 */
        mdio_write_bits(0xFFFFFFFF, 32);

        /* ST = 00 (Clause 45) */
        mdio_write_bits(0x00, 2);

        /* OP = 00 (Address) */
        mdio_write_bits(CL45_OP_ADDR, 2);

        /* DEVAD = 5 bit (设备类型, 如 1=PMD/PMD, 2=WIS, 3=PCS, ...) */
        mdio_write_bits(devad & 0x1F, 5);

        /* PRTAD = 5 bit (端口地址, 类似 PHYAD) */
        mdio_write_bits(port_addr & 0x1F, 5);

        /* TA = 10 (主机驱动) */
        mdio_write_bits(0x02, 2);

        /* DATA = 16 bit (寄存器地址) */
        mdio_write_bits(reg_addr, 16);

        /* IDLE */
        mdio_set_dir(MDIO_INPUT);
    }

    /* 帧间间隙 (Inter-Frame Gap, 至少 1 MDC 周期) */
    delay_us(2);

    /* Phase 2: Read 操作 */
    uint16_t value;
    {
        /* PRE: 32 个 1 */
        mdio_write_bits(0xFFFFFFFF, 32);

        /* ST = 00 (Clause 45) */
        mdio_write_bits(0x00, 2);

        /* OP = 11 (Read) */
        mdio_write_bits(CL45_OP_READ, 2);

        /* DEVAD = 5 bit */
        mdio_write_bits(devad & 0x1F, 5);

        /* PRTAD = 5 bit */
        mdio_write_bits(port_addr & 0x1F, 5);

        /* TA = Z0 (PHY 驱动) */
        mdio_set_dir(MDIO_INPUT);
        mdio_cycle();              /* Z */
        mdio_set_mdc(1);
        delay_us(1);
        mdio_set_mdc(0);           /* 0 (PHY 拉低) */
        delay_us(1);

        /* DATA = 16 bit (PHY 驱动) */
        value = mdio_read_bits(16);

        /* IDLE */
        mdio_set_dir(MDIO_INPUT);
    }

    return value;
}

/**
 * Clause 45 写操作
 */
void mdio_cl45_write(uint8_t devad, uint8_t port_addr,
                     uint16_t reg_addr, uint16_t data) {
    /* Phase 1: Address (同上) */
    {
        mdio_write_bits(0xFFFFFFFF, 32);
        mdio_write_bits(0x00, 2);        /* ST = 00 */
        mdio_write_bits(CL45_OP_ADDR, 2);
        mdio_write_bits(devad & 0x1F, 5);
        mdio_write_bits(port_addr & 0x1F, 5);
        mdio_write_bits(0x02, 2);        /* TA = 10 */
        mdio_write_bits(reg_addr, 16);
        mdio_set_dir(MDIO_INPUT);
    }

    delay_us(2);

    /* Phase 2: Write 操作 */
    {
        mdio_write_bits(0xFFFFFFFF, 32);
        mdio_write_bits(0x00, 2);        /* ST = 00 */
        mdio_write_bits(CL45_OP_WRITE, 2);
        mdio_write_bits(devad & 0x1F, 5);
        mdio_write_bits(port_addr & 0x1F, 5);
        mdio_write_bits(0x02, 2);        /* TA = 10 */
        mdio_write_bits(data, 16);
        mdio_set_dir(MDIO_INPUT);
    }
}

/* ============================================================ */
/* 使用示例: 读/写 PHY 基本寄存器                               */
/* ============================================================ */

/* IEEE 802.3 标准 PHY 寄存器 (Clause 22) */
#define PHY_REG_BMCR   0x00  /* 基本控制 */
#define PHY_REG_BMSR   0x01  /* 基本状态 */
#define PHY_REG_PHYID1 0x02  /* PHY ID 高 16 位 */
#define PHY_REG_PHYID2 0x03  /* PHY ID 低 16 位 */
#define PHY_REG_ANAR   0x04  /* Auto-Neg 公告 */
#define PHY_REG_ANLPAR 0x05  /* 链路伙伴能力 */
#define PHY_REG_ANER   0x06  /* Auto-Neg 扩展 */

/* BMCR 位 */
#define BMCR_RESET       0x8000
#define BMCR_LOOPBACK    0x4000
#define BMCR_SPEED_1000  0x0040  /* 1000Mbps 选择位 */
#define BMCR_AN_EN       0x1000  /* Auto-Neg 使能 */
#define BMCR_AN_RESTART  0x0200
#define BMCR_DUPLEX      0x0100  /* 1=全双工 */
#define BMCR_SPEED_MSB   0x2000  /* 速度: 00=10M, 01=100M, 10=1000M */

/* BMSR 位 */
#define BMSR_100T4      0x8000
#define BMSR_100TX_FDX  0x4000
#define BMSR_100TX_HDX  0x2000
#define BMSR_10T_FDX    0x1000
#define BMSR_10T_HDX    0x0800
#define BMSR_AN_COMP    0x0020  /* Auto-Neg 完成 */
#define BMSR_LINK_STATUS 0x0004 /* 链路 UP */
#define BMSR_AN_CAP     0x0008  /* 支持 Auto-Neg */

/* 初始化时检测 PHY */
uint16_t phy_detect(uint8_t phy_addr) {
    uint16_t id1 = mdio_cl22_read(phy_addr, PHY_REG_PHYID1);
    uint16_t id2 = mdio_cl22_read(phy_addr, PHY_REG_PHYID2);
    uint32_t oui = ((uint32_t)id1 << 6) | (id2 >> 10);

    /* 常用 PHY OUI:
     * LAN8720: 0x0007C1 (SMSC)
     * DP83848: 0x2000A0 (TI / National)
     * KSZ8081: 0x002216 (Microchip)
     * RTL8201: 0x000820 (Realtek)
     */
    (void)oui;
    return (id1 != 0 && id1 != 0xFFFF) ? 1 : 0;
}

/* PHY 复位 + 启动 Auto-Neg */
void phy_init(uint8_t phy_addr, uint16_t advertise) {
    /* 1. 软复位 */
    mdio_cl22_write(phy_addr, PHY_REG_BMCR, BMCR_RESET);

    /* 等待复位完成 */
    for (int i = 0; i < 1000; i++) {
        delay_us(10);
        uint16_t bmcr = mdio_cl22_read(phy_addr, PHY_REG_BMCR);
        if (!(bmcr & BMCR_RESET)) break;
    }

    /* 2. 设置 Auto-Neg 能力 */
    mdio_cl22_write(phy_addr, PHY_REG_ANAR, advertise);

    /* 3. 启动 Auto-Neg */
    mdio_cl22_write(phy_addr, PHY_REG_BMCR,
                    BMCR_AN_EN | BMCR_AN_RESTART);

    /* 4. 等待 Auto-Neg 完成 */
    for (int i = 0; i < 5000; i++) {
        delay_us(1000); /* 1ms */
        uint16_t bmsr = mdio_cl22_read(phy_addr, PHY_REG_BMSR);
        if (bmsr & BMSR_AN_COMP) break;
    }

    /* 5. 读取协商结果 */
    uint16_t lpar = mdio_cl22_read(phy_addr, PHY_REG_ANLPAR);
    (void)lpar;
}
```

---

## 附录：以太网帧类型速查

| 名称 | EtherType | 目的 MAC | 用途 |
|------|-----------|----------|------|
| IPv4 | `0x0800` | 单播/广播 | IP 数据报 |
| ARP | `0x0806` | `FF-FF-FF-FF-FF-FF` | 地址解析 |
| VLAN (802.1Q) | `0x8100` | 原始帧 | VLAN 标记 |
| IPv6 | `0x86DD` | 单播/多播 | IPv6 数据报 |
| MAC Control | `0x8808` | `01-80-C2-00-00-01` | Pause 帧 |
| LLDP | `0x88CC` | `01-80-C2-00-00-0E` | 链路发现 |
| STP BPDU | `0x42` (LLC) | `01-80-C2-00-00-00` | 生成树 |
| Q-in-Q (S-Tag) | `0x88A8` | 原始帧 | 运营商 VLAN |
| EAPoL | `0x888E` | `01-80-C2-00-00-03` | 802.1X 认证 |
| Precision Time | `0x88F7` | `01-1B-19-00-00-00` | PTP (1588) |

## 附录：TCP/UDP 常用端口速查

| 端口 | 协议 | 说明 |
|------|------|------|
| 20/21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | UDP/TCP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 123 | UDP | NTP |
| 161/162 | UDP | SNMP |
| 443 | TCP | HTTPS |
| 502 | TCP | Modbus TCP |
| 1900 | UDP | SSDP (UPnP) |
| 5353 | UDP | mDNS |
| 5683 | UDP | CoAP |

---

---

## 九、以太网 PHY 驱动模型与 IEEE 802.3 寄存器地图

> 从比特级寄存器操作到完整 PHY 驱动状态机，涵盖中断架构、Auto-MDIX、TDR 电缆诊断、WoL 唤醒、802.3az 节能以及 Linux PHY 子系统

---

### 9.1 IEEE 802.3 Clause 22 寄存器模型

IEEE 802.3 Clause 22 定义了 PHY 寄存器集的标准规范，地址空间为 0x00 - 0x1F 共 32 个 16 位寄存器。其中 0x00 - 0x0F 为强制/可选标准寄存器，0x10 - 0x1F 为厂商自定义。

#### 9.1.1 寄存器 0x00 -- BMCR (Basic Mode Control Register)

BMCR 是 PHY 的基本控制寄存器，复位值因厂商而异（LAN8720 复位值 0x3100，DP83848 复位值 0x3000）。

```
位域     位号    名称                    读写    说明
──────────────────────────────────────────────────────────────────────
Reset     15     Software Reset          R/W   写 1 触发软件复位，硬件自清除
Loopback  14     Loopback Mode           R/W   写 1 使能回环（MAC→PHY→MAC 内部环回测试）
Speed_LSB 13     Speed Select LSB        R/W   与位 6 组合选择速率
AutoNeg   12     Auto-Negotiation Enable R/W   写 1 使能自动协商（优先级高于手动配置）
PowerDown 11     Power Down              R/W   写 1 进入低功耗模式
Isolate   10     Isolate                 R/W   写 1 隔离 MAC 接口（MII 输出高阻）
RestartAN 9      Restart Auto-Neg        R/W   写 1 触发重新协商，自清除
Duplex    8      Duplex Mode             R/W   1=全双工, 0=半双工
ColTest   7      Collision Test          R/W   写 1 使能碰撞测试（仅半双工）
Speed_MSB 6      Speed Select MSB        R/W   与位 13 组合选择速率
[5:0]     5:0    Reserved                 R    保留/厂商自定义
```

**速率选择编码**（位 6 和位 13 组合）：

| 位 6 (MSB) | 位 13 (LSB) | 速率    |
|-----------|------------|--------|
| 0         | 0          | 10 Mbps |
| 0         | 1          | 100 Mbps |
| 1         | 0          | 1000 Mbps |
| 1         | 1          | 保留    |

**典型 BMCR 配置值**：

```c
/* BMCR 位定义 */
#define BMCR_RESET          (1U << 15)  /* 0x8000 */
#define BMCR_LOOPBACK       (1U << 14)  /* 0x4000 */
#define BMCR_SPEED_LSB      (1U << 13)  /* 0x2000 */
#define BMCR_AUTONEG        (1U << 12)  /* 0x1000 */
#define BMCR_POWERDOWN      (1U << 11)  /* 0x0800 */
#define BMCR_ISOLATE        (1U << 10)  /* 0x0400 */
#define BMCR_RESTART_ANEG   (1U << 9)   /* 0x0200 */
#define BMCR_DUPLEX_MODE    (1U << 8)   /* 0x0100 */
#define BMCR_COL_TEST       (1U << 7)   /* 0x0080 */
#define BMCR_SPEED_MSB      (1U << 6)   /* 0x0040 */

/* 常用配置组合宏 */
#define BMCR_RESET_VAL          (BMCR_RESET)
#define BMCR_AUTONEG_ENABLE     (BMCR_AUTONEG | BMCR_SPEED_LSB)
#define BMCR_SPEED_10_HD        (0x0000)
#define BMCR_SPEED_10_FD        (BMCR_DUPLEX_MODE)
#define BMCR_SPEED_100_HD       (BMCR_SPEED_LSB)
#define BMCR_SPEED_100_FD       (BMCR_SPEED_LSB | BMCR_DUPLEX_MODE)
#define BMCR_SPEED_1000_HD      (BMCR_SPEED_MSB)
#define BMCR_SPEED_1000_FD      (BMCR_SPEED_MSB | BMCR_DUPLEX_MODE)
```

**复位时序的注意事项**：BMCR 的 Reset 位是自清除的，但不同 PHY 的自清除时间差异很大。LAN8720 通常在 50us 内完成，DP83848 需要约 100us。正确做法是轮询 BMCR 直到 bit 15 归零，而非简单地延时等待：

```c
uint8_t phy_soft_reset(uint8_t phy_addr) {
    phy_write_reg(phy_addr, 0x00, BMCR_RESET);

    uint32_t timeout = 10000;  /* 最大等待 10ms */
    while (timeout--) {
        if (!(phy_read_reg(phy_addr, 0x00) & BMCR_RESET)) {
            return 1;  /* 复位完成 */
        }
        delay_us(1);
    }
    return 0;  /* 复位超时 */
}
```

#### 9.1.2 寄存器 0x01 -- BMSR (Basic Mode Status Register)

BMSR 是只读状态寄存器，其中位 2（Link Status）具有**延迟锁存（Latching Low）**特性：

```
位域     位号    名称                    读写    说明
──────────────────────────────────────────────────────────────────────
T4        15     100BASE-T4 Capable      R     支持 100BASE-T4
TX_FD     14     100BASE-TX FD Capable  R     支持 100BASE-TX 全双工
TX_HD     13     100BASE-TX HD Capable  R     支持 100BASE-TX 半双工
10_FD     12     10BASE-T FD Capable    R     支持 10BASE-T 全双工
10_HD     11     10BASE-T HD Capable   R     支持 10BASE-T 半双工
[10:9]    10:9   100BASE-T2 Capable      R     100BASE-T2 能力（保留）
ExtStatus 8      Extended Status         R     是否有扩展状态寄存器（0x0F）
[7]       7      Reserved                R     保留
MFPS      6      MF Preamble Suppress   R     是否支持省略 PREAMBLE
AN_Comp   5      Auto-Neg Complete       R     自动协商已完成
RF        4      Remote Fault            R     远端故障
AN_Cap    3      Auto-Neg Capable        R     支持自动协商
Link      2      Link Status             RC    链路状态（延迟锁存，读后清除锁存）
Jabber    1      Jabber Detect           R     Jabber 检测（超长帧发送）
ExtCap    0      Extended Capability     R     0=仅有基本寄存器, 1=有 0x00-0x0F
```

**关于 Link Status 延迟锁存（Latching Low）的深入解释**：

当物理链路断开时，PHY 硬件将 BMSR 的位 2 清零并锁存。这意味着**即使链路已经恢复，位 2 仍然保持 0**，直到软件读取该寄存器一次，锁存才会被解除。这就是为什么读取链路状态时，推荐的做法是连续读两次，第一次用于清除锁存，第二次获得真实值：

```c
uint8_t phy_read_link_status(uint8_t phy_addr) {
    /* 第一次读取：清除延迟锁存 */
    (void)phy_read_reg(phy_addr, 0x01);

    /* 短暂延时，让 PHY 更新真实状态 */
    delay_us(50);

    /* 第二次读取：获得真实的链路状态 */
    uint16_t bmsr = phy_read_reg(phy_addr, 0x01);
    return (bmsr & 0x0004) ? 1 : 0;
}
```

**BMSR 能力的典型解读**：

```c
typedef struct {
    uint8_t  t4_support   : 1;  /* 100BASE-T4 */
    uint8_t  tx_fd        : 1;  /* 100BASE-TX Full Duplex */
    uint8_t  tx_hd        : 1;  /* 100BASE-TX Half Duplex */
    uint8_t  t_fd         : 1;  /* 10BASE-T Full Duplex */
    uint8_t  t_hd         : 1;  /* 10BASE-T Half Duplex */
    uint8_t  aneg_capable : 1;  /* 支持自动协商 */
    uint8_t  ext_status   : 1;  /* 有扩展状态 */
} phy_capabilities_t;

void phy_read_capabilities(uint8_t phy_addr, phy_capabilities_t *cap) {
    uint16_t bmsr = phy_read_reg(phy_addr, 0x01);
    cap->t4_support  = (bmsr >> 15) & 1;
    cap->tx_fd       = (bmsr >> 14) & 1;
    cap->tx_hd       = (bmsr >> 13) & 1;
    cap->t_fd        = (bmsr >> 12) & 1;
    cap->t_hd        = (bmsr >> 11) & 1;
    cap->aneg_capable = (bmsr >> 3) & 1;
    cap->ext_status  = (bmsr >> 8) & 1;
}
```

LAN8720 的 BMSR 典型读回值为 0x786D（二进制 0111 1000 0110 1101），表示支持 100BASE-TX 全/半双工、10BASE-T 全/半双工、支持自动协商且有扩展能力。DP83848 的典型读回值为 0x782D。

#### 9.1.3 寄存器 0x02/0x03 -- PHYIDR1 / PHYIDR2 (PHY Identifier)

这两个寄存器共同编码了一个 32 位的 PHY 标识符，遵循 IEEE 标准的 OUI（Organizationally Unique Identifier）编码方式：

```
PHYIDR1 (0x02):
  Bits[15:0] = OUI[21:6]    -- OUI 的第 21 到第 6 位（共 16 位）

PHYIDR2 (0x03):
  Bits[15:10] = OUI[5:0]    -- OUI 的第 5 到第 0 位（共 6 位）
  Bits[9:4]   = Model Number  -- 厂商分配的型号编号（6 位）
  Bits[3:0]   = Revision Number -- 芯片版本号（4 位）
```

OUI 是 24 位的 IEEE 分配的厂商唯一标识符。完整的 32 位 PHY ID 可以这样汇编：

```c
uint32_t phy_read_id(uint8_t phy_addr) {
    uint16_t id1 = phy_read_reg(phy_addr, 0x02);
    uint16_t id2 = phy_read_reg(phy_addr, 0x03);
    return ((uint32_t)id1 << 16) | id2;
}
```

**常见 PHY 的 OUI 识别**：

```c
typedef enum {
    PHY_MODEL_UNKNOWN  = 0,
    PHY_MODEL_LAN8720  = 0x0007C0F1,  /* SMSC/Microchip */
    PHY_MODEL_DP83848  = 0x20005C90,  /* TI / National  */
    PHY_MODEL_KSZ8081  = 0x00221560,  /* Microchip       */
    PHY_MODEL_KSZ9031  = 0x00221622,  /* Microchip (GigE) */
    PHY_MODEL_RTL8201  = 0x001CC800,  /* Realtek         */
    PHY_MODEL_YT8512   = 0x000001A2,  /* Motorcomm       */
} phy_model_t;

phy_model_t phy_detect_model(uint8_t phy_addr) {
    uint32_t id = phy_read_id(phy_addr);

    switch (id) {
    case PHY_MODEL_LAN8720: return PHY_MODEL_LAN8720;
    case PHY_MODEL_DP83848: return PHY_MODEL_DP83848;
    case PHY_MODEL_KSZ8081: return PHY_MODEL_KSZ8081;
    case PHY_MODEL_KSZ9031: return PHY_MODEL_KSZ9031;
    case PHY_MODEL_RTL8201: return PHY_MODEL_RTL8201;
    default:                return PHY_MODEL_UNKNOWN;
    }
}

const char *phy_model_string(phy_model_t model) {
    switch (model) {
    case PHY_MODEL_LAN8720: return "LAN8720 (SMSC)";
    case PHY_MODEL_DP83848: return "DP83848 (TI)";
    case PHY_MODEL_KSZ8081: return "KSZ8081 (Microchip)";
    case PHY_MODEL_KSZ9031: return "KSZ9031 (Microchip GigE)";
    case PHY_MODEL_RTL8201: return "RTL8201 (Realtek)";
    default:                return "Unknown PHY";
    }
}
```

OUI 提取的细节：假设 OUI 为 24 位值 `0x080028`（TI 的 OUI），其编码方式为：
- OUI[23] 是 MSB，对应 PHYIDR1 的 bit 15
- OUI[22:7] 填充 PHYIDR1 的 bits[15:0]
- OUI[6:1] 填充 PHYIDR2 的 bits[15:10]
- OUI[0] 不使用

所以 PHYIDR1 = (OUI >> 6) & 0xFFFF，PHYIDR2 = ((OUI & 0x3F) << 10) | (model << 4) | revision。

#### 9.1.4 寄存器 0x04 -- ANAR (Auto-Negotiation Advertisement Register)

ANAR 用于通告本端 PHY 支持的能力。自动协商过程中，双方通过 FLP Burst 交换各自 ANAR 的内容。

```
位域     位号    名称                    读写    说明
──────────────────────────────────────────────────────────────────────
NP        15     Next Page               R/W   写 1 表示有后续页面需要交换
ACK       14     Acknowledge             R     硬件自动设置（收到 3 次一致 FLP 后）
RF        13     Remote Fault            R/W   写入 1 通告远端发生故障
[12]      12     Technology Ability      R/W   保留（某些 PHY 用于扩展）
PAUSE_ASM 11     Asymmetric Pause        R/W   非对称流控能力
PAUSE_SYM 10     Symmetric Pause         R/W   对称流控能力（IEEE 802.3x）
T4        9      100BASE-T4              R/W   支持 100BASE-T4
TX_FD     8      100BASE-TX FD          R/W   支持 100BASE-TX 全双工
TX_HD     7      100BASE-TX HD          R/W   支持 100BASE-TX 半双工
10_FD     6      10BASE-T FD            R/W   支持 10BASE-T 全双工
10_HD     5      10BASE-T HD            R/W   支持 10BASE-T 半双工
Selector  4:0    Protocol Selector       R/W   00001 = IEEE 802.3
```

**Pause 流控位的编码含义**（位 11 和位 10 组合）：

| 位 11 | 位 10 | 含义                              |
|-------|-------|-----------------------------------|
| 0     | 0     | 不支持流控                        |
| 0     | 1     | 对称 Pause（双方都能发/收 Pause 帧）|
| 1     | 0     | 非对称 Pause（本端能收但不能发）  |
| 1     | 1     | 对称 + 非对称（两者都支持）       |

**配置 ANAR 的推荐值**：

```c
/* 通告所有能力：100M FD/HD + 10M FD/HD + Pause */
#define ANAR_DEFAULT  (0x0001     |  /* Selector = IEEE 802.3 */    \
                       (1 << 5)   |  /* 10BASE-T HD */               \
                       (1 << 6)   |  /* 10BASE-T FD */               \
                       (1 << 7)   |  /* 100BASE-TX HD */             \
                       (1 << 8)   |  /* 100BASE-TX FD */             \
                       (1 << 10))    /* Symmetric Pause */

void phy_set_advertisement(uint8_t phy_addr, uint16_t anar) {
    phy_write_reg(phy_addr, 0x04, anar);
}
```

#### 9.1.5 寄存器 0x05 -- ANLPAR (Auto-Negotiation Link Partner Ability)

ANLPAR 反映链路对端 PHY 通告的能力。该寄存器为**只读**，由自动协商硬件自动填写，其位定义与 ANAR 完全一致。通过 ANLPAR 可以知道对端支持哪些速率和双工模式，然后通过优先级裁决算法确定最终的"最高公共能力（HCD, Highest Common Denominator）"。

```c
typedef struct {
    uint8_t speed;    /* 0=10M, 1=100M, 2=1000M */
    uint8_t duplex;   /* 0=half, 1=full */
    uint8_t pause;    /* 0=none, 1=symmetric, 2=asymmetric */
} aneg_result_t;

#define AN_10_HD  (1 << 5)
#define AN_10_FD  (1 << 6)
#define AN_100_HD (1 << 7)
#define AN_100_FD (1 << 8)
#define AN_T4     (1 << 9)
#define AN_PAUSE  (1 << 10)
#define AN_ASM_DIR (1 << 11)

aneg_result_t phy_resolve_aneg(uint16_t local_advert, uint16_t partner_advert) {
    aneg_result_t result = {0, 0, 0};
    uint16_t common = local_advert & partner_advert;

    /* 优先级裁决：按 802.3 定义的优先级从高到低选择 */
    if (common & AN_100_FD) {
        result.speed  = 100;
        result.duplex = 1;
    } else if (common & AN_100_HD) {
        result.speed  = 100;
        result.duplex = 0;
    } else if (common & AN_10_FD) {
        result.speed  = 10;
        result.duplex = 1;
    } else if (common & AN_10_HD) {
        result.speed  = 10;
        result.duplex = 0;
    }

    /* Pause 流控裁决（802.3 Table 28B-2） */
    uint8_t loc_pause  = (local_advert >> 10) & 0x03;
    uint8_t rem_pause  = (partner_advert >> 10) & 0x03;

    if ((loc_pause & 0x01) && (rem_pause & 0x01)) {
        result.pause = 1; /* 对称流控 */
    } else if ((loc_pause & 0x02) && (rem_pause & 0x01)) {
        result.pause = 2; /* 本端非对称 */
    }

    return result;
}
```

#### 9.1.6 寄存器 0x06 -- ANER (Auto-Negotiation Expansion Register)

ANER 提供自动协商过程的额外状态信息：

```
位域     位号    名称                            说明
────────────────────────────────────────────────────────────
LP_AN    0    Link Partner AN Capable        对端支持自动协商
LP_NP    1    Link Partner Next Page Able    对端支持下一页面
Local_NP 2    Local Next Page Able           本端支持下一页面
PD_Fault 3    Parallel Detection Fault       并行检测失败
LP_ACK   4    Link Partner Code Word Ack     对端已确认能力码字
[15:5]   15:5 Reserved                       保留
```

#### 9.1.7 寄存器标准布局总图

```
PHY 寄存器地图（IEEE 802.3 Clause 22）：
地址   名称          类型        说明
────────────────────────────────────────────
0x00   BMCR          控制      基本模式控制寄存器
0x01   BMSR          状态      基本模式状态寄存器
0x02   PHYIDR1       标识      PHY ID 高 16 位
0x03   PHYIDR2       标识      PHY ID 低 16 位
0x04   ANAR          控制      自动协商通告寄存器
0x05   ANLPAR        状态      链路对端能力寄存器
0x06   ANER          状态      自动协商扩展寄存器
0x07   ANNPTR        控制      Auto-Neg Next Page 发送
0x08   ANLPNPAR      状态      Auto-Neg Next Page 接收
0x09-0x0E            保留      IEEE 保留
0x0F   100BT_ESR     状态      100BASE-T 扩展状态（千兆相关）
0x10-0x1F            厂商      各 PHY 厂商自定义
```

---

### 9.2 PHY 驱动状态机

完整的 PHY 驱动需要实现一个状态机来管理 PHY 的生命周期。以下是一个通用 PHY 状态机设计，涵盖从上电到正常运行的完整路径：

#### 9.2.1 状态定义

```c
typedef enum {
    PHY_STATE_POWER_ON_RESET,   /* 上电或硬件复位 */
    PHY_STATE_HW_RESET_WAIT,    /* 等待硬件复位完成（PLL 锁定） */
    PHY_STATE_SW_RESET,         /* 软件复位进行中 */
    PHY_STATE_PHY_DETECT,       /* 检测 PHY ID 和型号 */
    PHY_STATE_CONFIG,           /* 配置基本参数 */
    PHY_STATE_START_ANEG,       /* 启动自动协商 */
    PHY_STATE_WAIT_ANEG,        /* 等待自动协商完成 */
    PHY_STATE_ANEG_COMPLETE,    /* 自动协商完成，读取结果 */
    PHY_STATE_LINK_UP,          /* 链路已建立，正常收发 */
    PHY_STATE_LINK_DOWN,        /* 链路断开 */
    PHY_STATE_ENERGY_SAVE,      /* 节能模式 */
    PHY_STATE_ERROR,            /* 错误状态 */
} phy_state_t;
```

#### 9.2.2 状态上下文数据结构

```c
typedef struct {
    phy_state_t  state;            /* 当前状态 */
    phy_model_t  model;            /* PHY 型号 */
    uint8_t      phy_addr;         /* PHY 地址 */
    uint32_t     phy_id;           /* 完整 32 位 PHY ID */

    /* 链路参数 */
    uint8_t      speed;            /* 0=10M, 1=100M, 2=1000M */
    uint8_t      duplex;           /* 0=half, 1=full */
    uint8_t      pause_enabled;    /* 流控使能 */
    uint8_t      link_up;          /* 链路当前状态 */

    /* 定时器与计数 */
    uint32_t     state_timer_ms;   /* 当前状态的停留时间 (ms) */
    uint32_t     aneg_timeout_ms;  /* 自动协商超时计数器 */
    uint8_t      retry_count;      /* 重试次数 */

    /* 中断与事件 */
    uint8_t      event_pending;    /* 待处理事件标志 */
    uint8_t      int_status;       /* 中断状态寄存器快照 */

    /* 厂商特定操作函数表 */
    struct {
        uint16_t (*read_reg)(uint8_t addr, uint8_t reg);
        void     (*write_reg)(uint8_t addr, uint8_t reg, uint16_t val);
        void     (*config_speed)(uint8_t speed, uint8_t duplex);
    } ops;

} phy_device_t;
```

#### 9.2.3 状态机实现

```c
void phy_state_machine_tick(phy_device_t *dev) {
    switch (dev->state) {

    case PHY_STATE_POWER_ON_RESET:
        /* 执行硬件复位：拉低 RST 引脚保持 20ms 后释放 */
        phy_hw_reset_assert();
        dev->state_timer_ms = 0;
        dev->state = PHY_STATE_HW_RESET_WAIT;
        break;

    case PHY_STATE_HW_RESET_WAIT:
        dev->state_timer_ms += PHY_TICK_MS;
        if (dev->state_timer_ms >= 50) {
            /* 50ms 后释放复位，等待 PLL 锁定 */
            phy_hw_reset_deassert();
            dev->state_timer_ms = 0;
            dev->state = PHY_STATE_SW_RESET;
        }
        break;

    case PHY_STATE_SW_RESET:
        /* 触发软件复位 */
        dev->ops.write_reg(dev->phy_addr, 0x00, BMCR_RESET);
        dev->state_timer_ms = 0;
        dev->state = PHY_STATE_PHY_DETECT;
        break;

    case PHY_STATE_PHY_DETECT:
        /* 检查软件复位完成 + 读取 PHY ID */
        {
            uint16_t bmcr = dev->ops.read_reg(dev->phy_addr, 0x00);
            if (bmcr & BMCR_RESET) {
                /* 仍在复位中，继续等待 */
                dev->state_timer_ms += PHY_TICK_MS;
                if (dev->state_timer_ms >= 1000) {
                    dev->state = PHY_STATE_ERROR;
                }
                break;
            }
            /* 复位完成，读取 PHY ID */
            dev->phy_id = phy_read_id(dev->phy_addr);
            dev->model = phy_detect_model(dev->phy_addr);
            if (dev->model == PHY_MODEL_UNKNOWN) {
                dev->state = PHY_STATE_ERROR;
                break;
            }
            dev->state = PHY_STATE_CONFIG;
        }
        break;

    case PHY_STATE_CONFIG:
        /* 配置 ANAR 通告能力 */
        dev->ops.write_reg(dev->phy_addr, 0x04, ANAR_DEFAULT);
        dev->state = PHY_STATE_START_ANEG;
        break;

    case PHY_STATE_START_ANEG:
        /* 使能自动协商 + 触发重新协商 */
        dev->ops.write_reg(dev->phy_addr, 0x00,
                            BMCR_AUTONEG | BMCR_RESTART_ANEG);
        dev->aneg_timeout_ms = 0;
        dev->state = PHY_STATE_WAIT_ANEG;
        break;

    case PHY_STATE_WAIT_ANEG:
        {
            dev->aneg_timeout_ms += PHY_TICK_MS;
            uint16_t bmsr = dev->ops.read_reg(dev->phy_addr, 0x01);

            /* BMSR 位 5 = Auto-Neg Complete */
            if (bmsr & (1U << 5)) {
                dev->state = PHY_STATE_ANEG_COMPLETE;
                break;
            }

            /* 超时处理：尝试强制模式 */
            if (dev->aneg_timeout_ms >= 4000) {
                dev->retry_count++;
                if (dev->retry_count < 3) {
                    /* 重试自动协商 */
                    dev->state = PHY_STATE_START_ANEG;
                } else {
                    /* 超过重试次数，使用并行检测结果或强制模式 */
                    dev->state = PHY_STATE_LINK_DOWN;
                }
            }
        }
        break;

    case PHY_STATE_ANEG_COMPLETE:
        {
            /* 从 ANLPAR 解析协商结果 */
            uint16_t local  = dev->ops.read_reg(dev->phy_addr, 0x04);
            uint16_t partner = dev->ops.read_reg(dev->phy_addr, 0x05);
            aneg_result_t result = phy_resolve_aneg(local, partner);

            dev->speed  = result.speed;
            dev->duplex = result.duplex;
            dev->pause_enabled = result.pause;

            /* 通知 MAC 层更新速率和双工 */
            dev->ops.config_speed(dev->speed, dev->duplex);

            /* 检查链路是否真正建立 */
            uint8_t link = phy_read_link_status(dev->phy_addr);
            if (link) {
                dev->link_up = 1;
                dev->state = PHY_STATE_LINK_UP;
            } else {
                dev->state = PHY_STATE_LINK_DOWN;
            }
        }
        break;

    case PHY_STATE_LINK_UP:
        /* 正常运行态，监控链路状态 */
        {
            uint8_t link = phy_read_link_status(dev->phy_addr);
            if (!link) {
                dev->link_up = 0;
                dev->state = PHY_STATE_LINK_DOWN;
            }
        }
        break;

    case PHY_STATE_LINK_DOWN:
        /* 链路断开，尝试恢复 */
        dev->retry_count++;
        if (dev->retry_count > 5) {
            dev->state = PHY_STATE_ENERGY_SAVE;
        } else {
            /* 重新启动自动协商 */
            dev->state = PHY_STATE_START_ANEG;
        }
        break;

    case PHY_STATE_ENERGY_SAVE:
        /* 进入节能模式，周期性唤醒检测链路 */
        {
            uint16_t bmcr = dev->ops.read_reg(dev->phy_addr, 0x00);
            dev->ops.write_reg(dev->phy_addr, 0x00,
                                bmcr | BMCR_POWERDOWN);
            /* 每 5 秒唤醒一次检测链路 */
            dev->state_timer_ms += PHY_TICK_MS;
            if (dev->state_timer_ms >= 5000) {
                /* 唤醒 */
                uint16_t bc = dev->ops.read_reg(dev->phy_addr, 0x00);
                dev->ops.write_reg(dev->phy_addr, 0x00,
                                    bc & ~BMCR_POWERDOWN);
                delay_ms(100);
                if (phy_read_link_status(dev->phy_addr)) {
                    dev->state = PHY_STATE_START_ANEG;
                }
                dev->state_timer_ms = 0;
            }
        }
        break;

    case PHY_STATE_ERROR:
        /* 错误处理：尝试完全复位 */
        {
            static uint32_t error_timer = 0;
            error_timer += PHY_TICK_MS;
            if (error_timer >= 10000) {
                /* 10 秒后尝试重新初始化 */
                error_timer = 0;
                dev->retry_count = 0;
                dev->state = PHY_STATE_POWER_ON_RESET;
            }
        }
        break;
    }
}
```

#### 9.2.4 状态转换图

```
                               +-----------------+
                               | POWER_ON_RESET  |
                               +--------+--------+
                                        |
                                  硬件复位 (RST=0, 保持 20ms)
                                        |
                                        v
                               +--------+--------+
                               | HW_RESET_WAIT   |  <-- 等待 PLL 锁定 (50ms)
                               +--------+--------+
                                        |
                                  软件复位 (BMCR.15=1)
                                        |
                                        v
                               +--------+--------+
                               | SW_RESET        |  <-- 等待自清除
                               +--------+--------+
                                        |
                                  读 PHYIDR1/2 验证
                                        |
                                        v
                               +--------+--------+
                               | PHY_DETECT      |
                               +--------+--------+
                                        |
                                  写入 ANAR + BMCR 启动 ANEG
                                        |
                                        v
                               +--------+--------+
                     +---------> START_ANEG      |
                     |         +--------+--------+
                     |                  |
                     |           等待 ANEG 完成
                     |                  |
                     |         +--------v--------+
                     |         | WAIT_ANEG       |  <-- 超时 4s 后重试或强制
                     |         +--------+--------+
                     |                  |
                     |           ANEG 完成 (BMSR.5=1)
                     |                  |
                     |         +--------v--------+
                     |         | ANEG_COMPLETE   |  <-- 读取 ANLPAR + 裁决 HCD
                     |         +--------+--------+
                     |                  |
                     |         +--------+--------+
                     |         | 配置 MAC 速率/双工
                     |         +--------+--------+
                     |                  |
                     |           /----+----\
                     |          /          \
                     |     链路 UP     链路 DOWN
                     |         |            |
                     |         v            v
                     |  +------+--+   +-----+------+
                     |  | LINK_UP  |   | LINK_DOWN  |
                     |  +---------+   +-----+------+
                     |                          |
                     |                  重试 > 5 次?
                     |                 /          \
                     |              是            否 (回到 START_ANEG)
                     |               |
                     |               v
                     |     +--------+--------+
                     |     | ENERGY_SAVE     |  <-- 关闭协商，周期唤醒
                     |     +--------+--------+
                     |               |
                     |         每 5 秒唤醒检测
                     |         链路恢复?
                     +---------( 是则重试 )
```

---

### 9.3 PHY 中断架构

PHY 在链路状态变化时可以通过中断引脚通知 MCU，避免轮询带来的 CPU 开销。不同 PHY 的中断寄存器布局不同，但原理相通。

#### 9.3.1 LAN8720 中断系统

LAN8720 的中断控制分布在两个寄存器中：

**寄存器 0x1B -- INT_MASK (Interrupt Mask)**：

```
位域     位号    说明
────────────────────────────────────────
15       Reserved
14       EnergyOn           检测到能量
13       Reserved
12       AutoNegComplete    自动协商完成
11       FarEndFault        远端故障
10       LinkDown           链路断开
9        Reserved
8        Line Error Count   RX 线路错误
7-0      Reserved
```

**寄存器 0x1C -- INT_SRC (Interrupt Source, 读即清零)**：

位定义与 INT_MASK 完全一致。读取该寄存器时，所有已挂起的中断源会返回，同时硬件自动清除挂起状态。

```c
/* LAN8720 中断寄存器地址 */
#define LAN8720_INT_MASK    0x1B
#define LAN8720_INT_SRC     0x1C

/* 中断源位定义 */
#define LAN8720_INT_ENERGY      (1U << 14)
#define LAN8720_INT_ANEG_DONE   (1U << 12)
#define LAN8720_INT_FAR_END_FA  (1U << 11)
#define LAN8720_INT_LINK_DOWN   (1U << 10)
#define LAN8720_INT_LINE_ERR    (1U << 8)

void lan8720_enable_interrupts(uint8_t phy_addr) {
    /* 使能链路状态变化 + 自动协商完成中断 */
    uint16_t mask = LAN8720_INT_LINK_DOWN |
                    LAN8720_INT_ANEG_DONE |
                    LAN8720_INT_ENERGY;
    phy_write_reg(phy_addr, LAN8720_INT_MASK, mask);

    /* 清除初始挂起状态 */
    (void)phy_read_reg(phy_addr, LAN8720_INT_SRC);
}

uint16_t lan8720_get_interrupt_source(uint8_t phy_addr) {
    return phy_read_reg(phy_addr, LAN8720_INT_SRC);
}
```

#### 9.3.2 DP83848 中断系统

DP83848 的中断系统更为复杂，有两个独立的屏蔽寄存器：

**寄存器 0x11 -- MISR1 (Miscellaneous Interrupt Status Register 1)**：

```
位域     位号    说明
────────────────────────────────────────
15-9     Reserved
8        ANEG_COMP          自动协商完成
7        ANEG_ACK           自动协商确认
6        FALSE_CARRIER      假载波检测
5        PARALLEL_FAULT     并行检测失败
4        POLARITY_CHANGED   极性反转检测
3        ENERGY_DETECT      能量检测
2        LINK_CHANGE        链路状态变化
1        REMOTE_FAULT       远端故障
0        JABBER             Jabber 检测
```

**寄存器 0x12 -- MISR2 (Miscellaneous Interrupt Status Register 2)**：

```
位域     位号    说明
────────────────────────────────────────
15-12    Reserved
11       RX_ER_COUNTER_HIT  RX_ER 计数溢出
10       FEF_CNT_HIT        FEF 计数溢出
9        SNR_FAULT          信噪比错误
8        SQE                信号质量错误
7-0      Reserved
```

DP83848 的使能方式是通过写入同一个寄存器（写 1 使能对应位的中断）：

```c
void dp83848_enable_interrupts(uint8_t phy_addr) {
    /* MISR1: 使能链路变化 + ANEG 完成 + 能量检测 */
    phy_write_reg(phy_addr, 0x11,
                  (1U << 2) |   /* LINK_CHANGE */
                  (1U << 8) |   /* ANEG_COMP   */
                  (1U << 3));   /* ENERGY_DETECT */

    /* 清除挂起的中断（读 MISR1 即清除） */
    (void)phy_read_reg(phy_addr, 0x11);
}
```

#### 9.3.3 完整中断服务程序（ISR + 延迟处理）

在 MCU 中，ISR 应尽可能短，将耗时操作推迟到主循环或任务中处理：

```c
/* 前向声明 */
extern void phy_deferred_handler(uint16_t int_src, uint8_t phy_addr);

/* 全局事件标志 */
volatile uint8_t g_phy_int_pending = 0;
volatile uint16_t g_phy_int_src = 0;

/* ── PHY 中断服务程序 (ISR) ── */
/* EXTI 中断入口 (假设 PHY_INT 连接到 PA8) */
void EXTI9_5_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_8) != RESET) {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_8);

        /* 读取中断源 (因 PHY 型号而异，这里以 LAN8720 为例) */
        uint16_t int_src = phy_read_reg(PHY_ADDR, LAN8720_INT_SRC);

        /* 保存中断信息，设置标志供主循环处理 */
        g_phy_int_src = int_src;
        g_phy_int_pending = 1;

        /* 在 RTOS 环境中通知任务：
         * BaseType_t xHigherPriorityTaskWoken = pdFALSE;
         * xSemaphoreGiveFromISR(xPhyEventSem, &xHigherPriorityTaskWoken);
         * portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
         */
    }
}

/* ── 延迟处理函数（在主循环或 RTOS 任务中调用） ── */
void phy_deferred_handler(uint16_t int_src, uint8_t phy_addr) {
    if (int_src & LAN8720_INT_LINK_DOWN) {
        /* 链路断开：停止网络收发 */
        printf("[PHY] Link DOWN\r\n");
        eth_stop_transmission();
        phy_device.link_up = 0;
        phy_device.state = PHY_STATE_LINK_DOWN;
    }

    if (int_src & LAN8720_INT_ANEG_DONE) {
        /* 自动协商完成：读取协商结果 */
        uint16_t local  = phy_read_reg(phy_addr, 0x04);
        uint16_t partner = phy_read_reg(phy_addr, 0x05);

        aneg_result_t result = phy_resolve_aneg(local, partner);
        phy_device.speed  = result.speed;
        phy_device.duplex = result.duplex;

        printf("[PHY] ANEG Done: %dM %s Duplex\r\n",
               result.speed, result.duplex ? "Full" : "Half");

        /* 检查链路状态 */
        if (phy_read_link_status(phy_addr)) {
            phy_device.link_up = 1;
            phy_device.state = PHY_STATE_LINK_UP;
            eth_start_transmission();
        }
    }

    if (int_src & LAN8720_INT_ENERGY) {
        /* 检测到能量（可能有设备接入） */
        printf("[PHY] Energy detected\r\n");
    }
}

/* ── 在主循环中处理挂起的中断 ── */
void phy_int_task(void) {
    if (g_phy_int_pending) {
        g_phy_int_pending = 0;
        phy_deferred_handler(g_phy_int_src, PHY_ADDR);
    }
}
```

#### 9.3.4 中断驱动 vs 轮询驱动对比

| 维度         | 中断驱动                          | 轮询驱动                        |
|-------------|-----------------------------------|--------------------------------|
| CPU 开销     | 仅在事件发生时执行                | 周期性占用 CPU 时间             |
| 响应延迟     | 几微秒（中断响应）                | 取决于轮询间隔（通常 10-500ms） |
| 代码复杂度   | 需要中断配置 + ISR + 延迟处理     | 简单的循环读取                  |
| 适用场景     | 低功耗、对延迟敏感                | 简单裸机系统，链路不频繁变化    |
| 链路切换感知 | 即时                               | 延迟一个轮询周期                |

---

### 9.4 Auto-MDIX (自动交叉检测)

Auto-MDIX 是 IEEE 802.3ab 定义的一项技术（2000 年以后的标准 PHY 基本都支持），用于自动检测并适配直通线（Straight-through）和交叉线（Crossover）。

#### 9.4.1 MDI 与 MDI-X 的定义

在传统以太网中：
- **MDI (Medium Dependent Interface)**：终端设备（PC、MCU）的接口。引脚定义：1=TX+, 2=TX-, 3=RX+, 6=RX-
- **MDI-X (MDI Crossover)**：交换机的接口。引脚定义：1=RX+, 2=RX-, 3=TX+, 6=TX-（TX 和 RX 互换）

直通线连接 MDI 到 MDI-X，交叉线连接 MDI 到 MDI。当两台 MDI 设备直连时，必须使用交叉线。

#### 9.4.2 Auto-MDIX 工作原理

支持 Auto-MDIX 的 PHY 在自动协商过程中，通过 FLP Burst 的特定编码来检测对端的传输对配置，然后在内部自动切换发送/接收对的映射关系：

```
  设备 A (Auto-MDIX 使能)          设备 B (Auto-MDIX 使能)
   ┌──────────────┐               ┌──────────────┐
   │ PHY          │  TX+/TX- ──── │  RX+/RX-     │
   │ (自动检测为   │  RX+/RX- ──── │  TX+/TX-     │
   │  MDI 角色)   │               │ (自动检测为   │
   └──────────────┘               │  MDI-X 角色) │
                                   └──────────────┘
```

Auto-MDIX 的检测流程：
1. PHY 发送 FLP Burst 的同时监听链路上的脉冲
2. 如果在一对未配置为发送的引脚上检测到 FLP，说明对端在对这些引脚发送 → 需要切换
3. 通过内部交换矩阵，将 TX 和 RX 对映射到正确的引脚

#### 9.4.3 Auto-MDIX 配置

大部分 PHY 通过厂商自定义寄存器使能/配置 Auto-MDIX：

```c
/* LAN8720 Auto-MDIX 配置 (通过 0x1F 寄存器) */
/* LAN8720 默认自动使能 Auto-MDIX，无需额外配置 */

/* DP83848 Auto-MDIX 配置 (通过 0x10 PHYSTS 寄存器的位 9) */
#define DP83848_PHYSTS       0x10
#define DP83848_PHYSTS_MDIX   (1U << 9)  /* 1=MDI-X mode, 0=MDI mode */

void dp83848_auto_mdix_enable(uint8_t phy_addr) {
    /* DP83848 Auto-MDIX 通过寄存器 0x17 (PHYCR) 控制 */
    uint16_t phycr = phy_read_reg(phy_addr, 0x17);
    phycr |= (1U << 6);   /* MDIX 模式使能 */
    phycr |= (1U << 5);   /* 自动 MDIX 使能 */
    phy_write_reg(phy_addr, 0x17, phycr);
}

uint8_t dp83848_get_mdix_status(uint8_t phy_addr) {
    uint16_t sts = phy_read_reg(phy_addr, DP83848_PHYSTS);
    return (sts & DP83848_PHYSTS_MDIX) ? 1 : 0;
}

/* KSZ8081 Auto-MDIX 配置 (通过 0x1F 寄存器) */
void ksz8081_auto_mdix_enable(uint8_t phy_addr) {
    uint16_t reg = phy_read_reg(phy_addr, 0x1F);
    reg |= (1U << 8);    /* Auto-MDIX 使能 */
    reg &= ~(1U << 7);   /* 自动模式，非手动 */
    phy_write_reg(phy_addr, 0x1F, reg);
}
```

---

### 9.5 Cable Diagnostics (TDR 电缆诊断)

电缆诊断功能基于时域反射计（TDR, Time Domain Reflectometry）原理，用于检测网线的断路、短路位置和长度。

#### 9.5.1 TDR 原理

TDR 的基本原理是：PHY 向网线发送一个已知的脉冲信号，然后测量反射信号返回的时间。由于信号在铜缆中的传播速度（VOP, Velocity of Propagation）是已知的（约为光速的 0.6-0.8 倍），可以精确计算故障点的距离：

```
                    发送脉冲
  PHY ──────────────────────────→ 网线 ──────→ 远端
       ←──────────────────────────
                    反射脉冲

  距离 = (反射时间 × VOP) / 2
```

故障类型判断：
- **短路**：反射脉冲极性相反（负反射），幅值大
- **断路**：反射脉冲极性相同（正反射），幅值大
- **阻抗不匹配**：反射脉冲较小，可能有多次反射

#### 9.5.2 电缆诊断的 C 实现

DP83848 内置电缆诊断功能，通过寄存器 0x13-0x16 访问：

```c
/* DP83848 电缆诊断寄存器 */
#define DP83848_BISCR     0x13   /* BIST Control Register */
#define DP83848_BISR      0x14   /* BIST Status Register  */
#define DP83848_LTSSR     0x16   /* Link Test Status & Control */

/* BISCR 位定义 */
#define BISCR_CABLE_DIAG_EN   (1U << 15)  /* 电缆诊断使能 */
#define BISCR_START_CABLE_DIAG (1U << 14)  /* 启动诊断 */
#define BISCR_CABLE_FAULT_CODE(x) ((x >> 10) & 0x0F)  /* 故障码 */
#define BISCR_CABLE_FAULT_DIST(x) (x & 0x03FF)          /* 故障距离 */

typedef enum {
    CABLE_OK               = 0,   /* 电缆正常 */
    CABLE_OPEN_CIRCUIT     = 1,   /* 断路 */
    CABLE_SHORT_CIRCUIT    = 2,   /* 短路 */
    CABLE_IMPEDANCE_MISMATCH = 3,  /* 阻抗不匹配 */
} cable_fault_type_t;

typedef struct {
    uint8_t  fault_type;    /* 故障类型 */
    uint16_t distance_m;    /* 故障距离（米） */
    uint8_t  valid;         /* 诊断是否有效 */
} cable_diag_result_t;

cable_diag_result_t dp83848_cable_diag(uint8_t phy_addr) {
    cable_diag_result_t result = {0, 0, 0};

    /* 启动电缆诊断 */
    phy_write_reg(phy_addr, DP83848_BISCR,
                  BISCR_CABLE_DIAG_EN | BISCR_START_CABLE_DIAG);

    /* 等待诊断完成（最长 100ms） */
    uint32_t timeout = 1000;
    while (timeout--) {
        uint16_t status = phy_read_reg(phy_addr, DP83848_BISCR);
        if (!(status & BISCR_START_CABLE_DIAG)) {
            /* 诊断完成 */
            uint16_t bscr = phy_read_reg(phy_addr, DP83848_BISCR);
            uint8_t  fault_code = BISCR_CABLE_FAULT_CODE(bscr);
            uint16_t raw_dist   = BISCR_CABLE_FAULT_DIST(bscr);

            /* 故障距离 = raw_dist × 0.4 米（DP83848 精度） */
            result.distance_m = (raw_dist * 4) / 10;
            result.valid = 1;

            switch (fault_code) {
            case 0:  result.fault_type = CABLE_OK; break;
            case 1:  result.fault_type = CABLE_OPEN_CIRCUIT; break;
            case 2:  result.fault_type = CABLE_SHORT_CIRCUIT; break;
            default: result.fault_type = CABLE_IMPEDANCE_MISMATCH; break;
            }
            break;
        }
        delay_us(100);
    }
    return result;
}

/* 输出诊断结果 */
void cable_diag_report(cable_diag_result_t *r) {
    static const char *fault_str[] = {
        "OK", "OPEN", "SHORT", "IMPEDANCE MISMATCH"
    };
    printf("Cable Diagnostic: %s", fault_str[r->fault_type]);
    if (r->fault_type != CABLE_OK) {
        printf(" at ~%dm", r->distance_m);
    }
    printf("\r\n");
}
```

LAN8720 不支持完整的 TDR 电缆诊断（入门级 PHY 通常没有此功能），而 DP83848 和 KSZ9031 等工业/千兆级 PHY 内置该功能。

---

### 9.6 Wake-on-LAN (WoL 网络唤醒)

Wake-on-LAN 允许处于低功耗状态的设备通过接收特殊格式的"魔术包（Magic Packet）"来唤醒。

#### 9.6.1 魔术包结构

魔术包是一个以太网帧，其 payload 包含以下特殊序列：

```
Ethernet 帧头:
  Dst MAC: FF:FF:FF:FF:FF:FF (广播 MAC)
  Src MAC: 任意 MAC
  EtherType: 0x0842 (WoL 专用，旧标准) 或 0x0800 (IP 广播)

Payload:
  [0xFFFFFFFFFFFF]    -- 6 字节全 0xFF (同步序列)
  [目标 MAC 地址 × 16] -- 目标设备的 MAC 地址重复 16 次 (共 96 字节)
  [可选: 密码 4/6 字节] -- 某些实现支持安全密码
```

魔术包的 payload 中，MAC 地址重复 16 次是识别魔术包的关键特征。

#### 9.6.2 WoL 的 PHY 配置

不同的 PHY 有不同的 WoL 配置方式。大部分现代 PHY 支持将 WoL 过滤卸载到 PHY 层，使得 MCU 可以在深度睡眠时由 PHY 检测魔术包并唤醒 MCU：

```c
/* LAN8720 WoL 配置 */
/* LAN8720 不支持硬件 WoL 过滤，需要 MAC 或 MCU 软件实现 */
/* 需要在 MCU 睡眠时保持 MAC 接收并使能特定中断 */

/* DP83848 WoL 配置 */
#define DP83848_WOL_CFG     0x1C   /* WoL 配置寄存器 */
#define DP83848_WOL_STS     0x1D   /* WoL 状态寄存器 */

/* WOL_CFG 位定义 */
#define WOL_CFG_ENABLE      (1U << 0)   /* WoL 使能 */
#define WOL_CFG_MAGIC_PKT   (1U << 1)   /* 魔术包检测 */
#define WOL_CFG_PATTERN     (1U << 2)   /* 模式匹配（自定义过滤） */
#define WOL_CFG_BCAST       (1U << 4)   /* 广播检测 */
#define WOL_CFG_LINK_EVENT  (1U << 5)   /* 链路事件唤醒 */

void dp83848_wol_enable(uint8_t phy_addr, const uint8_t *mac) {
    /* 1. 将 MAC 地址写入 WoL 寄存器 */
    uint16_t mac_low  = (mac[0] << 8) | mac[1];
    uint16_t mac_mid  = (mac[2] << 8) | mac[3];
    uint16_t mac_high = (mac[4] << 8) | mac[5];

    phy_write_reg(phy_addr, 0x18, mac_low);   /* MAC[1:0] */
    phy_write_reg(phy_addr, 0x19, mac_mid);   /* MAC[3:2] */
    phy_write_reg(phy_addr, 0x1A, mac_high);  /* MAC[5:4] */

    /* 2. 使能 WoL (魔术包检测) */
    phy_write_reg(phy_addr, DP83848_WOL_CFG,
                  WOL_CFG_ENABLE | WOL_CFG_MAGIC_PKT);

    /* 3. 清除状态 */
    (void)phy_read_reg(phy_addr, DP83848_WOL_STS);
}

uint8_t dp83848_wol_check(uint8_t phy_addr) {
    uint16_t sts = phy_read_reg(phy_addr, DP83848_WOL_STS);
    if (sts & (1U << 0)) {  /* Bit0 = Magic Packet Received */
        return 1;  /* WoL 魔术包已收到 */
    }
    return 0;
}
```

#### 9.6.3 软件 WoL 实现

对于不支持硬件 WoL 的 PHY（如 LAN8720），需要在 MCU 的 MAC 层或协议栈中软件实现魔术包检测：

```c
#define WOL_SYNC_SEQ_LEN    6
#define WOL_MAC_REPEAT      16
#define WOL_MAGIC_LEN       (WOL_SYNC_SEQ_LEN + (ETH_ALEN * WOL_MAC_REPEAT))

uint8_t wol_check_magic_packet(const uint8_t *frame, uint16_t len,
                                const uint8_t *target_mac) {
    /* 魔术包最小长度 */
    if (len < 14 + WOL_MAGIC_LEN) return 0;

    /* 跳过以太网头部 (14 字节) */
    const uint8_t *payload = frame + 14;
    uint16_t payload_len = len - 14;

    /* 扫描 payload 查找同步序列 */
    for (uint16_t off = 0; off < payload_len - WOL_MAGIC_LEN; off++) {
        /* 检查 6 字节的 0xFF 同步序列 */
        uint8_t found = 1;
        for (int i = 0; i < WOL_SYNC_SEQ_LEN; i++) {
            if (payload[off + i] != 0xFF) {
                found = 0;
                break;
            }
        }
        if (!found) continue;

        /* 检查 MAC 地址重复 16 次 */
        off += WOL_SYNC_SEQ_LEN;
        for (int rep = 0; rep < WOL_MAC_REPEAT; rep++) {
            for (int b = 0; b < ETH_ALEN; b++) {
                if (payload[off + rep * ETH_ALEN + b] != target_mac[b]) {
                    found = 0;
                    break;
                }
            }
            if (!found) break;
        }
        if (found) return 1;  /* 匹配！ */
    }
    return 0;
}

/* WoL 帧过滤 DMA 回调示例 */
uint8_t eth_dma_rx_callback_wol(const uint8_t *frame, uint16_t len) {
    static const uint8_t our_mac[6] = {0x0C, 0x29, 0x12, 0x34, 0x56, 0x78};

    if (wol_check_magic_packet(frame, len, our_mac)) {
        /* 收到唤醒包，退出低功耗模式 */
        system_wake_from_sleep();
        return 1;  /* 帧已消费 */
    }
    return 0;  /* 正常处理该帧 */
}
```

---

### 9.7 Energy Efficient Ethernet (802.3az EEE)

EEE（IEEE 802.3az-2010）是在网络空闲时降低 PHY 功耗的标准，目标是在链路利用率低的场景下减少 50% 以上的功耗。

#### 9.7.1 LPI (Low Power Idle) 模式

EEE 的核心机制是 LPI 模式，通过以下方式工作：

```
正常模式 (Active):
            ┌───────────┐     ┌───────────┐     ┌───────────┐
  数据      │  突发    │     │  突发    │     │  突发    │
            └───────────┘     └───────────┘     └───────────┘
            ↑───── 连续发送/接收 ────→

LPI 模式 (Eyes-Open, Sleep + Refresh):
  数据      ┌───┐ ┌─┐                  ┌─┐          ┌───┐
            │突发│ │S│   ── 空闲 ──   │R│   ──→   │突发│
            └───┘ └─┘                  └─┘          └───┘
                 ↑  ↑  ↑             ↑     ↑
              活动  Sleep            Refresh  Wake
              发送  (100us)          (周期)   (20us)
```

LPI 周期包含三个阶段：
1. **Sleep 阶段**：发送完成后，发送 Sleep 信号（约 100us），通知链路对端进入 LPI
2. **Refresh 阶段**：空闲期间，周期性地发送 Refresh 信号（维持时钟同步），间隔约 20ms
3. **Wake 阶段**：需要发送数据时，发送 Wake 信号（约 20us），恢复正常传输

#### 9.7.2 EEE 的 PHY 配置

大部分支持 EEE 的 PHY 通过标准寄存器 0x7C（EEE 控制/状态寄存器）来配置：

```c
/* IEEE 802.3az EEE 寄存器 (Clause 45, 设备类型 3=PCS, 寄存器 0x7C) */
/* 对于 Clause 22 的 PHY，EEE 配置通过厂商自定义寄存器 */

/* DP83848 EEE 配置 */
/* DP83848 不支持 EEE（较早的工业 PHY） */

/* KSZ9031 (千兆 PHY) EEE 配置 */
/* 寄存器 0x7C 用于 EEE 能力通告 */
#define KSZ9031_EEE_CAP      0x7C

#define EEE_CAP_100BASE_TX   (1U << 1)  /* 100BASE-TX 支持 EEE */
#define EEE_CAP_1000BASE_T   (1U << 2)  /* 1000BASE-T 支持 EEE */

void ksz9031_eee_enable(uint8_t phy_addr) {
    /* 读取 EEE 能力 */
    uint16_t eee = phy_read_reg(phy_addr, KSZ9031_EEE_CAP);

    /* 通告 EEE 能力 */
    eee |= EEE_CAP_100BASE_TX | EEE_CAP_1000BASE_T;
    phy_write_reg(phy_addr, KSZ9031_EEE_CAP, eee);

    /* 使能 EEE 的 LPI 模式 */
    /* 通过厂商特定寄存器 0x0E 控制 */
    uint16_t reg_0e = phy_read_reg(phy_addr, 0x0E);
    reg_0e |= (1U << 10);  /* LPI 使能 */
    phy_write_reg(phy_addr, 0x0E, reg_0e);
}

/* LPI 状态查询 */
uint8_t ksz9031_lpi_status(uint8_t phy_addr) {
    /* 寄存器 0x7D = EEE 状态 */
    uint16_t eee_sts = phy_read_reg(phy_addr, 0x7D);
    return (eee_sts & (1U << 0)) ? 1 : 0;  /* Bit0 = LPI 模式 */
}

/* EEE 省电统计 */
typedef struct {
    uint32_t active_time_ms;    /* 正常模式时间 */
    uint32_t lpi_time_ms;       /* LPI 模式时间 */
    uint32_t wake_count;        /* 唤醒次数 */
} eee_stats_t;

void eee_update_stats(eee_stats_t *stats, uint8_t phy_addr) {
    /* 通过厂商特定寄存器获取统计信息 */
    uint16_t lpi_counter = phy_read_reg(phy_addr, 0x7E);
    stats->lpi_time_ms += (lpi_counter * 100);  /* 每个计数 ≈ 100ms */
}
```

#### 9.7.3 EEE 的数据链路层配合

MAC 层也需要配合 EEE：

```c
/* 在 MAC 层控制 LPI 状态的使能和退出 */
/* 这通常由 MAC 硬件自动处理，软件只需配置使能 */

/* STM32 ETH MAC 的 LPI 配置 (如果 MAC 支持) */
void eth_mac_eee_config(void) {
    /* ETH_MACLPIACR (LPI 控制寄存器) */
    /* LPIEN: LPI 模式使能 */
    /* PLS: 完美 LPI 状态 */
    /* LPIACR 的典型配置： */

    /* LPI 定时器 */
    /* LPI 最小间隔 = 2000us (1000BASE-T) */
    /* LPI 唤醒定时器 = 20us */

    /* 对于 STM32F4/F7/H7 MAC，EEE 支持因型号而异 */
    /* 具体配置请参考对应参考手册的 MAC LPI 章节 */
}
```

---

### 9.8 Linux PHY 子系统

Linux 内核已经实现了完整的 PHY 驱动框架，位于 `drivers/net/phy/` 目录下。理解这个框架有助于在 MCU RTOS 中设计类似的 PHY 驱动抽象层。

#### 9.8.1 核心数据结构：phy_device

```c
/* Linux 内核 phy_device 结构体 (简化自 include/linux/phy.h) */

struct phy_device {
    /* PHY ID */
    uint32_t            phy_id;           /* 完整的 32 位 PHY 标识符 */
    struct phy_c45_device_ids *c45_ids;   /* Clause 45 设备 ID */

    /* 链路状态 */
    int                 link;             /* 当前链路状态: 0=down, 1=up */
    int                 autoneg;          /* 自动协商配置 */
    int                 speed;            /* SPEED_10, SPEED_100, SPEED_1000 */
    int                 duplex;           /* DUPLEX_HALF, DUPLEX_FULL */
    int                 pause;            /* 流控协商结果 */
    int                 asymmetric_pause; /* 非对称流控 */

    /* 状态机 */
    int                 state;            /* 当前 PHY 状态 */
    int                 dev_flags;        /* 设备特定标志 */
    phy_interface_t     interface;        /* MII/RMII/RGMII 等 */

    /* 地址 */
    uint32_t            addr;             /* MDIO 总线地址 (0-31) */

    /* 驱动 */
    struct phy_driver   *drv;             /* 指向 PHY 驱动的指针 */
    struct phy_device   *next;            /* 链表中的下一个设备 */

    /* 属性 */
    int                 irq;              /* 中断号 */
    uint32_t            supported[BITS_PER_LONG];  /* 支持的能力位图 */
    uint32_t            advertising[BITS_PER_LONG]; /* 通告的能力位图 */

    /* 自动协商 */
    unsigned int        autoneg_complete:1;  /* ANEG 完成标志 */

    /* 回调 */
    int (*adjust_link)(struct net_device *ndev);  /* 链路变化回调 */
};
```

#### 9.8.2 phy_driver 结构

```c
/* Linux 内核 phy_driver 结构体 (简化) */

struct phy_driver {
    uint32_t    phy_id;           /* PHY 标识符（匹配用） */
    uint32_t    phy_id_mask;      /* 掩码（例如 0xFFFFFFF0 忽略版本号） */
    const char *name;             /* 驱动名称 */

    /* 生命周期管理 */
    int (*probe)(struct phy_device *phydev);      /* 探测函数 */
    void (*remove)(struct phy_device *phydev);    /* 移除函数 */

    /* 配置函数 */
    int (*config_init)(struct phy_device *phydev);           /* 初始化 */
    int (*config_aneg)(struct phy_device *phydev);           /* 配置 ANEG */
    int (*config_intr)(struct phy_device *phydev);           /* 配置中断 */
    int (*did_interrupt)(struct phy_device *phydev);         /* 检查中断 */
    int (*ack_interrupt)(struct phy_device *phydev);         /* 确认中断 */

    /* 状态读取 */
    int (*read_status)(struct phy_device *phydev);           /* 读链路状态 */
    int (*config_speed_duplex)(struct phy_device *phydev);   /* 配置速率 */

    /* 电源管理 */
    int (*suspend)(struct phy_device *phydev);               /* 挂起 */
    int (*resume)(struct phy_device *phydev);                /* 恢复 */

    /* 厂商特定 */
    int (*soft_reset)(struct phy_device *phydev);            /* 软复位 */
    int (*link_change_notify)(struct phy_device *phydev);    /* 链路变化通知 */
};
```

#### 9.8.3 PHY 状态机 (phy_state_machine)

Linux 的 PHY 状态机通过 `phy_state_machine()` 工作队列实现，定期运行（由 `PHY_STATE_TIME = 1` 秒定时器触发）：

```c
/* Linux 内核 PHY 状态机 (drivers/net/phy/phy.c, 概念性展示) */
/* 简化示意，非逐行复制 */

enum phy_state {
    PHY_DOWN          = 0,  /* PHY 未初始化 */
    PHY_STARTING      = 1,  /* PHY 正在启动 */
    PHY_READY         = 2,  /* PHY 就绪但无连接 */
    PHY_PENDING       = 3,  /* PHY 待连接 */
    PHY_UP            = 4,  /* PHY 运行中且已连接 */
    PHY_AN            = 5,  /* 自动协商进行中 */
    PHY_RUNNING       = 6,  /* PHY 运行中 */
    PHY_NOLINK        = 7,  /* 链路断开 */
    PHY_FORCING       = 8,  /* 强制链路模式 */
    PHY_CHANGELINK    = 9,  /* 链路变化 */
    PHY_HALTED        = 10, /* PHY 停止 */
    PHY_RESUMING      = 11, /* PHY 正在恢复 */
};

/* 简化的状态机运行逻辑 */
void phy_state_machine(struct phy_device *phydev) {
    bool needs_aneg = false;

    switch (phydev->state) {
    case PHY_DOWN:
    case PHY_STARTING:
    case PHY_READY:
    case PHY_PENDING:
        /* 触发自动协商 */
        phy_start_aneg(phydev);
        phydev->state = PHY_AN;
        break;

    case PHY_UP:
        /* 不做任何事 */
        break;

    case PHY_AN:
        /* 检查 ANEG 完成状态 */
        err = phy_read_status(phydev);
        if (err || phydev->autoneg_complete) {
            phydev->state = PHY_RUNNING;
        }
        break;

    case PHY_NOLINK:
        /* 链路断开，尝试 ANEG */
        if (phydev->autoneg) {
            err = phy_read_status(phydev);
            if (!err && phydev->link) {
                phydev->state = PHY_RUNNING;
                phy_link_change(phydev, true);
            }
        }
        break;

    case PHY_RUNNING:
        /* 正常运行，监控链路 */
        if (phy_read_status(phydev)) {
            phydev->state = PHY_NOLINK;
            phy_link_change(phydev, false);
        }
        break;

    case PHY_HALTED:
        break;

    case PHY_CHANGELINK:
    case PHY_FORCING:
    case PHY_RESUMING:
        break;
    }
}
```

#### 9.8.4 phy_connect API 使用示例

```c
/* Linux 用户态或驱动中使用 PHY 子系统的典型流程 */

#include <linux/phy.h>
#include <linux/netdevice.h>

/* 网络设备驱动的 PHY 连接 */
static int eth_phy_connect(struct net_device *ndev, int phy_addr) {
    struct phy_device *phydev = NULL;
    int ret;

    /* 1. 获取 PHY 设备 */
    phydev = mdiobus_get_phy(ndev->mii_bus, phy_addr);
    if (!phydev) {
        pr_err("PHY at address %d not found\n", phy_addr);
        return -ENODEV;
    }

    /* 2. 连接到网络设备 */
    ret = phy_connect(ndev, dev_name(&phydev->mdio.dev),
                      eth_adjust_link,  /* 链路变化回调 */
                      PHY_INTERFACE_MODE_RMII);
    if (ret) {
        pr_err("phy_connect failed: %d\n", ret);
        return ret;
    }

    /* 3. 配置自动协商能力 */
    phydev->supported &= PHY_100BT_FEATURES | SUPPORTED_Pause;
    phydev->advertising = phydev->supported;

    /* 4. 配置自动协商模式 */
    phydev->autoneg = AUTONEG_ENABLE;

    /* 5. 启动 PHY */
    phy_start(phydev);

    pr_info("PHY connected: %s\n", phydev->drv->name);
    return 0;
}

/* 链路变化回调 */
static void eth_adjust_link(struct net_device *ndev) {
    struct phy_device *phydev = ndev->phydev;

    if (phydev->link) {
        /* 链路已建立 */
        pr_info("Link UP: %dMbps %s Duplex\n",
                phydev->speed,
                phydev->duplex == DUPLEX_FULL ? "Full" : "Half");

        /* 更新 MAC 寄存器 */
        eth_update_mac_speed(phydev->speed, phydev->duplex);
    } else {
        pr_info("Link DOWN\n");
    }
}

/* 断开 PHY */
static void eth_phy_disconnect(struct net_device *ndev) {
    phy_stop(ndev->phydev);    /* 停止状态机 */
    phy_disconnect(ndev->phydev); /* 断开连接 */
}
```

#### 9.8.5 现有 PHY 驱动列表

Linux 内核 `drivers/net/phy/` 中包含了几乎所有主流 PHY 的驱动：

| 驱动文件 | 支持的 PHY |
|---------|-----------|
| smsc.c | LAN8720, LAN8710, LAN8187 (SMSC/Microchip) |
| dp83848.c | DP83848 (TI) |
| micrel.c | KSZ8081, KSZ9031, KSZ9021 (Microchip) |
| realtek.c | RTL8201, RTL8211, RTL8366 (Realtek) |
| at803x.c | AR803x, QCA803x (Qualcomm/Atheros) |
| marvell.c | 88E1111, 88E151x (Marvell/Cavium) |
| broadcom.c | BCM5461, BCM5241 (Broadcom) |

#### 9.8.6 MCU 裸机 PHY 驱动抽象层设计建议

借鉴 Linux PHY 子系统的分层思想，MCU 裸机环境的 PHY 驱动可以设计为：

```c
/* 轻量级 PHY 驱动抽象层 (适用于 MCU RTOS) */

/* PHY 驱动结构体（类似 Linux 的 phy_driver） */
typedef struct {
    uint32_t    phy_id;                 /* 匹配的 PHY ID */
    uint32_t    phy_id_mask;            /* 掩码: 0xFFFFFFF0 */
    const char *name;                   /* 驱动名称 */

    /* 厂商特定函数 */
    uint8_t (*probe)(uint8_t phy_addr);
    uint8_t (*config_init)(uint8_t phy_addr);
    uint8_t (*config_aneg)(uint8_t phy_addr, uint16_t adv);
    uint8_t (*read_status)(uint8_t phy_addr, phy_link_info_t *info);
    uint8_t (*enable_intr)(uint8_t phy_addr);
    uint16_t (*read_intr_src)(uint8_t phy_addr);
    void     (*disable_intr)(uint8_t phy_addr);
} phy_driver_t;

/* LAN8720 驱动实例 */
static const phy_driver_t lan8720_driver = {
    .phy_id        = 0x0007C0F0,  /* 忽略版本号 */
    .phy_id_mask   = 0xFFFFFFF0,
    .name          = "LAN8720 (SMSC)",
    .probe         = lan8720_probe,
    .config_init   = lan8720_config_init,
    .config_aneg   = lan8720_config_aneg,
    .read_status   = lan8720_read_status,
    .enable_intr   = lan8720_enable_interrupts,
    .read_intr_src = lan8720_get_interrupt_source,
    .disable_intr  = lan8720_disable_interrupts,
};

/* DP83848 驱动实例 */
static const phy_driver_t dp83848_driver = {
    .phy_id        = 0x20005C90,
    .phy_id_mask   = 0xFFFFFFF0,
    .name          = "DP83848 (TI)",
    .probe         = dp83848_probe,
    .config_init   = dp83848_config_init,
    .config_aneg   = dp83848_config_aneg,
    .read_status   = dp83848_read_status,
    .enable_intr   = dp83848_enable_interrupts,
    .read_intr_src = dp83848_read_interrupt_source,
    .disable_intr  = dp83848_disable_interrupts,
};

/* 驱动表（注册所有支持的 PHY） */
static const phy_driver_t *phy_driver_table[] = {
    &lan8720_driver,
    &dp83848_driver,
    /* 可以继续添加 KSZ8081、RTL8201 等 */
    NULL  /* 终止符 */
};

/* 自动匹配 PHY 驱动 */
const phy_driver_t *phy_match_driver(uint32_t phy_id) {
    for (int i = 0; phy_driver_table[i] != NULL; i++) {
        const phy_driver_t *drv = phy_driver_table[i];
        if ((phy_id & drv->phy_id_mask) == drv->phy_id) {
            return drv;
        }
    }
    return NULL;
}

/* 通用 PHY 初始化（自动适配型号） */
uint8_t phy_generic_init(uint8_t phy_addr) {
    uint32_t phy_id = phy_read_id(phy_addr);
    const phy_driver_t *drv = phy_match_driver(phy_id);

    if (drv == NULL) {
        printf("Unknown PHY (ID=0x%08X)\r\n", phy_id);
        return 0;
    }

    printf("Found: %s (ID=0x%08X)\r\n", drv->name, phy_id);

    /* 调用厂商特定初始化序列 */
    if (!drv->probe(phy_addr)) return 0;
    if (!drv->config_init(phy_addr)) return 0;
    if (!drv->config_aneg(phy_addr, ANAR_DEFAULT)) return 0;

    return 1;
}
```

---

### 9.9 总结与实战建议

**IEEE 802.3 Clause 22 寄存器的三个核心认知**：

1. BMSR 位 2 的延迟锁存特性是所有 PHY 共有的，第一次读到的链路状态可能不反映真实情况，必须丢弃第一次结果
2. BMCR 的 Reset 位自清除时序因 PHY 而异，必须用轮询代替固定延时
3. ANAR 和 ANLPAR 的 Pause 流控位需要根据 IEEE 802.3 Table 28B-2 的规则裁决，不能简单做与运算

**PHY 驱动开发的推荐策略**：

- 低端 PHY（LAN8720）重点关注寄存器映射差异（0x1F 的厂商定义）
- 工业级 PHY（DP83848）额外关注中断系统（0x11/0x12）和电缆诊断
- 千兆 PHY（KSZ9031）需要支持 EEE（0x7C/0x7D）和 Clause 45 扩展寻址
- 在 MCU RTOS 中，建议实现类似 Linux phy_driver 的驱动表结构，通过 PHY ID 自动匹配驱动，实现源代码级的 PHY 透明替换

---

> **附注**：本节所有 PHY 寄存器地址和位定义来自 LAN8720A (Microchip/SMSC) 和 DP83848 (TI) 的官方数据手册。不同批次/版本的 PHY 可能在厂商自定义寄存器上存在差异，实际开发中请以对应芯片数据手册为准。Linux PHY 子系统的代码框架参考了内核 `drivers/net/phy/phy.c` 和 `include/linux/phy.h` 的设计思路。

---

## Appendix B: Network Protocol Beginner Learning Path

> 面向 MCU 开发者的网络协议入门学习路径，从字节流视角逐层拆解以太网协议栈，配合 5 个动手实验，帮助零基础开发者快速建立网络协议的核心概念和实践能力。

---

### B.1 前置知识 (Prerequisites)

在开始学习网络协议之前，需要掌握以下基础知识：

**二进制与十六进制流利度**
网络协议本质上是字节流的结构化排列。常见的数值表示包括：
- 十六进制表示法：`0x08 0x06` = ARP 的 EtherType，`0x08 0x00` = IPv4 的 EtherType
- 位操作：`(value >> 12) & 0x0F` 提取高 4 位，`value & 0x0FFF` 提取低 12 位
- 字节序转换：`0x1234` 在大端序网络中表现为 `0x12 0x34`，在内存中用小端序存储时需翻转

练习：将 `0x8100` 以大端序写入内存的两个字节 -- 结果应为 `{0x81, 0x00}`；将 `0xA0 0x64` 解析为 16 位值 -- 结果应为 `0xA064`。

**基本 C 语言能力**
- 结构体 (`struct`) 与 `__attribute__((packed))`：网络协议字段往往没有对齐间隙，packed 属性确保结构体布局与线缆上的字节流完全一致
- 指针 (`*`) 与类型转换 (`(type *)`)：将接收缓冲区强制转换为协议头部结构体指针，直接访问各字段
- 位运算 (`&`, `|`, `>>`, `<<`)：提取或设置协议头部中的位字段，如 VLAN TCI 中的 PCP、DEI、VID
- `memcpy()`：复制 MAC 地址等固定长度字段

**网络字节序 (Network Byte Order)**
- 网络协议使用 **大端序 (Big-Endian, 也称网络序)**：高位字节在前，低位字节在后
- 主机字节序因 CPU 架构而异：ARM Cortex-M 系列使用小端序
- 转换宏：`__builtin_bswap16()` / `__builtin_bswap32()` (GCC) 或 `htons()` / `htonl()` / `ntohs()` / `ntohl()` (标准 BSD Socket API)
- 规则：**多字节字段从网络接收后必须 ntoh 转换，发送前必须 hton 转换**

```c
// 小端序主机上，从网络接收 2 字节后的正确做法：
uint8_t raw[2] = {0x08, 0x06};  // 从网线接收的原始字节
uint16_t ethertype = ((uint16_t)raw[0] << 8) | raw[1];  // 手动拼装 = 0x0806
// 或将缓冲区强转后 ntohs：
uint16_t ethertype = ntohs(*(uint16_t *)raw);  // = 0x0806
```

---

### B.2 协议学习顺序 (Protocol Learning Order)

网络协议的学习应当遵循 **自底向上、逐层递进** 的原则。以下依赖关系图展示了推荐的协议学习路径：

```
                      ┌─────────────────────────────────────────┐
                      │         实验室 1: 以太网帧                │
                      │  (Ethernet Frame -- 一切的基础)           │
                      └────────────────┬────────────────────────┘
                                       │
                                       ▼
                      ┌─────────────────────────────────────────┐
                      │         实验室 2: ARP 协议                │
                      │  (Address Resolution Protocol)           │
                      └────────────────┬────────────────────────┘
                                       │
                                       ▼
                      ┌─────────────────────────────────────────┐
                      │         实验室 3: IPv4 + ICMP            │
                      │  (IP 协议 + Ping 响应)                    │
                      └────────────────┬────────────────────────┘
                                       │
                                       ▼
                      ┌─────────────────────────────────────────┐
                      │         实验室 4: UDP 协议                │
                      │  (UDP Echo Server -- 最简单的传输层)      │
                      └────────────────┬────────────────────────┘
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                          ▼                         ▼
          ┌─────────────────────────┐   ┌─────────────────────────┐
          │    实验室 5: VLAN       │   │   进阶: TCP 状态机      │
          │    (802.1Q 标记)         │   │   (三次握手 + 滑动窗口)  │
          └─────────────────────────┘   └─────────────────────────┘
```

**学习路径说明**：
1. 以太网帧是所有上层协议的基础载体，必须先理解帧结构
2. ARP 是网络层编址的核心协议，解决了"已知 IP 找 MAC"的问题
3. IPv4 提供跨网络的路由能力，ICMP (Ping) 是最简单的网络层交互
4. UDP 是无连接的传输层协议，实现最简单，适合初学者
5. VLAN 在二层扩展网络功能，TCP 实现可靠传输 -- 两条进阶路径可选

---

### B.3 实验室 1：以太网帧深度拆解 (Lab 1: Ethernet Frame Deep Dive)

#### 实验目标
- 理解以太网帧的逐字节结构
- 能够从原始字节流中手工解析出每个字段
- 编写 C 代码实现帧解析打印

#### 以太网帧结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Preamble (7 字节)                       |
|                   10101010 重复 7 次                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SFD |                       目的 MAC 地址                        |
|10101011|                    (6 字节)                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       源 MAC 地址 (6 字节)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         EtherType (2 字节)      |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                        Payload (46-1500 字节)                   |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        FCS (4 字节)                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**各字段详解**：

| 字段 | 长度 | 说明 | 值示例 |
|------|------|------|--------|
| Preamble (前导码) | 7 字节 | 10101010 重复 7 次，用于接收方同步时钟 | `AA AA AA AA AA AA AA` |
| SFD (帧起始定界符) | 1 字节 | 10101011，标志帧即将开始 | `AB` |
| Destination MAC (目的 MAC) | 6 字节 | 接收方 MAC 地址 | `FF FF FF FF FF FF` (广播) |
| Source MAC (源 MAC) | 6 字节 | 发送方 MAC 地址 | `00 11 22 33 44 55` |
| EtherType (以太类型) | 2 字节 | 指示上层协议类型 | `08 06` (ARP), `08 00` (IPv4) |
| Payload (数据载荷) | 46-1500 字节 | 上层协议数据 | 可变 |
| FCS (帧校验序列) | 4 字节 | CRC32 校验，覆盖 MAC 到 Payload | `XX XX XX XX` |

**注意**：Preamble + SFD 共 8 字节由 PHY 硬件添加和移除，软件通常在接收时看不到这两个字段。软件视角的以太网帧从目的 MAC 开始。

#### 带注释的十六进制转储

考虑一个最小 ARP 广播帧：

```
FF FF FF FF FF FF     ← 目的 MAC：广播地址 (6 字节)
00 11 22 33 44 55     ← 源 MAC：00-11-22-33-44-55 (6 字节)
08 06                 ← EtherType：0x0806 = ARP (2 字节)
00 01                 ← ARP HTYPE：以太网 (2 字节)
08 00                 ← ARP PTYPE：IPv4 (2 字节)
06                    ← HLEN：MAC 地址长度 6 (1 字节)
04                    ← PLEN：IP 地址长度 4 (1 字节)
00 01                 ← OPER：1 = 请求 (2 字节)
00 11 22 33 44 55     ← SHA：发送方 MAC (6 字节)
C0 A8 01 64           ← SPA：发送方 IP = 192.168.1.100 (4 字节)
00 00 00 00 00 00     ← THA：目标 MAC = 0 (未知，6 字节)
C0 A8 01 01           ← TPA：目标 IP = 192.168.1.1 (4 字节)
00 00 00 00 00 00...  ← 填充位 (Pad) 使帧达到 64 字节
XX XX XX XX           ← FCS：CRC32 (4 字节)
```

#### C 代码：以太网帧解析器

```c
#include <stdint.h>
#include <stdio.h>
#include <string.h>

/* 以太网帧头结构 (14 字节，不含 Preamble/SFD/FCS) */
typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[6];     /* 目的 MAC */
    uint8_t  src_mac[6];     /* 源 MAC */
    uint16_t ethertype;      /* EtherType (网络字节序) */
} eth_header_t;

/* 常用 EtherType */
#define ETHTYPE_IPV4  0x0800
#define ETHTYPE_ARP   0x0806
#define ETHTYPE_VLAN  0x8100
#define ETHTYPE_IPV6  0x86DD
#define ETHTYPE_PAUSE 0x8808
#define ETHTYPE_LLDP  0x88CC

/* MAC 地址转字符串 */
void mac_to_str(const uint8_t mac[6], char *str, int str_len) {
    if (str_len < 18) return;
    snprintf(str, str_len, "%02X-%02X-%02X-%02X-%02X-%02X",
             mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
}

/* 解析并打印以太网帧头 */
void eth_frame_print(const uint8_t *frame, int frame_len) {
    if (frame_len < 14) {
        printf("Frame too short: %d bytes (minimum 14)\r\n", frame_len);
        return;
    }

    const eth_header_t *eth = (const eth_header_t *)frame;
    char mac_str[18];

    mac_to_str(eth->dst_mac, mac_str, sizeof(mac_str));
    printf("Destination MAC : %s\r\n", mac_str);

    mac_to_str(eth->src_mac, mac_str, sizeof(mac_str));
    printf("Source MAC      : %s\r\n", mac_str);

    /* EtherType 是网络字节序，需要转换为本地字节序 */
    uint16_t etype = __builtin_bswap16(eth->ethertype);
    printf("EtherType       : 0x%04X ", etype);

    switch (etype) {
    case ETHTYPE_IPV4:  printf("(IPv4)\r\n");  break;
    case ETHTYPE_ARP:   printf("(ARP)\r\n");   break;
    case ETHTYPE_VLAN:  printf("(802.1Q VLAN)\r\n"); break;
    case ETHTYPE_IPV6:  printf("(IPv6)\r\n");  break;
    case ETHTYPE_LLDP:  printf("(LLDP)\r\n");  break;
    case ETHTYPE_PAUSE: printf("(MAC Control/Pause)\r\n"); break;
    default:            printf("(Unknown/LLC)\r\n"); break;
    }

    /* 判断是否为广播帧 */
    uint8_t broadcast_mac[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
    if (memcmp(eth->dst_mac, broadcast_mac, 6) == 0) {
        printf("  -> Broadcast frame\r\n");
    }

    /* 判断是否为组播帧 (MAC 第 1 字节最低位 = 1) */
    if (eth->dst_mac[0] & 0x01) {
        printf("  -> Multicast frame\r\n");
    }

    int payload_len = frame_len - 14;
    printf("Payload length  : %d bytes\r\n", payload_len);

    /* 打印前 16 字节 payload 的 hex dump */
    if (payload_len > 0) {
        printf("Payload hex     :");
        int dump_len = (payload_len > 16) ? 16 : payload_len;
        for (int i = 0; i < dump_len; i++) {
            if (i % 8 == 0) printf(" ");
            printf(" %02X", frame[14 + i]);
        }
        printf("\r\n");
    }
}
```

#### 动手练习 (Step-by-Step)

1. 用手工解析以下 hex 转储，写出每个字段的值：
   ```
   FF FF FF FF FF FF 00 0A 95 9D 68 18 08 06
   ```
   (答案：dstMAC=FF:FF:FF:FF:FF:FF, srcMAC=00:0A:95:9D:68:18, EtherType=0x0806=ARP)

2. 使用 Wireshark 捕获一个真实以太网帧，对比软件输出与 Wireshark 解析结果

3. 修改 `eth_frame_print` 函数，增加对 802.1Q VLAN 帧的检测（EtherType=0x8100 时打印 "VLAN tagged"）

---

### B.4 实验室 2：构建 ARP 请求 (Lab 2: Build an ARP Request)

#### 实验目标
- 理解 ARP 协议的消息结构
- 手动构建并发送 ARP 请求帧
- 解析 ARP 应答，提取对端 MAC 地址
- 实现一个简单的 ARP 缓存表

#### ARP 协议结构

ARP (Address Resolution Protocol) 用于将 IP 地址解析为 MAC 地址。ARP 报文直接封装在以太网帧中，EtherType = `0x0806`。

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Hardware Type (HTYPE)    |       Protocol Type (PTYPE)  |
|             0x0001 (Ethernet)   |          0x0800 (IPv4)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| HLEN  | PLEN  |         Operation (OPER)                      |
| 0x06  | 0x04  |     0x0001=Request, 0x0002=Reply             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Sender Hardware Address (SHA, 6 字节)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Sender Protocol Address (SPA, 4 字节)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Target Hardware Address (THA, 6 字节)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Target Protocol Address (TPA, 4 字节)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

ARP 报文共 28 字节（不含以太网头部）：

| 字段 | 长度 | 说明 | ARP 请求示例 | ARP 应答示例 |
|------|------|------|-------------|-------------|
| HTYPE | 2 字节 | 硬件类型，以太网=1 | `00 01` | `00 01` |
| PTYPE | 2 字节 | 协议类型，IPv4=0x0800 | `08 00` | `08 00` |
| HLEN | 1 字节 | 硬件地址长度，MAC=6 | `06` | `06` |
| PLEN | 1 字节 | 协议地址长度，IPv4=4 | `04` | `04` |
| OPER | 2 字节 | 操作码，1=请求，2=应答 | `00 01` | `00 02` |
| SHA | 6 字节 | 发送方 MAC | 本机 MAC | 本机 MAC |
| SPA | 4 字节 | 发送方 IP | 本机 IP | 本机 IP |
| THA | 6 字节 | 目标 MAC（请求中为 0） | `00 00 00 00 00 00` | 对端 MAC |
| TPA | 4 字节 | 目标 IP | 目标 IP | 目标 IP |

#### 完整的 ARP 请求构建器

```c
#include <stdint.h>
#include <stdio.h>
#include <string.h>

/* 以太网帧头 */
typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[6];
    uint8_t  src_mac[6];
    uint16_t ethertype;  /* 0x0806 */
} eth_header_t;

/* ARP 报文 (28 字节) */
typedef struct __attribute__((packed)) {
    uint16_t htype;       /* 硬件类型 */
    uint16_t ptype;       /* 协议类型 */
    uint8_t  hlen;        /* 硬件地址长度 */
    uint8_t  plen;        /* 协议地址长度 */
    uint16_t oper;        /* 操作码 */
    uint8_t  sha[6];      /* 发送方 MAC */
    uint8_t  spa[4];      /* 发送方 IP */
    uint8_t  tha[6];      /* 目标 MAC */
    uint8_t  tpa[4];      /* 目标 IP */
} arp_packet_t;

#define ARP_HTYPE_ETHERNET 1
#define ARP_PTYPE_IPV4     0x0800
#define ARP_OP_REQUEST     1
#define ARP_OP_REPLY       2

/* 广播 MAC */
static const uint8_t broadcast_mac[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

/**
 * 构建 ARP 请求帧
 * buffer:   输出缓冲区 (至少 42 字节: 14 + 28)
 * src_mac:  本机 MAC 地址
 * src_ip:   本机 IP 地址 (网络字节序)
 * target_ip: 目标 IP 地址 (网络字节序)
 * 返回: 帧总长度 (42 字节以太网帧头 + ARP，不含 FCS)
 */
int arp_build_request(uint8_t *buffer,
                      const uint8_t src_mac[6],
                      uint32_t src_ip,
                      uint32_t target_ip) {
    eth_header_t *eth = (eth_header_t *)buffer;
    arp_packet_t *arp = (arp_packet_t *)(buffer + sizeof(eth_header_t));

    /* ---- 填充以太网头部 ---- */
    memcpy(eth->dst_mac, broadcast_mac, 6);    /* 广播到所有设备 */
    memcpy(eth->src_mac, src_mac, 6);           /* 本机 MAC */
    eth->ethertype = __builtin_bswap16(0x0806); /* ARP EtherType */

    /* ---- 填充 ARP 报文 ---- */
    arp->htype = __builtin_bswap16(ARP_HTYPE_ETHERNET); /* 以太网 */
    arp->ptype = __builtin_bswap16(ARP_PTYPE_IPV4);     /* IPv4 */
    arp->hlen  = 6;                           /* MAC 地址长度 */
    arp->plen  = 4;                           /* IP 地址长度 */
    arp->oper  = __builtin_bswap16(ARP_OP_REQUEST); /* 请求 */

    /* TODO: 填写发送方硬件地址 (本机 MAC) */
    memcpy(arp->sha, src_mac, 6);

    /* TODO: 填写发送方协议地址 (本机 IP) */
    arp->spa[0] = (src_ip >> 24) & 0xFF;  /* 或者直接 memcpy */
    arp->spa[1] = (src_ip >> 16) & 0xFF;
    arp->spa[2] = (src_ip >> 8) & 0xFF;
    arp->spa[3] = src_ip & 0xFF;

    /* TODO: 填写目标硬件地址 (未知，填 0) */
    memset(arp->tha, 0, 6);

    /* TODO: 填写目标协议地址 (要查询的 IP) */
    arp->tpa[0] = (target_ip >> 24) & 0xFF;
    arp->tpa[1] = (target_ip >> 16) & 0xFF;
    arp->tpa[2] = (target_ip >> 8) & 0xFF;
    arp->tpa[3] = target_ip & 0xFF;

    return sizeof(eth_header_t) + sizeof(arp_packet_t);
}

/**
 * 解析 ARP 应答
 * buffer: 接收到的完整帧 (以太网头 + ARP)
 * len:    帧长度
 * peer_mac: 输出参数，对端的 MAC 地址
 * 返回: 1=有效ARP应答, 0=非ARP应答或ARP请求
 */
int arp_parse_reply(const uint8_t *buffer, int len, uint8_t peer_mac[6]) {
    if (len < (int)(sizeof(eth_header_t) + sizeof(arp_packet_t))) {
        return 0;
    }

    const eth_header_t *eth = (const eth_header_t *)buffer;
    const arp_packet_t *arp = (const arp_packet_t *)(buffer + sizeof(eth_header_t));

    /* 检查 EtherType */
    if (__builtin_bswap16(eth->ethertype) != 0x0806) return 0;

    /* 检查是否为 ARP 应答 (oper=2) */
    if (__builtin_bswap16(arp->oper) != ARP_OP_REPLY) return 0;

    /* 提取对端的 MAC 地址 (发送方硬件地址) */
    memcpy(peer_mac, arp->sha, 6);
    return 1;
}

/**
 * 打印 ARP 报文内容 (调试用)
 */
void arp_print(const uint8_t *buffer, int len) {
    if (len < (int)(sizeof(eth_header_t) + sizeof(arp_packet_t))) {
        printf("ARP packet too short\r\n");
        return;
    }

    const arp_packet_t *arp = (const arp_packet_t *)(buffer + sizeof(eth_header_t));
    uint16_t oper = __builtin_bswap16(arp->oper);

    printf("ARP %s:\r\n", (oper == ARP_OP_REQUEST) ? "Request" : "Reply");
    printf("  Sender MAC : %02X-%02X-%02X-%02X-%02X-%02X\r\n",
           arp->sha[0], arp->sha[1], arp->sha[2],
           arp->sha[3], arp->sha[4], arp->sha[5]);
    printf("  Sender IP  : %d.%d.%d.%d\r\n",
           arp->spa[0], arp->spa[1], arp->spa[2], arp->spa[3]);
    printf("  Target MAC : %02X-%02X-%02X-%02X-%02X-%02X\r\n",
           arp->tha[0], arp->tha[1], arp->tha[2],
           arp->tha[3], arp->tha[4], arp->tha[5]);
    printf("  Target IP  : %d.%d.%d.%d\r\n",
           arp->tpa[0], arp->tpa[1], arp->tpa[2], arp->tpa[3]);
}
```

#### 简单的 ARP 缓存实现

```c
#define ARP_CACHE_SIZE 8

typedef struct {
    uint32_t ip_addr;          /* IP 地址 (网络字节序) */
    uint8_t  mac_addr[6];      /* MAC 地址 */
    uint32_t timestamp_ticks;  /* 记录时间戳 (ticks) */
    uint8_t  valid;            /* 是否有效 */
} arp_cache_entry_t;

typedef struct {
    arp_cache_entry_t entries[ARP_CACHE_SIZE];
    int               count;
} arp_cache_t;

void arp_cache_init(arp_cache_t *cache) {
    memset(cache, 0, sizeof(arp_cache_t));
}

/**
 * 在 ARP 缓存中查找 IP 对应的 MAC
 * 返回: 1=找到, 0=未找到
 */
int arp_cache_lookup(arp_cache_t *cache, uint32_t ip, uint8_t mac[6]) {
    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (cache->entries[i].valid &&
            cache->entries[i].ip_addr == ip) {
            memcpy(mac, cache->entries[i].mac_addr, 6);
            return 1;
        }
    }
    return 0;
}

/**
 * 更新 ARP 缓存 (从 ARP 应答中学习)
 */
void arp_cache_update(arp_cache_t *cache, uint32_t ip, const uint8_t mac[6]) {
    /* 查找是否已有该 IP 的表项 */
    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (cache->entries[i].valid &&
            cache->entries[i].ip_addr == ip) {
            memcpy(cache->entries[i].mac_addr, mac, 6);
            cache->entries[i].timestamp_ticks = 0;
            return;
        }
    }

    /* 分配新表项 (覆盖最旧的) */
    int oldest = 0;
    for (int i = 1; i < ARP_CACHE_SIZE; i++) {
        if (!cache->entries[i].valid) { oldest = i; break; }
        if (cache->entries[i].timestamp_ticks >
            cache->entries[oldest].timestamp_ticks) {
            oldest = i;
        }
    }

    cache->entries[oldest].ip_addr = ip;
    memcpy(cache->entries[oldest].mac_addr, mac, 6);
    cache->entries[oldest].timestamp_ticks = 0;
    cache->entries[oldest].valid = 1;
}

/* ARP 缓存老化 (每秒调用一次) */
void arp_cache_tick(arp_cache_t *cache) {
    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (!cache->entries[i].valid) continue;
        cache->entries[i].timestamp_ticks++;
        /* ARP 缓存条目 300 秒后过期 (RFC 1122 建议) */
        if (cache->entries[i].timestamp_ticks > 300) {
            cache->entries[i].valid = 0;
        }
    }
}
```

#### ARP 工作流程 (动手理解)

1. **主机 A (192.168.1.100, MAC=00:11:22:33:44:55)** 需要向 **主机 B (192.168.1.1)** 发送数据
2. 主机 A 检查 ARP 缓存，未找到 192.168.1.1 对应的 MAC
3. 主机 A 构造 ARP 请求：
   - 以太网帧：dst=FF:FF:FF:FF:FF:FF (广播), src=00:11:22:33:44:55, EtherType=0x0806
   - ARP：OPER=1, SHA=00:11:22:33:44:55, SPA=192.168.1.100, THA=00:00:00:00:00:00, TPA=192.168.1.1
4. 交换机广播该帧到所有端口
5. 主机 B (192.168.1.1) 收到后，发现 TPA 是自己的 IP，发送 ARP 应答：
   - 以太网帧：dst=00:11:22:33:44:55 (单播), src=主机B的MAC, EtherType=0x0806
   - ARP：OPER=2, SHA=主机B的MAC, SPA=192.168.1.1, THA=00:11:22:33:44:55, TPA=192.168.1.100
6. 主机 A 收到应答，提取 SHA 得到主机 B 的 MAC，存入 ARP 缓存

#### 动手练习

1. 在纸上手写构建一个 ARP 请求：本机 MAC=00:11:22:33:44:55, 本机 IP=10.0.0.2, 目标 IP=10.0.0.1。写出完整的 42 字节十六进制值
2. 用 `arp_build_request()` 构建请求帧，假设底层有 `emac_send_frame(buffer, len)` 发送函数
3. 在接收回调中调用 `arp_parse_reply()` 提取对端 MAC，再调用 `arp_cache_update()` 存入缓存
4. 思考题：ARP 应答为什么用单播而不用广播？

---

### B.5 实验室 3：响应 Ping (ICMP Echo)

#### 实验目标
- 理解 IP 头部结构
- 理解 ICMP 协议 (Echo Request / Echo Reply)
- 实现从接收 Ping 请求到发送 Ping 响应的完整链路
- 掌握 IP 和 ICMP 校验和计算方法

#### 协议层次回顾

一个完整的 Ping 请求帧包含三个协议层：

```
+-------------------+     +----------------------------+     +-------------------+
| 以太网帧头 (14B)   | --> | IP 头部 (20B, 无选项)       | --> | ICMP Echo Request |
| dstMAC|srcMAC|0x0800|    | ver=4, proto=1, ...        |    | type=8, ...       |
+-------------------+     +----------------------------+     +-------------------+
```

#### 实现原理

响应 Ping 的步骤：
1. 接收帧，检查 EtherType = 0x0800 (IPv4)
2. 解析 IP 头部，检查 Protocol 字段 = 1 (ICMP)
3. 跳过 IP 头部，检查 ICMP Type = 8 (Echo Request)
4. **交换** 源和目的 MAC 地址
5. **交换** 源和目的 IP 地址
6. 将 ICMP Type 从 8 改为 0 (Echo Reply)
7. 将 ICMP Code 置为 0
8. 重新计算 ICMP 校验和
9. 重新计算 IP 头部校验和
10. 发送新帧

#### 完整 C 代码实现

```c
#include <stdint.h>
#include <string.h>

/* 以太网帧头 */
typedef struct __attribute__((packed)) {
    uint8_t  dst_mac[6];
    uint8_t  src_mac[6];
    uint16_t ethertype;
} eth_header_t;

/* IP 头部 (20 字节，无选项) */
typedef struct __attribute__((packed)) {
    uint8_t  ver_ihl;          /* 版本(4) + 头部长度(4, 单位4字节) */
    uint8_t  dscp_ecn;         /* 差分服务 + 显式拥塞通知 */
    uint16_t total_length;     /* 总长度 (包括头部) */
    uint16_t identification;   /* 标识 */
    uint16_t flags_offset;     /* 标志(3) + 片偏移(13) */
    uint8_t  ttl;              /* 生存时间 */
    uint8_t  protocol;         /* 协议 (1=ICMP, 6=TCP, 17=UDP) */
    uint16_t checksum;         /* 头部校验和 */
    uint8_t  src_ip[4];        /* 源 IP */
    uint8_t  dst_ip[4];        /* 目的 IP */
} ip_header_t;

/* ICMP 头部 (4 字节 + 可选数据) */
typedef struct __attribute__((packed)) {
    uint8_t  type;             /* 8=Echo Request, 0=Echo Reply */
    uint8_t  code;             /* 对 Echo 为 0 */
    uint16_t checksum;         /* ICMP 校验和 (覆盖整个 ICMP 报文) */
    uint16_t identifier;       /* 标识符 (用于匹配请求/回复) */
    uint16_t sequence_num;     /* 序列号 */
} icmp_header_t;

/* 协议常量 */
#define IP_PROTO_ICMP  1
#define ICMP_TYPE_ECHO_REQUEST 8
#define ICMP_TYPE_ECHO_REPLY   0

/**
 * 计算 Internet 校验和 (RFC 1071)
 * 以 16 位为单位的反码求和
 */
uint16_t internet_checksum(const uint8_t *data, int len) {
    uint32_t sum = 0;
    const uint16_t *p = (const uint16_t *)data;

    while (len >= 2) {
        sum += __builtin_bswap16(*p++);
        if (sum & 0x80000000) sum = (sum & 0xFFFF) + (sum >> 16);
        len -= 2;
    }

    /* 如果有奇数个字节，补 0 继续累加 */
    if (len == 1) {
        sum += __builtin_bswap16((uint16_t)(*(const uint8_t *)p) << 8);
    }

    /* 进位折叠 */
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }

    return (uint16_t)(~sum & 0xFFFF);
}

/**
 * 设置 IP 头部校验和 (校验和字段先置 0)
 */
void ip_set_checksum(ip_header_t *ip) {
    ip->checksum = 0;
    ip->checksum = internet_checksum((uint8_t *)ip, 20);
}

/**
 * 处理接收到的 Ping 请求并构建响应
 * rx_buffer: 接收到的完整帧 (将被原地修改为响应帧)
 * rx_len:    接收到的帧长度
 * 返回: 响应帧长度 (0=不是 Ping 请求)
 */
int icmp_handle_ping(uint8_t *rx_buffer, int rx_len) {
    if (rx_len < (int)(sizeof(eth_header_t) + sizeof(ip_header_t) +
                         sizeof(icmp_header_t))) {
        return 0; /* 帧太短 */
    }

    eth_header_t *eth = (eth_header_t *)rx_buffer;
    ip_header_t  *ip  = (ip_header_t  *)(rx_buffer + sizeof(eth_header_t));

    /* 步骤 1: 检查 EtherType */
    if (__builtin_bswap16(eth->ethertype) != 0x0800) return 0;

    /* 步骤 2: 检查 IP 协议 */
    if (ip->protocol != IP_PROTO_ICMP) return 0;

    /* 步骤 3: 计算 IP 头部长度 */
    int ip_hdr_len = (ip->ver_ihl & 0x0F) * 4;
    if (ip_hdr_len < 20) return 0;

    /* 步骤 4: 指向 ICMP 头部 */
    icmp_header_t *icmp = (icmp_header_t *)(rx_buffer +
                                             sizeof(eth_header_t) + ip_hdr_len);

    /* 步骤 5: 检查是否是 Echo Request */
    if (icmp->type != ICMP_TYPE_ECHO_REQUEST) return 0;
    if (icmp->code != 0) return 0;

    /* 步骤 6: 交换 MAC 地址 */
    uint8_t temp_mac[6];
    memcpy(temp_mac, eth->dst_mac, 6);
    memcpy(eth->dst_mac, eth->src_mac, 6);
    memcpy(eth->src_mac, temp_mac, 6);

    /* 步骤 7: 交换 IP 地址 */
    uint8_t temp_ip[4];
    memcpy(temp_ip, ip->dst_ip, 4);
    memcpy(ip->dst_ip, ip->src_ip, 4);
    memcpy(ip->src_ip, temp_ip, 4);

    /* 步骤 8: 修改 ICMP 类型为 Echo Reply */
    icmp->type = ICMP_TYPE_ECHO_REPLY;
    icmp->code = 0;

    /* 步骤 9: 重新计算 ICMP 校验和 */
    /* ICMP 报文长度 = IP 总长度 - IP 头部长度 */
    int icmp_len = __builtin_bswap16(ip->total_length) - ip_hdr_len;
    icmp->checksum = 0;
    icmp->checksum = internet_checksum((uint8_t *)icmp, icmp_len);

    /* 步骤 10: 重新计算 IP 头部校验和 */
    ip_set_checksum(ip);

    /* 步骤 11: 返回响应帧长度 (与接收帧相同) */
    return rx_len;
}
```

#### 动手练习

1. 理解为什么 ICMP 校验和覆盖整个 ICMP 报文（包括 Identifier 和 Sequence Number 以及后面的 Payload 数据），而 IP 校验和只覆盖 IP 头部
2. 修改代码，使其支持 Ping 请求中携带自定义数据的场景（标准 Ping 通常会发送 56 字节的填充数据）
3. 思考：如果需要同时支持 Ping 响应和 Ping 请求（即你的设备也可以主动发起 Ping），代码应该怎么组织？
4. 验证：用 Wireshark 捕获 PC ping MCU 的报文，观察请求和响应的字节差异

---

### B.6 实验室 4：UDP Echo 服务器 (Lab 4: UDP Echo Server)

#### 实验目标
- 理解 UDP 头部结构
- 实现一个简单的 UDP Echo 服务器（收到什么数据就原样返回）
- 掌握 MAC、IP、UDP 三层的地址交换

#### UDP 头部结构

UDP (User Datagram Protocol) 提供无连接的传输层服务，头部仅 8 字节：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         源端口 (Src Port)       |       目的端口 (Dst Port)      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             UDP 长度 (Length)    |          UDP 校验和 (Checksum) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 字段 | 长度 | 说明 |
|------|------|------|
| 源端口 | 2 字节 | 发送方端口 (客户端随机分配，如 49152) |
| 目的端口 | 2 字节 | 接收方端口 (如 DHCP=67, DNS=53, Echo=7) |
| UDP 长度 | 2 字节 | UDP 头部 + 数据的字节数 (最小 8) |
| UDP 校验和 | 2 字节 | 覆盖 UDP 伪头部 + 头部 + 数据 (可选在 IPv4 中) |

#### 协议层次回顾

一个完整的 UDP 数据报封装在 IP 和以太网中：

```
+-------------------+     +----------------------------+     +-------------------+
| 以太网帧头 (14B)   | --> | IP 头部 (20B, 无选项)       | --> | UDP 头部 (8B)      |
| dstMAC|srcMAC|0x0800|    | ver=4, proto=17(UDP), ...  |    | srcport|dstport|len|chk|
+-------------------+     +----------------------------+     +-------------------+
                                                                    |
                                                                    v
                                                               +-----------+
                                                               | Payload   |
                                                               | (可变长度)  |
                                                               +-----------+
```

#### 完整 C 代码：UDP Echo Server

```c
#include <stdint.h>
#include <string.h>

/* UDP 头部 (8 字节) */
typedef struct __attribute__((packed)) {
    uint16_t src_port;       /* 源端口 */
    uint16_t dst_port;       /* 目的端口 */
    uint16_t length;         /* UDP 头部 + 数据长度 */
    uint16_t checksum;       /* UDP 校验和 */
} udp_header_t;

#define IP_PROTO_UDP 17

extern void emac_send_frame(const uint8_t *frame, int len);

/**
 * UDP Echo Server 处理函数
 * 收到 UDP 报文后，交换 MAC/IP/端口，原样返回 payload
 *
 * 典型用例：PC 上运行 netcat:
 *   echo "Hello" | nc -u -p 12345 <MCU_IP> 7
 * 或 Python:
 *   import socket; s=socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
 *   s.sendto(b"Hello", ("<MCU_IP>", 7))
 *
 * rx_buffer: 接收到的完整帧 (将被原地修改)
 * rx_len:    接收帧长度
 * 返回: 1=已发送UDP Echo响应, 0=不是UDP包
 */
int udp_echo_server(uint8_t *rx_buffer, int rx_len) {
    if (rx_len < (int)(sizeof(eth_header_t) + sizeof(ip_header_t) +
                         sizeof(udp_header_t))) {
        return 0;
    }

    eth_header_t *eth = (eth_header_t *)rx_buffer;
    ip_header_t  *ip  = (ip_header_t  *)(rx_buffer + sizeof(eth_header_t));

    /* 检查 EtherType */
    if (__builtin_bswap16(eth->ethertype) != 0x0800) return 0;

    /* 计算 IP 头部长度 */
    int ip_hdr_len = (ip->ver_ihl & 0x0F) * 4;
    if (ip_hdr_len < 20) return 0;

    /* 检查 IP 协议 */
    if (ip->protocol != IP_PROTO_UDP) return 0;

    /* 指向 UDP 头部 */
    udp_header_t *udp = (udp_header_t *)(rx_buffer +
                                          sizeof(eth_header_t) + ip_hdr_len);

    /* 交换 MAC 地址 */
    uint8_t temp_mac[6];
    memcpy(temp_mac, eth->dst_mac, 6);
    memcpy(eth->dst_mac, eth->src_mac, 6);
    memcpy(eth->src_mac, temp_mac, 6);

    /* 交换 IP 地址 */
    uint8_t temp_ip[4];
    memcpy(temp_ip, ip->dst_ip, 4);
    memcpy(ip->dst_ip, ip->src_ip, 4);
    memcpy(ip->src_ip, temp_ip, 4);

    /* 交换 UDP 端口 (源端口 ↔ 目的端口) */
    uint16_t temp_port = udp->src_port;
    udp->src_port = udp->dst_port;
    udp->dst_port = temp_port;

    /* UDP 长度不变 (原样返回同样的 payload) */

    /* 重新计算 IP 头部校验和 */
    ip->checksum = 0;
    ip->checksum = internet_checksum((uint8_t *)ip, ip_hdr_len);

    /* 重新计算 UDP 校验和 */
    /* 注意：UDP 校验和包含伪头部 (src_ip + dst_ip + 0 + proto + udp_len) */
    udp->checksum = 0;  /* 设置为 0 表示校验和未计算 (IPv4 允许) */
    /* 为符合规范，最好正确计算，这里因篇幅暂略 */

    /* 发送响应 (与接收帧同长度) */
    emac_send_frame(rx_buffer, rx_len);

    return 1;
}
```

#### UDP 校验和计算（伪头部）

UDP 校验和与 TCP 校验和类似，包含一个 **伪头部 (Pseudo Header)**，用于验证数据报是否到达了正确的主机和端口：

```c
/**
 * 计算 UDP 校验和 (包含伪头部)
 * buf:      UDP 头部 + 数据的起始地址
 * udp_len:  UDP 头部 + 数据的长度
 * src_ip:   源 IP (网络字节序)
 * dst_ip:   目的 IP (网络字节序)
 */
uint16_t udp_checksum_calc(const uint8_t *buf, int udp_len,
                           uint32_t src_ip, uint32_t dst_ip) {
    uint32_t sum = 0;
    int i;

    /* 伪头部 (12 字节) */
    uint8_t pseudo[12];
    /* 源 IP */
    pseudo[0]  = (src_ip >> 24) & 0xFF;
    pseudo[1]  = (src_ip >> 16) & 0xFF;
    pseudo[2]  = (src_ip >> 8) & 0xFF;
    pseudo[3]  = src_ip & 0xFF;
    /* 目的 IP */
    pseudo[4]  = (dst_ip >> 24) & 0xFF;
    pseudo[5]  = (dst_ip >> 16) & 0xFF;
    pseudo[6]  = (dst_ip >> 8) & 0xFF;
    pseudo[7]  = dst_ip & 0xFF;
    /* 保留 (1 字节) + 协议 (1 字节) */
    pseudo[8]  = 0;
    pseudo[9]  = IP_PROTO_UDP;
    /* UDP 长度 */
    pseudo[10] = (udp_len >> 8) & 0xFF;
    pseudo[11] = udp_len & 0xFF;

    /* 累加伪头部 */
    for (i = 0; i < 12; i += 2) {
        sum += ((uint16_t)pseudo[i] << 8) | pseudo[i + 1];
    }

    /* 累加 UDP 头部 + 数据 */
    const uint16_t *p = (const uint16_t *)buf;
    int remaining = udp_len;
    while (remaining > 1) {
        sum += *p++;
        remaining -= 2;
    }
    if (remaining == 1) {
        sum += (uint16_t)(*(const uint8_t *)p) << 8;
    }

    /* 进位折叠 */
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }

    return (uint16_t)(~sum & 0xFFFF);
}
```

#### 动手练习

1. 用 netcat 测试 UDP Echo 服务器：`echo "Hello" | nc -u <MCU_IP> 7`，看是否能收到回显
2. 思考：为什么 `udp->checksum = 0` 可以工作？IPv4 对 UDP 校验和有什么特殊规定？
3. 扩展练习：将 UDP Echo 服务器改为只响应特定端口（如 8888），其他端口丢弃

---

### B.7 实验室 5：VLAN 标记 (Lab 5: VLAN Tagging)

#### 实验目标
- 理解 802.1Q VLAN Tag 的结构
- 实现 VLAN Tag 的插入和移除
- 理解 tagged 帧与 untagged 帧的区别

#### VLAN Tag 结构回顾

802.1Q Tag 是 4 字节的字段，插入在源 MAC 和 EtherType 之间：

```
原始以太网帧:
+----------------+----------------+----------+------------------+
|  Dst MAC (6B)  |  Src MAC (6B)  | EtherType|   Payload       |
|                |                |  (2B)    |   (46-1500B)    |
+----------------+----------------+----------+------------------+

带 802.1Q Tag:
+----------------+----------------+----------+----------+-------+
|  Dst MAC (6B)  |  Src MAC (6B)  | Tag(4B)  | EtherType|Payload|
|                |                |0x8100|TCI|  (2B)    |       |
+----------------+----------------+----------+----------+-------+
```

Tag 内部结构 (4 字节)：
```
 0                   1                   2                   3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     TPID (0x8100)              |PCP(3)|DEI(1)|  VID (12)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **TPID (16 位)** = `0x8100`，标志这是一个 802.1Q 帧
- **PCP (3 位)** = 优先级，0-7，7=最高
- **DEI (1 位)** = 是否可丢弃 (Drop Eligible Indicator)
- **VID (12 位)** = VLAN ID，1-4094

#### C 代码：VLAN Tag 插入和移除

```c
#include <stdint.h>
#include <string.h>

#define VLAN_TPID 0x8100

/**
 * 构建 VLAN TCI 值
 * pcp: 优先级 (0-7)
 * dei: 丢弃指示 (0-1)
 * vid: VLAN ID (0-4094)
 * 返回: 网络字节序的 TCI
 */
static inline uint16_t vlan_make_tci(uint8_t pcp, uint8_t dei, uint16_t vid) {
    uint16_t tci = ((uint16_t)(pcp & 0x07) << 13) |
                   ((uint16_t)(dei & 0x01) << 12) |
                   (vid & 0x0FFF);
    return __builtin_bswap16(tci);
}

/**
 * 插入 VLAN Tag
 *
 * 假设 buffer 布局 (插入前):
 *   [0-5] = dst_mac, [6-11] = src_mac, [12-13] = EtherType, [14+] = payload
 *
 * 插入后:
 *   [0-5] = dst_mac, [6-11] = src_mac,
 *   [12-13] = TPID (0x8100), [14-15] = TCI,
 *   [16-17] = 原 EtherType, [18+] = payload
 *
 * buffer: 帧缓冲区 (需有至少 4 字节额外空间)
 * len:    当前帧长度 (不含 FCS)
 * pcp:    Priority Code Point (0-7)
 * dei:    Drop Eligible Indicator (0-1)
 * vid:    VLAN ID (1-4094)
 * 返回:   插入后的帧长度 (-1=错误)
 */
int vlan_insert_tag(uint8_t *buffer, int len, uint8_t pcp, uint8_t dei, uint16_t vid) {
    if (len < 14) return -1;  /* 帧头不完整 */

    /* 检查是否已有 VLAN Tag */
    uint16_t tpid_check = (buffer[12] << 8) | buffer[13];
    if (tpid_check == VLAN_TPID) return -1;  /* 已标记 */

    /* 保存原 EtherType (位于 buffer[12-13]) */
    uint16_t orig_ethertype = (buffer[12] << 8) | buffer[13];

    /* 将 payload 向后移动 4 字节 (从末尾开始) */
    for (int i = len - 1; i >= 14; i--) {
        buffer[i + 4] = buffer[i];
    }

    /* 插入 VLAN Tag */
    buffer[12] = (VLAN_TPID >> 8) & 0xFF;     /* 0x81 */
    buffer[13] = VLAN_TPID & 0xFF;             /* 0x00 */

    uint16_t tci = vlan_make_tci(pcp, dei, vid);
    buffer[14] = (tci >> 8) & 0xFF;
    buffer[15] = tci & 0xFF;

    /* 恢复原 EtherType (在 Tag 之后) */
    buffer[16] = (orig_ethertype >> 8) & 0xFF;
    buffer[17] = orig_ethertype & 0xFF;

    return len + 4;
}

/**
 * 移除 VLAN Tag
 *
 * buffer: 帧缓冲区
 * len:    当前帧长度 (输入/输出参数)
 * 返回:   1=成功移除, 0=不是 VLAN 帧, -1=帧太短
 */
int vlan_remove_tag(uint8_t *buffer, int *len) {
    if (*len < 18) return -1;  /* 至少 14(帧头) + 4(Tag) */

    /* 检查 TPID */
    uint16_t tpid = (buffer[12] << 8) | buffer[13];
    if (tpid != VLAN_TPID) return 0;  /* 不是 VLAN 帧 */

    /* 读取插入的 EtherType (在 Tag 之后) */
    uint16_t inner_ethertype = (buffer[16] << 8) | buffer[17];

    /* 提取 TCI 信息 (可选: 记录 VID 等) */
    uint16_t tci = (buffer[14] << 8) | buffer[15];
    uint16_t vid = tci & 0x0FFF;
    (void)vid;  /* 如需使用 VID 可取消注释 */

    /* 将 payload 向前移动 4 字节 */
    int payload_start = 18;  /* dst(6)+src(6)+TPID(2)+TCI(2)+EtherType(2) */
    int payload_end = *len;  /* 原始帧结束 */
    for (int i = payload_start; i < payload_end; i++) {
        buffer[i - 4] = buffer[i];
    }

    /* 恢复 EtherType 到原始位置 */
    buffer[12] = (inner_ethertype >> 8) & 0xFF;
    buffer[13] = inner_ethertype & 0xFF;

    *len -= 4;
    return 1;
}

/* 示例：从 VLAN 帧中提取 VID */
uint16_t vlan_get_vid(const uint8_t *buffer) {
    uint16_t tpid = (buffer[12] << 8) | buffer[13];
    if (tpid != VLAN_TPID) return 0;
    uint16_t tci = (buffer[14] << 8) | buffer[15];
    return tci & 0x0FFF;
}

/* 示例：从 VLAN 帧中提取 PCP */
uint8_t vlan_get_pcp(const uint8_t *buffer) {
    uint16_t tpid = (buffer[12] << 8) | buffer[13];
    if (tpid != VLAN_TPID) return 0;
    uint16_t tci = (buffer[14] << 8) | buffer[15];
    return (tci >> 13) & 0x07;
}
```

#### 动手练习

1. 用 Wireshark 观察一个带 VLAN Tag 的帧，对比插入前后的 hex 变化
2. 编写一个测试函数：创建一个普通的 ARP 帧，调用 `vlan_insert_tag()` 插入 VID=100 的 Tag，然后用 `vlan_remove_tag()` 移除
3. 思考：如果一个交换机 Access 端口收到一个已标记 (tagged) 的帧，应该怎么处理？
4. 挑战：实现 Q-in-Q (802.1ad) 的双层 Tag 插入

#### VLAN Tag 对帧长度的影响

| 项目 | 无 Tag | 有 Tag |
|------|--------|--------|
| 最小帧长度 | 64 字节 | 68 字节 |
| 最大帧长度 | 1518 字节 | 1522 字节 |
| MAC 头部 | 14 字节 | 18 字节 (含 4 字节 Tag) |
| 最大 Payload | 1500 字节 | 1500 字节 (EtherType 后移) |

注意：交换机的 Trunk 端口会在转发时保持 Tag 不变，而 Access 端口在发送到终端前移除 Tag。

---

### B.8 参考速查表 (Reference Tables)

#### 常用 EtherType

| 值 | 协议 | 说明 |
|----|------|------|
| `0x0800` | IPv4 | Internet Protocol version 4 |
| `0x0806` | ARP | Address Resolution Protocol |
| `0x8100` | 802.1Q | VLAN Tagged Frame |
| `0x86DD` | IPv6 | Internet Protocol version 6 |
| `0x8808` | MAC Control | 802.3x Pause Frame |
| `0x88CC` | LLDP | Link Layer Discovery Protocol |
| `0x888E` | EAPoL | 802.1X Authentication |
| `0x88A8` | 802.1ad | Q-in-Q (Service VLAN) |
| `0x88F7` | PTP | Precision Time Protocol (1588) |

#### 常用 UDP/TCP 端口

| 端口 | 协议 | 说明 |
|------|------|------|
| 7 | UDP/TCP | Echo (回显服务) |
| 20/21 | TCP | FTP (数据/控制) |
| 22 | TCP | SSH (安全 Shell) |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP (邮件发送) |
| 53 | UDP/TCP | DNS (域名解析) |
| 67/68 | UDP | DHCP (地址分配) |
| 80 | TCP | HTTP (网页) |
| 123 | UDP | NTP (时间同步) |
| 161/162 | UDP | SNMP (网络管理) |
| 443 | TCP | HTTPS (加密网页) |
| 502 | TCP | Modbus TCP (工业自动化) |
| 1900 | UDP | SSDP (UPnP 设备发现) |
| 5353 | UDP | mDNS (多播 DNS) |
| 5683 | UDP | CoAP (物联网) |

#### 特殊 MAC 地址

| MAC 地址 | 名称 | 说明 |
|----------|------|------|
| `FF-FF-FF-FF-FF-FF` | 广播 MAC | 发送到所有设备 (IP 广播时使用) |
| `01-80-C2-00-00-00` | STP 组播 | 802.1D 生成树 BPDU 的目的 MAC |
| `01-80-C2-00-00-01` | MAC Control | 802.3x Pause 帧的目的 MAC |
| `01-80-C2-00-00-02` | LACP | 链路聚合控制 (802.3ad) |
| `01-80-C2-00-00-03` | 802.1X | EAPoL 认证帧 |
| `01-80-C2-00-00-0E` | LLDP 组播 | LLDP 协议的目的 MAC |
| `01-00-5E-xx-xx-xx` | IPv4 组播 | 以太网多播 (映射自 IP 组播 224.0.0.0/4) |
| `33-33-xx-xx-xx-xx` | IPv6 组播 | IPv6 多层多播 MAC |

#### MAC 地址类型快速判断

```c
/* 判断是否为广播 MAC */
int is_broadcast_mac(const uint8_t mac[6]) {
    uint8_t bcast[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
    return memcmp(mac, bcast, 6) == 0;
}

/* 判断是否为单播 MAC (第 1 字节最低位 = 0) */
int is_unicast_mac(const uint8_t mac[6]) {
    return (mac[0] & 0x01) == 0;
}

/* 判断是否为组播 MAC (第 1 字节最低位 = 1) */
int is_multicast_mac(const uint8_t mac[6]) {
    return (mac[0] & 0x01) != 0;
}
```

---

### B.9 自测题 (Self-Test Questions)

**Q1：为什么以太网帧最小长度为 64 字节？**

A：这是为了支持 **CSMA/CD 碰撞检测** 机制。在 10Mbps 以太网中，最长传输距离约 2500 米，信号在电缆上的往返时间 (RTT) 约为 51.2us。在 51.2us 内 10Mbps 可以传输 512 位 = 64 字节。如果帧长度小于 64 字节，发送方可能在数据发送完毕后才检测到碰撞，导致无法识别碰撞。因此最小帧长度被设定为 64 字节（从目的 MAC 到 FCS 结束，不含 Preamble 和 SFD）。

**Q2：如果我看到一个 60 字节的以太网帧，这是怎么回事？**

A：如果上层协议数据不足 64 字节（如 ARP 请求仅 42 字节），MAC 层会自动添加 **Padding (填充字节)** 使帧达到最小长度。60 字节的帧实际上缺少 FCS（4 字节），有些抓包工具会剥离 FCS。完整的帧应该是 64 字节：14 字节头部 + 28 字节 ARP + 18 字节填充 + 4 字节 FCS。Wireshark 可能显示为 60 字节（剥离了 FCS）。

**Q3：如何将收到的 ARP 应答与之前发出的 ARP 请求匹配？**

A：ARP 应答不包含与请求匹配的显式标识符。匹配依据是 **发送方 IP (SPA)**：如果收到的 ARP 应答中，SPA 等于你之前发送的 ARP 请求中的 TPA (目标 IP)，则认为匹配。还有一种方法是通过 ARP 缓存机制：发送请求时记录目标 IP 和发出时间，收到应答后查找缓存表中 IP 相同的条目进行匹配。建议的代码逻辑：

```c
/* 收到 ARP 应答后 */
if (arp->oper == ntohs(2)) {  /* ARP Reply */
    uint32_t resp_ip = (arp->spa[0] << 24) | (arp->spa[1] << 16) |
                       (arp->spa[2] << 8) | arp->spa[3];
    if (resp_ip == my_pending_target_ip) {
        /* 匹配成功！这就是我们等待的 ARP 应答 */
        memcpy(peer_mac, arp->sha, 6);
    }
}
```

**Q4：为什么 TCP 建立连接需要三次握手 (3-Way Handshake)，而不是两次？**

A：三次握手的主要目的是 **防止历史失效的 SYN 段建立意外连接**。考虑以下场景：

1. 客户端发送 SYN (seq=100) 到服务器，但该 SYN 在网络中滞留
2. 客户端超时重发 SYN (seq=200)，到达服务器，服务器回复 SYN+ACK (seq=300, ack=201)
3. 客户端收到 SYN+ACK，回复 ACK (seq=201, ack=301)，正常建立连接，数据传输后关闭
4. **此时**，旧 SYN (seq=100) 到达服务器！服务器认为这是新的连接请求，回复 SYN+ACK (seq=400, ack=101)
5. 客户端收到这个 SYN+ACK，发现 ack=101 与自己期望的序号不匹配（或连接已关闭），回复 RST

如果只有两次握手，在步骤 4 中服务器会直接建立连接并等待数据，浪费资源。三次握手使服务器停留在 SYN-RCVD 状态，直到收到客户端的 ACK 确认，在此期间如果收到不匹配的 ACK 或 RST，可以安全地释放连接。

另外，三次握手还实现了 **ISN (初始序列号) 的交换和确认**，双方都能确认对方的接收和发送能力正常。

**Q5：VLAN Tag 对帧结构有什么具体影响？**

A：VLAN Tag 在帧中插入 4 个额外字节，带来以下影响：

1. **帧长度增加 4 字节**：原始帧从 64-1518 字节变为 68-1522 字节
2. **结构变化**：原始的 EtherType 位置被 Tag 占用，原 EtherType 向后移动 4 字节
3. **校验范围不变**：FCS 的 CRC32 校验覆盖包括 Tag 在内的整个帧
4. **MTU 不变**：L2 的 Payload 最大仍为 1500 字节（EtherType 和 Payload 整体后移）
5. **兼容性问题**：不支持 802.1Q 的传统设备可能把 `0x8100` 当作 LLC 长度字段处理，无法正确解析后续数据

对于 Trunk 端口：允许多个 VLAN 的 tagged 帧通过，最大帧为 1522 字节。
对于 Access 端口：通常只发送 untagged 帧，最大帧仍为 1518 字节。
对于支持 Jumbo Frame 的网络：最大帧可达 9000+ 字节（含 Tag）。

#### 综合思考题

**Q6**：从 PC ping MCU (ICMP Echo)，在 MCU 端收到的帧与发送的帧之间，有哪些字节发生了变化？请逐字节列出。

提示：答案应该涵盖 MAC 地址交换、IP 地址交换、TTL 减 1、ICMP Type 从 8 变为 0、校验和重算。

**Q7**：UDP 校验和是强制性的吗？如果 UDP 校验和设置为 0，接收方应该怎么处理？

A：在 IPv4 中，UDP 校验和是**可选的**（校验和为 0 表示发送方未计算校验和）。但在 IPv6 中，UDP 校验和是**强制性的**。接收方收到校验和为 0 的 UDP 数据报时，在 IPv4 网络中可以信任该数据并传递给上层，但为了健壮性，生产代码中应当总是计算正确的 UDP 校验和。

**Q8**：如果 ARP 缓存中没有目标 IP 的条目，但上层应用需要发送 UDP 数据，MCU 应该按照什么顺序处理？

A：正确流程：
1. 检查 ARP 缓存，未找到条目
2. 构造 ARP 请求并发送（广播）
3. 将 UDP 数据报暂存到等待队列
4. 设置 ARP 等待超时（通常 1 秒）
5. 收到 ARP 应答后，更新 ARP 缓存
6. 构造完整的以太网帧（插入正确的目的 MAC），发送暂存的 UDP 数据报
7. 如果超时未收到 ARP 应答，通知上层"目标不可达"并丢弃数据

---

*附录 B 结束 -- 接下来可以在实际的 MCU 开发板上运行这些实验，配合 Wireshark 抓包验证每个步骤的输出结果。*

---

## Appendix C: Network Protocol State Machine Deep Dive

> 网络协议状态机深度解析 -- 涵盖 ARP 缓存状态机、自动协商 (Auto-Neg) 状态机、TCP 简化状态机、PHY 链路检测状态机，每节包含文本图、C 代码实现、定时器与回调机制。

---

协议状态机是嵌入式网络协议栈的核心骨架。在裸机或 RTOS 环境下，每个协议层都维护自己的状态机，由主循环或定时器驱动状态迁移。本节深入剖析四个关键状态机的设计模式与实现细节。

---

### C.1 ARP 缓存状态机 (ARP Cache State Machine)

ARP 缓存状态机管理 IP 到 MAC 地址映射的整个生命周期，是网络协议栈中最基础也最常用的状态机。与简单哈希表不同，带状态机的 ARP 缓存能够处理"查询中"的异步等待场景。

#### C.1.1 状态图

```
                             +-----------+
                     +------->  IDLE     <---------+
                     |       +----+------+         |
                     |            |                |
                     |    (需要发送数据报,         (STALE 超时,
                     |     缓存中无该 IP 的 MAC)    条目过期)
                     |            |
                     |            v
                     |       +----+---------+
         (3 次重试   |       |  RESOLVING   | ------+
          超时无应答) |       +----+---------+       |
                     |            |                 |
                     |            | (收到 ARP 应答,  |
                     |            |  学习到 MAC)     |
                     |            v                 |
                     |       +----+---------+       |
                     +-------+  RESOLVED    |       |
                             +----+---------+       |
                                  |                 |
                                  | (STALE 超时,    |
                                  |  标记为陈旧)    |
                                  v                 |
                             +----+---------+       |
                             |   STALE      |       |
                             +--------------+       |
                                  |                 |
                                  +-----------------+
                                  (需要再次发送时,
                                   从 STALE 回到 RESOLVING)
```

**状态定义**：

| 状态 | 说明 | 行为 |
|------|------|------|
| `IDLE` | 空闲态，该 IP 无缓存条目 | 不动作 |
| `RESOLVING` | 解析中，已发送 ARP 请求，等待应答 | 缓存 ARP 请求重试计时器，可发送最多 3 次 ARP 请求 |
| `RESOLVED` | 已解析，MAC 已知，缓存有效 | 可正常转发数据 |
| `STALE` | 陈旧态，MAC 仍保留但标记为可能过时 | 下次发送数据前需重新解析 |

**状态转换条件**：

| 当前状态 | 下一状态 | 触发条件 |
|----------|---------|----------|
| `IDLE` | `RESOLVING` | 上层需要发送 IP 数据报，缓存中无对应条目 |
| `RESOLVING` | `RESOLVED` | 收到匹配的 ARP 应答，提取到对端 MAC |
| `RESOLVING` | `IDLE` | 重试 3 次后仍未收到 ARP 应答，超时放弃 |
| `RESOLVED` | `STALE` | 条目存活超时（如 300s），变为陈旧 |
| `STALE` | `RESOLVING` | 需要发送数据时检测到条目为 STALE |
| `STALE` | `IDLE` | 短超时（如 30s）未使用，完全清除 |

#### C.1.2 C 代码实现

```c
#include <stdint.h>
#include <string.h>
#include <stdbool.h>

/* ============================================================ */
/* ARP 缓存状态机 -- 完整实现                                  */
/* ============================================================ */

#define ETH_ALEN            6
#define ARP_CACHE_SIZE      16
#define ARP_MAX_RETRIES     3
#define ARP_RESOLVE_TIMEOUT 1000   /* 每次重试间隔 ms */
#define ARP_RESOLVED_TTL    300000 /* RESOLVED -> STALE, 300s */
#define ARP_STALE_TTL       30000  /* STALE -> IDLE, 30s */

/* ----- 状态定义 ----- */
typedef enum {
    ARP_STATE_IDLE,       /* 无条目 */
    ARP_STATE_RESOLVING,  /* ARP 请求已发送，等待应答 */
    ARP_STATE_RESOLVED,   /* MAC 已知，缓存有效 */
    ARP_STATE_STALE,      /* 缓存陈旧，再次使用前需解析 */
} arp_state_t;

/* ----- 事件枚举 (状态机的输入) ----- */
typedef enum {
    ARP_EVENT_NEED_SEND,    /* 上层需要发送数据 */
    ARP_EVENT_REPLY_RCVD,   /* 收到 ARP 应答 */
    ARP_EVENT_TIMEOUT,      /* 计时器超时 */
    ARP_EVENT_AGING_TICK,   /* 老化滴答 */
} arp_event_t;

/* ----- 状态上下文结构 ----- */
typedef struct {
    uint32_t    ip_addr;           /* IP 地址 (网络字节序) */
    uint8_t     mac_addr[ETH_ALEN]; /* MAC 地址 (仅 RESOLVED 有效) */
    arp_state_t state;             /* 当前状态 */
    uint8_t     retry_count;       /* 重试计数 */
    uint32_t    state_timer_ms;    /* 当前状态停留时间 (ms) */
    uint32_t    entry_timer_ms;    /* 条目生存时间 (ms) */
    uint8_t     pending_tx;        /* 是否有待发送的数据等待解析完成 */
    bool        valid;             /* 条目是否有效 */
} arp_cache_entry_t;

/* ----- 完整 ARP 缓存表 ----- */
typedef struct {
    arp_cache_entry_t entries[ARP_CACHE_SIZE];
    uint32_t          tick_ms;  /* 系统 ms 时钟 */
} arp_cache_table_t;

/* ----- 动作回调函数指针类型 ----- */
typedef void (*arp_on_resolved_t)(uint32_t ip, const uint8_t mac[ETH_ALEN]);
typedef void (*arp_on_timeout_t)(uint32_t ip);
typedef void (*arp_send_request_t)(uint32_t target_ip);

/* ----- 状态机全局回调 ----- */
static arp_on_resolved_t  g_arp_on_resolved  = NULL;
typedef void (*arp_on_resolved_t)(uint32_t ip, const uint8_t mac[ETH_ALEN]);

/* ============================================================ */
/* 状态机核心: 单条目的状态机处理                               */
/* ============================================================ */

/**
 * ARP 条目状态机处理函数
 *
 * 对每个条目独立驱动状态机，接收事件后完成状态迁移，
 * 并在状态进入/退出时执行对应的动作回调。
 *
 * entry:     条目指针
 * event:     输入事件
 * ext_data:  事件附加数据 (如 ARP 应答中的 MAC 地址)
 * send_req:  发送 ARP 请求的函数回调
 * on_resolved: 解析完成回调
 * on_timeout:  超时放弃回调
 */
static void arp_entry_state_machine(
    arp_cache_entry_t *entry,
    arp_event_t event,
    const void *ext_data,
    arp_send_request_t send_req,
    arp_on_resolved_t on_resolved,
    arp_on_timeout_t on_timeout)
{
    arp_state_t old_state = entry->state;

    switch (entry->state) {

    /* ---------- IDLE 状态 ---------- */
    case ARP_STATE_IDLE:
        switch (event) {
        case ARP_EVENT_NEED_SEND:
            /* 状态进入: 重置重试计数，清空 MAC，进入 RESOLVING */
            entry->retry_count    = 0;
            entry->pending_tx     = 1;
            entry->state_timer_ms = 0;
            memset(entry->mac_addr, 0, ETH_ALEN);
            entry->state = ARP_STATE_RESOLVING;
            /* 动作: 发送第一个 ARP 请求 */
            if (send_req) send_req(entry->ip_addr);
            break;
        default:
            break;
        }
        break;

    /* ---------- RESOLVING 状态 ---------- */
    case ARP_STATE_RESOLVING:
        switch (event) {
        case ARP_EVENT_REPLY_RCVD:
            /* 状态进入: 保存 MAC，标记待发送数据可发送 */
            if (ext_data) {
                memcpy(entry->mac_addr, (const uint8_t *)ext_data, ETH_ALEN);
            }
            entry->pending_tx     = 0;
            entry->entry_timer_ms = 0;
            entry->state = ARP_STATE_RESOLVED;
            /* 退出动作: 通知上层解析完成 */
            if (on_resolved) on_resolved(entry->ip_addr, entry->mac_addr);
            break;

        case ARP_EVENT_TIMEOUT:
            entry->retry_count++;
            if (entry->retry_count < ARP_MAX_RETRIES) {
                /* 重试: 重新发送 ARP 请求 */
                entry->state_timer_ms = 0;
                if (send_req) send_req(entry->ip_addr);
            } else {
                /* 超过最大重试次数, 放弃 */
                entry->pending_tx  = 0;
                entry->valid       = 0;
                entry->state       = ARP_STATE_IDLE;
                entry->state_timer_ms = 0;
                /* 退出动作: 通知上层目标不可达 */
                if (on_timeout) on_timeout(entry->ip_addr);
            }
            break;
        default:
            break;
        }
        break;

    /* ---------- RESOLVED 状态 ---------- */
    case ARP_STATE_RESOLVED:
        switch (event) {
        case ARP_EVENT_AGING_TICK:
            /* 累积生存时间, 超时后进入 STALE */
            if (entry->entry_timer_ms >= ARP_RESOLVED_TTL) {
                entry->state = ARP_STATE_STALE;
                entry->state_timer_ms = 0;
            }
            break;

        case ARP_EVENT_REPLY_RCVD:
            /* 收到重复/更新的 ARP 应答, 刷新 MAC 和计时器 */
            if (ext_data) {
                memcpy(entry->mac_addr, (const uint8_t *)ext_data, ETH_ALEN);
            }
            entry->entry_timer_ms = 0;  /* 刷新老化计时器 */
            break;
        default:
            break;
        }
        break;

    /* ---------- STALE 状态 ---------- */
    case ARP_STATE_STALE:
        switch (event) {
        case ARP_EVENT_NEED_SEND:
            /* 需要发送数据, 从 STALE 回退到 RESOLVING */
            entry->retry_count    = 0;
            entry->pending_tx     = 1;
            entry->state_timer_ms = 0;
            entry->state = ARP_STATE_RESOLVING;
            if (send_req) send_req(entry->ip_addr);
            break;

        case ARP_EVENT_AGING_TICK:
            /* STALE 短暂存活后完全清除 */
            if (entry->entry_timer_ms >= ARP_STALE_TTL) {
                entry->valid = 0;
                entry->state = ARP_STATE_IDLE;
                entry->state_timer_ms = 0;
            }
            break;

        case ARP_EVENT_REPLY_RCVD:
            /* 收到 ARP 应答 (对端也在通信), 回到 RESOLVED */
            if (ext_data) {
                memcpy(entry->mac_addr, (const uint8_t *)ext_data, ETH_ALEN);
            }
            entry->entry_timer_ms = 0;
            entry->state = ARP_STATE_RESOLVED;
            break;
        default:
            break;
        }
        break;
    }
}

/* ============================================================ */
/* 定时器: 每个条目独立计时                                    */
/* ============================================================ */

/**
 * ARP 条目标记器更新 -- 每个条目独立计时
 * 在主循环中周期性调用, 更新各条目计时器并驱动超时事件
 *
 * tbl:     ARP 缓存表
 * elapsed_ms: 自上次调用以来的毫秒数
 * send_req:   发送 ARP 请求的回调
 * on_resolved: 解析完成回调
 * on_timeout:  超时放弃回调
 */
void arp_cache_timer_tick(arp_cache_table_t *tbl, uint32_t elapsed_ms,
                          arp_send_request_t send_req,
                          arp_on_resolved_t on_resolved,
                          arp_on_timeout_t   on_timeout)
{
    tbl->tick_ms += elapsed_ms;

    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (!tbl->entries[i].valid) continue;

        arp_cache_entry_t *entry = &tbl->entries[i];

        /* 累计状态计时器 (用于 RESOLVING 超时检测) */
        entry->state_timer_ms += elapsed_ms;

        /* 累计条目生存计时器 (用于 RESOLVED/STALE 老化) */
        entry->entry_timer_ms += elapsed_ms;

        /* 触发超时事件 */
        if (entry->state == ARP_STATE_RESOLVING) {
            /* RESOLVING 超时: 检查是否需要触发重试 */
            if (entry->state_timer_ms >= ARP_RESOLVE_TIMEOUT) {
                arp_entry_state_machine(entry, ARP_EVENT_TIMEOUT, NULL,
                                        send_req, on_resolved, on_timeout);
            }
        } else if (entry->state == ARP_STATE_RESOLVED) {
            /* RESOLVED 老化 */
            arp_entry_state_machine(entry, ARP_EVENT_AGING_TICK, NULL,
                                    send_req, on_resolved, on_timeout);
        } else if (entry->state == ARP_STATE_STALE) {
            /* STALE 老化 */
            arp_entry_state_machine(entry, ARP_EVENT_AGING_TICK, NULL,
                                    send_req, on_resolved, on_timeout);
        }
    }
}

/* ============================================================ */
/* 对外 API: 上层协议栈调用接口                                */
/* ============================================================ */

/**
 * 查找或创建条目, 触发 ARP 解析
 * 由 IP 层在发送数据报前调用
 *
 * 返回: true = MAC 已知, 可直接发送;
 *       false = 解析中, 需等待回调后发送
 */
bool arp_cache_resolve(arp_cache_table_t *tbl, uint32_t target_ip,
                       uint8_t out_mac[ETH_ALEN],
                       arp_send_request_t send_req,
                       arp_on_resolved_t on_resolved,
                       arp_on_timeout_t   on_timeout)
{
    /* 查找是否已有条目 */
    arp_cache_entry_t *entry = NULL;
    int empty_slot = -1;

    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (!tbl->entries[i].valid) {
            if (empty_slot < 0) empty_slot = i;
            continue;
        }
        if (tbl->entries[i].ip_addr == target_ip) {
            entry = &tbl->entries[i];
            break;
        }
    }

    /* 未找到, 创建新条目 */
    if (entry == NULL) {
        if (empty_slot < 0) return false; /* 缓存满 */
        entry = &tbl->entries[empty_slot];
        entry->ip_addr  = target_ip;
        entry->valid    = true;
        entry->state    = ARP_STATE_IDLE;
        entry->state_timer_ms = 0;
        entry->entry_timer_ms = 0;
        entry->retry_count    = 0;
        entry->pending_tx     = 0;
        memset(entry->mac_addr, 0, ETH_ALEN);
    }

    /* 如果已 RESOLVED, 直接返回 MAC */
    if (entry->state == ARP_STATE_RESOLVED) {
        memcpy(out_mac, entry->mac_addr, ETH_ALEN);
        return true;
    }

    /* 否则投递 NEED_SEND 事件, 触发状态机 */
    arp_entry_state_machine(entry, ARP_EVENT_NEED_SEND, NULL,
                            send_req, on_resolved, on_timeout);
    return false; /* 返回 false, 上层需等待回调 */
}

/**
 * 处理收到的 ARP 应答 -- 由网络接收路径调用
 */
void arp_cache_handle_reply(arp_cache_table_t *tbl,
                            uint32_t sender_ip,
                            const uint8_t sender_mac[ETH_ALEN],
                            arp_send_request_t send_req,
                            arp_on_resolved_t on_resolved,
                            arp_on_timeout_t   on_timeout)
{
    /* 查找匹配条目 */
    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (!tbl->entries[i].valid) continue;
        if (tbl->entries[i].ip_addr == sender_ip) {
            arp_entry_state_machine(&tbl->entries[i],
                                    ARP_EVENT_REPLY_RCVD,
                                    sender_mac,
                                    send_req, on_resolved, on_timeout);
            return;
        }
    }

    /* 未找到条目但收到应答: 学习新条目 (Gratuitous ARP 学习) */
    int empty_slot = -1;
    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (!tbl->entries[i].valid) { empty_slot = i; break; }
    }
    if (empty_slot >= 0) {
        arp_cache_entry_t *entry = &tbl->entries[empty_slot];
        entry->valid    = true;
        entry->ip_addr  = sender_ip;
        memcpy(entry->mac_addr, sender_mac, ETH_ALEN);
        entry->state    = ARP_STATE_RESOLVED;
        entry->state_timer_ms = 0;
        entry->entry_timer_ms = 0;
        entry->retry_count    = 0;
        entry->pending_tx     = 0;
    }
}

/**
 * 初始化 ARP 缓存表
 */
void arp_cache_init(arp_cache_table_t *tbl) {
    memset(tbl, 0, sizeof(arp_cache_table_t));
}
```

#### C.1.3 状态机设计要点

**何时引入 STALE 状态**：很多简单实现只有 RESOLVED -> IDLE 直接超时清除。引入 STALE 的好处是：如果对端只是短暂离线后恢复，ARP 请求不会立即广播（避免不必要的网络流量），但如果长期未使用则完全清除条目。

**pending_tx 标记**：当状态机处于 RESOLVING 时，上层可能需要发送的数据必须暂存。这个标记位通知协议栈"数据已暂存，等待 MAC 就绪后发送"。

**Gratuitous ARP 学习**：收到非请求的 ARP 应答（Gratuitous ARP）时，即使之前的缓存条目处于 IDLE 或不存在，也可以直接学习到 MAC 地址并进入 RESOLVED 状态，这是 ARP 协议中"免请求学习"机制的体现。

---

### C.2 Auto-Negotiation 状态机 (Auto-Neg State Machine)

自动协商状态机位于 PHY 内部，但 MCU 端需要监控和管理这个状态机。与 PHY 驱动状态机（第九章）不同，这里聚焦于自动协商协议本身的内部状态流转。

#### C.2.1 状态图

```
                      +----------------+
                      |   DISABLED     |
                      +-------+--------+
                              |
                     (使能 Auto-Neg,
                      写入 BMCR.12=1)
                              |
                              v
                      +-------+--------+
               +----->   ENABLED       |
               |     +-------+--------+
               |             |
               |     (触发重新协商,
               |      BMCR.9=1)
               |             |
               |             v
               |     +-------+--------+
               |     |  START_ANEG    | <-- 发送 FLP Burst
               |     +-------+--------+
               |             |
               |      (3 次一致 FLP
               |       Burst 接收完成)
               |             |
               |             v
               |     +-------+--------+
               |     | ANEG_WAIT      | <-- 等待 ANEG Complete
               |     | (BMSR.5 检测)   |     (BMSR.5=1)
               |     +-------+--------+
               |             |
               |      (协商完成)
               |             |
               |             v
               |     +-------+--------+
               |     | ANEG_COMPLETE  | <-- 读取 ANLPAR, 裁决 HCD
               |     +-------+--------+
               |             |
               |     (链路建立,
               |      BMSR.2=1)
               |             |
               |             v
               |     +-------+--------+
               |     |   LINK_UP      | <-- 正常运行
               |     +-------+--------+
               |             |
               |      (链路断开,
               |       BMSR.2=0)
               |             |
               |             v
               |     +-------+--------+
               |     |  LINK_DOWN     | <-- 链路断开处理
               |     +-------+--------+
               |             |
               |      (尝试恢复,
               |       重试 <= N)
               |             |
               +-------------+
               (重试 > N, 进入错误处理)
```

#### C.2.2 C 代码实现 (MCU 侧监控 PHY)

```c
#include <stdint.h>
#include <stdbool.h>

/* ============================================================ */
/* Auto-Neg 状态机 -- MCU 侧监控 PHY                            */
/* ============================================================ */

/* PHY 寄存器地址 */
#define PHY_REG_BMCR   0x00
#define PHY_REG_BMSR   0x01
#define PHY_REG_ANAR   0x04
#define PHY_REG_ANLPAR 0x05

/* BMCR 位 */
#define BMCR_RESET         0x8000
#define BMCR_AN_EN         0x1000
#define BMCR_RESTART_ANEG  0x0200

/* BMSR 位 */
#define BMSR_AN_COMP      0x0020
#define BMSR_LINK_STATUS  0x0004

/* ----- 状态定义 ----- */
typedef enum {
    ANEG_DISABLED,
    ANEG_ENABLED,
    ANEG_START,
    ANEG_WAIT,          /* 等待 ANEG 完成 (BMSR.5) */
    ANEG_COMPLETE,      /* 协商完成,读取结果 */
    ANEG_LINK_UP,       /* 链路已建立 */
    ANEG_LINK_DOWN,     /* 链路断开 */
    ANEG_ERROR,         /* 错误状态 */
} aneg_state_t;

/* ----- 协商结果 ----- */
typedef struct {
    uint8_t  speed;       /* 10, 100, 1000 */
    uint8_t  duplex;      /* 0=half, 1=full */
    bool     pause_sym;   /* 对称流控 */
    bool     pause_asym;  /* 非对称流控 */
} aneg_result_t;

/* ----- 状态机上下文 ----- */
typedef struct {
    aneg_state_t state;           /* 当前状态 */
    uint8_t      phy_addr;        /* PHY 地址 */
    uint8_t      retry_count;     /* 重试次数 */
    uint32_t     timer_ms;        /* 状态计时器 */
    uint32_t     wait_timeout_ms; /* 等待超时配置 */
    aneg_result_t result;         /* 协商结果 */

    /* PHY 寄存器读写回调 (由硬件抽象层提供) */
    uint16_t (*read_reg)(uint8_t phy_addr, uint8_t reg);
    void     (*write_reg)(uint8_t phy_addr, uint8_t reg, uint16_t val);

    /* 动作回调 */
    void (*on_link_up)(const aneg_result_t *result);
    void (*on_link_down)(void);
} aneg_context_t;

/* ----- 能力优先级裁决 ----- */
#define AN_10BT_HD  0x0020
#define AN_10BT_FD  0x0040
#define AN_100TX_HD 0x0080
#define AN_100TX_FD 0x0100
#define AN_PAUSE    0x0400
#define AN_ASM_DIR  0x0800

static aneg_result_t aneg_resolve_hcd(uint16_t local, uint16_t partner) {
    aneg_result_t res = {0, 0, false, false};
    uint16_t common = local & partner;

    if (common & AN_100TX_FD) { res.speed = 100; res.duplex = 1; }
    else if (common & AN_100TX_HD) { res.speed = 100; res.duplex = 0; }
    else if (common & AN_10BT_FD)  { res.speed = 10;  res.duplex = 1; }
    else if (common & AN_10BT_HD)  { res.speed = 10;  res.duplex = 0; }

    /* Pause 流控裁决 (802.3 Table 28B-2) */
    uint8_t loc_pause = ((local  >> 10) & 0x03);
    uint8_t rmt_pause = ((partner >> 10) & 0x03);
    if ((loc_pause & 0x01) && (rmt_pause & 0x01)) res.pause_sym = true;
    else if ((loc_pause & 0x02) && (rmt_pause & 0x01)) res.pause_asym = true;

    return res;
}

/* ============================================================ */
/* 状态机主循环                                                 */
/* ============================================================ */

/**
 * Auto-Neg 状态机主函数 -- 每次读取 PHY 寄存器后驱动一次状态迁移
 * 建议在主循环中每 10-50ms 调用一次
 *
 * 返回: 当前状态, 用于调试打印
 */
aneg_state_t aneg_state_machine_run(aneg_context_t *ctx) {
    uint16_t bmcr, bmsr, anlpar;
    aneg_result_t new_result;

    switch (ctx->state) {

    /* ======== DISABLED: 使能 Auto-Neg ======== */
    case ANEG_DISABLED:
        /* 状态入口动作: 写入 BMCR 使能 Auto-Neg */
        ctx->write_reg(ctx->phy_addr, PHY_REG_BMCR, BMCR_AN_EN);
        ctx->timer_ms = 0;
        ctx->retry_count = 0;
        ctx->state = ANEG_ENABLED;
        break;

    /* ======== ENABLED: 触发重新协商 ======== */
    case ANEG_ENABLED:
        /* 状态入口动作: 设置 ANAR + 触发重新协商 */
        bmcr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMCR);
        bmcr |= BMCR_AN_EN | BMCR_RESTART_ANEG;
        ctx->write_reg(ctx->phy_addr, PHY_REG_BMCR, bmcr);
        ctx->timer_ms = 0;
        ctx->state = ANEG_START;
        break;

    /* ======== START: 发起 FLP Burst ======== */
    case ANEG_START:
        ctx->timer_ms += 50; /* 假设每次调用间隔 50ms */

        /* 读取 BMSR 检查 ANEG 是否完成 */
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);

        /* 第二次读取: 清除锁存 + 获取真实状态 */
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);

        if (bmsr & BMSR_AN_COMP) {
            /* 自动协商完成, 进入 COMPLETE 读取结果 */
            ctx->state = ANEG_COMPLETE;
        }

        /* 超时检测: ANEG 应在 3 秒内完成 */
        if (ctx->timer_ms >= ctx->wait_timeout_ms) {
            ctx->retry_count++;
            if (ctx->retry_count < 3) {
                ctx->state = ANEG_ENABLED;  /* 重试 */
            } else {
                /* 并行检测: 尝试根据 LINK STATUS 强制模式 */
                if (bmsr & BMSR_LINK_STATUS) {
                    /* 有链路但 ANEG 未完成, 可能是并行检测 */
                    ctx->result.speed  = 100;  /* 假设 100M */
                    ctx->result.duplex = 0;    /* 假设半双工 */
                    ctx->state = ANEG_LINK_UP;
                    if (ctx->on_link_up) ctx->on_link_up(&ctx->result);
                } else {
                    ctx->state = ANEG_ERROR;
                }
            }
            ctx->timer_ms = 0;
        }
        break;

    /* ======== WAIT: 等 ANEG Complete (备选路径) ======== */
    case ANEG_WAIT:
        ctx->timer_ms += 50;
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR); /* 清除锁存 */

        if (bmsr & BMSR_AN_COMP) {
            ctx->state = ANEG_COMPLETE;
        }
        if (ctx->timer_ms >= ctx->wait_timeout_ms) {
            ctx->state = ANEG_LINK_DOWN;
        }
        break;

    /* ======== COMPLETE: 读取协商结果 ======== */
    case ANEG_COMPLETE:
        /* 读取本端和对端能力寄存器 */
        anlpar = ctx->read_reg(ctx->phy_addr, PHY_REG_ANLPAR);
        uint16_t anar = ctx->read_reg(ctx->phy_addr, PHY_REG_ANAR);

        /* 裁决最高公共能力 */
        new_result = aneg_resolve_hcd(anar, anlpar);
        ctx->result = new_result;

        /* 检查链路是否真正建立 */
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);
        if (bmsr & BMSR_LINK_STATUS) {
            ctx->state = ANEG_LINK_UP;
            /* 通知上层: 链路已建立, 速率为 xxx */
            if (ctx->on_link_up) ctx->on_link_up(&ctx->result);
        } else {
            ctx->state = ANEG_LINK_DOWN;
        }
        break;

    /* ======== LINK_UP: 正常运行态, 监控链路状态 ======== */
    case ANEG_LINK_UP:
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);
        bmsr = ctx->read_reg(ctx->phy_addr, PHY_REG_BMSR);
        if (!(bmsr & BMSR_LINK_STATUS)) {
            ctx->state = ANEG_LINK_DOWN;
            if (ctx->on_link_down) ctx->on_link_down();
        }
        break;

    /* ======== LINK_DOWN: 链路断开处理 ======== */
    case ANEG_LINK_DOWN:
        ctx->retry_count++;
        if (ctx->retry_count <= 5) {
            /* 重新尝试自动协商 */
            ctx->state = ANEG_ENABLED;
        } else {
            ctx->state = ANEG_ERROR;
        }
        break;

    /* ======== ERROR: 错误处理 ======== */
    case ANEG_ERROR:
        /* 错误状态, 10 秒后自动重试 */
        ctx->timer_ms += 50;
        if (ctx->timer_ms >= 10000) {
            ctx->timer_ms = 0;
            ctx->retry_count = 0;
            ctx->state = ANEG_DISABLED;
        }
        break;
    }

    return ctx->state;
}

/**
 * 初始化 Auto-Neg 状态机上下文
 */
void aneg_context_init(aneg_context_t *ctx, uint8_t phy_addr,
                       uint16_t (*read_reg)(uint8_t, uint8_t),
                       void (*write_reg)(uint8_t, uint8_t, uint16_t),
                       void (*on_link_up)(const aneg_result_t *),
                       void (*on_link_down)(void))
{
    ctx->state           = ANEG_DISABLED;
    ctx->phy_addr        = phy_addr;
    ctx->retry_count     = 0;
    ctx->timer_ms        = 0;
    ctx->wait_timeout_ms = 3000;  /* 默认 3 秒超时 */
    ctx->read_reg        = read_reg;
    ctx->write_reg       = write_reg;
    ctx->on_link_up      = on_link_up;
    ctx->on_link_down    = on_link_down;
}
```

#### C.2.3 与第九章 PHY 驱动状态机的关系

第九章的 PHY 驱动状态机（`phy_state_machine_tick`）是上层管理状态机，涵盖了从电源复位到链路建立的完整 PHY 生命周期（11 个状态）。本节的 Auto-Neg 状态机则聚焦于自动协商子过程本身（7 个状态）。

两者的关系：
- PHY 驱动状态机作为总控制器，在 `PHY_STATE_WAIT_ANEG` 子状态中调用本节的 ANEG 状态机
- ANEG 状态机专注于 FLP Burst 交换、ACK 确认、HCD 裁决等 ANEG 特有逻辑
- ANEG 完成后的结果（speed/duplex/pause）向上传递给 PHY 驱动状态机，最终通知 MAC 层

---

### C.3 TCP 连接状态机 (简化版, 面向 MCU 服务器)

完整的 TCP 状态机有 11 个状态（RFC 793）。在 MCU 服务器场景中，通常只涉及 CLOSED、LISTEN、SYN_RCVD、ESTABLISHED、FIN_WAIT_1、FIN_WAIT_2、CLOSE_WAIT、LAST_ACK、TIME_WAIT 中的子集。本节提供面向嵌入式服务器的简化 6 状态实现。

#### C.3.1 状态图 (MCU 服务器视角)

```
                           +--------------------+
                           |      CLOSED        |
                           +--------+-----------+
                                    |
                            (调用 tcp_bind + tcp_listen)
                                    |
                                    v
                           +--------+-----------+
                    +------>      LISTEN         |
                    |      +--------+-----------+
                    |               |
                    |       (收到 SYN 报文,
                    |        发送 SYN+ACK)
                    |               |
                    |               v
                    |      +--------+-----------+
                    |      |     SYN_RCVD        |
                    |      +--------+-----------+
                    |               |
                    |       (收到 ACK, 三次握手完成)
                    |               |
                    |               v
                    |      +--------+-----------+
                    |      |   ESTABLISHED      | <-- 正常运行,收发数据
                    |      +--------+-----------+
                    |               |
                    |       (收到 FIN,           (主动发送 FIN,
                    |        发送 ACK)           收到 ACK)
                    |               |               |
                    |               v               v
                    |      +--------+--------+   +--+-----------+
                    |      |   CLOSE_WAIT    |   | FIN_WAIT_1   |
                    |      +--------+--------+   +--+-----------+
                    |               |               |
                    |       (发送 FIN)        (收到 ACK)
                    |               |               |
                    |               v               v
                    |      +--------+--------+   +--+-----------+
                    |      |    LAST_ACK     |   | FIN_WAIT_2   |
                    |      +--------+--------+   +--+-----------+
                    |               |               |
                    |       (收到 ACK,         (收到 FIN,
                    |        连接关闭)          发送 ACK)
                    |               |               |
                    |               +-------+-------+
                    |                       |
                    |                       v
                    |               +-------+--------+
                    |               |   TIME_WAIT    | (2MSL 等待)
                    |               +-------+--------+
                    |                       |
                    +-----------------------+
                                    |
                            (超时或异常)
                                    v
                           +--------+-----------+
                           |      CLOSED        |
                           +--------------------+
```

#### C.3.2 C 代码实现 (简化 TCP 状态处理器)

```c
#include <stdint.h>
#include <string.h>
#include <stdbool.h>

/* ============================================================ */
/* 简化 TCP 连接状态机 -- 嵌入式服务器端                        */
/* ============================================================ */

/* TCP 标志位 */
#define TCP_FLAG_FIN  0x01
#define TCP_FLAG_SYN  0x02
#define TCP_FLAG_RST  0x04
#define TCP_FLAG_PSH  0x08
#define TCP_FLAG_ACK  0x10

/* ----- TCP 状态 (服务器相关子集) ----- */
typedef enum {
    TCP_CLOSED,
    TCP_LISTEN,
    TCP_SYN_RCVD,
    TCP_ESTABLISHED,
    TCP_CLOSE_WAIT,
    TCP_LAST_ACK,
    TCP_FIN_WAIT_1,
    TCP_FIN_WAIT_2,
    TCP_TIME_WAIT,
} tcp_state_t;

/* ----- TCP 事件 (来自报文解析 + 上层调用) ----- */
typedef enum {
    TCP_EVENT_RCV_SYN,       /* 收到 SYN */
    TCP_EVENT_RCV_SYNACK,    /* 收到 SYN+ACK (客户端模式) */
    TCP_EVENT_RCV_ACK,       /* 收到 ACK */
    TCP_EVENT_RCV_FIN,       /* 收到 FIN */
    TCP_EVENT_RCV_RST,       /* 收到 RST */
    TCP_EVENT_APP_SEND,      /* 上层请求发送数据 */
    TCP_EVENT_APP_CLOSE,     /* 上层请求关闭 */
    TCP_EVENT_TIMEOUT,       /* 计时器超时 */
} tcp_event_t;

/* ----- TCP 连接控制块 (简化) ----- */
typedef struct tcp_sock {
    tcp_state_t state;           /* 当前状态 */

    /* 序列号空间 */
    uint32_t snd_nxt;            /* 下一个待发送的序列号 */
    uint32_t rcv_nxt;            /* 下一个期望接收的序列号 */
    uint16_t local_port;         /* 本地端口 */
    uint16_t remote_port;        /* 远端端口 */
    uint8_t  local_mac[6];       /* 本机 MAC */
    uint8_t  remote_mac[6];      /* 远端 MAC */
    uint32_t local_ip;           /* 本机 IP (网络字节序) */
    uint32_t remote_ip;          /* 远端 IP (网络字节序) */

    /* 计时器 */
    uint32_t timewait_timer_ms;  /* TIME_WAIT 计时器 */

    /* 发送/接收缓冲区 (简化) */
    uint8_t  *rcv_buf;
    uint16_t  rcv_len;
    uint16_t  rcv_capacity;
    uint8_t  *snd_buf;
    uint16_t  snd_len;
    uint16_t  snd_capacity;

    /* 回调函数 */
    void (*on_connected)(struct tcp_sock *sock);
    void (*on_data)(struct tcp_sock *sock, const uint8_t *data, uint16_t len);
    void (*on_closed)(struct tcp_sock *sock);
    void (*on_error)(struct tcp_sock *sock);

    /* 报文发送回调 (由底层 MAC 层提供) */
    void (*send_frame)(const uint8_t *frame, uint16_t len);
} tcp_sock_t;

/* ============================================================ */
/* TCP 状态机核心处理函数                                       */
/* ============================================================ */

/**
 * TCP 状态机 -- 处理单个事件
 *
 * sock:    TCP 连接控制块
 * event:   输入事件
 * flags:   收到的 TCP 标志位 (仅报文事件有效)
 * ext_data: 事件附加数据 (如收到报文的序列号)
 *
 * 返回: true 表示事件被处理, false 表示事件在当前状态下非法
 */
bool tcp_state_machine_handle(tcp_sock_t *sock, tcp_event_t event,
                              uint8_t flags, uint32_t ext_seq)
{
    /* 记录旧状态用于调试和退出动作 */
    tcp_state_t old_state = sock->state;

    switch (sock->state) {

    /* ======== CLOSED ======== */
    case TCP_CLOSED:
        switch (event) {
        case TCP_EVENT_RCV_SYN:
            /* 收到 SYN: 建立被动连接 */
            sock->rcv_nxt = ext_seq + 1;   /* 期望下一个是 seq+1 */
            sock->snd_nxt = 1000;           /* 本端 ISN (实际应随机) */
            sock->state = TCP_SYN_RCVD;
            /* 动作: 发送 SYN+ACK */
            /* tcp_send_segment(sock, TCP_FLAG_SYN | TCP_FLAG_ACK); */
            return true;

        case TCP_EVENT_APP_CLOSE:
            /* 已经在 CLOSED, 无需动作 */
            return true;

        default:
            return false;  /* 非法事件 */
        }
        break;

    /* ======== LISTEN ======== */
    case TCP_LISTEN:
        switch (event) {
        case TCP_EVENT_RCV_SYN:
            /* 收到 SYN: 进入 SYN_RCVD, 回复 SYN+ACK */
            sock->rcv_nxt = ext_seq + 1;
            sock->snd_nxt = 1000;  /* ISN */
            sock->state = TCP_SYN_RCVD;
            /* 动作: 发送 SYN+ACK (seq=snd_nxt, ack=rcv_nxt) */
            /* tcp_send_segment(sock, TCP_FLAG_SYN | TCP_FLAG_ACK); */
            return true;

        case TCP_EVENT_RCV_RST:
            /* 收到 RST: 忽略, 保持监听 */
            return true;

        default:
            return false;
        }
        break;

    /* ======== SYN_RCVD ======== */
    case TCP_SYN_RCVD:
        switch (event) {
        case TCP_EVENT_RCV_ACK:
            /* 收到 ACK: 三次握手完成, 进入 ESTABLISHED */
            sock->snd_nxt = ext_seq;  /* 更新本端序列号 */
            sock->state = TCP_ESTABLISHED;
            /* 入口动作: 调用 on_connected 回调 */
            if (sock->on_connected) sock->on_connected(sock);
            return true;

        case TCP_EVENT_RCV_RST:
            /* 收到 RST: 回到 LISTEN */
            sock->state = TCP_LISTEN;
            return true;

        case TCP_EVENT_TIMEOUT:
            /* SYN+ACK 重传超时: 回到 CLOSED */
            sock->state = TCP_CLOSED;
            if (sock->on_error) sock->on_error(sock);
            return true;

        default:
            return false;
        }
        break;

    /* ======== ESTABLISHED ======== */
    case TCP_ESTABLISHED:
        switch (event) {
        case TCP_EVENT_RCV_ACK:
            /* 收到 ACK: 更新发送序列号 */
            sock->snd_nxt = ext_seq;
            return true;

        case TCP_EVENT_RCV_FIN:
            /* 收到 FIN: 回复 ACK, 进入 CLOSE_WAIT */
            sock->rcv_nxt = ext_seq + 1;
            sock->state = TCP_CLOSE_WAIT;
            /* 动作: 发送 ACK */
            /* tcp_send_segment(sock, TCP_FLAG_ACK); */
            return true;

        case TCP_EVENT_APP_CLOSE:
            /* 上层主动关闭: 发送 FIN, 进入 FIN_WAIT_1 */
            sock->state = TCP_FIN_WAIT_1;
            /* sock->snd_nxt = ...; */
            /* 动作: 发送 FIN (seq=snd_nxt) */
            /* tcp_send_segment(sock, TCP_FLAG_FIN | TCP_FLAG_ACK); */
            return true;

        case TCP_EVENT_RCV_RST:
            /* 收到 RST: 直接关闭 */
            sock->state = TCP_CLOSED;
            if (sock->on_error) sock->on_error(sock);
            if (sock->on_closed) sock->on_closed(sock);
            return true;

        default:
            return false;
        }
        break;

    /* ======== CLOSE_WAIT ======== */
    case TCP_CLOSE_WAIT:
        switch (event) {
        case TCP_EVENT_APP_CLOSE:
            /* 上层处理完数据后关闭: 发送 FIN, 进入 LAST_ACK */
            sock->state = TCP_LAST_ACK;
            /* 动作: 发送 FIN */
            /* tcp_send_segment(sock, TCP_FLAG_FIN | TCP_FLAG_ACK); */
            return true;

        default:
            return false;
        }
        break;

    /* ======== LAST_ACK ======== */
    case TCP_LAST_ACK:
        switch (event) {
        case TCP_EVENT_RCV_ACK:
            /* 收到对 FIN 的 ACK: 连接彻底关闭 */
            sock->state = TCP_CLOSED;
            if (sock->on_closed) sock->on_closed(sock);
            return true;

        case TCP_EVENT_TIMEOUT:
            /* 超时: 强制关闭 */
            sock->state = TCP_CLOSED;
            if (sock->on_closed) sock->on_closed(sock);
            return true;

        default:
            return false;
        }
        break;

    /* ======== FIN_WAIT_1 ======== */
    case TCP_FIN_WAIT_1:
        switch (event) {
        case TCP_EVENT_RCV_ACK:
            /* 收到对 FIN 的 ACK: 等待对端 FIN */
            sock->state = TCP_FIN_WAIT_2;
            return true;

        case TCP_EVENT_RCV_FIN:
            /* 收到 FIN (对端同时关闭): 回复 ACK, 进入 TIME_WAIT */
            sock->rcv_nxt = ext_seq + 1;
            sock->state = TCP_TIME_WAIT;
            sock->timewait_timer_ms = 0;
            /* 动作: 发送 ACK */
            /* tcp_send_segment(sock, TCP_FLAG_ACK); */
            return true;

        default:
            return false;
        }
        break;

    /* ======== FIN_WAIT_2 ======== */
    case TCP_FIN_WAIT_2:
        switch (event) {
        case TCP_EVENT_RCV_FIN:
            /* 收到对端 FIN: 回复 ACK, 进入 TIME_WAIT */
            sock->rcv_nxt = ext_seq + 1;
            sock->state = TCP_TIME_WAIT;
            sock->timewait_timer_ms = 0;
            /* 动作: 发送 ACK */
            /* tcp_send_segment(sock, TCP_FLAG_ACK); */
            return true;

        case TCP_EVENT_TIMEOUT:
            /* 等待超时: 直接关闭 */
            sock->state = TCP_CLOSED;
            if (sock->on_closed) sock->on_closed(sock);
            return true;

        default:
            return false;
        }
        break;

    /* ======== TIME_WAIT ======== */
    case TCP_TIME_WAIT:
        switch (event) {
        case TCP_EVENT_TIMEOUT:
            /* 2MSL 超时: 进入 CLOSED */
            sock->state = TCP_CLOSED;
            if (sock->on_closed) sock->on_closed(sock);
            return true;

        default:
            /* TIME_WAIT 状态只接受超时事件, 其他报文(如 FIN 重传) */
            /* 需要重新发送 ACK 但不改变状态 -- 这里简化处理 */
            return false;
        }
        break;
    }

    return false;
}

/* ============================================================ */
/* TCP 定时器管理                                               */
/* ============================================================ */

/**
 * TCP 连接定时器更新 -- 用于 TIME_WAIT 超时
 * 在主循环中周期性调用, 更新 TCP 连接的计时器
 */
void tcp_timer_tick(tcp_sock_t *sock, uint32_t elapsed_ms) {
    switch (sock->state) {
    case TCP_SYN_RCVD:
        /* SYN+ACK 重传定时器 (简化: 3 秒超时关闭) */
        /* 实际实现需要重传逻辑, 这里仅作示例 */
        break;

    case TCP_TIME_WAIT:
        /* TIME_WAIT: 2MSL = 2 * 2 * RTT, 通常取 60 秒 */
        sock->timewait_timer_ms += elapsed_ms;
        if (sock->timewait_timer_ms >= 60000) {
            tcp_state_machine_handle(sock, TCP_EVENT_TIMEOUT, 0, 0);
        }
        break;

    case TCP_FIN_WAIT_2:
        /* FIN_WAIT_2 超时: 防止对端不发送 FIN 导致死等 */
        sock->timewait_timer_ms += elapsed_ms;
        if (sock->timewait_timer_ms >= 60000) {
            tcp_state_machine_handle(sock, TCP_EVENT_TIMEOUT, 0, 0);
        }
        break;

    default:
        break;
    }
}

/**
 * 初始化 TCP 连接控制块 (服务器模式)
 */
void tcp_sock_init_server(tcp_sock_t *sock, uint16_t local_port,
                          void (*send_frame)(const uint8_t *, uint16_t)) {
    memset(sock, 0, sizeof(tcp_sock_t));
    sock->state       = TCP_LISTEN;  /* 服务器启动即进入 LISTEN */
    sock->local_port  = local_port;
    sock->send_frame  = send_frame;
}
```

#### C.3.3 MCU 场景 TCP 状态机简化点

**为什么去掉 SYN_SENT**：在 MCU 服务器场景中，设备通常不主动发起 TCP 连接，因此 SYN_SENT、SYN_RCVD（客户端侧）等状态可以省略。如果 MCU 同时作为客户端，需要补充这些状态。

**TIME_WAIT 的处理**：完整 TCP 实现中 TIME_WAIT 需要 2MSL（通常 2 分钟）等待，在 MCU 环境下往往简化为 30-60 秒或直接使用 RST 快速回收。有些轻量实现甚至直接跳过 TIME_WAIT 直接进入 CLOSED。

**重传机制**：简化状态机未实现完整的超时重传（RTO）。在生产代码中，SYN_RCVD 和 LAST_ACK 状态的超时事件应触发重传（通常重试 3 次，每次间隔翻倍）。

---

### C.4 链路检测状态机 (PHY Link Detection)

PHY 链路检测状态机专门处理物理链路的抖动（flapping）问题。当电缆接触不良或电磁干扰导致链路快速上下波动时，软件需要去抖（debouce）逻辑来防止状态频繁切换。

#### C.4.1 状态图

```
                      +-------------------+
                      |    LINK_WAIT      | <-- 初始状态 / 链路断开等待
                      +--------+----------+
                               |
                       (PHY BMSR.2=1,
                        链路物理 UP)
                               |
                               v
                      +--------+----------+
               +----->| PRE_LINK_UP      | <-- 去抖确认中
               |      +--------+----------+
               |               |
               |      (去抖计时器到时, 连续 N 次检测到 LINK UP)
               |               |
               |               v
               |      +--------+----------+
               |      |    LINK_UP        | <-- 链路稳定运行
               |      +--------+----------+
               |               |
               |      (PHY BMSR.2=0,
               |       链路断开)
               |               |
               |               v
               |      +--------+----------+
               |      |  PRE_LINK_DOWN   | <-- 去抖确认断开
               |      +--------+----------+
               |               |
               |      (去抖计时器到时, 连续 N 次检测到 LINK DOWN)
               |               |
               |               v
               |      +--------+----------+
               |      |   LINK_DOWN      | <-- 确认链路断开
               |      +--------+----------+
               |               |
               |      (电缆恢复,           (超过重试次数,
               |       BMSR.2=1)          进入错误处理)
               |               |               |
               |               v               v
               |      +--------+----------+   +---------+
               +------+   LINK_RECOVERY  |   |  ERROR  |
                      +------------------+   +---------+
```

#### C.4.2 C 代码实现 (带去抖逻辑)

```c
#include <stdint.h>
#include <stdbool.h>

/* ============================================================ */
/* PHY 链路检测状态机 (带去抖逻辑)                              */
/* ============================================================ */

/* ----- 去抖参数可配置 ----- */
#define LINK_DEBOUNCE_UP_MS   200   /* 链路 UP 去抖时间 (ms) */
#define LINK_DEBOUNCE_DOWN_MS 300   /* 链路 DOWN 去抖时间 (ms) */
#define LINK_POLL_MS          50    /* PHY 轮询间隔 (ms) */
#define LINK_RECOVERY_MAX     3     /* 最大恢复尝试次数 */

/* ----- 状态定义 ----- */
typedef enum {
    LINK_STATE_WAIT,          /* 初始等待态 */
    LINK_STATE_PRE_UP,        /* 链路 UP 去抖中 */
    LINK_STATE_UP,            /* 链路已建立且稳定 */
    LINK_STATE_PRE_DOWN,      /* 链路 DOWN 去抖中 */
    LINK_STATE_DOWN,          /* 链路确认断开 */
    LINK_STATE_RECOVERY,      /* 尝试恢复 */
    LINK_STATE_ERROR,         /* 错误态 */
} link_state_t;

/* ----- 状态机上下文 ----- */
typedef struct {
    link_state_t state;           /* 当前状态 */
    uint8_t      phy_addr;        /* PHY 地址 */
    uint32_t     debounce_timer;  /* 去抖计时器 (ms) */
    uint8_t      recovery_count;  /* 恢复尝试计数 */
    uint32_t     stable_time_ms;  /* LINK_UP 稳定时间 (统计用) */

    /* 回调函数 */
    void (*on_link_up_cb)(void);
    void (*on_link_down_cb)(void);

    /* PHY 寄存器读取 (由 MDIO 层提供) */
    uint16_t (*phy_read)(uint8_t addr, uint8_t reg);
} link_detect_t;

/* ============================================================ */
/* 状态机主循环                                                 */
/* ============================================================ */

/**
 * 链路检测状态机 -- 每次 PHY 轮询时调用
 *
 * ctx:    状态机上下文
 * poll_ms: 自上次调用以来的时间 (ms)
 *
 * 返回: true 表示链路当前处于 UP 状态
 */
bool link_detect_state_machine_run(link_detect_t *ctx, uint32_t poll_ms) {
    /* 读取 PHY 链路状态 (两次读取清除锁存) */
    (void)ctx->phy_read(ctx->phy_addr, 0x01);           /* 清除锁存 */
    uint16_t bmsr = ctx->phy_read(ctx->phy_addr, 0x01); /* 真实值 */
    bool phy_link_up = (bmsr & 0x0004) ? true : false;

    switch (ctx->state) {

    /* ======== WAIT: 等待链路首次 UP ======== */
    case LINK_STATE_WAIT:
        if (phy_link_up) {
            /* 检测到链路 UP: 启动去抖 */
            ctx->debounce_timer = 0;
            ctx->state = LINK_STATE_PRE_UP;
        }
        break;

    /* ======== PRE_UP: 链路 UP 去抖确认 ======== */
    case LINK_STATE_PRE_UP:
        ctx->debounce_timer += poll_ms;

        if (!phy_link_up) {
            /* 去抖期间链路又断了: 复位去抖, 回到 WAIT */
            ctx->debounce_timer = 0;
            ctx->state = LINK_STATE_WAIT;
            break;
        }

        if (ctx->debounce_timer >= LINK_DEBOUNCE_UP_MS) {
            /* 连续 LINK_DEBOUNCE_UP_MS 时间内始终 UP: 确认 */
            ctx->stable_time_ms    = 0;
            ctx->recovery_count    = 0;
            ctx->state = LINK_STATE_UP;
            /* 入口动作: 通知上层链路 UP */
            if (ctx->on_link_up_cb) ctx->on_link_up_cb();
        }
        break;

    /* ======== UP: 正常运行态 ======== */
    case LINK_STATE_UP:
        ctx->stable_time_ms += poll_ms;  /* 记录稳定运行时间 */

        if (!phy_link_up) {
            /* 检测到链路 DOWN: 启动去抖 */
            ctx->debounce_timer = 0;
            ctx->state = LINK_STATE_PRE_DOWN;
        }
        break;

    /* ======== PRE_DOWN: 链路 DOWN 去抖确认 ======== */
    case LINK_STATE_PRE_DOWN:
        ctx->debounce_timer += poll_ms;

        if (phy_link_up) {
            /* 去抖期间链路恢复了: 回到 UP */
            ctx->debounce_timer = 0;
            ctx->state = LINK_STATE_UP;
            break;
        }

        if (ctx->debounce_timer >= LINK_DEBOUNCE_DOWN_MS) {
            /* 确认链路断开 */
            ctx->state = LINK_STATE_DOWN;
            /* 入口动作: 通知上层链路 DOWN */
            if (ctx->on_link_down_cb) ctx->on_link_down_cb();
        }
        break;

    /* ======== DOWN: 链路确认断开 ======== */
    case LINK_STATE_DOWN:
        if (phy_link_up) {
            /* 链路恢复: 尝试重新建立 */
            ctx->state = LINK_STATE_RECOVERY;
            ctx->debounce_timer = 0;
        }
        break;

    /* ======== RECOVERY: 恢复后去抖 ======== */
    case LINK_STATE_RECOVERY:
        ctx->debounce_timer += poll_ms;

        if (!phy_link_up) {
            /* 恢复失败: 计数器 +1 */
            ctx->recovery_count++;
            ctx->debounce_timer = 0;

            if (ctx->recovery_count >= LINK_RECOVERY_MAX) {
                ctx->state = LINK_STATE_ERROR;
            } else {
                ctx->state = LINK_STATE_DOWN;  /* 等待下一次恢复 */
            }
            break;
        }

        if (ctx->debounce_timer >= LINK_DEBOUNCE_UP_MS) {
            /* 恢复成功 */
            ctx->recovery_count = 0;
            ctx->stable_time_ms = 0;
            ctx->state = LINK_STATE_UP;
            if (ctx->on_link_up_cb) ctx->on_link_up_cb();
        }
        break;

    /* ======== ERROR: 错误处理 ======== */
    case LINK_STATE_ERROR:
        /* 定时重试: 每 10 秒尝试一次 */
        ctx->debounce_timer += poll_ms;
        if (ctx->debounce_timer >= 10000) {
            ctx->debounce_timer  = 0;
            ctx->recovery_count  = 0;
            ctx->state = LINK_STATE_WAIT;  /* 从头开始检测 */
        }
        break;
    }

    return (ctx->state == LINK_STATE_UP);
}

/**
 * 初始化链路检测状态机
 */
void link_detect_init(link_detect_t *ctx, uint8_t phy_addr,
                      uint16_t (*phy_read)(uint8_t, uint8_t),
                      void (*on_up)(void), void (*on_down)(void))
{
    ctx->state          = LINK_STATE_WAIT;
    ctx->phy_addr       = phy_addr;
    ctx->debounce_timer = 0;
    ctx->recovery_count = 0;
    ctx->stable_time_ms = 0;
    ctx->phy_read       = phy_read;
    ctx->on_link_up_cb  = on_up;
    ctx->on_link_down_cb = on_down;
}
```

#### C.4.3 去抖时序图解

```
PHY BMSR.2 (物理链路):
   ┌──────┐         ┌──────┐           ┌──────┐
   │      │         │      │           │      │
───┘      └─────────┘      └───────────┘      └────
    ^ 干扰  ^               ^ 干扰
    (50ms)  (100ms)         (30ms)

无去抖时软件状态:
   ┌─┐   ┌─┐         ┌─┐   ┌─┐
   │ │   │ │         │ │   │ │
───┘ └───┘ └─────────┘ └───┘ └──────────────────
    UP DOWN UP DOWN   ...   (频繁跳变, 不可接受)

有去抖时软件状态 (PRE_UP=200ms, PRE_DOWN=300ms):
                            ┌──────────────
                            │
────────────────────────────┘
                            只有持续 UP 超过 200ms 才算 UP
                            只有持续 DOWN 超过 300ms 才算 DOWN
                            (去抖滤除了短暂的毛刺)
```

#### C.4.4 链路状态机与 PHY 驱动状态机的协作

链路检测状态机位于 PHY 驱动状态机之下，两者是协作关系：

```
┌─────────────────────────────────────────────────┐
│             应用层 (App Layer)                    │
├─────────────────────────────────────────────────┤
│           TCP/UDP 协议栈                         │
├─────────────────────────────────────────────────┤
│     C.1 ARP 缓存状态机 ←→ C.3 TCP 状态机          │
├─────────────────────────────────────────────────┤
│  第九章 PHY 驱动状态机 (phy_state_machine_tick)    │
│     ├── C.2 Auto-Neg 状态机 (ANEG子过程)          │
│     └── C.4 链路检测状态机 (Link 监控)            │
├─────────────────────────────────────────────────┤
│              PHY 硬件 (寄存器层)                  │
└─────────────────────────────────────────────────┘
```

- PHY 驱动状态机的 `LINK_UP` / `LINK_DOWN` 状态使用本节的链路检测状态机作为去抖过滤器
- 链路检测状态机的 `on_link_up_cb` 回调触发 PHY 驱动状态机中的链接恢复流程
- `on_link_down_cb` 回调通知协议栈 (ARP 缓存需要清除，TCP 连接需要断开)

---

### C.5 状态机通用设计模式总结

#### 嵌入式网络协议栈中状态机的共性设计模式

1. **枚举 + switch-case 模式**：使用 `typedef enum` 定义状态集，通过 `switch(state)` 分发事件，每个 case 内实现状态转换逻辑。这是 MCU 环境最高效、最可读的实现方式。

2. **状态进入/退出动作**：在状态转换的前后执行动作，典型实现方式：
   ```c
   /* 退出旧状态 */
   exit_action(old_state);
   /* 执行状态转换 */
   state = new_state;
   /* 进入新状态 */
   entry_action(new_state);
   ```

3. **定时器驱动**：每个状态维护独立的计时器（`state_timer_ms`），在状态机 tick 函数中累加并检查超时，驱动超时事件。

4. **事件驱动 + 轮询混合**：报文接收产生事件（如 ARP 应答到达、TCP SYN 到达），通过事件队列驱动状态机；定时超时通过轮询 tick 函数驱动。

5. **状态表查询 (可选优化)**：当状态-事件对较多时，可以用二维函数指针表替代 switch-case，提高可扩展性：
   ```c
   typedef bool (*state_handler_t)(void *ctx, event_t evt, void *data);
   state_handler_t state_table[MAX_STATE][MAX_EVENT];
   ```

6. **层次状态机 (HSM)**：TCP 状态机天然具有层次性（ESTABLISHED 和 CLOSE_WAIT 共享 "已连接" 的公共行为），在复杂应用中可以将公共行为提取到父状态。但在 MCU 场景中，扁平状态机通常已经足够且更易调试。

---

### C.6 状态机调试技巧

**调试状态机最有效的方法**：日志 + 状态转储。

```c
/* 通用状态机日志宏 */
#define SM_DEBUG(state, event, format, ...) \
    do { \
        printf("[SM] %s -> event=%s: " format "\r\n", \
               state_name(state), event_name(event), ##__VA_ARGS__); \
    } while (0)

/* 状态转储: 打印所有 ARP 缓存表的状态 */
void arp_cache_dump(arp_cache_table_t *tbl) {
    const char *state_str[] = {"IDLE","RESOLVING","RESOLVED","STALE"};
    printf("--- ARP Cache Dump ---\r\n");
    for (int i = 0; i < ARP_CACHE_SIZE; i++) {
        if (!tbl->entries[i].valid) continue;
        printf("  [%d] IP=%08lX State=%s Retry=%d Timer=%lu\r\n",
               i, tbl->entries[i].ip_addr,
               state_str[tbl->entries[i].state],
               tbl->entries[i].retry_count,
               tbl->entries[i].state_timer_ms);
    }
}
```

**状态机调试清单**：
- 每个状态都有意料之外的输入时如何处理（防御性编程）
- 状态转换后计时器是否正确复位（防止残留计时器导致意外超时）
- 去抖参数是否与 PHY 硬件特性匹配（不同 PHY 的 Link Status 稳定时间不同）
- 状态机在 RESET/INIT 时是否进入确定的初始状态
- 是否存在死锁状态（无任何事件能离开的状态）

---

## Appendix D: Copy-Paste C Code Templates

> 本章节提供了一套完整的 C 语言代码模板，覆盖 MCU 以太网编程中最常用的基础功能。每个模板都是独立可编译的，可以直接复制到您的项目中修改使用。代码以 **ARM Cortex-M** (小端序, 32位) 为默认目标平台，但大部分代码是平台无关的。

---

### D.1 CRC32 Calculation (Ethernet FCS)

**功能**：计算 IEEE 802.3 标准 CRC32，用于以太网帧的 FCS (Frame Check Sequence) 校验。

**算法说明**：
- 多项式：`0x04C11DB7`（反射表示为 `0xEDB88320`）
- 初始值：`0xFFFFFFFF`
- 异或输出：`0xFFFFFFFF`
- 使用查表法加速（256 项查找表）

```c
/**
 * crc32.h - IEEE 802.3 CRC32 实现
 *
 * 使用方法：
 *   uint8_t data[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, ...};
 *   uint32_t crc = crc32_calculate(data, sizeof(data));
 *   // crc 即为 FCS 值（4 字节），以小端序附加到帧尾
 *
 * 编译测试：
 *   gcc -DTEST_CRC32 -o test_crc32 crc32.c && ./test_crc32
 */

#include <stdint.h>
#include <stddef.h>

/* CRC32 查找表 */
static uint32_t crc32_table[256];
static uint8_t  crc32_table_initialized = 0;

/**
 * 生成 CRC32 查找表（仅执行一次）
 * 多项式：0xEDB88320（0x04C11DB7 的反射表示）
 */
static void crc32_init_table(void)
{
    uint32_t crc;
    for (uint32_t i = 0; i < 256; i++)
    {
        crc = i;
        for (int j = 0; j < 8; j++)
        {
            if (crc & 1)
                crc = (crc >> 1) ^ 0xEDB88320;
            else
                crc = (crc >> 1);
        }
        crc32_table[i] = crc;
    }
    crc32_table_initialized = 1;
}

/**
 * 计算 IEEE 802.3 CRC32
 * @param data  输入数据指针
 * @param len   数据长度（字节）
 * @return      32 位 CRC 值
 */
uint32_t crc32_calculate(const uint8_t *data, uint32_t len)
{
    uint32_t crc = 0xFFFFFFFF;

    if (!crc32_table_initialized)
        crc32_init_table();

    for (uint32_t i = 0; i < len; i++)
    {
        uint8_t index = (uint8_t)((crc ^ data[i]) & 0xFF);
        crc = (crc >> 8) ^ crc32_table[index];
    }

    return crc ^ 0xFFFFFFFF;
}

#if defined(TEST_CRC32)
#include <stdio.h>

int main(void)
{
    /* 测试向量："123456789" */
    const uint8_t test[] = "123456789";
    uint32_t crc = crc32_calculate(test, 9);
    printf("CRC32(\"123456789\") = 0x%08lX (expect 0xCBF43926)\n",
           (unsigned long)crc);

    /* 测试空数据 */
    crc = crc32_calculate(NULL, 0);
    printf("CRC32(empty) = 0x%08lX (expect 0x00000000)\n",
           (unsigned long)crc);

    return 0;
}
#endif /* TEST_CRC32 */
```

**用法示例**：
```c
/* 计算以太网帧的 CRC32 并附加 FCS */
void ethernet_send_frame(uint8_t *frame, uint32_t len)
{
    uint32_t fcs = crc32_calculate(frame, len);

    /* 将 FCS 以小端序写入帧尾部 */
    frame[len + 0] = (uint8_t)( fcs        & 0xFF);
    frame[len + 1] = (uint8_t)((fcs >> 8)  & 0xFF);
    frame[len + 2] = (uint8_t)((fcs >> 16) & 0xFF);
    frame[len + 3] = (uint8_t)((fcs >> 24) & 0xFF);

    /* 发送 len + 4 字节到 MAC */
    MAC_send(frame, len + 4);
}

/* 验证收到的帧 CRC 是否正确 */
int ethernet_verify_fcs(const uint8_t *frame, uint32_t len)
{
    if (len < 4) return 0;

    uint32_t received_fcs = (uint32_t)frame[len - 4] |
                           ((uint32_t)frame[len - 3] << 8) |
                           ((uint32_t)frame[len - 2] << 16) |
                           ((uint32_t)frame[len - 1] << 24);

    uint32_t calc_fcs = crc32_calculate(frame, len - 4);

    return (calc_fcs == received_fcs);
}
```

---

### D.2 MAC Address Utilities

**功能**：MAC 地址的格式化、解析、类型判断和比较。

```c
/**
 * mac_utils.h - MAC 地址工具函数
 *
 * 所有函数均不依赖外部库（不调用 sprintf/sscanf），
 * 适用于裸机 MCU 环境。
 *
 * 编译测试：
 *   gcc -DTEST_MAC -o test_mac mac_utils.c && ./test_mac
 */

#include <stdint.h>
#include <stdbool.h>
#include <string.h>

#define MAC_ADDR_LEN    6
#define MAC_STR_LEN     18  /* "AA:BB:CC:DD:EE:FF\0" */

/**
 * 将 MAC 地址转换为字符串 "AA:BB:CC:DD:EE:FF"
 * @param mac  6 字节 MAC 地址
 * @param str  输出缓冲区（至少 18 字节）
 */
void mac_to_string(const uint8_t *mac, char *str)
{
    const char hex[] = "0123456789ABCDEF";
    str[0]  = hex[(mac[0] >> 4) & 0x0F];
    str[1]  = hex[ mac[0]       & 0x0F];
    str[2]  = ':';
    str[3]  = hex[(mac[1] >> 4) & 0x0F];
    str[4]  = hex[ mac[1]       & 0x0F];
    str[5]  = ':';
    str[6]  = hex[(mac[2] >> 4) & 0x0F];
    str[7]  = hex[ mac[2]       & 0x0F];
    str[8]  = ':';
    str[9]  = hex[(mac[3] >> 4) & 0x0F];
    str[10] = hex[ mac[3]       & 0x0F];
    str[11] = ':';
    str[12] = hex[(mac[4] >> 4) & 0x0F];
    str[13] = hex[ mac[4]       & 0x0F];
    str[14] = ':';
    str[15] = hex[(mac[5] >> 4) & 0x0F];
    str[16] = hex[ mac[5]       & 0x0F];
    str[17] = '\0';
}

/**
 * 将字符串 "AA:BB:CC:DD:EE:FF" 解析为 MAC 地址
 * @param str  输入字符串（支持大写/小写十六进制）
 * @param mac  输出缓冲区（6 字节）
 * @return     成功返回 1，失败返回 0
 */
int string_to_mac(const char *str, uint8_t *mac)
{
    for (int i = 0; i < 6; i++)
    {
        uint8_t nibble_high = 0, nibble_low = 0;

        char c = str[i * 3];
        if      (c >= '0' && c <= '9') nibble_high = (uint8_t)(c - '0');
        else if (c >= 'A' && c <= 'F') nibble_high = (uint8_t)(c - 'A' + 10);
        else if (c >= 'a' && c <= 'f') nibble_high = (uint8_t)(c - 'a' + 10);
        else return 0;

        c = str[i * 3 + 1];
        if      (c >= '0' && c <= '9') nibble_low = (uint8_t)(c - '0');
        else if (c >= 'A' && c <= 'F') nibble_low = (uint8_t)(c - 'A' + 10);
        else if (c >= 'a' && c <= 'f') nibble_low = (uint8_t)(c - 'a' + 10);
        else return 0;

        mac[i] = (nibble_high << 4) | nibble_low;

        if (i < 5 && str[i * 3 + 2] != ':')
            return 0;
    }
    return 1;
}

/**
 * 判断是否为广播地址 (FF:FF:FF:FF:FF:FF)
 */
bool mac_is_broadcast(const uint8_t *mac)
{
    return (mac[0] == 0xFF && mac[1] == 0xFF &&
            mac[2] == 0xFF && mac[3] == 0xFF &&
            mac[4] == 0xFF && mac[5] == 0xFF);
}

/**
 * 判断是否为组播地址
 * 组播标志：MAC 第一字节的最低位为 1
 */
bool mac_is_multicast(const uint8_t *mac)
{
    return (mac[0] & 0x01) != 0;
}

/**
 * 判断是否为单播地址
 * 单播标志：MAC 第一字节的最低位为 0
 */
bool mac_is_unicast(const uint8_t *mac)
{
    return (mac[0] & 0x01) == 0;
}

/**
 * 比较两个 MAC 地址是否相等
 * @return 相等返回 1，不等返回 0
 */
int mac_compare(const uint8_t *mac1, const uint8_t *mac2)
{
    return (mac1[0] == mac2[0] && mac1[1] == mac2[1] &&
            mac1[2] == mac2[2] && mac1[3] == mac2[3] &&
            mac1[4] == mac2[4] && mac1[5] == mac2[5]);
}

#if defined(TEST_MAC)
#include <stdio.h>

int main(void)
{
    uint8_t mac[6];
    char    str[MAC_STR_LEN];

    uint8_t test_mac[] = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};
    mac_to_string(test_mac, str);
    printf("mac_to_string: %s (expect AA:BB:CC:DD:EE:FF)\n", str);

    int ret = string_to_mac("12:34:56:78:9A:BC", mac);
    printf("string_to_mac ret=%d: %02X:%02X:%02X:%02X:%02X:%02X\n",
           ret, mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);

    uint8_t bc[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
    uint8_t mc[] = {0x01, 0x00, 0x5E, 0x00, 0x00, 0x01};
    uint8_t uc[] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55};

    printf("is_broadcast(FF:...):  %d (expect 1)\n", mac_is_broadcast(bc));
    printf("is_multicast(01:...):  %d (expect 1)\n", mac_is_multicast(mc));
    printf("is_unicast(00:...):    %d (expect 1)\n", mac_is_unicast(uc));
    printf("is_multicast(FF:...):  %d (expect 1, broadcast is also multicast)\n",
           mac_is_multicast(bc));

    uint8_t mac1[] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55};
    uint8_t mac2[] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55};
    uint8_t mac3[] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x56};
    printf("mac_compare(equal):  %d (expect 1)\n", mac_compare(mac1, mac2));
    printf("mac_compare(differ): %d (expect 0)\n", mac_compare(mac1, mac3));

    return 0;
}
#endif /* TEST_MAC */
```

**用法示例**：
```c
void eth_process_received_frame(const uint8_t *frame, uint32_t len)
{
    const uint8_t *dst_mac = &frame[0];
    const uint8_t *src_mac = &frame[6];

    if (mac_is_broadcast(dst_mac))
    {
        printf("RX: broadcast frame\n");
        process_arp_request(frame, len);
    }
    else if (mac_is_multicast(dst_mac))
    {
        if (mac_compare(dst_mac, our_mcast_group))
            process_igmp_frame(frame, len);
    }
    else if (mac_is_unicast(dst_mac))
    {
        if (mac_compare(dst_mac, our_mac))
            process_unicast_frame(frame, len);
    }

    char mac_str[MAC_STR_LEN];
    mac_to_string(src_mac, mac_str);
    printf("  from %s\n", mac_str);
}
```

---

### D.3 IP Utilities

**功能**：IP 协议栈核心工具函数 -- 互联网校验和、IP 地址字符串互转和组播判断。

```c
/**
 * ip_utils.h - IP 层工具函数
 *
 * 编译测试：
 *   gcc -DTEST_IP -o test_ip ip_utils.c && ./test_ip
 */

#include <stdint.h>
#include <stdbool.h>
#include <string.h>

/**
 * 计算互联网校验和 (RFC 1071)
 *
 * 对 16 位字进行反码求和，结果取反。
 * 使用前将校验和字段清零，计算后填入返回值。
 *
 * @param buf    数据缓冲区
 * @param len    数据长度（字节）
 * @return       16 位校验和（已取反）
 */
uint16_t ip_checksum(const uint8_t *buf, int len)
{
    uint32_t sum = 0;

    for (int i = 0; i < len; i += 2)
    {
        uint16_t word;
        word  = (uint16_t)((uint16_t)buf[i] << 8);
        if (i + 1 < len)
            word |= (uint16_t)buf[i + 1];
        sum += word;
    }

    while (sum >> 16)
        sum = (sum & 0xFFFF) + (sum >> 16);

    return (uint16_t)(~sum & 0xFFFF);
}

/**
 * 将 32 位 IP 地址（网络字节序）转换为字符串 "192.168.1.1"
 * @param ip   网络字节序的 IP 地址
 * @param str  输出缓冲区（至少 16 字节）
 */
void ip_to_string(uint32_t ip, char *str)
{
    uint8_t *bytes = (uint8_t *)&ip;
    uint8_t octets[4] = {bytes[0], bytes[1], bytes[2], bytes[3]};
    char *p = str;

    for (int i = 0; i < 4; i++)
    {
        uint8_t val = octets[i];

        if (val >= 100) { *p++ = (char)('0' + val / 100); val %= 100; }
        if (val >= 10)   *p++ = (char)('0' + val / 10);
        else if (octets[i] < 10 && i > 0 && p > str && *(p-1) >= '0')
            { /* 中间八位组小于 10 时需补 0 */ }
        val %= 10;
        *p++ = (char)('0' + val);
        if (i < 3) *p++ = '.';
    }
    *p = '\0';
}

/**
 * 将字符串 "192.168.1.1" 解析为 32 位 IP 地址（网络字节序）
 * @param str  输入字符串
 * @param ip   输出 IP 地址（网络字节序）
 * @return     成功返回 1，失败返回 0
 */
int string_to_ip(const char *str, uint32_t *ip)
{
    uint8_t octets[4] = {0, 0, 0, 0};
    int     octet_idx = 0;
    uint16_t value    = 0;
    char    c;

    while ((c = *str++) != '\0')
    {
        if (c >= '0' && c <= '9')
        {
            value = (uint16_t)(value * 10 + (c - '0'));
            if (value > 255) return 0;
        }
        else if (c == '.')
        {
            if (octet_idx >= 3) return 0;
            octets[octet_idx++] = (uint8_t)value;
            value = 0;
        }
        else
        {
            return 0;
        }
    }

    if (octet_idx != 3) return 0;

    octets[3] = (uint8_t)value;

    *ip = ((uint32_t)octets[0] << 24) |
          ((uint32_t)octets[1] << 16) |
          ((uint32_t)octets[2] <<  8) |
          ((uint32_t)octets[3]      );

    return 1;
}

/**
 * 判断 IP 地址是否为组播地址
 * 组播范围：224.0.0.0 ~ 239.255.255.255 (224.0.0.0/4)
 */
bool ip_is_multicast(uint32_t ip)
{
    uint8_t first_byte = (uint8_t)(ip >> 24);
    return (first_byte >= 224 && first_byte <= 239);
}

#if defined(TEST_IP)
#include <stdio.h>

int main(void)
{
    char str[16];

    /* 测试校验和 */
    {
        uint8_t ip_header[] = {
            0x45, 0x00, 0x00, 0x3C,
            0x00, 0x00, 0x40, 0x00,
            0x40, 0x01, 0x00, 0x00,
            0xC0, 0xA8, 0x01, 0x01,
            0xC0, 0xA8, 0x01, 0x02
        };
        uint16_t cksum = ip_checksum(ip_header, sizeof(ip_header));
        ip_header[10] = (uint8_t)((cksum >> 8)  & 0xFF);
        ip_header[11] = (uint8_t)( cksum        & 0xFF);
        printf("IP checksum = 0x%04X\n", cksum);
        printf("Verify = 0x%04X (expect 0x0000)\n",
               ip_checksum(ip_header, sizeof(ip_header)));
    }

    /* 测试 IP 地址转换 */
    {
        uint32_t ip;
        string_to_ip("192.168.1.1", &ip);
        ip_to_string(ip, str);
        printf("ip_to_string: %s (expect 192.168.1.1)\n", str);

        string_to_ip("10.0.0.255", &ip);
        ip_to_string(ip, str);
        printf("ip_to_string: %s (expect 10.0.0.255)\n", str);
    }

    /* 测试组播判断 */
    {
        uint32_t ip;
        string_to_ip("224.0.0.1", &ip);
        printf("224.0.0.1 multicast: %d (expect 1)\n", ip_is_multicast(ip));

        string_to_ip("192.168.1.1", &ip);
        printf("192.168.1.1 multicast: %d (expect 0)\n", ip_is_multicast(ip));
    }

    return 0;
}
#endif /* TEST_IP */
```

**用法示例**：
```c
void build_ip_header(uint8_t *buf,
                     uint32_t src_ip, uint32_t dst_ip,
                     uint8_t protocol, uint16_t total_len)
{
    buf[0]  = 0x45;
    buf[1]  = 0x00;
    buf[2]  = (uint8_t)((total_len >> 8) & 0xFF);
    buf[3]  = (uint8_t)( total_len       & 0xFF);
    buf[4]  = 0x00;  buf[5]  = 0x00;
    buf[6]  = 0x40;  buf[7]  = 0x00;
    buf[8]  = 64;
    buf[9]  = protocol;
    buf[10] = 0x00;  buf[11] = 0x00;

    uint32_t src_net = src_ip;
    uint32_t dst_net = dst_ip;
    memcpy(&buf[12], &src_net, 4);
    memcpy(&buf[16], &dst_net, 4);

    uint16_t cksum = ip_checksum(buf, 20);
    buf[10] = (uint8_t)((cksum >> 8) & 0xFF);
    buf[11] = (uint8_t)( cksum       & 0xFF);
}
```

---

### D.4 Network Byte Order

**功能**：手动实现 htons/htonl/ntohs/ntohl，适用于没有标准 BSD Socket API 的裸机 MCU 环境。

```c
/**
 * byteorder.h - 网络字节序转换（手动实现）
 *
 * 目标平台：小端 CPU（ARM Cortex-M, x86 等）
 * 对于 GCC/Clang，推荐用内建函数：
 *   __builtin_bswap16() 和 __builtin_bswap32()
 *
 * 编译测试：
 *   gcc -DTEST_BYTEORDER -o test_byteorder byteorder.c && ./test_byteorder
 */

#include <stdint.h>

/**
 * 检测当前平台字节序
 * 返回 1 表示小端，0 表示大端
 */
static inline int platform_is_little_endian(void)
{
    const uint16_t test = 0x0001;
    const uint8_t *byte = (const uint8_t *)&test;
    return (int)(byte[0] != 0);
}

static inline uint16_t swap16(uint16_t val)
{
    return (uint16_t)((val << 8) | (val >> 8));
}

static inline uint32_t swap32(uint32_t val)
{
    return (uint32_t)(((val << 24)) |
                      ((val <<  8) & 0x00FF0000) |
                      ((val >>  8) & 0x0000FF00) |
                      ((val >> 24)));
}

static inline uint16_t htons(uint16_t val)
{
    if (platform_is_little_endian())
        return swap16(val);
    return val;
}

static inline uint16_t ntohs(uint16_t val)
{
    return htons(val);
}

static inline uint32_t htonl(uint32_t val)
{
    if (platform_is_little_endian())
        return swap32(val);
    return val;
}

static inline uint32_t ntohl(uint32_t val)
{
    return htonl(val);
}

/*
 * 确定小端平台时，可用宏版本消除函数调用开销：
 * #define HTONS(x)  (uint16_t)((((x) & 0xFF) << 8) | (((x) >> 8) & 0xFF))
 * #define HTONL(x)  (uint32_t)((((x) & 0xFF000000) >> 24) | \
 *                              (((x) & 0x00FF0000) >>  8) | \
 *                              (((x) & 0x0000FF00) <<  8) | \
 *                              (((x) & 0x000000FF) << 24))
 * #define NTOHS(x)  HTONS(x)
 * #define NTOHL(x)  HTONL(x)
 */

#if defined(TEST_BYTEORDER)
#include <stdio.h>

int main(void)
{
    printf("Platform is %s-endian\n",
           platform_is_little_endian() ? "little" : "big");

    uint16_t host16 = 0x1234;
    uint16_t net16  = htons(host16);
    printf("htons(0x%04X) = 0x%04X\n", host16, net16);
    printf("ntohs(0x%04X) = 0x%04X\n", net16, ntohs(net16));

    uint32_t host32 = 0x12345678;
    uint32_t net32  = htonl(host32);
    printf("htonl(0x%08lX) = 0x%08lX\n",
           (unsigned long)host32, (unsigned long)net32);
    printf("ntohl(0x%08lX) = 0x%08lX\n",
           (unsigned long)net32, (unsigned long)ntohl(net32));

    uint16_t udp_port = ntohs(0x0035);
    printf("UDP port (DNS) = %u (expect 53)\n", udp_port);

    return 0;
}
#endif /* TEST_BYTEORDER */
```

**用法示例**：
```c
void parse_udp_packet(const uint8_t *buf)
{
    typedef struct {
        uint16_t src_port;    /* 网络字节序 */
        uint16_t dst_port;
        uint16_t length;
        uint16_t checksum;
    } __attribute__((packed)) udp_header_t;

    const udp_header_t *udp = (const udp_header_t *)buf;

    uint16_t src_port = ntohs(udp->src_port);
    uint16_t dst_port = ntohs(udp->dst_port);
    uint16_t len      = ntohs(udp->length);

    if (dst_port == 53) { /* DNS */ }
}
```

---

### D.5 ARP Cache Implementation

**功能**：完整的 ARP 缓存实现，支持 IP 到 MAC 的映射管理、自动老化。

```c
/**
 * arp_cache.h - ARP 缓存实现
 *
 * 编译测试：
 *   gcc -DTEST_ARP_CACHE -o test_arp arp_cache.c && ./test_arp
 */

#include <stdint.h>
#include <string.h>
#include <stdbool.h>

/* ---------- 配置宏 ---------- */
#define ARP_CACHE_SIZE         8
#define ARP_TIMEOUT_SEC        300  /* RFC 1122 建议 >= 60s */
#define MAC_ADDR_LEN           6

#define ARP_STATE_INVALID      0
#define ARP_STATE_RESOLVING    1
#define ARP_STATE_VALID        2

/* ---------- 数据结构 ---------- */

typedef struct {
    uint32_t ip_addr;
    uint8_t  mac_addr[MAC_ADDR_LEN];
    uint8_t  state;
    uint16_t age_seconds;
} arp_entry_t;

typedef struct {
    arp_entry_t entries[ARP_CACHE_SIZE];
    uint32_t    lookups;
    uint32_t    hits;
} arp_cache_t;

static arp_cache_t arp_cache;

/* ---------- 函数实现 ---------- */

void arp_cache_init(void)
{
    memset(&arp_cache, 0, sizeof(arp_cache));
    for (int i = 0; i < ARP_CACHE_SIZE; i++)
        arp_cache.entries[i].state = ARP_STATE_INVALID;
}

int arp_cache_lookup(uint32_t ip, uint8_t *mac_out)
{
    arp_cache.lookups++;

    for (int i = 0; i < ARP_CACHE_SIZE; i++)
    {
        if (arp_cache.entries[i].state == ARP_STATE_VALID &&
            arp_cache.entries[i].ip_addr == ip)
        {
            memcpy(mac_out, arp_cache.entries[i].mac_addr, MAC_ADDR_LEN);
            arp_cache.hits++;
            return 1;
        }
    }

    return 0;
}

void arp_cache_update(uint32_t ip, const uint8_t *mac, uint8_t state)
{
    int empty_idx = -1;
    int oldest_idx = 0;
    uint16_t oldest_age = 0;

    for (int i = 0; i < ARP_CACHE_SIZE; i++)
    {
        if (arp_cache.entries[i].ip_addr == ip)
        {
            arp_cache.entries[i].state = state;
            memcpy(arp_cache.entries[i].mac_addr, mac, MAC_ADDR_LEN);
            arp_cache.entries[i].age_seconds = 0;
            return;
        }

        if (arp_cache.entries[i].state == ARP_STATE_INVALID && empty_idx < 0)
            empty_idx = i;

        if (arp_cache.entries[i].age_seconds > oldest_age)
        {
            oldest_age = arp_cache.entries[i].age_seconds;
            oldest_idx = i;
        }
    }

    int idx = (empty_idx >= 0) ? empty_idx : oldest_idx;

    arp_cache.entries[idx].ip_addr = ip;
    memcpy(arp_cache.entries[idx].mac_addr, mac, MAC_ADDR_LEN);
    arp_cache.entries[idx].state = state;
    arp_cache.entries[idx].age_seconds = 0;
}

int arp_cache_start_resolve(uint32_t ip)
{
    uint8_t zero_mac[MAC_ADDR_LEN] = {0};
    arp_cache_update(ip, zero_mac, ARP_STATE_RESOLVING);

    for (int i = 0; i < ARP_CACHE_SIZE; i++)
    {
        if (arp_cache.entries[i].ip_addr == ip)
            return i;
    }
    return -1;
}

void arp_cache_age(void)
{
    for (int i = 0; i < ARP_CACHE_SIZE; i++)
    {
        if (arp_cache.entries[i].state == ARP_STATE_INVALID)
            continue;

        arp_cache.entries[i].age_seconds++;

        if (arp_cache.entries[i].state == ARP_STATE_VALID &&
            arp_cache.entries[i].age_seconds >= ARP_TIMEOUT_SEC)
        {
            arp_cache.entries[i].state = ARP_STATE_INVALID;
        }
        else if (arp_cache.entries[i].state == ARP_STATE_RESOLVING &&
                 arp_cache.entries[i].age_seconds >= 3)
        {
            arp_cache.entries[i].state = ARP_STATE_INVALID;
        }
    }
}

void arp_cache_print(void)
{
#if !defined(NO_PRINTF)
    extern int printf(const char *fmt, ...);
    printf("--- ARP Cache (lookups=%lu, hits=%lu, hitrate=%lu%%) ---\n",
           (unsigned long)arp_cache.lookups,
           (unsigned long)arp_cache.hits,
           arp_cache.lookups > 0
               ? (unsigned long)(arp_cache.hits * 100 / arp_cache.lookups)
               : 0);
    for (int i = 0; i < ARP_CACHE_SIZE; i++)
    {
        arp_entry_t *e = &arp_cache.entries[i];
        const char *s;
        switch (e->state) {
            case ARP_STATE_INVALID:   s = "INV"; break;
            case ARP_STATE_RESOLVING: s = "RSV"; break;
            case ARP_STATE_VALID:     s = "VAL"; break;
            default:                  s = "???"; break;
        }
        if (e->state != ARP_STATE_INVALID)
        {
            uint8_t *ipb = (uint8_t *)&e->ip_addr;
            printf("  [%d] %d.%d.%d.%d -> %02X:%02X:%02X:%02X:%02X:%02X [%s] age=%u\n",
                   i, ipb[0], ipb[1], ipb[2], ipb[3],
                   e->mac_addr[0], e->mac_addr[1], e->mac_addr[2],
                   e->mac_addr[3], e->mac_addr[4], e->mac_addr[5],
                   s, e->age_seconds);
        }
    }
#endif
}

#if defined(TEST_ARP_CACHE)
#include <stdio.h>

int main(void)
{
    arp_cache_init();

    uint8_t mac[6];
    uint8_t mac1[6] = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0x01};
    uint8_t mac2[6] = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0x02};
    uint32_t ip1 = 0xC0A80101;
    uint32_t ip2 = 0xC0A80102;

    arp_cache_update(ip1, mac1, ARP_STATE_VALID);
    arp_cache_update(ip2, mac2, ARP_STATE_VALID);

    int found = arp_cache_lookup(ip1, mac);
    printf("Lookup 192.168.1.1: %s\n", found ? "FOUND" : "NOT FOUND");

    found = arp_cache_lookup(0xC0A80103, mac);
    printf("Lookup 192.168.1.3: %s\n", found ? "FOUND" : "NOT FOUND");

    arp_cache_print();

    /* 模拟老化 */
    printf("\n--- After 301 seconds ---\n");
    for (int i = 0; i < 301; i++)
        arp_cache_age();
    found = arp_cache_lookup(ip1, mac);
    printf("Lookup after aging: %s\n", found ? "FOUND" : "NOT FOUND");

    return 0;
}
#endif /* TEST_ARP_CACHE */
```

**用法示例**：
```c
/* 发送数据前的 ARP 处理流程 */
int eth_send_udp(uint32_t dst_ip, const uint8_t *data, uint16_t len)
{
    uint8_t dst_mac[MAC_ADDR_LEN];

    if (arp_cache_lookup(dst_ip, dst_mac))
    {
        return build_and_send_eth_frame(dst_mac, 0x0800, data, len);
    }

    arp_cache_start_resolve(dst_ip);

    uint8_t broadcast_mac[MAC_ADDR_LEN] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
    send_arp_request(broadcast_mac, dst_ip);
    queue_pending_packet(dst_ip, data, len);
    return 0;
}

void on_arp_reply(uint32_t sender_ip, const uint8_t *sender_mac)
{
    arp_cache_update(sender_ip, sender_mac, ARP_STATE_VALID);
    dequeue_and_send_pending(sender_ip);
}
```

---

### D.6 Packet Buffer Management

**功能**：轻量级数据包缓冲区管理，类似 lwIP 的 pbuf 机制，适用于资源受限的 MCU。

```c
/**
 * pbuf.h - 轻量级数据包缓冲区管理
 *
 * 设计思路：
 * - 静态内存池分配，避免动态内存碎片
 * - 支持单 buffer 和链式 buffer
 * - 减少数据拷贝（通过指针操作）
 *
 * 编译测试：
 *   gcc -DTEST_PBUF -o test_pbuf pbuf.c && ./test_pbuf
 */

#include <stdint.h>
#include <string.h>
#include <stddef.h>

/* ---------- 配置 ---------- */
#define PBUF_POOL_SIZE      8
#define PBUF_POOL_BUF_SIZE  1536

/* ---------- 数据结构 ---------- */

typedef struct pbuf {
    struct pbuf *next;
    uint8_t      *payload;
    uint16_t     len;
    uint16_t     tot_len;
    uint16_t     ref;
    uint8_t      in_use;
    uint8_t      data[PBUF_POOL_BUF_SIZE];
} pbuf_t;

static pbuf_t pbuf_pool[PBUF_POOL_SIZE];

void pbuf_init(void)
{
    memset(pbuf_pool, 0, sizeof(pbuf_pool));
}

pbuf_t *pbuf_alloc(uint16_t size)
{
    if (size > PBUF_POOL_BUF_SIZE || size == 0)
        return NULL;

    for (int i = 0; i < PBUF_POOL_SIZE; i++)
    {
        if (!pbuf_pool[i].in_use)
        {
            pbuf_t *p = &pbuf_pool[i];
            memset(p, 0, sizeof(pbuf_t));
            p->in_use  = 1;
            p->ref     = 1;
            p->payload = p->data;
            p->len     = size;
            p->tot_len = size;
            p->next    = NULL;
            return p;
        }
    }
    return NULL;
}

void pbuf_free(pbuf_t *p)
{
    if (p == NULL) return;

    if (p->ref > 1) { p->ref--; return; }

    pbuf_t *cur = p;
    while (cur != NULL)
    {
        pbuf_t *nxt = cur->next;
        cur->in_use = 0;
        cur->ref    = 0;
        cur->next   = NULL;
        cur = nxt;
    }
}

void pbuf_ref(pbuf_t *p)
{
    if (p) p->ref++;
}

uint16_t pbuf_copy(pbuf_t *dst, const uint8_t *src, uint16_t len)
{
    uint16_t written = 0;
    while (dst && len)
    {
        uint16_t c = (len < dst->len) ? len : dst->len;
        memcpy(dst->payload, src + written, c);
        written += c;
        len -= c;
        dst = dst->next;
    }
    return written;
}

uint16_t pbuf_copy_to_buf(uint8_t *dst, const pbuf_t *src, uint16_t len)
{
    uint16_t copied = 0;
    while (src && len)
    {
        uint16_t c = (len < src->len) ? len : src->len;
        memcpy(dst + copied, src->payload, c);
        copied += c;
        len -= c;
        src = src->next;
    }
    return copied;
}

int pbuf_header(pbuf_t *p, int16_t size)
{
    if (p == NULL) return -1;

    if (size < 0)
    {
        int16_t shift = (int16_t)(-size);
        if ((uint8_t *)(p->payload) - shift < p->data)
            return -1;
        p->payload -= shift;
        p->len     += (uint16_t)shift;
        p->tot_len += (uint16_t)shift;
    }
    else
    {
        if ((uint16_t)size > p->len) return -1;
        p->payload += size;
        p->len     -= (uint16_t)size;
        p->tot_len -= (uint16_t)size;
    }
    return 0;
}

#if defined(TEST_PBUF)
#include <stdio.h>

int main(void)
{
    pbuf_init();

    pbuf_t *p = pbuf_alloc(64);
    if (!p) { printf("alloc failed\n"); return 1; }
    printf("Allocated: len=%u\n", p->len);

    const char *test = "Hello Ethernet!";
    memcpy(p->payload, test, strlen(test) + 1);

    pbuf_t *p2 = pbuf_alloc(100);
    pbuf_header(p2, -14);
    pbuf_header(p2, -20);
    printf("After header reserve: len=%u, tot_len=%u\n",
           p2->len, p2->tot_len);

    pbuf_t *c1 = pbuf_alloc(10);
    pbuf_t *c2 = pbuf_alloc(20);
    memcpy(c1->payload, "HEAD", 4);
    memcpy(c2->payload, "TAIL", 4);
    c1->next = c2;
    c1->tot_len = c1->len + c2->len;

    uint8_t linear[64];
    uint16_t copied = pbuf_copy_to_buf(linear, c1, sizeof(linear));
    linear[copied] = '\0';
    printf("Chained: %s (expect HEADTAIL)\n", linear);

    pbuf_free(p);
    pbuf_free(p2);
    pbuf_free(c1);

    int used = 0;
    for (int i = 0; i < PBUF_POOL_SIZE; i++)
        if (pbuf_pool[i].in_use) used++;
    printf("Pool: %d/%d in use\n", used, PBUF_POOL_SIZE);

    return 0;
}
#endif /* TEST_PBUF */
```

**用法示例**：
```c
/* 协议栈分层处理流程 */
pbuf_t *build_udp_packet(uint32_t dst_ip, uint16_t dst_port,
                         const uint8_t *data, uint16_t len)
{
    uint16_t total = 14 + 20 + 8 + len;  /* Eth + IP + UDP + payload */
    pbuf_t *p = pbuf_alloc(total);
    if (p == NULL) return NULL;

    memcpy(p->payload + 14 + 20 + 8, data, len);

    /* 填充 UDP 头部 */
    uint16_t udp_len = 8 + len;
    p->payload[14+20+0] = (uint8_t)((dst_port >> 8) & 0xFF);
    p->payload[14+20+1] = (uint8_t)( dst_port       & 0xFF);
    p->payload[14+20+4] = (uint8_t)((udp_len  >> 8) & 0xFF);
    p->payload[14+20+5] = (uint8_t)( udp_len        & 0xFF);

    /* 填充 IP 头部 (调用 D.3 的 build_ip_header) */
    build_ip_header(&p->payload[14], 0xC0A80101, dst_ip, 17, 20 + 8 + len);

    /* 填充以太网头部 ... */
    return p;
}
```

---

### D.7 Simple Statistics

**功能**：网络接口的统计计数器，用于调试和性能监控。

```c
/**
 * eth_stats.h - 以太网接口统计计数器
 *
 * 支持多端口（用于交换芯片场景）。
 *
 * 编译测试：
 *   gcc -DTEST_STATS -o test_stats eth_stats.c && ./test_stats
 */

#include <stdint.h>
#include <string.h>
#include <stdbool.h>

/* ---------- 配置 ---------- */
#define ETH_MAX_PORTS    4

/* ---------- 错误标志 ---------- */
#define ERR_CRC          0x00000001
#define ERR_ALIGNMENT    0x00000002
#define ERR_COLLISION    0x00000004
#define ERR_OVERSIZE     0x00000008
#define ERR_UNDERSIZE    0x00000010
#define ERR_RUNT         0x00000020
#define ERR_JABBER       0x00000040
#define ERR_FRAGMENT     0x00000080
#define ERR_LATE_COLL    0x00000100
#define ERR_EXCESS_COLL  0x00000200
#define ERR_FIFO_UNDER   0x00000400
#define ERR_FIFO_OVER    0x00000800
#define ERR_DROPPED      0x00001000

/* ---------- 统计数据结构 ---------- */

typedef struct {
    uint32_t rx_packets;
    uint32_t tx_packets;
    uint32_t rx_bytes;
    uint32_t tx_bytes;
    uint32_t rx_errors;
    uint32_t tx_errors;
    uint32_t rx_dropped;
    uint32_t tx_dropped;
    uint32_t crc_errors;
    uint32_t alignment_errors;
    uint32_t collisions;
    uint32_t late_collisions;
    uint32_t excessive_collisions;
    uint32_t oversize_errors;
    uint32_t undersize_errors;
    uint32_t fragments;
    uint32_t jabber_errors;
    uint32_t fifo_overruns;
    uint32_t fifo_underruns;
    uint32_t multicast_rx;
    uint32_t broadcast_rx;
    uint32_t multicast_tx;
    uint32_t broadcast_tx;
    uint32_t rx_bps;
    uint32_t tx_bps;
    uint32_t rx_pps;
    uint32_t tx_pps;
    uint32_t last_rx_bytes;
    uint32_t last_tx_bytes;
    uint32_t last_rx_packets;
    uint32_t last_tx_packets;
} eth_port_stats_t;

static eth_port_stats_t eth_stats[ETH_MAX_PORTS];

void eth_stats_init(void)
{
    memset(eth_stats, 0, sizeof(eth_stats));
}

void eth_stats_rx_ok(uint8_t port, uint32_t byte_count,
                     bool is_multicast, bool is_broadcast)
{
    if (port >= ETH_MAX_PORTS) return;
    eth_port_stats_t *s = &eth_stats[port];
    s->rx_packets++;
    s->rx_bytes += byte_count;
    if (is_multicast)  s->multicast_rx++;
    if (is_broadcast)  s->broadcast_rx++;
}

void eth_stats_tx_ok(uint8_t port, uint32_t byte_count,
                     bool is_multicast, bool is_broadcast)
{
    if (port >= ETH_MAX_PORTS) return;
    eth_port_stats_t *s = &eth_stats[port];
    s->tx_packets++;
    s->tx_bytes += byte_count;
    if (is_multicast) s->multicast_tx++;
    if (is_broadcast) s->broadcast_tx++;
}

void eth_stats_rx_error(uint8_t port, uint32_t err_flags)
{
    if (port >= ETH_MAX_PORTS) return;
    eth_port_stats_t *s = &eth_stats[port];
    s->rx_errors++;
    if (err_flags & ERR_CRC)       s->crc_errors++;
    if (err_flags & ERR_ALIGNMENT) s->alignment_errors++;
    if (err_flags & ERR_COLLISION) s->collisions++;
    if (err_flags & ERR_OVERSIZE)  s->oversize_errors++;
    if (err_flags & ERR_UNDERSIZE) s->undersize_errors++;
    if (err_flags & ERR_FRAGMENT)  s->fragments++;
    if (err_flags & ERR_JABBER)    s->jabber_errors++;
    if (err_flags & ERR_FIFO_OVER) s->fifo_overruns++;
    if (err_flags & ERR_FIFO_UNDER)s->fifo_underruns++;
    if (err_flags & ERR_DROPPED)   s->rx_dropped++;
}

void eth_stats_update_rates(void)
{
    for (int port = 0; port < ETH_MAX_PORTS; port++)
    {
        eth_port_stats_t *s = &eth_stats[port];
        s->rx_bps = (s->rx_bytes - s->last_rx_bytes) * 8;
        s->tx_bps = (s->tx_bytes - s->last_tx_bytes) * 8;
        s->rx_pps = (s->rx_packets - s->last_rx_packets);
        s->tx_pps = (s->tx_packets - s->last_tx_packets);
        s->last_rx_bytes   = s->rx_bytes;
        s->last_tx_bytes   = s->tx_bytes;
        s->last_rx_packets = s->rx_packets;
        s->last_tx_packets = s->tx_packets;
    }
}

void eth_stats_print(uint8_t port)
{
#if !defined(NO_PRINTF)
    extern int printf(const char *fmt, ...);
    if (port >= ETH_MAX_PORTS) return;
    eth_port_stats_t *s = &eth_stats[port];

    printf("==== Port %u Stats ====\n", port);
    printf("  RX: %lu pkts, %lu bytes\n",
           (unsigned long)s->rx_packets, (unsigned long)s->rx_bytes);
    printf("  TX: %lu pkts, %lu bytes\n",
           (unsigned long)s->tx_packets, (unsigned long)s->tx_bytes);
    printf("  Rates: RX %lu bps/%lu pps, TX %lu bps/%lu pps\n",
           (unsigned long)s->rx_bps, (unsigned long)s->rx_pps,
           (unsigned long)s->tx_bps, (unsigned long)s->tx_pps);
    printf("  MCast RX: %lu, BCast RX: %lu\n",
           (unsigned long)s->multicast_rx, (unsigned long)s->broadcast_rx);
    printf("  Errors: CRC=%lu, Align=%lu, Coll=%lu\n",
           (unsigned long)s->crc_errors, (unsigned long)s->alignment_errors,
           (unsigned long)s->collisions);
#endif
}

void eth_stats_print_summary(void)
{
#if !defined(NO_PRINTF)
    extern int printf(const char *fmt, ...);
    printf("Port | RX Pkts  | TX Pkts  | RX Err | TX Err | RX bps\n");
    printf("-----+----------+----------+--------+--------+----------\n");
    for (int port = 0; port < ETH_MAX_PORTS; port++)
    {
        eth_port_stats_t *s = &eth_stats[port];
        printf("  %u  | %8lu | %8lu | %6lu | %6lu | %8lu\n",
               port,
               (unsigned long)s->rx_packets,
               (unsigned long)s->tx_packets,
               (unsigned long)s->rx_errors,
               (unsigned long)s->tx_errors,
               (unsigned long)s->rx_bps);
    }
#endif
}

void eth_stats_clear(uint8_t port)
{
    if (port >= ETH_MAX_PORTS) return;
    memset(&eth_stats[port], 0, sizeof(eth_port_stats_t));
}

void eth_stats_clear_all(void)
{
    memset(eth_stats, 0, sizeof(eth_stats));
}

#if defined(TEST_STATS)
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(void)
{
    srand((unsigned)time(NULL));
    eth_stats_init();

    for (int sec = 0; sec < 5; sec++)
    {
        int rx = rand() % 1000;
        int tx = rand() % 800;
        for (int i = 0; i < rx; i++)
        {
            eth_stats_rx_ok(0, 64 + (rand() % 1400),
                           (rand() % 3 == 0), (rand() % 10 == 0));
            if (rand() % 100 == 0)
                eth_stats_rx_error(0, ERR_CRC);
        }
        for (int i = 0; i < tx; i++)
            eth_stats_tx_ok(0, 64 + (rand() % 1400),
                           (rand() % 3 == 0), (rand() % 10 == 0));
        eth_stats_update_rates();
    }

    eth_stats_print(0);
    eth_stats_print_summary();
    return 0;
}
#endif /* TEST_STATS */
```

**用法示例**：
```c
/* MAC 接收中断中集成统计 */
void MAC_IRQHandler(void)
{
    uint8_t rx_data[1536];
    uint16_t rx_len;

    if (MAC_read_frame(rx_data, &rx_len))
    {
        eth_stats_rx_ok(0, rx_len,
                       mac_is_multicast(rx_data),
                       mac_is_broadcast(rx_data));
        process_eth_frame(rx_data, rx_len);
    }
    else
    {
        eth_stats_rx_error(0, ERR_CRC);
    }
}
```

---

### D.8 Timer Utilities

**功能**：基于 SysTick 的毫秒计时、微秒延时、超时检测和周期性任务调度器。

```c
/**
 * timer_utils.h - MCU 定时器工具函数
 *
 * 基于 ARM Cortex-M SysTick 实现。
 * 其他 MCU 平台替换 get_ms_tick() 为同级接口即可。
 *
 * 编译测试（PC 模拟）：
 *   gcc -DTEST_TIMER -o test_timer timer_utils.c && ./test_timer
 */

#include <stdint.h>
#include <stdbool.h>

/* ---------- 系统滴答 ---------- */

static volatile uint32_t system_ms_ticks = 0;

/**
 * SysTick 中断处理函数（每 1ms 调用一次）
 * 初始化：SysTick_Config(SystemCoreClock / 1000);
 */
void SysTick_Handler(void)
{
    system_ms_ticks++;
}

/**
 * 获取当前毫秒值
 * 溢出周期：约 49.7 天（32 位计数器）
 */
uint32_t get_ms_tick(void)
{
    uint32_t tick;
    __disable_irq();
    tick = system_ms_ticks;
    __enable_irq();
    return tick;
}

/**
 * 阻塞延时（毫秒）
 */
void delay_ms(uint32_t ms)
{
    uint32_t start = get_ms_tick();
    while ((get_ms_tick() - start) < ms) { }
}

/**
 * 微秒延时（忙等待，用于 MDIO 等精确时序）
 * 需根据 MCU 主频调整 MICROSECOND_LOOPS
 */
#define MICROSECOND_LOOPS  72  /* 72MHz 约 72 周期/us */

void delay_us(uint32_t us)
{
    for (uint32_t i = 0; i < us; i++)
    {
        for (volatile uint32_t j = 0; j < MICROSECOND_LOOPS; j++) { }
    }
}

/* ---------- 超时检测 ---------- */

typedef struct {
    uint32_t start_tick;
    uint32_t timeout_ms;
    bool     running;
} timeout_t;

void timeout_start(timeout_t *t, uint32_t timeout_ms)
{
    t->start_tick = get_ms_tick();
    t->timeout_ms = timeout_ms;
    t->running    = true;
}

bool timeout_expired(timeout_t *t)
{
    if (!t->running) return false;
    if ((get_ms_tick() - t->start_tick) >= t->timeout_ms)
    {
        t->running = false;
        return true;
    }
    return false;
}

void timeout_stop(timeout_t *t)
{
    t->running = false;
}

uint32_t timeout_remaining(timeout_t *t)
{
    if (!t->running) return 0;
    uint32_t elapsed = get_ms_tick() - t->start_tick;
    if (elapsed >= t->timeout_ms) return 0;
    return t->timeout_ms - elapsed;
}

void timeout_reset(timeout_t *t)
{
    t->start_tick = get_ms_tick();
    t->running    = true;
}

/* ---------- 周期性任务调度器 ---------- */

#define SCHEDULER_MAX_TASKS   8

typedef enum {
    TASK_READY,
    TASK_PAUSED,
    TASK_STOPPED
} task_state_t;

typedef struct {
    void     (*func)(void);
    uint32_t  interval_ms;
    uint32_t  last_run;
    task_state_t state;
    const char *name;
} task_t;

typedef struct {
    task_t tasks[SCHEDULER_MAX_TASKS];
    int    task_count;
} scheduler_t;

static scheduler_t scheduler;

void scheduler_init(void)
{
    scheduler.task_count = 0;
}

int scheduler_add(void (*func)(void), uint32_t interval_ms, const char *name)
{
    if (scheduler.task_count >= SCHEDULER_MAX_TASKS) return -1;
    task_t *t = &scheduler.tasks[scheduler.task_count++];
    t->func        = func;
    t->interval_ms = interval_ms;
    t->last_run    = get_ms_tick();
    t->state       = TASK_READY;
    t->name        = name;
    return scheduler.task_count - 1;
}

void scheduler_pause(int idx)
{
    if (idx >= 0 && idx < scheduler.task_count)
        scheduler.tasks[idx].state = TASK_PAUSED;
}

void scheduler_resume(int idx)
{
    if (idx >= 0 && idx < scheduler.task_count)
    {
        scheduler.tasks[idx].state = TASK_READY;
        scheduler.tasks[idx].last_run = get_ms_tick();
    }
}

void scheduler_run(void)
{
    uint32_t now = get_ms_tick();
    for (int i = 0; i < scheduler.task_count; i++)
    {
        task_t *t = &scheduler.tasks[i];
        if (t->state != TASK_READY) continue;
        if ((now - t->last_run) >= t->interval_ms)
        {
            t->last_run = now;
            t->func();
        }
    }
}

#if defined(TEST_TIMER)
#include <stdio.h>

/* PC 测试：模拟 get_ms_tick */
static uint32_t simulated_tick = 0;
uint32_t get_ms_tick(void) { return simulated_tick; }
void __disable_irq(void) {}
void __enable_irq(void)  {}

void blink_led(void) { printf("[%lu] Blink LED\n", (unsigned long)simulated_tick); }
void arp_age(void)   { printf("[%lu] ARP age\n", (unsigned long)simulated_tick); }

int main(void)
{
    printf("=== Timeout Test ===\n");
    timeout_t t;
    timeout_start(&t, 50);
    while (!timeout_expired(&t))
    {
        simulated_tick += 10;
        printf("  remaining=%lu\n",
               (unsigned long)timeout_remaining(&t));
    }
    printf("  Expired at tick=%lu!\n", (unsigned long)simulated_tick);

    printf("\n=== Scheduler Test ===\n");
    simulated_tick = 0;
    scheduler_init();
    scheduler_add(blink_led, 500, "LED");
    scheduler_add(arp_age,   1000, "ARP");

    for (int s = 0; s < 3; s++)
    {
        for (int ms = 0; ms < 1000; ms += 10)
        {
            simulated_tick += 10;
            scheduler_run();
        }
    }

    printf("\n=== Done ===\n");
    return 0;
}
#endif /* TEST_TIMER */
```

**用法示例**：
```c
/* MCU 主程序典型结构 */
int main(void)
{
    SystemInit();
    SysTick_Config(SystemCoreClock / 1000);

    ethernet_init();
    arp_cache_init();
    scheduler_init();

    scheduler_add(led_toggle,       500,  "LED");
    scheduler_add(arp_cache_age,    1000, "ARP");
    scheduler_add(eth_stats_update, 1000, "Stats");
    scheduler_add(check_phy_link,   2000, "PHY");

    while (1)
    {
        if (MAC_has_frame())
        {
            uint8_t buf[1536];
            uint16_t len = MAC_read_frame(buf);
            process_ethernet_frame(buf, len);
        }
        scheduler_run();
    }
}

/* 超时示例：等待 ARP 应答 */
void wait_for_arp(uint32_t target_ip)
{
    uint8_t mac[6];
    timeout_t t;
    timeout_start(&t, 3000);

    while (!timeout_expired(&t))
    {
        if (arp_cache_lookup(target_ip, mac))
        {
            printf("ARP resolved!\n");
            return;
        }
        process_network_events();
    }
    printf("ARP timeout!\n");
}
```

---

## Appendix E: Timing Diagrams and Protocol Visualizations

> 本章节使用 ASCII 艺术图直观展示以太网协议各层面的时序关系，帮助初学者建立从"协议文本描述"到"实际总线信号/交互流程"的视觉映射。每个图表均附有详细的时序参数说明和 C 代码示例，可用于嵌入式调试中的时序验证。

---

### E.1 以太网帧传输时序（Ethernet Frame Transmission Timing）

以太网帧在物理介质上的传输是严格串行的——从 Preamble 开始，到 FCS 结束，帧间至少保持 96 bit-time 的 IFG（Inter-Frame Gap）。对于 100Mbps 速率，1 bit-time = 10ns。

```
      IDLE         PREAMBLE (7 bytes)         SFD       DEST MAC (6B)     SRC MAC (6B)     ETYPE (2B)
        │    ┌──┬──┬──┬──┬──┬──┬──┬──┐    ┌──┐    ┌──┬──┬──┬──┬──┬──┐    ┌──┬──┬──┬──┬──┬──┐    ┌──┬──┐
   ......    │55│55│55│55│55│55│55│55│    │D5│    │AA│BB│CC│DD│EE│FF│    │11│22│33│44│55│66│    │08│00│    ......
        │    └──┴──┴──┴──┴──┴──┴──┴──┘    └──┘    └──┴──┴──┴──┴──┴──┘    └──┴──┴──┴──┴──┴──┘    └──┴──┘
        │    10101010 重复 7 次           10101011   MAC 目标地址           MAC 源地址            0x0800 = IPv4

              PAYLOAD (46-1500 bytes)              FCS (4B)          IFG (12B idle)
        ┌──┬──┬──┬──┬──┬──┬──┬──┐    ┌──┬──┬──┐    ┌──┬──┬──┬──┐    ┌──┬──┬──┬──┐
   ......│45│00│00│2E│..│..│..│..│....│..│..│..│    │XX│XX│XX│XX│    │  │  │  │  │......
        └──┴──┴──┴──┴──┴──┴──┴──┘    └──┴──┴──┘    └──┴──┴──┴──┘    └──┴──┴──┴──┘
        IP header + TCP/UDP + data   填充至 >= 46B   CRC32 校验和      最小 96 bit-time
```

**关键时序参数（100BASE-TX）**：

| 字段 | 长度 | 时间（@100Mbps） | 说明 |
|------|------|-------------------|------|
| Preamble | 7 字节 | 560 ns | 10MHz 方波，接收端用于时钟同步 |
| SFD | 1 字节 | 80 ns | 起始帧定界符，标志帧数据开始 |
| 最小帧 | 64 字节 | 5.12 μs | 含 14B 头部 + 46B 净荷 + 4B FCS |
| 最大帧 | 1518 字节 | 121.44 μs | 不含 Preamble 和 SFD |
| IFG | 12 字节 | 960 ns | 帧间隔，供接收端处理完成前一帧 |

```c
#include <stdint.h>
#include <stdio.h>

/* 计算以太网帧传输时间（微秒） */
float ethernet_frame_time_us(uint16_t payload_len, int is_100mbps)
{
    /* 帧总字节：14(头部) + payload + 4(FCS) + 8(Preamble+SFD) + 12(IFG) */
    uint16_t total_bytes = 14 + payload_len + 4 + 8 + 12;
    float    bit_time_us = is_100mbps ? 0.01f : 0.1f;   /* 100M:10ns, 10M:100ns */
    return (float)total_bytes * 8 * bit_time_us;
}

/* 打印帧时序诊断信息 */
void print_frame_timing(uint16_t payload_len)
{
    float t_10m  = ethernet_frame_time_us(payload_len, 0);
    float t_100m = ethernet_frame_time_us(payload_len, 1);

    printf("Payload: %u bytes\n", payload_len);
    printf("  10Mbps: %.2f us  (%.1f pps)\n", t_10m, 1000000.0f / t_10m);
    printf("  100Mbps: %.2f us  (%.1f pps)\n", t_100m, 1000000.0f / t_100m);

    if (payload_len < 46)
        printf("  *** 警告：净荷 < 46，需填充至 46 字节！\n");
}
```

**初学者常见误区**：
- Preamble 和 SFD 不在 Wireshark 中显示——网卡在接收时已剥离它们
- IFG 是强制要求，不是"建议"——违反 IFG 的帧会被丢弃
- 帧最小 64 字节（不含 Preamble）——小于 64B 为 runts（残帧），正常网络不应出现

---

### E.2 ARP 请求/应答交换时序（ARP Request/Response Exchange）

ARP 的核心是"一问一答"：发送者广播查询，目标单播回复。关键特征是请求使用广播 MAC（FF-FF-FF-FF-FF-FF），而应答使用单播 MAC。

```
      PC1 (192.168.1.1)                          PC2 (192.168.1.2)
      MAC: 00:0C:29:12:34:56                     MAC: 00:11:22:33:44:55
           │                                            │
           │  --- [ARP Request] 广播 --->                │
           │  Dest MAC: FF-FF-FF-FF-FF-FF                │
           │  Sender MAC: 00:0C:29:12:34:56              │
           │  Sender IP:  192.168.1.1                    │
           │  Target IP:  192.168.1.2                    │
           │  "Who has 192.168.1.2? Tell 192.168.1.1"    │
           │────────────────────────────────────────────>│
           │                                            │
           │  PC2 收到广播，检查 Target IP == 自身 IP     │
           │  匹配！PC2 将 PC1 的 IP+MAC 加入 ARP 缓存    │
           │  构造 ARP Reply（交换 Sender/Target 字段）    │
           │                                            │
           │  <--- [ARP Reply] 单播 ---                   │
           │  Dest MAC: 00:0C:29:12:34:56                │
           │  Sender MAC: 00:11:22:33:44:55              │
           │  Sender IP:  192.168.1.2                    │
           │  Target IP:  192.168.1.1                    │
           │  "192.168.1.2 is at 00:11:22:33:44:55"      │
           │<────────────────────────────────────────────│
           │                                            │
           │  PC1 收到 Reply，将 PC2 加入 ARP 缓存        │
           │  现在可以发送 IP 数据包到 PC2                │
```

**ARP 缓存状态转换图**：

```
                  +-----------+
                  |   EMPTY   |   (初始状态：无条目)
                  +-----------+
                       │
                       │ 发送 ARP Request（或收到请求）
                       ▼
                  +-----------+
           +----->| INCOMPLETE|   (请求已发出，等待回复)
           |      +-----------+
           |           │
           |           │ 收到 ARP Reply
           |           ▼
           |      +-----------+        +----------+
           |      | VALID     |------->| STALE    |  (超过生存时间)
           |      +-----------+        +----------+
           |           │                    │
           |           │ 应用层查询          │ 尝试使用该条目
           |           ▼                    ▼
           |      +-----------+        +----------+
           |      | RESOLVED  |        | DELAY    |  (发送单播探测)
           |      +-----------+        +----------+
           |                              │
           |                              │ 收到回复
           +------------------------------+
```

**ARP 超时重传（典型嵌入式实现）**：

```c
#define ARP_MAX_RETRIES  3
#define ARP_TIMEOUT_MS   1000   /* 每次重试间隔（ms） */

typedef struct {
    uint32_t ip_addr;
    uint8_t  mac_addr[6];
    int      state;         /* 0=EMPTY, 1=INCOMPLETE, 2=VALID */
    int      retry_count;
    uint32_t timer_start_ms;
} arp_entry_t;

/* ARP 状态机轮询（在 main loop 中调用） */
void arp_poll(arp_entry_t *entry, uint32_t now_ms)
{
    if (entry->state != 1)          /* 不是 INCOMPLETE，无需处理 */
        return;

    if (now_ms - entry->timer_start_ms < ARP_TIMEOUT_MS)
        return;                     /* 尚未超时 */

    if (entry->retry_count >= ARP_MAX_RETRIES) {
        printf("ARP failed for %d.%d.%d.%d\n",
               (entry->ip_addr >> 24) & 0xFF,
               (entry->ip_addr >> 16) & 0xFF,
               (entry->ip_addr >> 8) & 0xFF,
               entry->ip_addr & 0xFF);
        entry->state = 0;           /* 回退到 EMPTY */
        return;
    }

    /* 重发 ARP Request */
    entry->retry_count++;
    entry->timer_start_ms = now_ms;
    arp_send_request(entry->ip_addr);
    printf("ARP retry %d/3 for %d.%d.%d.%d\n",
           entry->retry_count,
           (entry->ip_addr >> 24) & 0xFF,
           (entry->ip_addr >> 16) & 0xFF,
           (entry->ip_addr >> 8) & 0xFF,
           entry->ip_addr & 0xFF);
}
```

**典型时序实测值**（在 Cortex-M4 + LAN8720 平台上测得）：

| 操作 | 耗时 | 说明 |
|------|------|------|
| 发送 ARP Request | ~50 μs | 构造帧 + MAC 发送（含 DMA 描述符操作） |
| 对方处理 + 回复 | ~200-500 μs | 取决于对方 CPU 负载 |
| 收到 Reply 到缓存更新 | < 10 μs | 中断上下文中完成 |
| 一次成功 ARP 总耗时 | ~0.3-1 ms | 同一网段内 |
| 三次超时总耗时 | ~3 秒 | 1s × 3 次重试 |

---

### E.3 ICMP Ping 时序（ICMP Echo Sequence）

Ping 是最常用的网络连通性测试工具，其本质是 ICMP Echo Request/Echo Reply 交换。ICMP 协议直接承载在 IP 之上（Protocol = 1），不经过 TCP/UDP 端口。

```
      PC1 (ping -c 1 192.168.1.2)             Device (responder)
      IP: 192.168.1.1                         IP: 192.168.1.2
           │                                          │
           │  +-- Ethernet Frame (ARP 已解析) --+      │
           │  | Dest MAC: 00:11:22:33:44:55     |      │
           │  | Src MAC:  00:0C:29:12:34:56     |      │
           │  | EtherType: 0x0800 (IPv4)        |      │
           │  |                                |      │
           │  | +-- IP Header (20 bytes) ---+  |      │
           │  | | Version=4, IHL=5           |  |      │
           │  | | Total Length: 60 (0x003C)  |  |      │
           │  | | Protocol: 1 (ICMP)         |  |      │
           │  | | TTL: 64                    |  |      │
           │  | | Src: 192.168.1.1           |  |      │
           │  | | Dst: 192.168.1.2           |  |      │
           │  | | Checksum: 0xXXXX          |  |      │
           │  | +----------------------------+  |      │
           │  |                                |      │
           │  | +-- ICMP Echo Request ------+  |      │
           │  | | Type: 8 (Echo Request)    |  |      │
           │  | | Code: 0                   |  |      │
           │  | | Checksum: 0xXXXX          |  |      │
           │  | | Identifier: 0x1234        |  |      │
           │  | | Sequence: 1               |  |      │
           │  | | Data: 56 bytes of payload |  |      │
           │  | +----------------------------+  |      │
           │  +--------------------------------+      │
           │                                          │
           │  === 以太网传输：最快 ~5μs（同子网直连）===   │
           │                                          │
           │        ┌─────────────────────┐            │
           │        │ 设备收到 Echo Req    │            │
           │        │ 1. 检验 Ethernet FCS│            │
           │        │ 2. IP 校验和验证     │            │
           │        │ 3. ICMP 校验和验证   │            │
           │        │ 4. Type==8, 构造 Reply│           │
           │        │ 5. 交换 Src/Dest    │            │
           │        │ 6. Type=0, 重算校验和│            │
           │        │ 处理时间: ~1-2ms    │            │
           │        └─────────────────────┘            │
           │                                          │
           │  <--- ICMP Echo Reply (type=0, seq=1) --- │
           │<──────────────────────────────────────────│
           │                                          │
           │  PC1 收到 Reply，计算 RTT：               │
           │  RTT = T_receive - T_send               │
           │  = 2.345 ms (典型值，同网段直连)          │
           │                                          │
           │  Ping 统计输出：                           │
           │  64 bytes from 192.168.1.2: icmp_seq=1    │
           │  ttl=64 time=2.34 ms                      │
```

```c
#include <stdint.h>
#include <stdio.h>

/* ICMP Echo Request 头部结构（网络字节序） */
typedef struct __attribute__((packed)) {
    uint8_t  type;        /* 8: Echo Request, 0: Echo Reply */
    uint8_t  code;        /* 始终为 0 */
    uint16_t checksum;
    uint16_t identifier;
    uint16_t sequence;
    uint8_t  data[];      /* 可选数据负载 */
} icmp_echo_t;

/* ICMP 校验和计算（16-bit one's complement sum） */
uint16_t icmp_checksum(void *buf, int len)
{
    uint16_t *p   = (uint16_t *)buf;
    uint32_t  sum = 0;

    while (len > 1) {
        sum += *p++;
        len -= 2;
    }
    if (len == 1)
        sum += *(uint8_t *)p;

    sum  = (sum >> 16) + (sum & 0xFFFF);
    sum += (sum >> 16);
    return (uint16_t)(~sum);
}

/* 打印详细 Ping 时序（用于调试） */
void ping_timing_report(uint32_t t_start_us, uint32_t t_end_us,
                        uint8_t seq, int success)
{
    uint32_t rtt_us = t_end_us - t_start_us;
    float    rtt_ms = rtt_us / 1000.0f;

    printf("[PING] seq=%d ", seq);
    if (success) {
        printf("RTT = %.3f ms (%.0f us)\n", rtt_ms, (float)rtt_us);
        if (rtt_us < 500)
            printf("       >> 同网段直连，信号质量佳\n");
        else if (rtt_us < 5000)
            printf("       >> 可能有交换机跳转或轻微拥塞\n");
        else
            printf("       >> RTT 偏高，检查链路质量!\n");
    } else {
        printf("TIMEOUT (>1000ms)\n");
    }
}

/* 简化版 RTT 测量（基于 SysTick） */
uint32_t ping_send_and_measure(uint32_t target_ip, uint8_t seq)
{
    uint32_t t_send = get_systick_us();

    icmp_send_request(target_ip, seq);
    /* 等待应答（实际应用中应有超时机制） */
    icmp_wait_reply(target_ip, seq, 1000);

    uint32_t t_recv = get_systick_us();
    return t_recv - t_send;    /* 返回 RTT（微秒） */
}
```

---

### E.4 TCP 三次握手时序（TCP Three-Way Handshake）

TCP 是面向连接的协议，通信前必须通过三次握手建立连接。理解握手机制对调试 TCP 连接问题至关重要——很多"连接不上"问题都出在握手阶段。

```
  主动打开（Client）                       被动打开（Server）
  端口：任意（如 12345）                     端口：已知（如 80）
       │                                          │
       │  === 状态：CLOSED ===                     │  === 状态：LISTEN ===
       │                                          │
       │    --- 第 1 步：SYN --->                  │
       │    Flags: SYN=1, ACK=0                    │
       │    seq=1000         (初始序列号 ISN=C)    │
       │    MSS=1460, Window=65535                 │
       │    SACK Permitted, Timestamp              │
       │─────────────────────────────────────────> │  === 状态：SYN_RCVD ===
       │                                          │
       │    <--- 第 2 步：SYN+ACK ---              │
       │    Flags: SYN=1, ACK=1                    │
       │    seq=2000         (初始序列号 ISN=S)    │
       │    ack=1001  (期待收到的下一个字节)        │
       │    MSS=1460, Window=65535                 │
       │<───────────────────────────────────────── │
       │                                          │
       │  === 状态：ESTABLISHED ===                │
       │                                          │
       │    --- 第 3 步：ACK --->                  │
       │    Flags: SYN=0, ACK=1                    │
       │    seq=1001, ack=2001                     │
       │    （可捎带数据，此步之后才能发送应用数据） │
       │─────────────────────────────────────────> │  === 状态：ESTABLISHED ===
       │                                          │
       │    === 数据传输阶段 ===                    │
       │    Client -> Server: seq=1001, len=100    │
       │    Server -> Client: ack=1101             │
       │    Server -> Client: seq=2001, len=200    │
       │    Client -> Server: ack=2201             │
       │                                          │
       │    === 四次挥手关闭（FIN 交换）===         │
```

**握手时序参数详细说明**：

```c
#include <stdint.h>
#include <stdio.h>

/* TCP 头部头部结构 */
typedef struct __attribute__((packed)) {
    uint16_t src_port;
    uint16_t dst_port;
    uint32_t seq_num;
    uint32_t ack_num;
    uint8_t  data_offset;   /* 高 4 位: 头部长度(×4) */
    uint8_t  flags;         /* URG|ACK|PSH|RST|SYN|FIN */
    uint16_t window;
    uint16_t checksum;
    uint16_t urgent_ptr;
} tcp_header_t;

/* TCP 标志位定义 */
#define TCP_FLAG_FIN  0x01
#define TCP_FLAG_SYN  0x02
#define TCP_FLAG_RST  0x04
#define TCP_FLAG_PSH  0x08
#define TCP_FLAG_ACK  0x10
#define TCP_FLAG_URG  0x20

/* 打印三次握手详细时序 */
void tcp_handshake_dump(uint32_t t_syn, uint32_t t_synack,
                        uint32_t t_ack, uint32_t rtt)
{
    printf("=== TCP Three-Way Handshake Timing ===\n");
    printf("  T(SYN):     t = %u us\n", t_syn);
    printf("  T(SYN+ACK): t = %u us  (delta = %u us)\n",
           t_synack, t_synack - t_syn);
    printf("  T(ACK):     t = %u us  (delta = %u us)\n",
           t_ack, t_ack - t_synack);
    printf("  RTT:        %u us = %.2f ms\n", rtt, rtt / 1000.0f);

    if (rtt > 100000)
        printf("  *** 警告：RTT > 100ms，可能存在 WAN 延迟\n");
    if ((t_synack - t_syn) > 50000)
        printf("  *** 警告：Server 响应时间 > 50ms，Server 可能过载\n");
}

/* TCP 连接超时（典型嵌入式中断/轮询实现） */
#define TCP_SYN_TIMEOUT_MS  3000   /* 3 秒超时（RFC 6298） */
#define TCP_SYN_RETRIES     3

int tcp_connect_with_retry(uint32_t server_ip, uint16_t port)
{
    for (int retry = 0; retry < TCP_SYN_RETRIES; retry++)
    {
        /* 发送 SYN */
        tcp_send_syn(server_ip, port, get_isn());
        uint32_t timeout = get_tick_ms() + TCP_SYN_TIMEOUT_MS;

        while (get_tick_ms() < timeout)
        {
            if (tcp_has_synack(server_ip, port))
            {
                tcp_send_ack(server_ip, port);
                printf("TCP connected after %d retries\n", retry);
                return 1;   /* 成功 */
            }
            process_network_events();
        }
        printf("TCP SYN timeout (retry %d/%d)\n",
               retry + 1, TCP_SYN_RETRIES);
    }
    return 0;   /* 连接失败 */
}
```

**为什么是三次而不是两次**：
- 防止已失效的 SYN 请求在 Server 端建立"半开连接"（两次握手无法消除）
- 两次握手可能导致 Server 资源在收到 SYN 时即分配，而 Client 可能并未真正准备好通信
- 第三次 ACK 是 Client 对 Server ISN 的确认——如果没有这步，Server 无法确认 Client 收到了自己的 SYN+ACK

**常见的握手失败根因**：
- **SYN 无响应**：防火墙/ACL 过滤，或目标端口未开放（无 LISTEN 服务）
- **RST 响应**：目标端口有服务，但 SYN 中包含不受支持的选项（如 MSS 过小）
- **SYN+ACK 丢失**：对称 NAT 或 IP 欺骗防护拦截了非对称路由的回复包

---

### E.5 MDIO Clause 22 读操作时序（MDIO Read Timing with Measurements）

MDIO（Management Data Input/Output）是 IEEE 802.3 定义的 MAC 与 PHY 之间的管理接口。Clause 22 帧格式包含前导码、起始位、操作码、PHY 地址、寄存器地址、转向状态和数据字段。

```
MDC 时钟（2.5MHz 典型，最大可配置）：
                         +---+   +---+   +---+   +---+   +---+   +---+   +---+
                         |   |   |   |   |   |   |   |   |   |   |   |   |   |
                  -------+   +---+   +---+   +---+   +---+   +---+   +---+   +---+

MDIO 数据线（双向，需要上拉电阻）：
读操作帧格式（32 个 MDC 周期）：

        preamble (32个1)      ST   OP     PHYAD     REGAD      TA       DATA (读)
        ─────────────        ──   ──   ────────   ────────   ────   ────────────
MDIO: 111111111111111111...  01   10   AAAAAPPPP   RRRRR       Z0     DDDDDDDDDDDDDDDD
       ↑                     ↑    ↑    ↑           ↑           ↑      ↑
       32 个逻辑"1"           01 为起始  10=读操作   5位 PHY 地址  5位寄存器地址  Z=高阻,0=驱动 16 位数据
       用于同步                                                                 由 PHY 驱动

时序图（详细每周期标注）：
                                                                                    
MDC  ──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──
       │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
       └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──┘  └──
       ├── 前导码 (32bit) ──┤├ST├OP├── PHYAD ──├── REGAD ──├┤TA├───── DATA (16bit) ─────┤
       ↑                   ↑                    ↑                          ↑
       MDC 周期 1          周期 33              周期 38                    周期 48-63
       MAC 驱动 MDIO       MAC 驱动              MAC 驱动                   PHY 驱动 MDIO
```

**MDC 频率与周期关系**：

| MDC 频率 | 周期 | 单次读操作时间 | 说明 |
|----------|------|----------------|------|
| 2.5 MHz（标准） | 400 ns | 64 × 400ns = 25.6 μs | Clause 22 最大频率 |
| 1.0 MHz | 1000 ns | 64 μs | 兼容性最佳 |
| 500 KHz | 2000 ns | 128 μs | 长布线场景 |
| 12.5 MHz | 80 ns | 5.12 μs | Clause 45 支持更高速率 |

```c
#include <stdint.h>
#include <stdio.h>

/* 位操作宏 */
#define MDIO_READ   0x02   /* OP=10 (二进制) */
#define MDIO_WRITE  0x01   /* OP=01 (二进制) */

/* GPIO 操作抽象（平台相关，此处为示例） */
extern void mdio_set_mdc(int level);
extern void mdio_set_mdio(int level);
extern int  mdio_get_mdio(void);

/* 软件模拟 MDIO Clause 22 读操作 */
uint16_t mdio_read_clause22(uint8_t phy_addr, uint8_t reg_addr)
{
    uint16_t data = 0;
    int i;

    /* 1. 前导码：32 个连续"1" */
    for (i = 0; i < 32; i++) {
        mdio_set_mdio(1);
        mdio_set_mdc(1);  mdio_set_mdc(0);  /* MDC 上升沿采样 */
    }

    /* 2. 起始位 (ST=01) */
    mdio_set_mdio(0);  mdio_set_mdc(1);  mdio_set_mdc(0);
    mdio_set_mdio(1);  mdio_set_mdc(1);  mdio_set_mdc(0);

    /* 3. 操作码 (OP=10 = 读) */
    mdio_set_mdio(1);  mdio_set_mdc(1);  mdio_set_mdc(0);
    mdio_set_mdio(0);  mdio_set_mdc(1);  mdio_set_mdc(0);

    /* 4. PHY 地址 (5-bit, MSB first) */
    for (i = 4; i >= 0; i--) {
        mdio_set_mdio((phy_addr >> i) & 1);
        mdio_set_mdc(1);  mdio_set_mdc(0);
    }

    /* 5. 寄存器地址 (5-bit, MSB first) */
    for (i = 4; i >= 0; i--) {
        mdio_set_mdio((reg_addr >> i) & 1);
        mdio_set_mdc(1);  mdio_set_mdc(0);
    }

    /* 6. 转向状态 (TA)：MAC 释放总线(Z)，PHY 驱动第 2 个周期为 0 */
    mdio_set_mdio(1);  mdio_set_mdc(1);  mdio_set_mdc(0);  /* Z 状态 */
    mdio_set_mdio(0);  mdio_set_mdc(1);  mdio_set_mdc(0);  /* PHY 驱动为 0 */

    /* 7. 数据 (16-bit, MSB first), PHY 驱动 */
    for (i = 15; i >= 0; i--) {
        mdio_set_mdc(1);
        data |= (mdio_get_mdio() << i);
        mdio_set_mdc(0);
    }

    return data;
}

/* 测量 MDIO 读操作耗时 */
void mdio_benchmark_read(uint8_t phy_addr, uint8_t reg_addr, int iterations)
{
    uint32_t t_start = get_systick_us();

    for (int i = 0; i < iterations; i++)
        mdio_read_clause22(phy_addr, reg_addr);

    uint32_t t_end   = get_systick_us();
    float    avg_us  = (float)(t_end - t_start) / iterations;

    printf("MDIO Read Benchmark (%d iterations):\n", iterations);
    printf("  Total: %u us\n",   t_end - t_start);
    printf("  Avg:   %.2f us\n", avg_us);
    printf("  Rate:  %.0f reads/sec\n", 1000000.0f / avg_us);
}

/* 时序验证：测量 MDC 周期是否符合预期 */
void mdio_verify_timing(void)
{
    uint32_t t1 = get_systick_ns();    /* 高精度计时器 */
    mdio_set_mdc(1);
    uint32_t t2 = get_systick_ns();
    mdio_set_mdc(0);
    uint32_t t3 = get_systick_ns();

    printf("MDC high time: %u ns  (应 > 160ns @ 2.5MHz)\n", t2 - t1);
    printf("MDC low time:  %u ns  (应 > 160ns @ 2.5MHz)\n", t3 - t2);
    printf("MDC period:    %u ns  (应为 400ns @ 2.5MHz)\n", t3 - t1);
}
```

---

### E.6 自动协商 FLP Burst 时序（Auto-Negotiation FLP Burst Timing）

自动协商使用 FLP（Fast Link Pulse）Burst 在 PHY 之间交换能力信息。每个 FLP Burst 包含 17 个时钟脉冲和最多 16 个数据脉冲，持续约 2ms，随后间隔约 16ms 发送下一个 Burst。

```
       FLP Burst #1                     Idle (~16ms)              FLP Burst #2
   ┌──────────────────────┐    ├──────────────────────┤    ┌──────────────────────┐
   │    17 clock +        │    │     Link 空闲          │    │    17 clock +        │
   │    16 data pulses    │    │    无脉冲传输          │    │    16 data pulses    │
   │                      │    │                        │    │                      │
   │ ┌─┐ ┌─┐ ┌─┐ ┌─┐    │    │                        │    │ ┌─┐ ┌─┐ ┌─┐ ┌─┐    │
   │ │ │ │ │ │ │ │ │ │...│    │                        │    │ │ │ │ │ │ │ │ │ │...│
   │─┘ └─┘ └─┘ └─┘ └─┘  └────│────────────────────────│────┘ └─┘ └─┘ └─┘ └─┘  └──
   ├── 约 2ms ──┤             ├── 16ms ± 8ms ──────────┤   ├── 约 2ms ──┤
   ↑                         ↑                         ↑
   链接建立后，PHY 持续发送    第一个 Burst 包含         第二个 Burst 继续
   第一个 FLP Burst           Base Page (能力声明)      交换能力或发送 Next Page

单个脉冲的细节（100BASE-TX）：
        ┌───────┐                    
        │       │  100ns 宽脉冲（正脉冲）               
        │       │                    
   ─────┘       └─────────────────────────────
   ↑           ↑
   脉冲开始     脉冲结束
   宽度: 100ns ± 15ns（IEEE 802.3 规范）
```

**FLP Burst 时序参数表**：

| 参数 | 最小值 | 典型值 | 最大值 | 单位 |
|------|--------|--------|--------|------|
| Burst 长度 | 1.4 | 2.0 | 2.8 | ms |
| Burst 间隔 | 5.0 | 16.0 | 24.0 | ms |
| 时钟脉冲宽度 | 85 | 100 | 115 | ns |
| 数据脉冲宽度 | 85 | 100 | 115 | ns |
| 脉冲间距 | 55 | 62.5 | 70 | μs |
| 完整协商时间 | — | 0.5-3 | 6 | s |

**Base Page 位域定义**：

```
Base Page (16-bit, 通过 FLP Burst 的 16 个数据脉冲传输):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ D15│ D14│ D13│ D12│ D11│ D10│ D9 │ D8 │ D7 │ D6 │ D5 │ D4 │ D3 │ D2 │ D1 │ D0 │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│ S1 │ S0 │ E5 │ E4 │ E3 │ E2 │ E1 │ E0 │FD  │HD  │PS1 │PS0 │ T4 │ RF │ Ack│ NP │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  │    │    │    │    │    │    │    │    │    │    │    │    │    │    │    │
  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────
  Selector Field       Technology Ability         100BTX FD  100BTX HD   远程故障  确认 下一页
  S1:S0 = 0b01         E[5:0] 位图声明能力        PS1:PS0 = 10          T4 = 1      = 1 支持
  (IEEE 802.3)        (D8=1=100BASE-T4)           (100BASE-TX FD)      (表示支持)
```

```c
#include <stdint.h>
#include <stdio.h>

/* Base Page 位域定义 */
#define AN_NP     (1 << 15)   /* Next Page 指示 */
#define AN_ACK    (1 << 14)   /* 确认收到对方能力 */
#define AN_RF     (1 << 13)   /* 远程故障 */
#define AN_T4     (1 << 12)   /* 100BASE-T4 支持 */
#define AN_PS_MASK (0x03 << 10) /* 端口类型 */
#define AN_100TX_FD (0x02 << 10) /* 100BASE-TX 全双工 */
#define AN_100TX_HD (0x01 << 10) /* 100BASE-TX 半双工 */
#define AN_10T_FD  (1 << 9)   /* 10BASE-T 全双工 */
#define AN_10T_HD  (1 << 8)   /* 10BASE-T 半双工 */

/* 计算自动协商超时时间 */
uint32_t aneg_timeout_ms(uint8_t num_pages, int remote_fault)
{
    /* 每个 FLP Burst 约 2ms + 16ms 间隔 = 18ms/轮 */
    /* Base Page 交换需要 3 轮（发送、应答、确认） */
    uint32_t time_ms = 3000;         /* 基础等待 3 秒 */

    if (num_pages > 1)
        time_ms += num_pages * 500;  /* Next Page 每页加 500ms */

    if (remote_fault)
        time_ms += 2000;             /* 远程故障处理加 2 秒 */

    return time_ms;
}

/* 解析协商结果并打印 */
void aneg_result_dump(uint16_t local_ability, uint16_t peer_ability,
                      uint16_t negotiated)
{
    printf("=== Auto-Negotiation Result ===\n");
    printf("Local:  0x%04X\n", local_ability);
    printf("Peer:   0x%04X\n", peer_ability);
    printf("Result: 0x%04X\n", negotiated);

    printf("\nNegotiated Speed/Duplex: ");
    if (negotiated & AN_100TX_FD)
        printf("100BASE-TX Full Duplex\n");
    else if (negotiated & AN_100TX_HD)
        printf("100BASE-TX Half Duplex\n");
    else if (negotiated & AN_10T_FD)
        printf("10BASE-T Full Duplex\n");
    else if (negotiated & AN_10T_HD)
        printf("10BASE-T Half Duplex\n");
    else
        printf("UNKNOWN (fallback to 10T-HD)\n");

    int local_highest = 0, peer_highest = 0;
    uint32_t priorities[] = {AN_100TX_FD, AN_100TX_HD, AN_10T_FD, AN_10T_HD};

    for (int i = 0; i < 4; i++) {
        if (local_ability & priorities[i] && !local_highest)
            local_highest = priorities[i];
        if (peer_ability & priorities[i] && !peer_highest)
            peer_highest = priorities[i];
    }
    printf("Local best:  0x%04X\n", local_highest);
    printf("Peer best:   0x%04X\n", peer_highest);
}

/* FLP Burst 超时检测（用于调试自动协商问题） */
int aneg_wait_complete(uint32_t timeout_ms)
{
    uint32_t t_start = get_tick_ms();

    printf("Waiting for ANEG complete...\n");
    while (get_tick_ms() - t_start < timeout_ms)
    {
        int status = phy_read_aneg_status();

        if (status == 1) {   /* 协商完成 */
            uint32_t elapsed = get_tick_ms() - t_start;
            printf("ANEG complete after %u ms\n", elapsed);
            return 1;
        }
        if (status == -1) {  /* 协商失败 */
            printf("ANEG failed!\n");
            return -1;
        }
        delay_ms(10);        /* 每 10ms 轮询一次 */
    }

    printf("ANEG timeout after %u ms!\n", timeout_ms);
    return 0;
}
```

---

### E.7 自动协商优先级裁决（Auto-Negotiation Priority Resolution）

当两端 PHY 支持多种速率/双工模式组合时，自动协商的优先级裁决算法保证选择双方都支持的"最优"模式。裁决规则遵循 IEEE 802.3 定义的优先级表。

```
  本地 PHY 能力                       对端 PHY 能力
  (PC 侧)                              (交换机侧)
      │                                      │
      │  100BASE-TX Full Duplex   ✓           │  100BASE-TX Full Duplex   ✓
      │  100BASE-TX Half Duplex   ✓           │  100BASE-TX Half Duplex   ✓
      │  10BASE-T Full Duplex     ✓           │  10BASE-T Full Duplex     ✓
      │  10BASE-T Half Duplex     ✓           │  10BASE-T Half Duplex     ✓
      │                                      │
      └─────────────── 交换能力 ──────────────┘
                              │
                     Priority Resolution
                     (双方按相同算法独立计算)
                              │
                              ▼
                    ┌─────────────────────┐
                    │  共同支持的能力集    │
                    │                     │
                    │  100TX-FD  ← 优先级 1│ ← 最高优先级
                    │  100TX-HD      优先级 2│
                    │  10T-FD        优先级 3│
                    │  10T-HD        优先级 4│ ← 最低优先级
                    │                     │
                    │  取最高优先级的交集  │
                    │  = 100BASE-TX FD    │
                    └─────────────────────┘

  优先级裁决算法（双方独立执行，结果一致）：

  1. 本地启动 Base Page 交换（发送本端能力位图）
  2. 等待对方完成 ACK（Ack 位置 1）
  3. 列表如下优先级（从高到低）：

     优先级 | 能力
     ───────┼──────────────────
       1    | 100BASE-T2 FD
       2    | 100BASE-T2 HD
       3    | 100BASE-TX FD
       4    | 100BASE-T4
       5    | 100BASE-TX HD
       6    | 10BASE-T FD
       7    | 10BASE-T HD

  4. 如果无共同能力 → 协商失败，Link Down
  5. 如果只有一方声明 HD → 即使另一方支持 FD，也降级为 HD
```

```c
#include <stdint.h>
#include <stdio.h>

/* IEEE 802.3 优先级表（索引从 0=最高到 6=最低） */
static const uint16_t aneg_priority_table[] = {
    (0 << 10) | (0 << 9) | (0 << 8) | 0x0020,  /* 100BASE-T2 FD  */
    (0 << 10) | (0 << 9) | (0 << 8) | 0x0010,  /* 100BASE-T2 HD  */
    AN_100TX_FD,                                 /* 100BASE-TX FD  */
    AN_T4,                                       /* 100BASE-T4     */
    AN_100TX_HD,                                 /* 100BASE-TX HD  */
    AN_10T_FD,                                   /* 10BASE-T FD    */
    AN_10T_HD,                                   /* 10BASE-T HD    */
};

/* 自动协商优先级裁决函数 */
uint16_t aneg_resolve(uint16_t local_ability, uint16_t peer_ability)
{
    printf("=== Auto-Negotiation Priority Resolution ===\n");
    printf("  Local:  0x%04X\n", local_ability);
    printf("  Peer:   0x%04X\n", peer_ability);

    for (int priority = 0; priority < 7; priority++)
    {
        uint16_t tech = aneg_priority_table[priority];

        if ((local_ability & tech) && (peer_ability & tech))
        {
            printf("  >> Match at priority %d: 0x%04X\n", priority + 1, tech);

            /* 特殊情况：如果一方声明 HD 而对方声明 FD，降级为 HD */
            if ((tech == AN_100TX_FD || tech == AN_10T_FD) &&
                !(local_ability & tech) != !(peer_ability & tech))
            {
                /* 不应该到达这里——优先级表中 FD 高于 HD */
                /* 但如果协商双方能力集不完整，需额外处理 */
            }

            return tech;
        }
    }

    printf("  >> No common technology! Negotiation FAILED.\n");
    return 0;   /* 协商失败 */
}

/* 显示完整协商诊断信息 */
void aneg_diagnostic(uint8_t phy_addr)
{
    printf("\n========== ANEG Diagnostic ==========\n");

    /* 读取 ANEG 状态寄存器（Clause 22, Reg 0x01 的 bit 5 和 bit 6） */
    uint16_t bmsr = mdio_read_clause22(phy_addr, 0x01);
    uint16_t lpa  = mdio_read_clause22(phy_addr, 0x05);  /* Link Partner Ability */

    int autoneg_complete = (bmsr >> 5) & 1;
    int link_up          = (bmsr >> 2) & 1;
    int partner_ack      = (lpa  >> 14) & 1;  /* 对端 ACK */

    printf("  PHY Addr:    0x%02X\n", phy_addr);
    printf("  Autoneg:     %s\n",     autoneg_complete ? "COMPLETE" : "IN PROGRESS/FAILED");
    printf("  Link Status: %s\n",     link_up ? "UP" : "DOWN");
    printf("  Partner ACK: %s\n",     partner_ack ? "YES" : "NO");

    if (!link_up)
    {
        printf("\n  *** Link DOWN 可能原因：\n");
        printf("      1. 网线未连接或损坏\n");
        printf("      2. 对端未上电或端口禁用\n");
        printf("      3. PHY 地址配置错误\n");
        printf("      4. 自动协商不匹配（强制设置速率>对端不支持）\n");
        printf("      5. MDI/MDIX 交叉问题（现代 PHY 通常自动翻转）\n");
    }
    else if (!partner_ack)
    {
        printf("\n  *** 对端未 ACK 自动协商，仍在交换能力中...\n");
    }
}

/* 模拟自动协商过程（纯软件演示） */
void aneg_simulate(void)
{
    printf("\n=== Auto-Negotiation Simulation ===\n\n");

    /* 定义典型场景 */
    typedef struct {
        char *desc;
        uint16_t local;
        uint16_t peer;
    } aneg_scenario_t;

    aneg_scenario_t scenarios[] = {
        {"PC(100TX-FD) <-> Switch(100TX-FD+HD)", AN_100TX_FD | AN_100TX_HD | AN_10T_FD | AN_10T_HD,
                                                   AN_100TX_FD | AN_100TX_HD | AN_10T_FD | AN_10T_HD},
        {"PC(100TX-HD) <-> Switch(100TX-FD)",     AN_100TX_HD | AN_10T_HD,
                                                   AN_100TX_FD | AN_100TX_HD | AN_10T_FD | AN_10T_HD},
        {"PC(10T-HD)   <-> Switch(100TX-FD)",     AN_10T_HD,
                                                   AN_100TX_FD},
        {"PC(100TX-FD) <-> Switch(100TX-FD)",     AN_100TX_FD | AN_100TX_HD,
                                                   AN_100TX_FD | AN_100TX_HD},
    };

    for (int i = 0; i < 4; i++) {
        printf("Scenario %d: %s\n", i + 1, scenarios[i].desc);
        uint16_t result = aneg_resolve(scenarios[i].local, scenarios[i].peer);
        printf("  --> ");
        if (result & AN_100TX_FD) printf("100BASE-TX FD\n");
        else if (result & AN_100TX_HD) printf("100BASE-TX HD\n");
        else if (result & AN_10T_FD) printf("10BASE-T FD\n");
        else if (result & AN_10T_HD) printf("10BASE-T HD\n");
        else printf("FAILED (no common mode)\n");
        printf("\n");
    }
}
```

---

### E.8 时序综合测量工具（Timing Measurement Utility）

以下代码提供了一个通用的时序测量和调试输出工具，可用于验证上述所有协议的时序表现：

```c
#include <stdint.h>
#include <stdio.h>

/* ======================== 时序测量工具 ======================== */

/* 高精度计时器抽象（需根据平台实现） */
#if defined(__ARM_ARCH_7M__) || defined(__ARM_ARCH_7EM__)
    /* Cortex-M3/M4/M7：使用 DWT 周期计数器 */
    static inline uint32_t timer_get_us(void)
    {
        extern uint32_t SystemCoreClock;
        static uint32_t dwt_cycles_per_us = 0;
        if (!dwt_cycles_per_us) {
            dwt_cycles_per_us = SystemCoreClock / 1000000;
            CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
            DWT->CYCCNT = 0;
            DWT->CTRL  |= DWT_CTRL_CYCCNTENA_Msk;
        }
        return DWT->CYCCNT / dwt_cycles_per_us;
    }
#else
    /* 通用 fallback：使用标准库时钟 */
    #include <time.h>
    static inline uint32_t timer_get_us(void)
    {
        return (uint32_t)(clock() * 1000000 / CLOCKS_PER_SEC);
    }
#endif

/* 时序统计结构 */
typedef struct {
    uint32_t min_us;      /* 最短耗时 */
    uint32_t max_us;      /* 最长耗时 */
    uint32_t total_us;    /* 累计耗时 */
    uint32_t count;       /* 采样次数 */
    char     label[32];   /* 测量标签 */
} timing_stats_t;

/* 初始化时序统计 */
void timing_stats_init(timing_stats_t *stats, const char *label)
{
    stats->min_us   = 0xFFFFFFFF;
    stats->max_us   = 0;
    stats->total_us = 0;
    stats->count    = 0;
    snprintf(stats->label, sizeof(stats->label), "%s", label);
}

/* 记录一次测量 */
void timing_stats_record(timing_stats_t *stats, uint32_t elapsed_us)
{
    if (elapsed_us < stats->min_us) stats->min_us = elapsed_us;
    if (elapsed_us > stats->max_us) stats->max_us = elapsed_us;
    stats->total_us += elapsed_us;
    stats->count++;
}

/* 打印统计报告 */
void timing_stats_report(timing_stats_t *stats)
{
    float avg_us = stats->count ?
                   (float)stats->total_us / stats->count : 0;

    printf("=== Timing: %s ===\n", stats->label);
    printf("  Samples: %u\n",  stats->count);
    printf("  Min:     %u us (%.3f ms)\n",
           stats->min_us, stats->min_us / 1000.0f);
    printf("  Avg:     %.1f us (%.3f ms)\n",
           avg_us, avg_us / 1000.0f);
    printf("  Max:     %u us (%.3f ms)\n",
           stats->max_us, stats->max_us / 1000.0f);
    printf("  Rate:    %.0f ops/sec\n",
           1000000.0f / (avg_us > 0 ? avg_us : 1));
}

/* ================ 以太网时序综合测试 ================ */

void ethernet_timing_test(void)
{
    timing_stats_t arp_stats, icmp_stats, mdio_stats, tcp_stats;

    timing_stats_init(&arp_stats,  "ARP Request-Reply");
    timing_stats_init(&icmp_stats, "ICMP Ping Echo");
    timing_stats_init(&mdio_stats, "MDIO Register Read");
    timing_stats_init(&tcp_stats,  "TCP Connect (3-way handshake)");

    printf("\n========== Ethernet Timing Test Suite ==========\n");
    printf("Note: Results depend on link speed and CPU load.\n\n");

    /* 测试 ARP */
    for (int i = 0; i < 5; i++) {
        uint32_t t1 = timer_get_us();
        arp_resolve(0xC0A80102);    /* 192.168.1.2 */
        uint32_t t2 = timer_get_us();
        timing_stats_record(&arp_stats, t2 - t1);
    }
    timing_stats_report(&arp_stats);

    /* 测试 ICMP Ping */
    for (int i = 0; i < 5; i++) {
        uint32_t t1 = timer_get_us();
        ping_send(0xC0A80102);      /* 192.168.1.2 */
        ping_wait_reply(1000);
        uint32_t t2 = timer_get_us();
        timing_stats_record(&icmp_stats, t2 - t1);
    }
    timing_stats_report(&icmp_stats);

    /* 测试 MDIO 读 */
    for (int i = 0; i < 100; i++) {
        uint32_t t1 = timer_get_us();
        mdio_read_clause22(0x01, 0x01);   /* PHY addr=1, BMSR reg */
        uint32_t t2 = timer_get_us();
        timing_stats_record(&mdio_stats, t2 - t1);
    }
    timing_stats_report(&mdio_stats);

    printf("\n========== Test Complete ==========\n");
}
```

> **使用建议**：
> 1. 在 main loop 或 RTOS 任务中运行 `ethernet_timing_test()`，可快速了解当前平台的时序基线
> 2. 将 `timing_stats_t` 集成到网络协议栈的各个关键路径（发送、接收、中断处理等）
> 3. 对比不同优化级别（-O0 vs -O2）下的时序差异，定位性能瓶颈
> 4. 当 MCU 主频或以太网速率变更时，重新运行测试以验证时序是否仍满足协议要求

---

## 参考资源
